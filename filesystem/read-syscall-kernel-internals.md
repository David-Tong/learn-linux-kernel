# Linux 内核中的 read() POSIX 系统调用

> 本文基于 Linux Kernel 7.0.0 版本作为参考。

## 概述

`read()` 系统调用是 POSIX 兼容系统中最基础的 I/O 操作之一。它允许用户空间程序从文件描述符读取数据到缓冲区。本文详细解释了 `read()` 调用从用户空间经过内核到实际从文件系统或设备检索数据的完整过程。

> **相关参考**：想了解 Python read() 和内核 read() 的关系？请参考 [python-read-reference.md](python-read-reference.md)

## 1. read() 系统调用接口

### POSIX 定义

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

**参数：**
- `fd`：文件描述符（通过 open()、pipe()、socket() 等打开）
- `buf`：用户空间缓冲区，用于读取数据
- `count`：要读取的最大字节数

**返回值：**
- `> 0`：实际读取的字节数
- `0`：到达文件末尾
- `< 0`：发生错误（设置 errno）

### 使用示例

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("file.txt", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }
    
    char buffer[1024];
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
    
    if (bytes_read < 0) {
        perror("read");
        close(fd);
        return 1;
    }
    
    printf("读取了 %ld 字节\n", bytes_read);
    close(fd);
    return 0;
}
```

## 2. 用户空间到内核空间的转换

### CPU 的两个权限级别

现代 CPU 提供两个不同的权限级别：

- **Ring 3（用户态）**：普通用户程序运行在这里。在这个级别，程序无法直接操作硬件（如磁盘、网络卡）或访问某些内存区域。
- **Ring 0（内核态）**：操作系统内核运行在这里。内核有完全的硬件访问权限。

这种权限隔离是为了**保护系统**：一个用户程序的错误或恶意代码无法直接摧毁系统或访问其他程序的数据。

### 从用户态进入内核态的唯一通道：syscall 指令

用户程序无法直接跳转到内核代码。唯一的合法方式是**执行 syscall 指令**。

当 CPU 看到 `syscall` 指令时，硬件会自动执行以下操作：

1. **权限切换**：CPU 自动从 Ring 3 切换到 Ring 0
2. **保存返回信息**：CPU 自动保存当前的指令地址和 CPU 标志位，以便后续返回
3. **栈切换**：CPU 自动切换到内核栈（每个进程/CPU 核心都有独立的栈）
4. **跳转**：CPU 跳转到事先设定的内核入口函数

这一切都是**硬件强制完成**的，用户程序无法干预。这样的设计确保了**只有指定的内核代码能被执行**。

### 参数如何传递

用户程序和内核都需要一种方式来传递参数。系统调用使用 **CPU 寄存器**来传递参数：

- 用户程序将参数放在指定的寄存器中
- 执行 `syscall` 指令
- 内核从这些寄存器中读取参数

**为什么必须用寄存器？** 因为 `syscall` 时会自动切换栈。如果参数在栈上，内核就无法访问用户栈。而 CPU 寄存器是 CPU 内部的，不受栈切换影响。

具体使用哪些寄存器有严格规定（x86-64 Linux 的标准），详见 [系统调用详细参考](references/syscall-detailed-reference.md)。

### glibc 包装器的角色

当你写 `read(3, buf, 1024)` 时，这个函数来自 **glibc（GNU C Library）**，而非内核。

glibc 中的 `read()` 是一个预先写好的**汇编包装器**：

1. 它接收你的参数
2. 将参数放入指定的 CPU 寄存器
3. 调用 `syscall` 指令进入内核
4. 返回内核的返回值给你

这个包装器是**静态编写**的（不是实时生成），早已编译在 glibc 库中。你的程序编译时会将这段代码链接进去。

### 从内核返回到用户态

内核完成 `read()` 操作后，需要返回到用户程序。这个过程是 `syscall` 的反向操作：

1. **准备返回值**：内核将结果放在寄存器中
2. **恢复保存的信息**：CPU 恢复 syscall 时保存的指令地址和 CPU 标志位
3. **权限切换**：CPU 执行 `sysret` 指令，自动从 Ring 0 切换回 Ring 3
4. **继续执行**：用户程序从 `syscall` 的下一条指令继续执行

和进入内核一样，这一切也都是**硬件强制完成**的。

### 完整的过程概述

```
用户程序执行 read(3, buf, 1024)
  ↓
