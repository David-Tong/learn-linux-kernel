# Linux 文件系统挂载：ext4 如何连接到 VFS

本文以 Linux 7.0 的 ext4 块设备挂载为例，解释 VFS 如何找到文件系统实现、建立文件系统实例，并把根 inode 和操作表接入目录树。挂载后的路径查找和读取见 [cd、open 和 read 如何找到 inode](path-lookup-syscalls.md)。

## 1. 先认识几个对象

| 对象 | 作用 | 在哪里 |
| --- | --- | --- |
| ext4 超级块 | 记录文件系统的块大小、特性等信息 | 磁盘 |
| ext4 目录项 | 保存名字与 inode 号的对应关系 | 目录数据中 |
| ext4 磁盘 inode | 保存文件类型、权限、大小、数据映射等元数据 | 磁盘 inode 表中 |
| `struct super_block` | 表示内核中一个文件系统实例 | 内存 |
| `struct dentry` | 表示并缓存一个路径分量及其与 inode 的关联 | 内存 |
| `struct inode` | 表示加载到内存的文件系统对象及其操作 | 内存 |

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

## 6. 根 inode 和操作表如何连接 VFS 与 ext4？

挂载之前，ext4 向 VFS 注册 `file_system_type`，让 VFS 能通过名字找到文件系统实现。挂载时，ext4 建立或复用 superblock，并加载根目录 inode；内存 inode 所属的 superblock 表示文件系统实例。

```text
挂载命名空间
  → /mnt/data 的挂载对象 → ext4 文件系统根 dentry
                            → 根目录 inode → i_sb → ext4 superblock
                                ├─ i_op → ext4_dir_inode_operations
                                │             └─ lookup → ext4_lookup
                                └─ i_fop → ext4_dir_operations
```

挂载点原先是父文件系统中的目录，挂载对象把它与被挂载文件系统的根关联；路径查找跨越挂载点后，使用 ext4 根目录的 inode。不同挂载对象可以指向同一 superblock，所以挂载位置与文件系统实例不是同一个对象。

ext4 在加载内存 inode 时，根据对象类型设置对应操作表。简化示意如下；它不是可直接编译的源码：

```c
if (S_ISDIR(inode->i_mode)) {
    inode->i_op = &ext4_dir_inode_operations;
    inode->i_fop = &ext4_dir_operations;
} else if (S_ISREG(inode->i_mode)) {
    inode->i_op = &ext4_file_inode_operations;
    inode->i_fop = &ext4_file_operations;
    ext4_set_aops(inode);
}
```

`i_op` 定义 inode 操作，目录的 `lookup` 用于查找子项；`i_fop` 为打开后的文件提供默认操作；`i_mapping->a_ops` 提供页缓存相关操作。它们是内存中的函数指针，指向已加载的 ext4 实现，不存放在磁盘 inode 里。其他文件系统会安装自己的操作表。对于普通文件，打开过程中通常从 `i_fop` 设置 `struct file` 的 `f_op`，实际打开处理可能调整它。

因此，VFS 不需要从 inode 号猜测文件系统类型：挂载先建立文件系统实例及根 inode，之后路径查找可通过已知父目录的操作表进入对应实现。按需获取其他 inode 的过程见[路径查找文章](path-lookup-syscalls.md)。源码参见 [ext4 inode 初始化](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)和 [VFS 文档](https://docs.kernel.org/7.0/filesystems/vfs.html)。
