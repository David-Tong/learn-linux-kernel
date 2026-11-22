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

这一章介绍 `read()` 调用是如何从用户程序穿过权限边界进入内核的。这是系统调用的核心机制。

### 第 1 层：你写的用户代码

用户程序中的调用看起来很简单：

```c
#include <unistd.h>

char buf[1024];
ssize_t n = read(fd, buf, 1024);
```

你以为在调用一个普通的 C 函数。但实际上不是。

**为什么不能是普通函数？**

因为 `read()` 最终需要：
- 访问磁盘硬件
- 访问设备驱动程序
- 操作内核数据结构（文件表、缓存等）

用户程序没有权限直接做这些。所以必须进入内核。

### 第 2 层：glibc 包装器

用户程序链接的 `read()` 实际上是 glibc 中的包装函数：

```c
// 来自 glibc 源码 - sysdeps/unix/sysv/linux/read.c
ssize_t read(int fd, void *buf, size_t count) {
    // 调用系统调用包装宏
    return SYSCALL_CANCEL(read, fd, buf, count);
}
```

这个宏做的事情是：
1. 准备参数
2. 执行 `syscall` 指令

本质相当于：

```c
syscall(SYS_read, fd, buf, count);
```

### 第 3 层：参数如何传给内核

x86-64 Linux 有一个固定的规则：**系统调用参数放在特定寄存器中**。

对于 `read(fd, buf, count)` 这个系统调用：

| 参数 | 值 | 目标寄存器 |
|------|-----|----------|
| 系统调用号 | 0（表示 read） | `RAX` |
| 第 1 个参数 | `fd` | `RDI` |
| 第 2 个参数 | `buf` | `RSI` |
| 第 3 个参数 | `count` | `RDX` |

所以在执行 `syscall` 指令前，glibc 包装器会生成类似这样的汇编代码：

```asm
; 假设调用 read(3, buf, 1024)
mov $0, %rax        ; RAX = 0（read 的系统调用号）
mov $3, %rdi        ; RDI = 3（文件描述符）
mov buf_addr, %rsi  ; RSI = buf 的地址
mov $1024, %rdx     ; RDX = 1024（字节数）
syscall             ; 执行系统调用指令
```

### 第 4 层：系统调用号

前面的 `RAX = 0` 是什么意思？

Linux 维护了一张表，每个系统调用都有一个唯一的号码：

```
系统调用号  名字      x86-64 内核函数
0           read      __x64_sys_read
1           write     __x64_sys_write
2           open      __x64_sys_open
3           close     __x64_sys_close
...
```

这张表的位置是：`arch/x86/entry/syscalls/syscall_64.tbl`

在 Linux 7.0.0 中，对于 x86-64 架构：

```
# 来自 arch/x86/entry/syscalls/syscall_64.tbl
0    read    sys_read    __x64_sys_read
```

所以：
- `read()` 的系统调用号就是 `0`
- 内核会通过 `RAX` 中的值 `0` 查表，找到应该调用的函数 `__x64_sys_read`

### 第 5 层：CPU 权限级别

x86-64 CPU 有权限级别的概念：

```
Ring 0  - 内核（最高权限）
Ring 1  - （保留，通常不用）
Ring 2  - （保留，通常不用）
Ring 3  - 用户程序（最低权限）
```

用户程序一直在 **Ring 3** 运行。大部分指令都可以执行，但某些**特权指令**只能在 Ring 0 执行。

例如：
- `CLI`（禁用中断）- 只能在 Ring 0
- `MOV CR3, RAX`（修改页表） - 只能在 Ring 0
- 访问磁盘 I/O 端口 - 只能在 Ring 0

如果用户程序尝试执行这些指令，CPU 会抛出异常。

**那怎么办？使用系统调用！**

### 第 6 层：syscall 指令

最关键的指令是 `syscall`：

```asm
syscall
```

这个指令的作用：
1. CPU 检测到 `syscall` 指令
2. CPU 自动从 Ring 3 切换到 Ring 0
3. CPU 执行以下操作，然后跳入内核

这是 CPU 硬件提供的**权限转换机制**。用户程序无法直接进入 Ring 0，只有通过 `syscall` 指令才能请求 CPU 完成这个转换。

### 第 7 层：CPU 执行 syscall 时做的三件事

#### 1. 保存用户环境（User Context）

CPU 执行 `syscall` 后，需要记住用户程序当前的状态，这样才能在系统调用返回后继续执行用户代码。

CPU 自动保存：
- `RIP` - 下一条用户指令的地址
- `RSP` - 用户栈指针
- `RFLAGS` - CPU 标志寄存器（包含进位标志、零标志等）

这些值被保存到 **Ring 0 安全的位置**（通常是内核栈或特殊寄存器）。

#### 2. 切换堆栈（Stack Switching）

用户程序有自己的栈：

```
User Mode Stack
┌─────────────────┐
│   局部变量      │
│   返回地址      │
│   参数          │
└─────────────────┘
```

内核也有自己的栈（每个 CPU 核心一个，或每个进程一个）：

```
Kernel Mode Stack
┌─────────────────┐
│   内核数据      │
│   中断帧        │
│   函数变量      │
└─────────────────┘
```

执行 `syscall` 后，CPU 自动将 `RSP`（栈指针）从用户栈切换到内核栈。这防止用户代码破坏内核栈。

#### 3. 跳入内核入口函数

CPU 查看一个特殊的 MSR（Model-Specific Register）寄存器：

```
MSR_LSTAR (IA32_LSTAR)
```

这个寄存器中预先存储了系统调用入口函数的地址。对于 Linux：

```
MSR_LSTAR = entry_SYSCALL_64
```