glibc 中的 read() 包装器准备寄存器
  ↓
执行 syscall 指令
  ↓
CPU 自动：权限切换 → 保存返回地址 → 切换栈 → 跳转到内核入口
  ↓
进入内核的系统调用处理程序
  ↓
内核查看系统调用号和参数，执行实际的 read 操作
  ↓
内核准备返回值
  ↓
执行 sysret 指令返回
  ↓
CPU 自动：恢复权限 → 恢复返回地址 → 恢复栈 → 跳回用户程序
  ↓
用户程序获得数据和返回值，继续执行
```

### 为什么需要这个机制？

如果用户程序可以直接调用内核函数或操作硬件，会导致：
- 一个程序的错误会导致整个系统崩溃
- 恶意程序可以访问其他用户的数据
- 无法对系统资源进行管理和保护

通过 `syscall` 机制，所有硬件访问都必须经过内核的检查和授权，这样内核就能：
- **验证权限**：用户是否有权限执行这个操作
- **隔离资源**：每个程序独立管理自己的资源
- **安全控制**：防止恶意或错误的操作

> 想要了解 CPU 寄存器的具体映射、汇编代码细节、硬件操作等技术细节，请参考 [系统调用详细参考](references/syscall-detailed-reference.md)。

### 从 POSIX read() 到内核的 sys_read()

现在我们知道用户调用 `read()` 后会通过 syscall 进入内核。但进入内核后，到底调用的是哪个内核函数？

**转换的三个层次：**

1. **POSIX 层**：用户程序调用 `read(3, buf, 1024)`
   - 这是 glibc 中的包装函数
   - 参数：fd=3，buf，count=1024

2. **系统调用层**：CPU 切换到内核后，内核根据系统调用号查表
   - 系统调用号被放在 RAX 寄存器中：RAX = 0
   - 内核查表：0 对应 `__x64_sys_read()`
   - 这个函数位置：`arch/x86/include/asm/syscalls.h`

3. **内核实现层**：真正的读取逻辑
   - `__x64_sys_read()` 解析参数后调用 `ksys_read()`
   - `ksys_read()` 才是真正执行读文件的函数
   - 位置：`fs/read_write.c`

**源代码位置（Linux Kernel 7.0.0）：**

系统调用号表：`arch/x86/entry/syscalls/syscall_64.tbl`
```
0   read   sys_read   __x64_sys_read
```

实现函数：`fs/read_write.c`
```c
SYSCALL_DEFINE3(read, unsigned int, fd, 
                      char __user *, buf, 
                      size_t, count) {
    return ksys_read(fd, buf, count);
}
```

下一章会详细讲解 `ksys_read()` 和它调用的函数链。

## 3. 文件描述符和文件结构

### 数据结构

内核为每个打开的文件维护几个关键结构：

#### 文件描述符表（每个进程）

```c
// 来自 include/linux/fdtable.h
struct fdtable {
    struct file **fd;      // 文件指针数组
    fd_set *close_on_exec;
    fd_set *open_fds;
    struct rcu_head rcu;
    struct fdtable *next;
};
```

#### 文件结构

```c
// 来自 include/linux/fs.h - Kernel 7.0.0
struct file {
    union {
        struct llist_node       fu_llist;
        struct rcu_head         fu_rcuhead;
    } f_u;
    
    struct path             f_path;        // 路径名
    struct inode           *f_inode;       // inode 指针
    const struct file_operations *f_op;    // 文件操作（read、write 等）
    
    atomic_long_t           f_count;       // 引用计数
    unsigned int            f_flags;       // 打开标志（O_RDONLY、O_WRONLY 等）
    fmode_t                 f_mode;        // 文件访问模式
    struct mutex            f_pos_lock;    // 保护 f_pos
    loff_t                  f_pos;         // 文件中的当前位置
    struct fown_struct      f_owner;       // 用于关闭时的信号
    const struct cred      *f_cred;        // 凭据
    struct file_ra_state    f_ra;          // 预读状态
    
    // ... 其他字段
};
```

#### Inode 结构

```c
// 来自 include/linux/fs.h
struct inode {
    umode_t                 i_mode;        // 文件类型和权限
    unsigned short          i_opflags;
    kuid_t                  i_uid;         // 用户 ID
    kgid_t                  i_gid;         // 组 ID
    
