# read() 内核实现与文件系统读取分派

本文保留 Linux 7.0 中 read 进入 VFS 的函数级说明。入口表和包装宏见 [系统调用包装机制](references/SYSCALL_UNDERSTANDING.md)，路径与 fd 的关系见 [路径查找](path-lookup-syscalls.md)。代码为说明性摘录，具体实现请对照对应版本源码。

## 1. ksys_read() - 内核的 read 实现

### 核心理解：ksys_read 是真实实现

系统调用包装器进入 `ksys_read()`，由它获取文件描述符并调用 VFS 读取。

**关键点：**
- `sys_read` 是系统调用表中的入口点定义
- `__x64_sys_read()` 是编译时自动生成的 stub 函数（系统调用入口点）
- `ksys_read()` 处理 fd 和文件偏移，再将读取交给 `vfs_read()`
- 这个分离设计让内核代码也能直接调用 `ksys_read()` 而不用经过系统调用机制

### 函数位置
- **源文件**：`fs/read_write.c`
- **类型**：C 语言函数（可被系统调用 stub 或内核代码直接调用）

### 函数签名

```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
```

**参数：**
- `fd`：文件描述符
- `buf`：用户空间缓冲区（`__user` 表示这是用户空间的指针）
- `count`：要读取的字节数

**返回值：**
- `> 0`：实际读取的字节数
- `0`：到达文件末尾
- `< 0`：错误（负的错误代码）

### 函数的作用

这是真正的**内核实现函数**，完成以下工作：

1. **获取文件描述符对应的 file 结构**
2. **检查权限**：用户是否有权限读取这个文件
3. **调用虚拟文件系统层**（vfs_read）
4. **处理返回值**

### 实现流程

```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count) {
    // 步骤 1：从文件描述符表中获取 file 结构
    struct fd f = fdget_pos(fd);  // 获取并锁定文件位置
    ssize_t ret = -EBADF;  // 默认错误代码

    if (f.file) {
        // 步骤 2：获取当前文件位置
        loff_t pos = file_pos_read(f.file);

        // 步骤 3：调用虚拟文件系统读取函数
        ret = vfs_read(f.file, buf, count, &pos);

        // 步骤 4：如果成功，更新文件位置
        if (ret >= 0)
            file_pos_write(f.file, pos);

        // 步骤 5：释放文件描述符
        fdput_pos(f);
    }

    return ret;
}
```

### 关键数据结构

**struct fd（文件描述符包装）：**
```c
struct fd {
    struct file *file;  // 指向实际的 file 结构
    // 其他字段...
};
```

**struct file（文件结构）：**

这是内核表示打开文件实例的核心结构，定义在 `include/linux/fs.h` 中。下面是与读取过程相关的简化定义；具体字段和布局因内核版本及配置而异：

```c
struct file {
    struct path                  f_path;       // 路径（挂载点和目录项）
    struct inode                *f_inode;      // 关联的 inode
    const struct file_operations *f_op;        // 文件操作函数表
    atomic_long_t                f_count;      // 引用计数
    unsigned int                 f_flags;      // 文件状态标志
    fmode_t                      f_mode;       // 内核中的访问模式
    struct mutex                 f_pos_lock;   // 文件偏移的锁
    loff_t                       f_pos;        // 当前文件偏移
    void                        *private_data; // 文件系统或驱动的私有数据
    // 其他字段...
};
```

`f_op` 决定具体的读取操作，`f_pos` 记录普通文件的当前位置。多个文件描述符可以通过 `dup()` 或 `fork()` 共享同一个打开文件实例，因此也会共享其文件偏移。

### 调用下一个函数

`ksys_read()` 调用 `vfs_read(f.file, buf, count, &pos)`

---

## 2. vfs_read() - 虚拟文件系统读取

### 函数位置
- **源文件**：`fs/read_write.c`
- **类型**：C 语言函数

### 函数签名

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
```

**参数：**
- `file`：file 结构指针
- `buf`：用户空间缓冲区
- `count`：要读取的字节数
- `pos`：文件当前位置指针

**返回值：**
- 读取的字节数，或错误代码

### 函数的作用

这一层是虚拟文件系统（VFS）层，它：

1. **检查文件类型**：是普通文件、目录、块设备还是字符设备
2. **调用相应的处理函数**：每种文件类型有不同的处理方式

### 实现流程

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos) {
    // 检查读权限
    if (!(file->f_mode & FMODE_READ))
        return -EBADF;

    // 根据文件的操作函数表选择读取函数
    if (file->f_op->read)
        return file->f_op->read(file, buf, count, pos);  // 传统接口
    else if (file->f_op->read_iter)
        return new_sync_read(file, buf, count, pos);  // 迭代器接口
    else
        return -EINVAL;
}
```

---

## VFS 如何调用具体文件系统的读取实现

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

- [文件读取和块 I/O 参考](references/vfs-read-io-reference.md)：通用读取、页缓存与块 I/O。
