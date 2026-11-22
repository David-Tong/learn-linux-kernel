# Linux 内核中的 read() 系统调用

> 本文基于 Linux Kernel 7.0.0 版本作为参考。
> 
> **文档结构**：本文按照函数调用链组织，每一章都围绕一个函数展开，从用户空间的 `read()` 一直追踪到内核的具体实现。

## 概述

当用户程序调用 `read(fd, buf, count)` 时，会经过以下函数调用链：

```
用户态
└─ read() (glibc)
     ↓ [syscall 指令 - 权限切换]
内核态
└─ 系统调用表查询（syscall_64.tbl）
     ↓ RAX = 0 → sys_read
     ↓
└─ __x64_sys_read() (编译时自动生成的 stub)
     ↓ [直接转发参数]
     ↓
└─ ksys_read() (真正的内核实现)
     ↓
└─ vfs_read() (虚拟文件系统)
     ↓
└─ 具体文件系统操作
     ↓
└─ (页缓存、块设备等)
```

**核心理解：**
- 系统调用表定义 `sys_read`，编译时自动生成 `__x64_sys_read()` stub
- `__x64_sys_read()` 是系统调用入口点，只负责参数转发
- `ksys_read()` 是**真实实现**，完成所有读取工作

> **相关参考**：想了解 Python read() 和内核 read() 的关系？请参考 [python-read-reference.md](references/python-read-reference.md)

---

## 1. read() - glibc 中的 POSIX 包装函数

### 函数位置
- **源文件**：`glibc/sysdeps/unix/sysv/linux/x86_64/syscall.S`
- **类型**：汇编函数（预先编写，链接时复制到程序中）

### 函数签名

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

**参数：**
- `fd`：文件描述符（整数，由 open() 等系统调用返回）
- `buf`：用户空间缓冲区指针，用于接收读取的数据
- `count`：要读取的最大字节数

**返回值：**
- `> 0`：实际读取的字节数
- `0`：到达文件末尾（EOF）
- `< 0`：发生错误（返回 -1，错误代码在 errno 中）

### 函数的作用

这是 POSIX 标准定义的文件读取接口。由于用户程序无法直接访问磁盘硬件或内核数据结构，所以这个函数的主要作用是：

1. **打包参数**：将用户的三个参数打包
2. **切换权限**：通过 syscall 指令请求进入内核
3. **返回结果**：内核处理完毕后，返回读取的字节数

### 实现细节

这是一个**汇编包装函数**，完成以下工作：

```asm
read:
    # 准备系统调用号
    mov $0, %rax           # RAX = 0（read 的系统调用号）
    
    # 参数已经在相应的寄存器中（来自调用者）
    # RDI = fd
    # RSI = buf
    # RDX = count
    
    # 执行系统调用
    syscall                # 进入内核
    
    # 检查返回值
    cmp $-4095, %rax       # 检查是否为负数（-4095 到 -1 表示错误）
    jae .L_error
    
    # 成功：返回值在 RAX 中
    ret

.L_error:
    # 错误处理：设置 errno，返回 -1
    # ...
```

### 调用下一个函数

当 `read()` 执行 `syscall` 指令时：

1. **CPU 硬件**自动进行权限切换（Ring 3 → Ring 0）
2. **硬件**跳转到内核入口：`entry_SYSCALL_64()`
3. **内核**通过系统调用号（RAX = 0）查表，在 `syscall_64.tbl` 中找到 `sys_read`
4. **调用**：跳转到编译时自动生成的 `__x64_sys_read()` stub 函数

