# Linux 如何挂载文件系统、加载 inode 并找到文件内容

本文以 **Linux 7.0、ext4 普通文件、缓冲读取**为例，回答三个问题：

1. 挂载时，内核怎么知道设备上是 ext4？
2. 打开路径时，内核怎么找到文件的 inode？
3. inode 里的文件系统操作和文件内容从哪里来？

**核心答案：inode 的操作表是文件系统自己设置的。** 挂载 ext4 时，ext4 建立根目录 inode，并给它设置 ext4 的目录操作表。之后 VFS 查找文件名时，通过父目录 inode 的 `i_op->lookup` 进入 ext4；ext4 找到子文件的 inode 号并加载 inode，再根据文件类型给新 inode 设置 ext4 对应的操作表。这个过程逐级传递，不需要从 inode 号猜测文件系统类型。

挂载时建立文件系统实例并加载根目录；打开文件时按路径查找，按需加载 inode；读取文件时再按需获取数据。缓存命中时可以复用已有对象，预读和日志恢复等也可能提前读取数据。

```text
mount /dev/vdb1 /mnt/data
  → 建立 ext4 文件系统实例，获得根目录

open /mnt/data/docs/a.txt
  → 逐级查找目录项，找到文件 inode
  → 创建 struct file，返回 fd

read(fd, ...)
  → 通过 struct file 的操作表调用 ext4
  → 查页缓存；需要时查块映射并读取数据
```

本文的路径是一个例子。网络文件系统、内存文件系统、直接 I/O、DAX 或不同内核版本的细节会不同。

## 1. 先认识几个对象

| 对象 | 作用 | 在哪里 |
| --- | --- | --- |
| ext4 超级块 | 记录文件系统的块大小、特性等信息 | 磁盘 |
| ext4 目录项 | 保存名字与 inode 号的对应关系 | 目录数据中 |
| ext4 磁盘 inode | 保存文件类型、权限、大小、数据映射等元数据 | 磁盘 inode 表中 |
| `struct super_block` | 表示内核中一个文件系统实例 | 内存 |
| `struct dentry` | 表示并缓存一个路径分量及其与 inode 的关联 | 内存 |
| `struct inode` | 表示加载到内存的文件系统对象及其操作 | 内存 |
| `struct file` | 表示一次打开文件的状态，如偏移和操作表 | 内存 |
| fd | 进程文件描述符表中的编号，引用打开文件 | 进程的描述符表 |

**磁盘 inode 和内存 `struct inode` 不是同一个结构。** 磁盘上保存的是文件系统格式定义的数据；内核读取它后，创建或复用内存 inode，并由文件系统代码设置内存中的函数指针。

## 2. 用户态怎么知道要挂载 ext4？

可以手动指定类型：

```bash
sudo mount -t ext4 /dev/vdb1 /mnt/data
```

`/dev/vdb1` 是示例设备，`/mnt/data` 是已存在的挂载点。没有 `-t` 时，常见的 util-linux `mount` 会尝试用 `libblkid` 识别设备上的格式签名和元数据。可以先独立观察：

```bash
lsblk -f                    # 查看设备、分区和已识别的类型
sudo blkid -p /dev/vdb1    # 直接探测指定设备，名称按实际情况替换
```

