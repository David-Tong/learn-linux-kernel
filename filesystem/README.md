# 文件系统和 I/O 操作（filesystem）

本目录深入讲解 Linux 内核如何处理文件系统操作和 I/O 请求。通过 `read()` 系统调用作为典范，我们将追踪用户空间请求如何在内核中传递，最终从硬盘读取数据。

## 📚 文章列表

### [read-syscall-kernel-internals.md](read-syscall-kernel-internals.md) - read() 系统调用内核实现

**学习内容**：
- 🔹 POSIX read() 接口定义和用法
- 🔹 系统调用的用户态到内核态转换
- 🔹 系统调用入口点和 syscall 表
- 🔹 文件描述符和 file 结构体
- 🔹 虚拟文件系统（VFS）层抽象
- 🔹 页缓存管理机制
- 🔹 块 I/O 子系统
- 🔹 中断处理和 I/O 完成
- 🔹 数据从内核空间拷贝到用户空间

**关键代码位置**：
- `arch/x86/entry/entry_64.S` - 系统调用入口（第 1-150 行）
- `fs/read_write.c` - sys_read() 实现（第 400-500 行）
- `include/linux/fs.h` - 文件系统核心结构体
- `mm/filemap.c` - 页缓存实现（第 1-100 行）
- `block/blk-core.c` - 块 I/O 核心（第 1-100 行）
- `drivers/ata/` 或 `drivers/scsi/` - 存储设备驱动

**预计时间**：5-6 小时

**难度**：⭐⭐⭐⭐（高度复杂）

**先决条件**：
- 理解进程和虚拟内存概念
- 熟悉 Linux 文件系统基础
- 理解中断和异常处理
- 完成 kernel-boot 目录的学习

---

## 🎓 核心概念

### 系统调用的层次

```
用户程序
   ↓
libc: read(fd, buf, count)
   ↓
系统调用指令 (syscall)
   ↓
arch/x86/entry/entry_64.S: entry_SYSCALL_64
   ↓
kernel/sys.c: SYSCALL_DEFINE3(read, ...)
   ↓
fs/read_write.c: vfs_read()
   ↓
inode->i_fop->read_iter() (VFS 操作)
   ↓
ext4_file_read_iter()
   ↓
mm/filemap.c: generic_file_read_iter() (页缓存)
   ↓
block I/O layer
   ↓
存储驱动
   ↓
硬件中断
   ↓
数据返回到用户空间
```

### I/O 路径概览

```
Page Cache
    ├─ 缓存命中：直接返回数据
    └─ 缓存缺失：
        ↓
    Block I/O Layer
        ├─ 请求合并
        ├─ I/O 调度
        └─ 提交到驱动
        ↓
    Storage Driver (ATA/SCSI/NVMe)
        └─ 生成硬件命令
        ↓
    Storage Device (HDD/SSD)
        ↓
    Hardware Interrupt
        ↓
    Interrupt Handler
        ↓
    I/O Completion
        ↓
    Wake up Process
```

---

## 🔍 关键数据结构

### 文件描述符和 file 结构体

```c
// kernel/fs/file.h - 进程的文件描述符表
struct files_struct {
    struct fdtable *fdt;        // 文件描述符表
    struct fdtable fdtab;       // 内联的 fdtable
    // ...
};

// 文件描述符表的入口
struct fdtable {
    struct file **fd;           // 指向 file 结构体的指针数组
    fd_set *close_on_exec;      // exec() 时关闭的标志
    // ...
};

// kernel/fs/file.h - 表示一个打开的文件
struct file {
    struct path f_path;         // 文件路径
    struct inode *f_inode;      // 指向 inode
    const struct file_operations *f_op;  // 文件操作
    atomic_long_t f_count;      // 引用计数
    unsigned int f_flags;       // 打开标志（O_RDONLY 等）
    fmode_t f_mode;             // 访问模式
    loff_t f_pos;               // 当前文件位置
    // ...
};
```

### inode 和 dentry 结构体

```c
// kernel/include/linux/fs.h - 文件元数据
struct inode {
    umode_t i_mode;             // 文件类型和权限
    unsigned int i_nlink;       // 硬链接数
    kuid_t i_uid;               // 所有者 UID
    kgid_t i_gid;               // 所有者 GID
    loff_t i_size;              // 文件大小
    struct timespec64 i_atime;  // 访问时间
    struct timespec64 i_mtime;  // 修改时间
    struct timespec64 i_ctime;  // 变更时间
    const struct inode_operations *i_op;      // inode 操作
    const struct file_operations *i_fop;      // 文件操作
    struct address_space i_mapping;           // 页缓存
    // ...
};

// kernel/include/linux/dcache.h - 目录项缓存
struct dentry {
    unsigned int d_flags;       // 标志
    seqcount_t d_seq;           // 序列计数（读-复制-更新）
    struct hlist_bl_node d_hash;        // 哈希表节点
    struct dentry *d_parent;    // 父目录项
    struct qstr d_name;         // 目录名
    struct inode *d_inode;      // 指向 inode
    // ...
};
```

### 页缓存

