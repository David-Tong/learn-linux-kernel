# cd、open 和 read 如何找到文件：从系统调用到 inode

本文以 Linux 7.0、ext4 普通文件为例，解释路径如何变成内存中的 inode，以及 `read()` 如何使用已经打开的文件。挂载如何把 ext4 接入 VFS，请先看[挂载和 VFS 的连接](mount-inode-file-lookup.md)。具体函数随内核版本、架构和文件系统变化；以下调用链省略了权限检查、错误处理和若干中间步骤。

## 先区分三个操作

| 用户操作 | 内核要做什么 | 查找的起点 |
| --- | --- | --- |
| shell 的 `cd /mnt/data/docs` | 通常调用 `chdir()`，解析目录并更新当前工作目录 | 绝对路径从进程的根目录开始 |
| `open("a.txt", O_RDONLY)` | 解析路径，打开文件，返回 fd | 相对路径通常从当前工作目录开始 |
| `read(fd, buf, n)` | 根据 fd 找到已打开的文件并读取 | fd 表；不重新解析路径 |

`cd` 是 shell 内建命令，因为它必须改变 shell 自己的当前目录，才能影响之后启动的程序。`chdir()` 和 `fchdir()` 是 POSIX 接口，也是 Linux 系统调用；`read()` 读取文件描述符，不接收文件名。shell 可能先处理 `~`、逻辑路径和 `CDPATH`，所以命令行文本不一定原样传给内核。`cd -` 等行为还有 shell 自己的处理。

## 1. cd 和 chdir：把当前目录改成一个路径对象

例如在 shell 中执行 `cd /mnt/data/docs`，典型过程是：

```text
shell 的 cd 内建命令
  → chdir("/mnt/data/docs")             用户态接口 / 系统调用
  → SYSCALL_DEFINE1(chdir, ...)         fs/open.c
  → filename_lookup(..., LOOKUP_FOLLOW | LOOKUP_DIRECTORY, ...)
  → VFS 路径查找                         fs/namei.c
  → path_permission(..., MAY_EXEC | MAY_CHDIR)
  → set_fs_pwd(current->fs, &path)      更新当前工作目录
```

这里的箭头是概念上的主要步骤，并非都直接调用。查找到的 `struct path` 包含挂载对象和 dentry；目标必须是目录，并通过相应的搜索权限和安全检查。调用失败时工作目录不变。`fchdir(fd)` 则从已打开的目录 fd 获得路径，检查后更新工作目录，不需要再次按名字解析这个目录。

当前目录是进程文件系统上下文中的路径引用，并非仅保存一个字符串。子进程通常继承它；在子 shell 或单独进程中执行 `cd` 不会改变父 shell 的工作目录。绝对路径从进程的根目录开始，该根目录可能受容器或 `chroot` 环境影响。

## 2. VFS 如何逐级把名字解析成 inode

假设 `/dev/vdb1` 上的 ext4 挂载在 `/mnt/data`，解析 `/mnt/data/docs/a.txt`：

```text
进程根目录 → mnt → data [跨过挂载点] → ext4 根目录 → docs → a.txt
                                                    ↑ 父目录     ↑ 目标
```

VFS 按路径分量查找，先尝试 dentry 缓存；缓存未能满足查找时，在适当的慢路径中调用父目录 inode 的 `i_op->lookup`。对于 ext4 目录，这通常进入 `ext4_lookup()`：它在目录数据中查找名字对应的 inode 号，再通过 `ext4_iget()` 获取内存 inode。inode 不在缓存时，需要从磁盘 inode 表加载；查找目录本身也可能产生 I/O。

```text
父目录 dentry → 父目录 inode → i_op->lookup → ext4_lookup()
  → 目录中 "a.txt" 对应的 inode 号 → ext4_iget()
  → 目标 dentry ↔ 内存 inode
```

