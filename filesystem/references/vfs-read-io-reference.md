# 文件读取和块 I/O 参考

本文补充 [VFS 读取流程](../vfs-filesystem-read-dispatch.md)中的通用读取、页缓存和块 I/O 细节。具体调用取决于内核版本及文件系统。

## 后续函数与块 I/O 参考


### 调用下一个函数

`new_sync_read()` 不直接调用 `generic_file_read_iter()`，而是通过文件操作表中的函数指针间接调用。它把用户缓冲区包装成 `iov_iter`，把文件和偏移量放入 `kiocb`，然后执行：

```c
// fs/read_write.c，省略初始化和错误处理
ret = file->f_op->read_iter(&kiocb, &iter);
```

例如，FAT 普通文件的操作表直接将该指针设置为通用读取函数：

```c
// fs/fat/file.c，省略其他成员
const struct file_operations fat_file_operations = {
    .read_iter = generic_file_read_iter,
    // ...
};
```

打开文件时，VFS 将文件对应的操作表关联到 `file->f_op`。因此对于这个例子，调用链是：

```text
vfs_read()
  → new_sync_read()
    → file->f_op->read_iter(&kiocb, &iter)
      → generic_file_read_iter(&kiocb, &iter)
```

这取决于具体文件系统：有些操作表指向文件系统自己的 `read_iter`，它可能再调用通用函数，也可能走其他路径。实际代码会将更新后的偏移量写回，并处理返回值；不同内核版本的细节可能不同。参见 [VFS 读取实现](https://github.com/torvalds/linux/blob/master/fs/read_write.c)和 [FAT 文件操作表](https://github.com/torvalds/linux/blob/master/fs/fat/file.c)。

---

## 6. generic_file_read_iter() - 普通文件读取

### 函数位置
- **源文件**：`mm/filemap.c`
- **类型**：C 语言函数

### 函数签名

```c
ssize_t generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
```

这是供文件系统使用的通用读取实现：它根据请求选择直接 I/O 或经页缓存的读取，并把读取结果写入 `iter` 描述的目标缓冲区。具体实现参见 [Linux 源码 mm/filemap.c](https://github.com/torvalds/linux/blob/master/mm/filemap.c)。

**参数：**

- `iocb`：本次 I/O 的控制信息。`iocb->ki_filp` 指向打开的文件，`iocb->ki_pos` 是当前读取偏移；读取成功后偏移会随读取量推进。`iocb->ki_flags` 包含直接 I/O 等请求标志。
- `iter`：描述数据写入目标和剩余长度的迭代器。对于前面的 `read(fd, buf, count)`，`new_sync_read()` 将用户空间的 `buf` 和 `count` 包装成迭代器；复制数据时迭代器也会向前推进。

**返回值：**正数是已读取字节数，可能小于请求长度；`0` 通常表示没有数据可读（例如到达文件末尾），也可能是请求长度为零；负数是内核错误码。异步调用还可能返回 `-EIOCBQUEUED`，表示请求已排队，稍后完成；前面的 `new_sync_read()` 是同步调用。

### 主要执行流程

下面是帮助理解的伪代码，省略了边界检查、异步处理和错误处理；它不是某个版本的源码原文：

```c
generic_file_read_iter(iocb, iter)
{
    if (目标长度为 0)
        return 0;

    if (请求直接 I/O) {
        // 交给文件系统的直接 I/O 实现，通常绕过页缓存的数据复制
        // 如只读到一部分，符合条件时可继续走下面的缓冲读取
        尝试直接读取();
    }

    // 缓冲读取：按文件偏移从页缓存获取数据，必要时发起读取或预读
    return filemap_read(iocb, iter, 已经读取的字节数);
}
```

缓冲读取时，`filemap_read()` 查找文件的页缓存；缓存未命中或数据尚未准备好时，会通过文件系统的读取路径获取数据，并可能触发预读。随后它把可用数据复制到 `iter` 指定的目标，推进迭代器和 `ki_pos`。缓存未命中不一定意味着访问本地磁盘：实际数据来源取决于文件系统。

### 什么时候请求直接 I/O？

通常由应用程序主动指定 `O_DIRECT`，例如：

```c
int fd = open("data.bin", O_RDONLY | O_DIRECT);
// 使用满足该文件直接 I/O 对齐要求的 buf、count 和文件偏移
ssize_t n = read(fd, buf, count);
```

对于前面的同步读取路径，打开标志会反映到 `kiocb` 的 `IOCB_DIRECT` 标志中。若文件系统的读取实现进入 `generic_file_read_iter()`，它据此尝试直接 I/O；未指定时通常走页缓存读取。文件系统也可能使用自己的 `read_iter` 实现，因此具体调用链取决于文件系统和内核版本。

直接 I/O 主要用于应用希望自行管理数据缓存的场景，例如一些数据库；它通常绕过文件数据的页缓存读写路径，但不意味着绕过内核或所有缓存，也不保证比普通读取更快。

直接 I/O 可能要求缓冲区地址、读取长度和文件偏移满足对齐限制。要求因文件系统和内核版本而异，不满足时可能返回 `EINVAL` 或退回缓冲 I/O；文件系统也可能不支持 `O_DIRECT`。Linux 6.1 起，可在支持的文件系统上通过 `statx()` 的 `STATX_DIOALIGN` 查询对齐要求。详见 [`open(2)`](https://man7.org/linux/man-pages/man2/open.2.html)。

例如 `read(fd, buf, 1024)` 从偏移 0 读取了 600 字节时，返回值为 600，目标缓冲区前 600 字节被填充，`ki_pos` 推进到 600；对于普通的 `read()` 调用，成功后的偏移最终会写回打开文件的当前位置。

### 函数的作用

普通文件（存储在磁盘上的文件）的读取操作，包括：

1. **检查页缓存**：数据是否已经在内存中
2. **预读**：提前读入更多数据以提高性能
3. **缓存未命中处理**：如果数据不在页缓存，通过文件系统读取数据
4. **数据复制**：将数据从内核缓冲区复制到用户缓冲区

### 页缓存

页缓存（Page Cache）是 Linux 内核的一个关键机制：

- 文件数据可以按需缓存在内存中，不保证整个文件都在缓存中
- 如果数据已在缓存中，直接返回（快速路径）
- 如果数据不在缓存中，通过文件系统读取（可能涉及存储设备或网络）
- 由内核自动管理，用户程序看不到

---

## 7. 块 I/O 和磁盘读取

### 与前面读取代码的关系

这一节是前面 `filemap_read()` 在**缓冲读取、页缓存数据未就绪**时的一种后续路径。以下以使用块设备的文件系统为例，箭头表示可能经过的层次，不代表每个箭头都是一次直接函数调用：

```text
generic_file_read_iter()
  → filemap_read()
    → 查找页缓存
      ├─ 数据已就绪 → 复制到用户缓冲区 → 返回
      └─ 数据未就绪 → 获取或创建缓存 folio
          → 文件系统的 readahead / read_folio 操作
          → 将文件偏移映射到块设备位置
          → 构造并提交 bio → 块层 → 设备驱动 → 存储设备
          → I/O 完成，缓存数据就绪，唤醒等待者
          → filemap_read() 复制数据到用户缓冲区 → 返回
```

`filemap_read()` 通过文件系统的 `address_space_operations`（如 `readahead` 或 `read_folio`）获取缺失数据；文件系统负责文件位置到存储位置的映射，并通过自己的实现或通用辅助代码提交 I/O。等待发生在需要的数据尚未就绪时；预读也可能提前异步提交。具体函数和路径因文件系统及内核版本而异。参见 [内核对 filemap_read 的说明](https://www.kernel.org/doc/html/v6.8/core-api/mm-api.html)。

缓存未命中不总是产生磁盘 I/O：稀疏文件的空洞可以返回零，网络文件系统可能通过网络获取数据，内存文件系统也不一定需要块设备。直接 I/O 也可能使用块层和 `bio`，但它不经过上面的页缓存数据复制路径。

### 块 I/O（Bio）

位置：`include/linux/bio.h`

```c
struct bio {
    struct bvec_iter    bi_iter;      // 包含设备扇区位置和剩余长度等
    struct bio_vec     *bi_io_vec;    // 描述内存数据段的数组
    unsigned short      bi_vcnt;      // bio_vec 数量
    // ... 更多字段
};
```

这是简化示意，实际字段随内核版本变化。`bio` 描述一次**块设备 I/O 操作**，包括设备上的位置和内存数据段。

---
