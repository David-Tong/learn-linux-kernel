# VFS 如何调用具体文件系统的读取实现

本文接续 [路径查找和打开文件](path-lookup-syscalls.md)，以 Linux 7.0 的 ext4 普通文件缓冲读取为例，说明操作表如何连接 VFS、文件系统和页缓存。实际路径取决于文件类型、文件系统和内核版本。

## 从 file 操作表进入 ext4

打开文件时，VFS 通常从 inode 的 `i_fop` 设置 `file->f_op`。ext4 加载普通文件 inode 时将 `i_fop` 设置为 `ext4_file_operations`；该表的 `read_iter` 指向 `ext4_file_read_iter`。这两个函数指针在内存中由内核设置，不存放在磁盘 inode 中。

```text
vfs_read → new_sync_read
  → file->f_op->read_iter(&kiocb, &iter)
  → ext4_file_read_iter
  → generic_file_read_iter → filemap_read（普通缓冲读取）
```

ext4 的直接 I/O 和 DAX 有不同分支；其他文件系统也可以设置不同的操作函数。参见 [ext4/file.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/file.c)和 [ext4/inode.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)。

## ext4 页缓存未命中后的读取

`filemap_read()` 查找文件的页缓存。数据未就绪时，页缓存可以通过 `file->f_mapping->a_ops` 调用 ext4 的 `readahead` 或 `read_folio`：

```text
filemap_read → 页缓存未命中
  → a_ops->readahead / a_ops->read_folio
  → ext4_readahead / ext4_read_folio
  → ext4_mpage_readpages → ext4_map_blocks
  → 构造并提交 bio → 块层和设备驱动
  → I/O 完成，folio 就绪 → 复制到用户缓冲区
```

箭头表示可能经过的层次，并非全部为直接调用。`ext4_map_blocks()` 查询文件逻辑块到块设备物理块的映射；ext4 根据映射设置 BIO 的设备位置，并将页缓存 folio 加入 BIO。完成回调处理结果并解锁 folio，等待读取的调用者随后可以继续。预读可能异步提交；空洞可补零，内联数据可直接由 ext4 读取，并非每次缓存未命中都产生数据 BIO。参见 [Linux 7.0 ext4/readpage.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/readpage.c)。

## 深入阅读

- [ksys_read 和 vfs_read 参考](references/read-syscall-kernel-internals.md)：从 fd 到 VFS 的函数细节。
- [文件读取和块 I/O 参考](references/vfs-read-io-reference.md)：通用读取、页缓存与块 I/O。