### 使用示例

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    // 打开文件
    int fd = open("file.txt", O_RDONLY);
    if (fd < 0) {
        perror("open failed");
        return 1;
    }
    
    // 分配缓冲区
    char buffer[1024];
    
    // 调用 read()
    ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
    
    // 处理结果
    if (bytes_read < 0) {
        perror("read failed");
        close(fd);
        return 1;
    }
    
    if (bytes_read == 0) {
        printf("End of file\n");
    } else {
        printf("Read %ld bytes\n", bytes_read);
        // 使用 buffer 中的数据...
    }
    
    close(fd);
    return 0;
}
```

---

## 2. 用户态到内核态的权限切换

### 硬件机制概述

这一章不涉及具体的函数，而是解释 CPU 的硬件功能如何实现用户态和内核态之间的转换。

### CPU 的两个权限级别

现代 CPU（x86-64）提供两个不同的权限级别：

- **Ring 3（用户态）**：普通用户程序运行在这里，无法直接操作硬件或访问内核内存
- **Ring 0（内核态）**：操作系统内核运行在这里，拥有完全的硬件访问权限

这种权限隔离的作用：
- **系统保护**：单个用户程序的错误不会崩溃整个系统
- **数据安全**：用户程序无法访问其他用户的数据
- **资源管理**：内核可以管理和限制资源使用

### syscall 指令的作用

`syscall` 是 x86-64 CPU 的一条特殊指令，作用是**从用户态切换到内核态**。

当 CPU 执行 `syscall` 时，硬件自动完成以下操作（这些都是强制性的，用户程序无法改变）：

**权限切换：**
```
执行前：Ring 3（用户态）
执行后：Ring 0（内核态）
```

**保存返回信息：**
```
CPU 自动保存：
  - RCX ← RIP（下一条用户指令的地址）
  - R11 ← RFLAGS（CPU 状态标志寄存器）
这样 sysret 时才知道回到哪里
```

**栈切换：**
```
用户程序有自己的栈
内核也有自己的栈
RSP 自动指向内核栈
（这样内核代码的栈操作不会影响用户栈）
```

**跳转到内核入口：**
```
CPU 查看 MSR_LSTAR 寄存器
这个寄存器保存系统调用入口地址
CPU 自动跳转到这个地址：entry_SYSCALL_64()
```

### 参数如何传递

由于栈会被切换，参数**不能**放在栈上。系统调用使用 **CPU 寄存器**来传递参数：

**x86-64 系统调用参数规则：**

| 参数 | 寄存器 | 用途 |
|------|--------|------|
| 系统调用号 | RAX | 告诉内核要调用哪个系统调用 |
| 参数 1 | RDI | fd |
| 参数 2 | RSI | buf |
| 参数 3 | RDX | count |
| 参数 4 | RCX | （不使用） |
| 参数 5 | R8 | （不使用） |
| 参数 6 | R9 | （不使用） |
| 返回值 | RAX | read 返回的字节数 |

详细内容见 [系统调用详细参考](references/syscall-detailed-reference.md)。

### 返回用户态

内核完成 `read()` 操作后，执行 `sysret` 指令返回：

**sysret 指令的作用：**

```
执行前：Ring 0（内核态）
执行后：Ring 3（用户态）

同时恢复：
  - RIP ← RCX（恢复到用户程序的下一条指令）
  - RFLAGS ← R11（恢复 CPU 标志）
```

### 完整的权限切换过程

```
用户程序
  ↓
read() 准备参数并执行 syscall
  ↓
CPU 硬件自动：权限切换 + 保存返回地址 + 切换栈 + 跳转
  ↓
内核入口函数
  ↓
（内核处理读取请求）
  ↓
准备返回值
  ↓
CPU 硬件自动：恢复权限 + 恢复返回地址 + 恢复栈
  ↓
用户程序继续执行，获得返回值
```

---

## 3. sys_read / __x64_sys_read() - 系统调用包装器

### 核心问题：sys_read 和 ksys_read 怎么连接的？

这是理解系统调用的关键问题。让我详细解释：

#### 🔗 三层关系

```
系统调用表 (arch/x86/entry/syscalls/syscall_64.tbl)
     ↓
  0  common  read  sys_read
     ↓
  __x64_sys_read() (编译时由 sys_read 动态生成的 stub)
     ↓
  ksys_read (真正的内核实现，位于 fs/read_write.c)
```

#### 📋 系统调用号表的真实格式

位置：`arch/x86/entry/syscalls/syscall_64.tbl`

**表头注释（包含完整定义）：**

```
# SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note
#
# 64-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point> [<compat entry point> [noreturn]]
#
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
#
# The abi is "common", "64" or "x32" for this file.
#
```

**实际的 read 系统调用条目：**

```
0	common	read			sys_read
```

**表中各列的含义：**

| 列 | 名称 | 含义 | 例子 |
|---|------|------|------|
| 1 | `<number>` | 系统调用号（放在 RAX 寄存器） | 0（read） |
| 2 | `<abi>` | 应用场景（common = 通用，64 = x86-64专用，x32 = 32位模式） | common |
| 3 | `<name>` | 系统调用的文字名称 | read |
| 4 | `<entry point>` | 表驱动生成的入口点前缀名 | sys_read |
| 5+ | `[<compat>]` | 可选：兼容模式入口点 | （无） |

#### ⚙️ 编译时动态生成的关键秘密

这是理解整个系统的关键：

**系统调用表文件中的定义：**
```
0  common  read  sys_read
```

**编译时会自动生成：**
```c
// Linux build系统根据 sys_read 前缀自动生成
// 生成一个名为 __x64_sys_read() 的 stub 函数
asmlinkage long __x64_sys_read(unsigned int fd, 
                               char __user *buf, 
                               size_t count);
