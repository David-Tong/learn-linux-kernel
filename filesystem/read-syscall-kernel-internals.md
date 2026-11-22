# Linux 内核中的 read() POSIX 系统调用

> 本文基于 Linux Kernel 7.0.0 版本作为参考。

## 概述

`read()` 系统调用是 POSIX 兼容系统中最基础的 I/O 操作之一。它允许用户空间程序从文件描述符读取数据到缓冲区。本文详细解释了 `read()` 调用从用户空间经过内核到实际从文件系统或设备检索数据的完整过程。

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

### 系统调用机制（x86-64）

在用户空间调用 `read()` 时，通常是 C 库（glibc）中的一个包装器：

```c
// 来自 glibc - 简化版
ssize_t read(int fd, void *buf, size_t count) {
    // 准备参数并触发系统调用
    return INLINE_SYSCALL(read, 3, fd, buf, count);
}
```

这个宏触发实际的系统调用入口：

```asm
; x86-64 系统调用入口
; 参数：RDI=fd, RSI=buf, RDX=count
; RAX 包含系统调用号（x86-64 上 read 是 0）

mov $0, %rax           ; read 的系统调用号
syscall                ; 触发系统调用（转换到内核模式）
```

### 系统调用表

`syscall` 指令触发内核执行以下操作：
1. 从用户模式切换到内核模式（通过 CPU 特权模式）
2. 保存当前用户上下文（寄存器、堆栈指针）
3. 跳转到内核的系统调用处理器

在 Linux Kernel 7.0.0 中，read 系统调用在 `fs/read_write.c` 中定义：

```c
// 来自 arch/x86/entry/syscalls/syscall_64.tbl
0   read    sys_read    __x64_sys_read
```

## 3. 系统调用入口点

### 位置：`fs/read_write.c` 和 `arch/x86/entry/`

执行 `syscall` 指令时，内核的低级处理器接收控制。这个处理器定义在 `arch/x86/entry/entry_64.S`：

```asm
; arch/x86/entry/entry_64.S - 系统调用入口
ENTRY(entry_SYSCALL_64)
    # 保存用户寄存器
    swapgs              ; 交换 GS 寄存器为内核
    mov %rsp, %gs:0x5700 ; 保存用户 RSP
    
    # 设置内核堆栈
    mov $entry_SYSCALL_64_safe_stack, %rsp
    
    # 保存用户上下文
    push %r15
    push %r14
    push %r13
    push %r12
    # ... 保存其他寄存器
    
    # 调用高级系统调用处理器
    call do_syscall_64
ENDPROC(entry_SYSCALL_64)
```

`do_syscall_64()` 函数：

```c
// 来自 arch/x86/entry/common.c - 简化版
__visible noinstr void do_syscall_64(struct pt_regs *regs) {
    // 检查系统调用号
    int nr = syscall_get_nr(current, regs);
    
    if (nr < NR_syscalls) {
        // 调用实际的系统调用处理器
        regs->ax = sys_call_table[nr](regs);
    }
}
```

## 4. sys_read() 系统调用处理器

### 位置：`fs/read_write.c`

这是 Kernel 7.0.0 中 read 系统调用的主要入口点：

```c
// 来自 fs/read_write.c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count) {
    return ksys_read(fd, buf, count);
}
```

### ksys_read() 实现

```c
// 来自 fs/read_write.c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count) {
    struct fd f = fdget_pos(fd);  // 获取文件描述符结构
    ssize_t ret = -EBADF;
    
    if (f.file) {
        loff_t pos = file_pos_read(f.file);
        ret = vfs_read(f.file, buf, count, &pos);
        
        if (ret >= 0)
            file_pos_write(f.file, pos);  // 更新文件位置
        
        fdput_pos(f);
    }
    
    return ret;
}
```

**关键函数：**
1. `fdget_pos()`：检索文件描述符结构并锁定文件位置
2. `vfs_read()`：虚拟文件系统读取操作
3. `file_pos_write()`：更新当前文件位置
4. `fdput_pos()`：释放文件描述符并解锁

## 5. 文件描述符和文件结构

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

## 6. VFS 读取操作

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

## 7. 常规文件读取路径

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

## 8. 页缓存

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

## 9. 从磁盘读取

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

## 10. 块 I/O 层

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
