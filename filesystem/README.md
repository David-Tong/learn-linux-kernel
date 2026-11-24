# Linux 文件系统与 I/O

本目录以 Linux 7.0 和 ext4 为主要示例，从挂载开始，经过路径查找和打开文件，最后追踪读取。内核版本、文件系统类型和 I/O 模式不同，具体实现也会变化。

## 正文

1. [文件系统挂载与 VFS](mount-inode-file-lookup.md)：文件系统注册、挂载、根 inode 和操作表。
2. [路径查找与系统调用](path-lookup-syscalls.md)：shell 的 `cd`、`chdir()` 和 `open()` 如何找到 inode；`read()` 如何从 fd 继续。
3. [read 内核实现与文件系统读取分派](vfs-filesystem-read-dispatch.md)：从 `ksys_read()` 到 ext4、页缓存和块 I/O。

## 深入参考

- [文件读取和块 I/O](references/vfs-read-io-reference.md)：通用读取、页缓存和块 I/O。
- [系统调用包装机制](references/SYSCALL_UNDERSTANDING.md)：系统调用表、包装宏与 `ksys_read()` 的关系。
- [系统调用底层细节](references/syscall-detailed-reference.md)：x86-64 参数传递、特权级切换和入口。
- [Bash 文件描述符实验](references/bash-file-handle-analysis.md)：fd、inode、打开状态和偏移共享。
- [Python read 参考](references/python-read-reference.md)：Python 文件读取与系统调用的关系。

建议按正文顺序阅读；需要跟踪具体实现或动手实验时，再查阅相应参考。
