# Linux 如何挂载文件系统、加载 inode 并找到文件内容

本文承接 [read() 系统调用内核实现](read-syscall-kernel-internals.md)，以 Linux 7.0 的 ext4 普通磁盘文件为例，回答三个问题：内核什么时候知道文件系统的格式？什么时候加载某个文件的 inode？又怎么根据 inode 找到文件内容？

先看时间顺序：**挂载时建立文件系统实例并加载根目录；路径查找时按需获取各级目录和目标文件的 inode；读取时再按需获取文件内容。** 缓存命中时可以复用已有对象，实际过程也可能因预读、日志恢复等提前读取更多元数据。

```text
mount /dev/vdb1 /mnt/data
  → 选择 ext4 实现，读取并验证文件系统元数据
  → 建立内存中的 super_block，加载根目录 inode 和根 dentry
  → 把该文件系统接入 /mnt/data

open("/mnt/data/docs/a.txt", O_RDONLY)
  → 沿路径查目录项；必要时从 ext4 目录数据找到 inode 号
  → 按需加载 docs 和 a.txt 的 inode，安装对应操作函数表
  → 创建打开文件的 struct file 和文件描述符

read(fd, buf, count)
  → 使用 file->f_op 进入读取实现，查询页缓存
  → 缓存未命中时，通过 ext4 块映射找到内容并提交 I/O
```

这不是所有文件系统的通用磁盘布局：例如 NFS 从服务器获得信息，`procfs` 的对象通常由内核生成。ext4 的函数名和具体实现也会随内核版本改变。

## 1. 先分清磁盘结构和内存对象

| 名称 | 作用 | 存在位置 |
| --- | --- | --- |
| ext4 超级块 | 记录块大小、inode 数量、特性等文件系统信息 | 磁盘上的 ext4 格式数据 |
| ext4 inode | 记录文件类型、权限、大小和数据映射等元数据 | 磁盘 inode 表中 |
| ext4 目录项 | 将目录中的名字关联到 inode 号 | 目录文件的数据中 |
| VFS `struct super_block` | 表示内核中一个文件系统实例 | 内存 |
| VFS `struct inode` | 表示已加载的文件系统对象，持有元数据和操作表指针 | 内存 |
| VFS `struct dentry` | 缓存路径分量及其与 inode 的关联 | 内存 |
| `struct file` | 表示一次打开的文件实例，保存偏移和操作表等 | 内存 |

磁盘 inode 不是把内存中的 `struct inode` 原样写到磁盘；特别是**内核函数指针不会存储在磁盘 inode 中**。内核解析磁盘元数据后，根据文件系统类型和对象类型，将内核代码中的操作表赋给内存 inode。

## 2. mount 时内核知道了什么？

假设管理员执行：

```bash
mount -t ext4 /dev/vdb1 /mnt/data
```

这里的 `ext4` 指定文件系统类型，`/dev/vdb1` 是块设备，`/mnt/data` 是挂载点。若不写 `-t`，用户空间的 mount 工具通常可以探测设备内容并选择类型；探测结果并不替代内核的格式校验。

ext4 的实现已编译进内核或作为模块加载，并向 VFS 注册文件系统类型。内核按 `ext4` 找到它的挂载实现，打开设备，读取并验证 ext4 超级块、块组描述符等必要元数据，检查格式和特性，必要时执行日志恢复，然后初始化内存中的 `struct super_block` 及 ext4 私有状态。接着加载 ext4 根目录 inode（通常 inode 号为 2），创建根 dentry，再将挂载实例接入指定的挂载点。路径穿过该挂载点时，后续查找从新文件系统的根目录继续。