    unsigned long           i_ino;         // Inode 号
    dev_t                   i_rdev;        // 设备 ID（对于设备文件）
    loff_t                  i_size;        // 文件大小
    struct timespec64       i_atime;       // 最后访问时间
    struct timespec64       i_mtime;       // 最后修改时间
    struct timespec64       i_ctime;       // 最后状态更改时间
    
    struct file_operations  *i_fop;        // 默认文件操作
    struct super_block      *i_sb;         // 超级块
    struct address_space    *i_mapping;    // 页缓存映射
    
    // ... 其他字段
};
```

## 4. VFS 读取操作

### vfs_read() 实现

```c
// 来自 fs/read_write.c
ssize_t vfs_read(struct file *file, char __user *buf,
                 size_t count, loff_t *pos) {
    // 检查文件权限
    if (!(file->f_mode & FMODE_READ))
        return -EBADF;
    
    // 处理特殊文件类型
    if (file->f_op->read_iter)
        return new_sync_read(file, buf, count, pos);
    
    if (file->f_op->read)
        return file->f_op->read(file, buf, count, pos);
    
    return -EINVAL;
}
```

### 文件操作结构

```c
// 来自 include/linux/fs.h
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    // ... 其他操作
};
```

不同文件系统类型（ext4、btrfs 等）实现自己的 file_operations 结构。

## 5. 常规文件读取路径

### 对于 ext4 文件系统上的常规文件

从 ext4 文件系统上的常规文件读取时，会发生以下情况：

```c
// 来自 fs/ext4/file.c - 简化版
const struct file_operations ext4_file_operations = {
    .read_iter = generic_file_read_iter,
    // ... 其他操作
};
```

### generic_file_read_iter() - 主要读取处理器

```c
// 来自 mm/filemap.c - Kernel 7.0.0
ssize_t generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter) {
    // 检查文件是否内存映射
    if (iocb->ki_flags & IOCB_DIRECT) {
        return generic_file_direct_read(iocb, iter);
    }
    
    // 常规缓冲读取
    return generic_file_buffered_read(iocb, iter, retval);
}
```

#### 缓冲读取路径

```c
// 来自 mm/filemap.c - 简化版
static ssize_t generic_file_buffered_read(struct kiocb *iocb,
                                          struct iov_iter *iter,
                                          ssize_t written) {
    struct file *filp = iocb->ki_filp;
    struct address_space *mapping = filp->f_mapping;
    struct inode *inode = filp->f_inode;
    
    for (;;) {
        // 计算页索引和页内偏移
        pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;
        unsigned offset = iocb->ki_pos & (PAGE_SIZE - 1);
        
        // 尝试在页缓存中查找页面
        struct page *page = find_get_page(mapping, index);
        
        if (!page) {
            // 页面不在缓存中，需要从磁盘读取
            page = page_cache_read_async(filp, index);
            if (!page) {
                ret = -EIO;
                break;
            }
        }
        
        // 如果需要，等待页面就绪
        if (PageReadahead(page)) {
            page_cache_async_readahead(mapping, filp, page, index, pos, PAGE_SIZE);
        }
        
        // 从页缓存复制数据到用户缓冲区
        ret = copy_page_to_iter(page, offset, iter);
        
        put_page(page);
        iocb->ki_pos += ret;
        
        if (!iov_iter_count(iter))
            break;
    }
    
    return written + ret;
}
```

## 6. 页缓存

### 什么是页缓存？

**页缓存**是文件系统页的内存缓存。它的目的是：

1. **性能**：避免对相同数据的重复磁盘 I/O
2. **一致性**：确保内存和磁盘之间的一致性
3. **写缓冲**：在刷新到磁盘前积累写入

### 页缓存结构

```c
// 来自 include/linux/fs.h
struct address_space {
    struct inode            *host;         // 所有者 inode
    struct xarray           i_pages;       // 页树/基数树
    struct rb_root_cached   i_mmap;        // 内存映射区域
    unsigned long           nrpages;       // 缓存中的页数
    pgoff_t                 writeback_index;
    
    const struct address_space_operations *a_ops;
    unsigned long           flags;
    