这是简化图：负 dentry 可以缓存不存在的名字，符号链接会引入额外解析，`O_CREAT` 等操作对最后一个分量有特殊处理；缓存命中时也不必调用 ext4 的 `lookup`。目录搜索权限、挂载跨越、并发重命名及 RCU-walk / REF-walk 都会影响实际过程。硬链接使多个 dentry 指向同一个 inode；inode 号只在所属文件系统内有意义。

ext4 使用 inode 号和每块组 inode 数算出块组及组内索引，通过块组描述符定位磁盘 inode 表。磁盘 inode 是元数据记录；内存 `struct inode` 是 VFS 对象，由 ext4 初始化其操作表。加载 inode 不意味着已经读取全部文件内容。

## 3. open：路径解析后，建立打开文件和 fd

```c
int fd = open("a.txt", O_RDONLY);
```

这是 POSIX `open()` 库接口。在 Linux 上 libc 可能以 `openat(AT_FDCWD, ...)` 实现；`strace` 看到 `openat` 不代表应用源码一定调用了 `openat()`。`openat()` 的相对路径可从目录 fd 开始，`AT_FDCWD` 则使用当前工作目录。

```text
open() / openat()                用户态接口
  → openat 系统调用入口           fs/open.c
  → do_sys_openat2() → do_file_open()
  → path_openat()                fs/namei.c；解析路径、检查权限并打开
  → struct file → fd 表项
  → 返回 fd
```

`openat2()` 也会进入相关打开流程，但其解析限制和参数与 `openat()` 不同。针对普通文件，打开过程把已解析的路径和 inode 与新的 `struct file` 关联，并通常从 `inode->i_fop` 取得文件操作表；文件系统打开方法可以调整它。两个独立的 `open()` 即使指向同一 inode，也通常得到各自的打开文件状态和偏移；`dup()` 则共享同一个打开文件描述及偏移。

## 4. read 从哪里继续？

`read(fd, buf, count)` 使用 fd 找到打开时保存的 `struct file`，不重新按路径查找 inode。读取如何通过 VFS 和文件操作表进入 ext4、页缓存和块 I/O，见 [VFS 如何调用文件系统的读取实现](vfs-filesystem-read-dispatch.md)。文件描述符共享偏移的实验见 [Bash 文件描述符参考](references/bash-file-handle-analysis.md)。

## 5. 亲自观察

在 Linux 上运行以下命令，观察 shell 及其子进程的系统调用：

```bash
strace -f -e trace=chdir,fchdir,openat,read bash -c 'cd /etc && cat passwd > /dev/null'
```

输出会包含动态加载器和其他文件的 `openat`、`read`；找到 `chdir("/etc")`、打开 `passwd` 以及对应 fd 的 `read` 即可。`cat` 可能使用其他优化路径；系统、libc 和工具版本也会影响结果。`strace` 展示用户态与内核的边界，不直接展示 `ext4_lookup()` 等内部函数，也不能证明发生了磁盘 I/O。

可以比较路径和已打开 fd 引用的 inode：

```bash
stat -c 'device=%d inode=%i %n' /etc/passwd
bash -c 'exec 3</etc/passwd; stat -Lc "device=%d inode=%i %n" /proc/$$/fd/3'
```

设备号和 inode 号应相同；这种观察展示对象身份，但不直接展示内核操作表或缓存状态。

## 源码与参考

- [Linux 7.0 fs/open.c](https://github.com/torvalds/linux/blob/v7.0/fs/open.c)：`chdir`、`fchdir` 和打开入口。
- [Linux 7.0 fs/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/namei.c)：路径查找和打开。
- [Linux 7.0 fs/read_write.c](https://github.com/torvalds/linux/blob/v7.0/fs/read_write.c)：`read` 系统调用和 VFS 读取。
- [Linux 7.0 ext4/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/namei.c)、[ext4/inode.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)：ext4 查找及 inode 加载。
- [内核路径查找文档](https://docs.kernel.org/7.0/filesystems/path-lookup.html)、[VFS 文档](https://docs.kernel.org/7.0/filesystems/vfs.html)。