CPU 自动跳转到这个地址，进入内核的低级汇编处理程序。

类似：
```asm
; CPU 硬件做的：
Ring3 → Ring0
RSP = KernelStack
RIP = entry_SYSCALL_64
```

### 第 8 层：内核的低级处理（汇编）

位置：`arch/x86/entry/entry_64.S`

当 CPU 跳入 `entry_SYSCALL_64` 时，这是一个汇编函数，负责：

```asm
; arch/x86/entry/entry_64.S - 简化版本

ENTRY(entry_SYSCALL_64)
    # 保存更多寄存器（CPU 只自动保存了 RIP、RSP、RFLAGS）
    # 现在需要保存其他寄存器以供高级 C 代码使用
    
    swapgs                      ; 交换 GS 寄存器为内核 GS
    
    # 获取当前进程的内核栈
    mov %rsp, %gs:0x5700        ; 保存用户 RSP 到特殊位置
    mov %gs:0x5708, %rsp        ; 加载内核 RSP
    
    # 创建一个 pt_regs 结构（寄存器现场）
    push %r15
    push %r14
    push %r13
    push %r12
    push %rbx
    push %rcx
    # ... 保存所有寄存器
    
    # 进入 C 代码
    call do_syscall_64
    
    # 恢复寄存器并返回用户态
    pop %rcx
    pop %rbx
    # ... 恢复所有寄存器
    
    # 返回用户态的指令
    sysret                      ; 或其他返回指令
ENDPROC(entry_SYSCALL_64)
```

关键点：
- **pt_regs 结构**：一个数据结构，包含所有保存的寄存器值
- 这个结构被传递给 C 代码 `do_syscall_64()`

### 第 9 层：内核的高级处理（C 代码）

位置：`arch/x86/entry/common.c`

```c
// 简化版本
__visible noinstr void do_syscall_64(struct pt_regs *regs) {
    // regs 包含用户程序的所有寄存器值
    
    // 获取系统调用号
    int syscall_nr = regs->rax;  // RAX 中的值
    
    // 检查系统调用号是否有效
    if (syscall_nr >= NR_syscalls) {
        regs->rax = -ENOSYS;  // 系统调用不存在
        return;
    }
    
    // 通过系统调用号查表，调用对应的系统调用处理函数
    // sys_call_table[0] = __x64_sys_read
    // sys_call_table[1] = __x64_sys_write
    // ...
    
    long result = sys_call_table[syscall_nr](regs);
    
    // 将返回值放入 RAX 寄存器
    regs->rax = result;
}
```

关键点：
- **sys_call_table** 是一个数组，索引就是系统调用号
- `sys_call_table[0]` 指向 `__x64_sys_read` 函数
- 系统调用号从 `regs->rax` 读取

### 第 10 层：调用实际的系统调用处理函数

根据系统调用号，内核调用对应的处理函数。对于 `read()`，就是 `__x64_sys_read()`。

这个函数位置在：`fs/read_write.c`

```c
// 来自 arch/x86/entry/syscalls/syscall_64.tbl
// 0    read    sys_read    __x64_sys_read

SYSCALL_DEFINE3(read, unsigned int, fd, 
                      char __user *, buf, 
                      size_t, count) {
    return ksys_read(fd, buf, count);
}
```

这里：
- `fd` 来自 `RDI`
- `buf` 来自 `RSI`
- `count` 来自 `RDX`

寄存器值被自动转换为函数参数。

### 第 11 层：返回用户态

读取完成后，系统调用处理函数返回一个值（读取的字节数）：

```c
return bytes_read;  // 假设是 1024
```

这个返回值被放入 `RAX` 寄存器：

```
RAX = 1024
```

然后，内核的返回路径（在 `entry_SYSCALL_64`）执行：

```asm
sysret  ; 或 syscall 返回指令
```

这个指令：
1. CPU 自动从 Ring 0 切换回 Ring 3
2. 恢复之前保存的 `RIP`（继续执行用户代码）
3. 恢复之前保存的 `RSP`（恢复用户栈）
4. 恢复 `RFLAGS`（恢复 CPU 标志）

用户程序继续执行，就像没有离开过一样。唯一的区别是 `RAX` 中现在有了返回值：

```c
// 用户程序
ssize_t n = read(fd, buf, 1024);  // n = 1024（来自 RAX）
```

### 完整流程总结

```
用户程序
  ↓
read(3, buf, 1024)
  ↓
glibc 包装器
  RAX = 0 (read 号)
  RDI = 3
  RSI = buf
  RDX = 1024
  syscall 指令
  ↓
CPU 权限转换 (Ring 3 → Ring 0)
CPU 保存用户环境 (RIP, RSP, RFLAGS)
CPU 切换到内核栈
CPU 跳入 entry_SYSCALL_64
  ↓
entry_SYSCALL_64 (汇编)
  创建 pt_regs 结构
  保存所有寄存器
  调用 do_syscall_64()
  ↓
do_syscall_64() (C 代码)
  从 RAX 读取系统调用号 (0)
  查表：sys_call_table[0] = __x64_sys_read
  调用 __x64_sys_read(regs)
  ↓
__x64_sys_read() (内核处理函数)
  从参数读取 fd、buf、count
  执行实际的文件读取逻辑
  ↓
返回用户态
  返回值放入 RAX (1024 字节)
  sysret 指令
  CPU 权限转换 (Ring 0 → Ring 3)
  恢复用户寄存器和栈
  ↓
用户程序
  继续执行下一行
  n = 1024 (从 RAX 读取)
```

现在你了解了系统调用的基本机制。接下来的章节会深入内核的文件读取逻辑。
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