```c
// kernel/include/linux/fs.h - 地址空间（页缓存管理）
struct address_space {
    struct inode *host;         // 拥有的 inode
    struct radix_tree_root pages;  // 页面树（Linux 5.0 后改为 xarray）
    struct rw_semaphore i_mmap_rwsem;  // 映射保护
    // ...
};

// 页缓存的核心概念：
// - 每个 inode 有一个 address_space 结构
// - address_space 维护该文件的所有缓存页面
// - 以文件偏移量为索引快速查找页面
```

---

## 🛠️ 实践操作

### 追踪 read() 系统调用

#### 1. 在系统上观察 read() 调用

```bash
# 使用 strace 追踪系统调用
strace -e read cat /etc/passwd

# 输出类似：
# read(3, "root:x:0:0:root:/root:/bin/bash\n"..., 4096) = 1024
# read(3, "user:x:1000:1000:user:/home/user"..., 4096) = 2048
# read(3, "", 4096) = 0
```

#### 2. 查看源代码中的 read() 实现

```bash
# 获取内核源代码
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout v7.0.0

# 查看系统调用定义
grep -n "SYSCALL_DEFINE3(read" fs/read_write.c

# 查看 sys_read() 函数
sed -n '400,450p' fs/read_write.c

# 查看 vfs_read() 函数
grep -n "^ssize_t vfs_read" fs/read_write.c
sed -n '415,470p' fs/read_write.c
```

#### 3. 追踪页缓存

```bash
# 编译内核时启用页缓存追踪
make menuconfig
# 在 Kernel hacking → Tracers 中启用 Function tracer

# 或使用现有系统追踪
# 在 /sys/kernel/debug/tracing 下工作（需要 root）
echo "generic_file_read_iter" > /sys/kernel/debug/tracing/set_ftrace_filter
echo "function" > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace
```

### 添加调试代码

在 `fs/read_write.c` 的 `vfs_read()` 函数中添加调试信息：

```bash
# 编辑文件
vim fs/read_write.c

# 在 vfs_read() 中添加：
printk(KERN_INFO "read: file=%s, pos=%lld, count=%zu\n", 
       file->f_path.dentry->d_name.name, file->f_pos, count);

# 重新编译
make -j$(nproc)

# 在 QEMU 中运行并查看日志
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -nographic
# 在虚拟机中运行命令
# 查看日志
dmesg | tail -20
```

---

## 📌 关键函数速查

| 函数 | 文件 | 功能 |
|------|------|------|
| `ksys_read()` | `fs/read_write.c` | 系统调用处理程序 |
| `vfs_read()` | `fs/read_write.c` | VFS 层读取 |
| `generic_file_read_iter()` | `mm/filemap.c` | 通用文件读取（页缓存） |
| `filemap_fault()` | `mm/filemap.c` | 页缺失处理 |
| `readahead_control()` | `mm/readahead.c` | 预读机制 |
| `generic_make_request()` | `block/blk-core.c` | 块 I/O 请求提交 |
| `blk_mq_submit_bio()` | `block/blk-mq.c` | 多队列 I/O 提交 |
| `submit_bio()` | `block/blk-core.c` | 生物请求提交 |

---

## 💡 性能优化知识

### 页缓存优化

1. **预读（Read-ahead）**
   - 预测性地加载后续页面
   - 文件：`mm/readahead.c`
   - 位置：`ondemand_readahead()`

2. **脏页回写（Write-back）**
   - 异步将脏页写回磁盘
   - 文件：`mm/page-writeback.c`
   - 位置：`balance_dirty_pages()`

3. **缓存驱逐（Eviction）**
   - LRU 策略驱逐不常用页面
   - 文件：`mm/vmscan.c`
   - 位置：`shrink_inactive_list()`

### I/O 调度优化

Linux 7.0.0 使用多队列块 I/O 调度（blk-mq）：

```bash
# 查看当前 I/O 调度器
cat /sys/block/sda/queue/scheduler

# 修改 I/O 调度器（需要 root）
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# 相关代码位置
# block/mq-deadline.c    - deadline 调度器
# block/blk-mq-sched.c   - 调度框架
# block/blk-mq.c         - 多队列核心
```

---

## 🔗 相关资源

- **Linux VFS 文档**：`Documentation/filesystems/vfs.rst`
- **Linux I/O 子系统**：`Documentation/block/`
- **页缓存详解**：LWN.net 上的文章
- **内核源代码在线浏览**：https://elixir.bootlin.com/linux/v7.0.0/source/

---

## ❓ 常见问题

**Q: read() 调用总是从硬盘读取吗？**
A: 不会。如果数据在页缓存中，直接返回；只有缺失时才从硬盘读取。

**Q: 为什么需要 VFS 层？**
A: VFS 提供统一的接口，使内核支持多种文件系统（ext4、btrfs、NFS 等）。

**Q: 页缓存的大小是固定的吗？**
A: 不是。页缓存大小是动态的，系统会根据内存可用情况自动调整。

**Q: read() 是否线程安全的？**
A: 单个 read() 是原子的，但多个线程共享一个文件描述符时需要加锁。

**Q: 如何提高 I/O 性能？**
A: 使用预读、异步 I/O (aio)、直接 I/O (O_DIRECT) 或内存映射 (mmap) 等技术。

---

**下一步**：准备好深入内核了吗？[开始阅读文章](read-syscall-kernel-internals.md)！