假设结果中有 `TYPE="ext4"`，表示探测结果为 ext4。探测并不遍历全部文件；签名冲突、权限或设备内容等也可能导致无法识别。识别出类型后，内核 ext4 实现仍会校验实际元数据，探测成功不保证挂载成功。详见 [mount(8)](https://man7.org/linux/man-pages/man8/mount.8.html)和 [blkid(8)](https://man7.org/linux/man-pages/man8/blkid.8.html)。

## 3. 从 mount 命令进入内核

`mount` 命令运行在用户态，它解析参数、读取必要的配置，再请求内核完成挂载。以传统 `mount(2)` 接口为例，过程可以概括为：

```text
用户态 mount 命令
  → mount("/dev/vdb1", "/mnt/data", "ext4", flags, options)
  → 系统调用指令，进入内核态

内核态 mount 系统调用入口
  → 复制设备名、类型和选项等用户参数
  → do_mount()：解析挂载点并处理挂载请求
  → VFS 根据 "ext4" 查找注册的文件系统类型
  → 初始化 ext4 挂载上下文，获取文件系统根目录
  → 将挂载接入当前挂载命名空间
  → 返回结果给用户态
```

图中是阶段关系，不表示每个箭头都是直接函数调用。Linux 7.0 中传统入口位于 [`fs/namespace.c`](https://github.com/torvalds/linux/blob/v7.0/fs/namespace.c) 的 `SYSCALL_DEFINE5(mount, ...)`。内核还检查权限、安全策略、路径和选项，任一步都可能失败。

较新的挂载 API 可以使用 `fsopen()`、`fsconfig()`、`fsmount()` 和 `move_mount()` 分阶段完成配置、创建和接入；具体命令使用哪套接口取决于工具版本和环境。下面的命令可用于**测试环境**观察，但它会真正尝试挂载：

```bash
sudo strace -f -e trace=mount,fsopen,fsconfig,fsmount,move_mount \
    mount -t ext4 /dev/vdb1 /mnt/data
```

设备和路径要替换为实际测试对象。详见 [内核挂载 API 文档](https://docs.kernel.org/7.0/filesystems/mount_api.html)。

## 4. ext4 实现怎样进入内核？

ext4 既可以编译进内核，也可以构建为模块。初始化时，ext4 向 VFS 注册一个名字为 `ext4` 的文件系统类型及其挂载入口：

```text
ext4 内建代码或模块初始化
  → register_filesystem(&ext4_fs_type)
  → VFS 可以通过名字 "ext4" 找到对应实现
```

挂载时如果类型尚未注册，内核可能请求用户空间模块加载机制加载相应模块，再重新查找；是否成功取决于系统配置和可用模块。已注册不等于已挂载。正常卸载也要受引用计数约束，不能移除仍在使用的模块代码。参见 [文件系统类型注册](https://github.com/torvalds/linux/blob/v7.0/fs/filesystems.c)和 [ext4 初始化](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/super.c)。

可以观察当前系统的配置和状态：

```bash
# 以下两种配置来源不一定都存在
grep '^CONFIG_EXT4_FS=\|^# CONFIG_EXT4_FS is not set' /boot/config-"$(uname -r)"
zgrep '^CONFIG_EXT4_FS=\|^# CONFIG_EXT4_FS is not set' /proc/config.gz

grep -w ext4 /proc/filesystems  # 当前是否注册了 ext4 类型
lsmod | grep '^ext4 '           # 当前是否作为模块加载
modinfo ext4 | head             # 可用模块的信息（若有）
```

`CONFIG_EXT4_FS=y` 表示内建，`m` 表示构建为模块，`not set` 表示未启用；配置文件不存在或无输出时不能据此判断。`lsmod` 不显示内建代码，`modinfo` 能找到模块也不代表它已经加载。

## 5. 挂载时，ext4 读取了什么？

以首次建立普通 ext4 块设备挂载为例，内核的大致步骤是：

1. **初始化上下文**：VFS 找到 ext4 注册的类型，调用 `ext4_init_fs_context()`，并解析源设备和挂载选项。
2. **获取文件系统树**：通过 ext4 的 `ext4_get_tree()` 进入 `get_tree_bdev()`，打开块设备，查找可复用的 superblock 或准备新实例。
3. **读取必要元数据**：对新实例调用 `ext4_fill_super()` 及辅助函数，读取和验证 ext4 超级块、块组描述符等，建立内存 superblock 和 ext4 私有状态；需要且允许时进行日志恢复。
4. **加载根目录**：获取 ext4 根 inode（inode 号为 2），建立根 dentry，作为该文件系统的根。
5. **接入挂载点**：VFS 将挂载接到目标挂载命名空间，后续路径查找到 `/mnt/data` 时进入这个根目录。

`struct super_block` 表示文件系统实例；挂载对象表示它接在目录树的什么位置。多个挂载可以共享一个实例。重复挂载、绑定挂载或错误路径可能改变上述步骤，并不一定重新读取元数据。

**挂载时通常不会遍历所有目录，也不会把所有 inode 和文件内容读进内存。** 它加载根目录和建立实例所需的信息；以后访问别的文件时再按需加载。可参见 [Linux 7.0 ext4/super.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/super.c)。

查看已有挂载，无需重新挂载：

```bash
findmnt -T /etc/passwd -o TARGET,SOURCE,FSTYPE,OPTIONS
cat /proc/self/mountinfo
```

`findmnt` 显示路径所在挂载；`mountinfo` 展示当前进程的挂载命名空间。容器内看到的结果可能与宿主机不同。

## 6. open 时，路径怎么找到 inode？

假设 ext4 挂载在 `/mnt/data`，程序打开 `/mnt/data/docs/a.txt`：

```text
系统根目录 → mnt → data（跨越挂载点）→ ext4 根目录 → docs → a.txt
```

VFS 按路径分量逐级查找，优先使用 dentry 缓存。需要查找 `docs` 时，如果缓存不能满足请求，VFS 调用**父目录 inode 的 `i_op->lookup`**；ext4 目录的该操作进入 `ext4_lookup()`。它检查目录数据中 `docs` 对应的 inode 号，获得子目录 inode，再以同样方式找到 `a.txt`。

```text
ext4 根目录的数据："docs" → inode 100
  → docs 的内存 inode
     → docs 的目录数据："a.txt" → inode 200
        → a.txt 的内存 inode
```

数字是示意。目录项将名字关联到 inode 号；硬链接允许多个名字关联同一个 inode。查找目录本身可能触发 I/O；dentry 缓存也可能记录不存在的名字。符号链接、权限和并发修改使实际路径更复杂。参见 [VFS 文档](https://docs.kernel.org/7.0/filesystems/vfs.html)和 [ext4/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/namei.c)。

## 7. ext4 怎样通过 inode 号加载 inode？

ext4 把磁盘 inode 表分布在多个块组中。给定 inode 号 `ino` 和超级块里的 `inodes_per_group`，可计算：

```text
块组号            = (ino - 1) / inodes_per_group
该组内的 inode 索引 = (ino - 1) % inodes_per_group
```

ext4 通过块组描述符找到该组 inode 表，再根据索引和 inode 大小定位记录。`ext4_iget()` / `__ext4_iget()` 会利用 inode 缓存；需要加载时，读取磁盘记录、检查数据并初始化内存 inode。inode 号只在对应文件系统内有意义。详见 [ext4 inode 格式](https://docs.kernel.org/7.0/filesystems/ext4/inodes.html)和 [ext4/inode.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)。

磁盘 inode 可能包含 extent 树根节点，用来寻找文件数据；较大的树还需要读取其他元数据块。启用 inline data 时，小文件内容也可能内联保存。所以加载 inode 后获得的是**找到内容的入口**，并不代表完整内容已在内存。

## 8. inode 的操作函数怎么连接到 ext4？

假设 `/dev/vdb1` 上的 ext4 已挂载到 `/mnt/data`，程序要打开 `/mnt/data/docs/a.txt`。关键问题是：**还没有加载 `a.txt` 的 inode 时，内核怎么知道要让 ext4 加载它？**

答案是从已知的**父目录 inode**开始。挂载时 ext4 已经加载根目录 inode，并设置它的目录操作表；路径查找使用父目录的操作表进入 ext4，由 ext4 查找并初始化子 inode。之后查找下一级时，继续使用刚得到的目录 inode。具体过程如下：

### 第一步：挂载产生带 ext4 操作的根 inode

内核根据挂载请求中的文件系统类型找到 ext4。ext4 的挂载实现建立 superblock，加载 ext4 根目录 inode 并创建根 dentry。加载目录 inode 时，ext4 设置它的 `i_op`，使之指向 ext4 的目录操作表：

```text
挂载点 /mnt/data
  → ext4 根 dentry
    → 根目录 inode
      → i_op → ext4_dir_inode_operations
                  → lookup: ext4_lookup
```

挂载点路径本来属于父文件系统；跨过挂载点后，路径查找从**新挂载的 ext4 根目录**继续。因此下一次查找使用的是这个 ext4 根目录 inode 的操作表。

### 第二步：通过父目录的 lookup 进入 ext4

VFS 需要在根目录中查找 `docs`。如果 dentry 缓存未能满足查找，VFS 调用根目录的 `i_op->lookup`，也就是 ext4 的 `ext4_lookup()`。它读取或查询该目录的数据，找到名字 `docs` 对应的 inode 号，然后调用 `ext4_iget()` 获取该 inode。

```text
VFS 查找 "docs"
  → 根目录 inode->i_op->lookup
  → ext4_lookup()
  → 找到 docs 的 inode 号
  → ext4_iget(superblock, inode 号, ...)
```

这里传入的 superblock 标识了要在哪个文件系统实例中查找 inode 号：不同文件系统可以有相同的 inode 号。若 inode 已在缓存中，ext4 可以复用它；否则 ext4 读取磁盘 inode 并初始化内存 inode。查找与加载代码可对照 [ext4/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/namei.c)和 [ext4/inode.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)。

### 第三步：ext4 给新加载的 inode 安装操作表

如果 `docs` 是目录，ext4 给它设置目录操作表；VFS 随后使用 `docs` 的 `i_op->lookup` 查找 `a.txt`。找到 `a.txt` 的 inode 号后，同样由 ext4 获取其 inode。如果 `a.txt` 是普通文件，ext4 设置普通文件操作表。下面是简化示意：

```c
// 加载 inode 时由 ext4 执行；省略检查和其他文件类型
if (S_ISDIR(inode->i_mode)) {
    inode->i_op = &ext4_dir_inode_operations;
    inode->i_fop = &ext4_dir_operations;
} else if (S_ISREG(inode->i_mode)) {
    inode->i_op = &ext4_file_inode_operations;
    inode->i_fop = &ext4_file_operations;
    ext4_set_aops(inode);
}
```

所以内核不是单凭数字 inode 号选择操作表，而是先根据挂载关系进入正确的文件系统，再由该文件系统的代码加载 inode 并根据类型设置操作表。不同文件系统会设置自己的操作表。

### 第四步：打开后，read 使用文件操作表

`a.txt` 查找成功后，VFS 创建本次打开的 `struct file`，通常根据 `inode->i_fop` 设置 `file->f_op`。对 ext4 普通文件，后续调用关系可以概括为：

```text
read(fd, ...)
  → fd 找到 struct file
  → file->f_op → ext4_file_operations
  → read_iter → ext4_file_read_iter()
```

`file->f_op` 可能在打开过程中被文件系统调整；此处是普通 ext4 文件的典型路径。读取到这里才通过文件操作表分派；前面的目录查找使用的是目录 inode 的 `i_op->lookup`。

### 三种操作表分别做什么？

关键是：**ext4 加载磁盘 inode 后，为内存 inode 设置函数表指针。** 函数地址属于内核中已注册的 ext4 代码，不存放在磁盘 inode 里。

例如读取到普通文件 inode 后，ext4 的初始化逻辑可以简化为：

```c
// 示意：省略文件类型判断和其他处理
inode->i_op = &ext4_file_inode_operations;
inode->i_fop = &ext4_file_operations;
ext4_set_aops(inode);
```

这三项职责不同：`i_op` 是 inode 操作；`i_fop` 是打开文件时使用的默认文件操作表；`i_mapping->a_ops` 用于页缓存读取和预读。目录 inode 会选择目录操作表，例如 `lookup`；普通文件会选择普通文件操作表。同类型对象通常共享操作表，不需要每个文件复制一份代码。

打开普通文件时，VFS 创建 `struct file`，通常根据 `inode->i_fop` 设置 `file->f_op`。文件系统的打开处理也可能调整它。对普通 ext4 文件，简化关系如下：

```text
fd → struct file
       ├─ f_inode → 文件的内存 inode → i_fop ─┐
       └─ f_op ──────────────────────────────┴→ ext4_file_operations
                                                  └─ read_iter
                                                      → ext4_file_read_iter()
```

因此，挂载时用名字 `ext4` 找到实现；之后 `read()` 用 fd 找到 `struct file`，通过已经保存的函数指针分派，不必每次重新搜索名字。参见 [ext4 inode 初始化](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)和 [ext4 文件操作](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/file.c)。

## 9. read 时怎样找到内容？

普通 ext4 缓冲读取可能经过：

```text
read(fd, ...)
  → VFS 根据 fd 找到 struct file
  → file->f_op->read_iter → ext4_file_read_iter
  → generic_file_read_iter → filemap_read
  → 查页缓存
    ├─ 数据已就绪：复制到用户缓冲区
    └─ 数据未就绪：通过 mapping->a_ops 进入 ext4 的读页或预读
       → ext4 查询逻辑块到物理块的映射
       → 需要时提交块 I/O，完成后填充页缓存
       → 将数据复制到用户缓冲区
```

箭头表示主要层次，不都是直接调用。空洞可以补零，inline data 有自己的处理，直接 I/O 和 DAX 的路径也不同。具体读取函数和 BIO 过程见 [VFS 如何调用具体文件系统](vfs-filesystem-read-dispatch.md)；系统调用到 VFS 的过程见 [read() 系统调用](read-syscall-kernel-internals.md)。

## 10. 动手观察挂载、inode 和 fd

下面在 Linux Bash 中逐行执行；以系统中已有的普通文件为例：

```bash
file=/etc/passwd
findmnt -T "$file" -o TARGET,SOURCE,FSTYPE,OPTIONS
stat -c 'device=%d inode=%i type=%F size=%s' "$file"
```

`findmnt` 显示该路径所在文件系统；它不一定是 ext4。`stat` 显示设备号、inode 号和文件大小。比较两个文件时，inode 号要与设备号一起考虑。

接着观察 fd。示例使用 fd 3，请先确认当前 shell 没有在使用它：

```bash
exec 3<"$file"
readlink /proc/$$/fd/3
cat /proc/$$/fdinfo/3
stat -Lc 'device=%d inode=%i size=%s' /proc/$$/fd/3

IFS= read -r -u 3 line
printf '第一行：%s\n' "$line"
cat /proc/$$/fdinfo/3
exec 3<&-
```

`fdinfo` 的 `ino` 对应打开对象的 inode 号，`pos` 是此次打开的文件偏移，按字节计数；读取一行后通常会推进，包括读过的换行符。空文件、没有末尾换行的文件或读取错误会有不同结果。`mnt_id` 可与 `/proc/$$/mountinfo` 第一列的挂载 ID 对照。`stat -L` 观察打开的对象，在路径被替换时更能说明 fd 当前引用什么。偏移共享的实验见 [Bash 文件描述符](bash-file-handle-analysis.md)。

还可以用硬链接观察两个名字对应同一个 inode：

```bash
dir=$(mktemp -d)
printf 'hello\n' > "$dir/first"
ln "$dir/first" "$dir/second"
stat -c '%n: device=%d inode=%i links=%h' "$dir/first" "$dir/second"
rm -- "$dir/second" "$dir/first"
rmdir -- "$dir"
```

两个文件名的设备号和 inode 号相同，链接数通常为 2。以上命令展示用户可见状态，不直接显示 inode 中的函数指针，也不能单独证明数据是否命中缓存。