    // ... 其他字段
};
```

### 页缓存查找

```c
// 来自 mm/filemap.c
struct page *find_get_page(struct address_space *mapping, pgoff_t offset) {
    // 使用 XArray（可扩展数组）查找页面
    // XArray 提供快速的 O(log n) 查找
    struct page *page;
    
    rcu_read_lock();
    page = xas_load(&xas);  // 从 XArray 加载
    
    if (page && !page_cache_get_speculative(page)) {
        page = NULL;
    }
    rcu_read_unlock();
    
    return page;
}
```

## 7. 从磁盘读取

### 当页面不在缓存中

如果页面不在页缓存中，内核需要从磁盘读取它：

```c
// 来自 mm/filemap.c - 简化版
static int page_cache_read_async(struct file *file, pgoff_t index) {
    struct address_space *mapping = file->f_mapping;
    struct page *page;
    
    // 分配新页面
    page = page_cache_alloc_cold(mapping, index);
    if (!page)
        return -ENOMEM;
    
    // 创建读请求
    int ret = mapping->a_ops->readpage(file, page);
    // 或 mapping->a_ops->read_folio(file, folio);
    
    return ret;
}
```

### 文件操作：readpage()

每个文件系统定义如何读取页面。对于 ext4：

```c
// 来自 fs/ext4/inode.c
static int ext4_readpage(struct file *file, struct page *page) {
    // 将页面转换为 folio（5.15+ 中新的页缓存抽象）
    return ext4_read_folio(file, page_folio(page));
}

static int ext4_read_folio(struct file *file, struct folio *folio) {
    // 获取此 folio 的磁盘块
    struct ext4_map_blocks map;
    int err = ext4_map_blocks(NULL, inode, &map, 0);
    
    if (map.m_flags & EXT4_MAP_MAPPED) {
        // 数据在磁盘上，提交 I/O 请求
        err = ext4_bio_read_page(&io_end, folio, nr_to_read, is_async);
    }
    
    // ext4_bio_read_page 向块层提交 bio 请求
    return err;
}
```

## 8. 块 I/O 层

### Bio 结构

文件系统确定要读取哪些磁盘块后，它创建一个 **bio**（块 I/O）结构：

```c
// 来自 include/linux/blk_types.h
struct bio {
    struct bio          *bi_next;         // 链接到下一个 bio
    struct gendisk      *bi_disk;         // 磁盘设备
    u16                 bi_opf;           // 操作（READ/WRITE）
    
    struct bio_vec      *bi_io_vec;       // 段数组
    unsigned short      bi_vcnt;          // 段数
    unsigned short      bi_max_vecs;      // 最大段容量
    
    sector_t            bi_sector;        // 磁盘上的扇区
    
    struct bio_set      *bi_pool;         // 内存池
    bio_end_io_t        *bi_end_io;       // I/O 完成时的回调
    void                *bi_private;      // 回调的私有数据
    
    // ... 其他字段
};
```

### I/O 提交

```c
// 来自 fs/ext4/inode.c - 简化版
static int ext4_bio_read_page(struct bio *bio, struct folio *folio,
                              unsigned nr_to_read, bool is_async) {
    struct inode *inode = folio->mapping->host;
    
    for (;;) {
        // 获取下一个磁盘块
        blkcnt_t block = folio->index << (PAGE_SHIFT - inode->i_blkbits);
        
        // 映射逻辑块到物理扇区
        sector_t sector = ext4_map_block_to_sector(inode, block);
        
        if (sector) {
            // 将页面段添加到 bio
            if (!bio_add_page(bio, &folio->page, blocksize, offset)) {
                // Bio 已满，提交当前的并创建新 bio
                submit_bio(bio);
                bio = bio_alloc(...);
            }
        }
    }
    
    // 提交最后的 bio 请求
    submit_bio(bio);
}
```

### submit_bio() - 块设备驱动程序

```c
// 来自 block/blk-core.c
void submit_bio(struct bio *bio) {
    // 检查 bio 有效性
    if (!bio_flagged(bio, BIO_REMAPPED))
        bio_set_dev(bio, bdev_get_by_dev(bio->bi_bdev->bd_dev));
    
    // 将 bio 加入队列到块设备队列
    if (should_fail_bio(bio)) {
        bio_io_error(bio);
        return;
    }
    
    // 提交到块设备
    generic_make_request(bio);
}
```

继续阅读第二部分...