```

**关键注释说明：**
> `The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls`

这意味着：
- 表中填 `sys_read`
- 编译时自动生成 `__x64_sys_read()` stub
- 这个 stub 会调用 `ksys_read()` 的实现

#### 🔄 完整的查询和执行流程

```
用户程序执行 syscall 指令（RAX = 0）
   ↓
CPU 切换到内核态
   ↓
内核入口函数 entry_SYSCALL_64 读取 RAX
   ↓
查询系统调用表 syscall_64.tbl
   ↓ 找到：0  common  read  sys_read
   ↓
调用自动生成的 __x64_sys_read() stub
   ↓
这个 stub 调用真实实现 ksys_read()
   ↓
ksys_read() 执行真正的读取逻辑
```

### 函数位置和类型

- **源文件**：
  - 表定义：`arch/x86/entry/syscalls/syscall_64.tbl`（手写）
  - 包装器实现：`fs/read_write.c`（SYSCALL_DEFINE3(read, ...) 调用）
- **生成位置**：
  - Stub 生成：`arch/x86/include/asm/syscalls.h`（由 Linux build 系统自动生成）
  - 宏定义：`include/linux/syscalls.h`（SYSCALL_DEFINE3 定义）
- **类型**：
  - `sys_read`：表中定义的入口点名称
  - `__x64_sys_read()`：编译时自动生成的 stub 函数（C 语言）

### sys_read 和 __x64_sys_read 的关系

这两个名称的关系是**自动生成的**：

| 名称 | 来源 | 生成时机 | 用途 |
|------|------|---------|------|
| `sys_read` | 系统调用表手写定义 | 编译前存在 | 编译时用来生成 stub 名称 |
| `__x64_sys_read` | 自动生成的 stub | 编译时生成 | 实际被系统调用表调用 |

**生成规则：**
系统调用表前缀 `sys_*` → 自动生成 `__x64_sys_*()` stub

这样做的好处是：
- 一行表定义就能自动生成不同架构的 stub 函数
- x86-64 生成 `__x64_sys_read()`
- ARM64 生成 `__arm64_sys_read()`
- RISC-V 生成 `__riscv_sys_read()`

### 函数的作用

当内核收到系统调用时（RAX = 0）：

1. **表查询**：根据 RAX = 0 在 `syscall_64.tbl` 中查找
2. **找到条目**：`0  common  read  sys_read`
3. **调用生成的 stub**：跳转到自动生成的 `__x64_sys_read()` 函数
4. **参数转发**：这个 stub 接收来自寄存器的参数
5. **调用实现**：直接转发给 `ksys_read()`

### SYSCALL_DEFINE3 在其中的角色

在 `fs/read_write.c` 中：

```c
SYSCALL_DEFINE3(read, unsigned int, fd, 
                      char __user *, buf, 
                      size_t, count) {
    return ksys_read(fd, buf, count);
}
```

这个宏做的是：
1. **定义参数类型**：告诉编译器参数有哪些以及类型是什么
2. **创建 ksys_read 的包装**：让自动生成的 stub 能正确调用 `ksys_read()`
3. **兼容性检查**：确保参数传递的正确性

**展开后的职责：**
1. 接收来自 CPU 寄存器的参数
2. 进行基础的参数检查和转换
3. 转发给 `ksys_read()`，让它执行真正的工作

### 为什么这样设计？

| 层级 | 名称 | 职责 | 优点 |
|------|------|------|------|
| **表驱动** | 系统调用表 | 定义系统调用号和入口点 | 一个表支持多个架构，自动生成 stub |
| **自动生成** | `__x64_sys_read()` stub | 被系统调用表调用，参数转发 | 架构相关的统一入口点，自动适配不同架构 |
| **真实实现** | `ksys_read()` | 真正的内核实现 | 代码复用，内核代码可直接调用，无需经过系统调用 |

**好处：**
- **表驱动设计**：一份表支持多个架构，编译时为每个架构生成对应的 stub
- **架构隔离**：x86-64、ARM、RISC-V 等不同架构编译器会生成不同的 stub
- **代码复用**：内核内部代码可以直接调用 `ksys_read()` 而不用经过系统调用机制
- **清晰责任**：表定义选路，自动生成的 stub 转发，实现层做实事

### 调用下一个函数

自动生成的 `__x64_sys_read()` stub 直接调用 `ksys_read(fd, buf, count)`

---

## 4. ksys_read() - 内核的 read 实现

### 核心理解：ksys_read 是真实实现

在第3章我们讲解了系统调用表和自动生成的 stub。现在我们来看 `ksys_read()` 这个**真实的实现函数**。

**关键点：**
- `sys_read` 是系统调用表中的入口点定义
- `__x64_sys_read()` 是编译时自动生成的 stub 函数（系统调用入口点）
- `ksys_read()` 是**真实实现**，完成所有内核工作
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

这是内核表示打开文件的核心结构（位置：`include/linux/fs.h`），包含：
- 文件操作函数指针（`f_op`）
- 当前读取位置（`f_pos`）
- 文件权限（`f_mode`）
- 关联的 inode（`f_inode`）
- 等等

### 调用下一个函数

`ksys_read()` 调用 `vfs_read(f.file, buf, count, &pos)`

---

## 5. vfs_read() - 虚拟文件系统读取

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
    
    // 根据文件类型选择不同的读取函数
    if (file->f_op->read_iter)
        return new_sync_read(file, buf, count, pos);  // 现代读取路径
    else if (file->f_op->read)
        return file->f_op->read(file, buf, count, pos);  // 兼容性路径
    else
        return -EINVAL;
}
```