现代内核通过 filesystem context 完成挂载；在 Linux 7.0 ext4 路径中可追踪 `ext4_init_fs_context()`、`ext4_get_tree()`、`get_tree_bdev()` 和 `ext4_fill_super()`，具体调用还包含辅助函数。重复挂载同一实例、绑定挂载及不同挂载命名空间会有不同的对象关系，并不一定重新读取全部元数据。参见 [VFS 文档](https://docs.kernel.org/7.0/filesystems/vfs.html)和 [ext4 挂载源码](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/super.c)。

**mount 不会遍历并加载所有目录和文件内容，也不会通常把每个 inode 都装入内存。** 它读取启动该文件系统实例所需的信息；根 inode、日志恢复或其他初始化可能带来额外读取。挂载后内核知道如何解释文件系统，并不意味着已经知道每个路径的内容。

## 3. open 时如何从名字找到 inode？

假设已挂载 `/mnt/data`，程序调用 `open("/mnt/data/docs/a.txt", O_RDONLY)`：

1. VFS 从进程的根目录开始逐分量查找；走到 `/mnt/data` 时跨越挂载点，转到 ext4 的根 dentry。
2. 查找 `docs`：先检查 dentry 缓存。若缓存不能满足查找，VFS 调用**父目录 inode 的 `i_op->lookup`**，进入该文件系统的目录查找实现。
3. 对 ext4 而言，`ext4_lookup()` 查找父目录的数据，根据名字找到目录项中的 inode 号。如果目录数据不在缓存，读取目录本身也可能产生 I/O。
4. ext4 根据 inode 号获取子对象的内存 inode；若不在 inode 缓存，就从磁盘 inode 表读取并初始化。VFS 将 dentry 与 inode 关联。
5. 用同样方法在 `docs` 中查找 `a.txt`。成功后执行打开操作，建立 `struct file`，再把它放进进程的文件描述符表。

```text
ext4 根目录 inode
  └─ 目录数据："docs" → inode 号 100
       └─ docs 的内存 inode
            └─ 目录数据："a.txt" → inode 号 200
                 └─ a.txt 的内存 inode → 打开得到 struct file → fd
```

数字仅为示意。dentry 缓存可以缓存不存在的名字（负 dentry）；inode 缓存可让不同路径的硬链接共享同一个内存 inode。inode 号只在对应文件系统实例内有意义。符号链接、权限检查、并发修改和挂载命名空间会使实际路径解析更复杂。详见 [VFS 路径和 inode 文档](https://docs.kernel.org/7.0/filesystems/vfs.html)及 [ext4/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/namei.c)。

## 4. 知道 inode 号以后，如何从磁盘加载？

ext4 按块组组织 inode 表。给定 inode 号 `ino` 和超级块中的每组 inode 数 `inodes_per_group`，可确定：

```text
块组号          = (ino - 1) / inodes_per_group
该组内 inode 索引 = (ino - 1) % inodes_per_group
```

通过块组描述符找到该组 inode 表的位置，再结合 inode 大小和组内索引定位磁盘记录。内核先查 inode 缓存，未命中时分配并初始化内存 inode，读取磁盘记录，校验并解析类型、权限、大小、时间和数据映射等；具体实现见 [`__ext4_iget()` / `ext4_iget()`](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)。相关磁盘格式见 [ext4 inode 文档](https://docs.kernel.org/7.0/filesystems/ext4/inodes.html)。

磁盘 inode 有时包含 extent 树的根节点；树较大时其他节点在额外的元数据块中，按需读取。文件很小时，若启用了 inline data，内容也可能内联在 inode 相关区域。因而加载 inode 后，内核已经有**找到内容所需的起点**，但不一定已读到完整块映射或文件数据。

## 5. 操作函数表到底从哪里来？

以 ext4 普通文件为例，读取磁盘 inode 并确定文件类型后，ext4 的 inode 初始化代码安装内核中的操作表，关系可简化为：

```c
// 示意：省略类型判断、配置分支和错误处理，并非源码原文
inode->i_op = &ext4_file_inode_operations;
inode->i_fop = &ext4_file_operations;
ext4_set_aops(inode);  // 为 inode->i_mapping 选择页缓存操作表
```

目录 inode 使用目录操作表，包含 ext4 的 `lookup`；普通文件的 `i_fop` 包含 `ext4_file_read_iter`。不同类型和配置会选择不同表。打开时 VFS 通常根据 `inode->i_fop` 设置 `file->f_op`，文件系统的 `open` 等机制也可能调整它；随后的 `read()` 通过**打开文件的 `file->f_op`** 分派。页缓存缺失时使用的是 `file->f_mapping->a_ops`，其中的 `read_folio` 和 `readahead` 又指向 ext4 实现。

```text
磁盘 inode 的文件类型 + 所属文件系统
   → ext4 初始化内存 inode 时选择内核代码中的表
   → inode->i_op     ：查找等 inode/目录操作
   → inode->i_fop    ：打开该对象时的默认文件操作
   → inode->i_mapping->a_ops：页缓存读取、预读等操作
   → open 后 file->f_op：read/write 等实际分派入口
```

注意这些表属于文件系统实现，而不是为每个文件复制一套函数；多个同类型 inode 通常指向同一个表。参考 [ext4 inode 初始化](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)、[ext4 文件操作](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/file.c)和 [VFS 文件对象](https://docs.kernel.org/7.0/filesystems/vfs.html)。

## 6. read 时如何从 inode 找到内容？

对普通 ext4 缓冲读取，`file->f_op->read_iter` 进入 `ext4_file_read_iter()`，再进入通用的 `generic_file_read_iter()` 和 `filemap_read()`。页缓存命中时直接从缓存复制；数据未就绪时，页缓存通过 `a_ops` 调用 `ext4_readahead()` 或 `ext4_read_folio()`。ext4 依据文件偏移查询逻辑块到物理块的映射，并在需要时构造 BIO 提交块设备。I/O 完成后，数据进入缓存 folio，再复制到用户缓冲区。对应 Linux 7.0 源码见 [ext4/file.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/file.c)、[ext4/readpage.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/readpage.c)和 [mm/filemap.c](https://github.com/torvalds/linux/blob/v7.0/mm/filemap.c)。

空洞可补零、inline data 可从内联区域读取，直接 I/O 和 DAX 也有不同路径。具体函数链、BIO 和完成过程可参见 [read() 文章的 ext4 章节](read-syscall-kernel-internals.md)；该章节目前以 Linux 6.8 为例，阅读时请留意版本差异。

## 7. 自己观察

以下命令只读取系统信息；把路径换成自己机器上已有的文件：

```bash
findmnt -T /path/to/file -o TARGET,SOURCE,FSTYPE,OPTIONS
stat -c 'device=%d inode=%i type=%F size=%s' /path/to/file
ls -li /path/to/file
```

`findmnt` 显示该路径所在挂载及文件系统类型；`stat` 和 `ls -i` 显示 inode 号。它们不能单独证明 inode 或数据是否来自缓存。若两个路径位于同一文件系统且指向同一个 inode，可能是同一文件的硬链接；比较时还应检查设备号。容器或其他挂载命名空间中的结果可能不同。