### 调用下一个函数

根据文件类型，VFS 层会调用具体的读取函数。对于普通文件，通常调用：
- `generic_file_read_iter()`（现代路径）
- 或 `generic_file_read()`（兼容路径）

---

## 6. generic_file_read_iter() - 普通文件读取

### 函数位置
- **源文件**：`mm/filemap.c`
- **类型**：C 语言函数

### 函数签名

```c
ssize_t generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
```

这个函数较复杂，处理页缓存、预读等。

### 函数的作用

普通文件（存储在磁盘上的文件）的读取操作，包括：

1. **检查页缓存**：数据是否已经在内存中
2. **预读**：提前读入更多数据以提高性能
3. **缺页处理**：如果数据不在内存，从磁盘读入
4. **数据复制**：将数据从内核缓冲区复制到用户缓冲区

### 页缓存

页缓存（Page Cache）是 Linux 内核的一个关键机制：

- 每个文件的内容都被缓存在内存中
- 如果数据已在缓存中，直接返回（快速路径）
- 如果数据不在缓存中，从磁盘读入（慢速路径）
- 由内核自动管理，用户程序看不到

---

## 7. 文件描述符和 inode 结构

### 文件描述符表

每个进程都有一个**文件描述符表**（位置：`include/linux/fdtable.h`）：

```c
struct fdtable {
    struct file **fd;      // 文件指针数组（索引 = fd 号）
    fd_set *close_on_exec; // exec 时关闭的标志
    fd_set *open_fds;      // 打开的文件标志
    // ...
};
```

**工作原理：**

```
用户调用：read(3, buf, 1024)
  ↓
内核查找：fdtable->fd[3]
  ↓
获得：struct file *file
  ↓
使用这个 file 结构进行读取
```

### 文件结构（struct file）

位置：`include/linux/fs.h`

这个结构代表**打开的文件实例**（每打开一次就创建一个新的 file 结构）：

```c
struct file {
    struct path             f_path;        // 文件路径
    struct inode           *f_inode;       // 指向 inode
    const struct file_operations *f_op;    // 文件操作函数（read/write 等）
    
    atomic_long_t           f_count;       // 引用计数
    unsigned int            f_flags;       // 打开标志（O_RDONLY 等）
    fmode_t                 f_mode;        // 访问模式（读/写等）
    
    struct mutex            f_pos_lock;    // 保护 f_pos
    loff_t                  f_pos;         // 当前读取位置（文件偏移）
    
    struct fown_struct      f_owner;       // 信号接收者
    const struct cred      *f_cred;        // 权限凭据
    struct file_ra_state    f_ra;          // 预读状态
    // ... 更多字段
};
```

**重要字段说明：**

- `f_inode`：指向代表文件本身的 inode 结构
- `f_op`：函数指针表，包含 read()、write()、mmap() 等操作
- `f_pos`：当前的读取位置。每次读取后会更新
- `f_flags`：打开时的标志（O_RDONLY、O_WRONLY、O_APPEND 等）

### Inode 结构（struct inode）

位置：`include/linux/fs.h`

这个结构代表**文件本身**（无论打开多少次，只有一个 inode）：

```c
struct inode {
    umode_t                 i_mode;        // 文件类型和权限
    unsigned short          i_opflags;     // 操作标志
    kuid_t                  i_uid;         // 文件所有者的 UID
    kgid_t                  i_gid;         // 文件所有者的 GID
    
    loff_t                  i_size;        // 文件大小（字节）
    
    const struct inode_operations *i_op;   // inode 操作
    const struct file_operations  *i_fop;  // 默认文件操作
    struct super_block     *i_sb;          // 指向所在的超块
    
    unsigned long           i_ino;         // inode 号
    // ... 更多字段
};
```

**重要字段说明：**

- `i_mode`：文件类型（普通文件、目录等）和权限（rwxrwxrwx）
- `i_size`：文件的总大小
- `i_sb`：指向超块（super_block），代表文件系统

### file vs inode

```
文件描述符表（进程）
  ↓
struct file （打开实例）
  ├─ f_pos         ← 每个 open 都有自己的读取位置
  ├─ f_flags       ← 打开时的标志
  └─ *f_inode ───→ struct inode （文件本身）
                     ├─ i_size      ← 文件大小（所有打开实例共享）
                     ├─ i_mode      ← 权限（所有打开实例共享）
                     └─ i_sb        ← 文件系统信息
```

---

## 8. 块 I/O 和磁盘读取

### 缺页处理

当需要的数据不在页缓存中时，内核需要从磁盘读入。这涉及：

1. **分配页面**：为数据分配一个物理内存页
2. **创建 Bio 请求**：生成块 I/O 请求
3. **提交给块设备**：交给磁盘控制器
4. **等待完成**：等待磁盘返回数据
5. **唤醒进程**：数据到达后唤醒等待的进程

### 块 I/O（Bio）

位置：`include/linux/bio.h`

```c
struct bio {
    sector_t            bi_sector;    // 磁盘上的扇区号
    struct bvec_iter    bi_iter;      // 数据在内存中的位置
    struct bio_vec     *bi_io_vec;    // 内存页的数组
    int                 bi_vcnt;      // 页数
    // ... 更多字段
};
```

这个结构描述一次**块设备 I/O 操作**。

---

## 总结：完整的调用链

```
用户程序
  ↓
1. read(fd, buf, count)           [glibc, 用户态]
  ↓ syscall
2. CPU 权限切换和栈切换            [硬件]
  ↓
3. __x64_sys_read()               [内核入口]
  ↓
4. ksys_read()                    [内核 read 实现]
  ↓
5. vfs_read()                     [虚拟文件系统]
  ↓
6. generic_file_read_iter()       [普通文件读取]
  ↓
7. 检查页缓存
  ├─ 命中：直接返回              [快速路径]
  └─ 未命中：
      ↓
8. 创建 Bio 请求                  [块 I/O]
  ↓
9. submit_bio()                   [提交到块设备层]
  ↓
10. 磁盘控制器处理                [硬件]
  ↓
11. 中断/回调：数据到达            [硬件]
  ↓
12. copy_to_user()                [复制到用户缓冲区]
  ↓
13. sysret                        [返回到用户态]
  ↓
用户程序获得数据
```

---

## 附录：源代码位置快速查询

| 函数 | 文件 | 行号范围 |
|------|------|---------|
| `read()` | glibc/sysdeps/unix/sysv/linux/x86_64/syscall.S | - |
| `entry_SYSCALL_64()` | arch/x86/entry/entry_64.S | - |
| `do_syscall_64()` | arch/x86/entry/common.c | - |
| `__x64_sys_read()` | arch/x86/include/asm/syscalls.h | (宏生成) |
| `ksys_read()` | fs/read_write.c | - |
| `vfs_read()` | fs/read_write.c | - |
| `generic_file_read_iter()` | mm/filemap.c | - |
| `struct file` | include/linux/fs.h | - |
| `struct inode` | include/linux/fs.h | - |
| `syscall_64.tbl` | arch/x86/entry/syscalls/syscall_64.tbl | - |

---

> 想要了解 CPU 寄存器的具体映射、汇编代码细节、硬件操作等技术细节，请参考 [系统调用详细参考](references/syscall-detailed-reference.md)。
