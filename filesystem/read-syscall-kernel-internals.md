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

#### ⚙️ 宏的真实展开过程

这是理解整个系统的关键。Linux 内核使用宏来自动生成系统调用 stub。

**源文件中的 SYSCALL_DEFINE3 宏定义：**

位置：`include/linux/syscalls.h`

```c
#define SYSCALL_DEFINE3(name, ...) \
    SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
```

这个宏对 `read` 系统调用的调用：

```c
// 位置：fs/read_write.c
SYSCALL_DEFINE3(read, unsigned int, fd, 
                      char __user *, buf, 
                      size_t, count) {
    return ksys_read(fd, buf, count);
}
```

会展开成：

```c
SYSCALL_DEFINEx(3, _read, unsigned int, fd,
                        char __user *, buf,
                        size_t, count) {
    return ksys_read(fd, buf, count);
}
```

**关键注释说明（来自 syscall_64.tbl）：**
> `The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls`

这意味着：
1. 在 `syscall_64.tbl` 中指定 `sys_read` 作为入口点
2. 编译系统根据这个前缀生成 `__x64_sys_read()` 函数
3. `SYSCALL_DEFINE3` 宏为这个生成的 `__x64_sys_read()` 提供参数类型和实现
4. 这个 stub 函数体就是 `{ return ksys_read(...); }`

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

### 三方互动：系统调用表、宏、和实现

这三个部分协同工作：

| 组件 | 位置 | 职责 |
|------|------|------|
| **系统调用表** | `syscall_64.tbl` | 定义系统调用号和入口点名 `sys_read` |
| **SYSCALL_DEFINE3 宏** | `include/linux/syscalls.h` + `fs/read_write.c` | 为自动生成的 stub 提供参数类型和函数体 |
| **自动生成的 stub** | 编译时生成的 `__x64_sys_read()` | 被系统调用表指向，只转发参数给 ksys_read() |
| **真实实现** | `fs/read_write.c` 中的 `ksys_read()` | 完成所有内核工作 |

**具体流程：**

1. **编译时**：
   - 编译系统读取 `syscall_64.tbl`，看到 `0  common  read  sys_read`
   - 编译系统根据 `sys_read` 前缀生成 `__x64_sys_read()` 函数声明
   - `SYSCALL_DEFINE3` 宏为这个生成的 `__x64_sys_read()` 提供参数类型和实现
   - 最终 `__x64_sys_read()` 的函数体是 `{ return ksys_read(fd, buf, count); }`

2. **运行时**：
   - 用户调用 `read()`，执行 `syscall` 指令进入内核
   - 内核根据 RAX = 0 查系统调用表
   - 找到 `__x64_sys_read()` 并跳转
   - `__x64_sys_read()` 转发参数给 `ksys_read()`
   - `ksys_read()` 执行真正的文件读取

### 为什么这样设计？

| 层级 | 名称 | 职责 | 优点 |
|------|------|------|------|
| **表驱动** | 系统调用表 | 定义系统调用号和入口点 | 一个表支持多个架构，自动生成 stub |
| **参数适配** | SYSCALL_DEFINE3 宏 | 参数类型转换和语义检查 | 确保参数正确转换，编译时检查 |
| **自动生成** | `__x64_sys_read()` stub | 被系统调用表调用，参数转发 | 架构相关的统一入口点，自动适配不同架构 |
| **真实实现** | `ksys_read()` | 真正的内核实现 | 代码复用，内核代码可直接调用，无需经过系统调用 |

**好处：**
- **表驱动设计**：一份表支持多个架构，编译时为每个架构生成对应的 stub
- **架构隔离**：x86-64、ARM、RISC-V 等不同架构编译器会生成不同的 stub（`__x64_sys_read()`、`__arm64_sys_read()` 等）
- **参数检查**：宏在编译时确保参数类型正确
- **代码复用**：内核内部代码可以直接调用 `ksys_read()` 而不用经过系统调用机制
- **清晰责任**：表定义选路，宏提供参数适配，自动生成的 stub 转发，实现层做实事

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
    
    // 根据文件的操作函数表选择读取函数
    if (file->f_op->read)
        return file->f_op->read(file, buf, count, pos);  // 传统接口
    else if (file->f_op->read_iter)
        return new_sync_read(file, buf, count, pos);  // 迭代器接口
    else
        return -EINVAL;
}
```

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

## 7. ext4：页缓存未命中后如何读取磁盘

下面以 **Linux 6.8、ext4 普通文件、缓冲读取**为例，假设文件数据已分配在块设备上，文件未启用 DAX，也没有指定 `O_DIRECT`。函数细节会随内核版本变化；示意代码省略了检查和错误处理。

### 7.1 从 VFS 进入 ext4，再进入页缓存

打开 ext4 普通文件后，`file->f_op` 指向 ext4 的文件操作表；它的 `read_iter` 指向 `ext4_file_read_iter`：

```c
// fs/ext4/file.c，简化
const struct file_operations ext4_file_operations = {
    .read_iter = ext4_file_read_iter,
    // ...
};
```

因此，前面的 `new_sync_read()` 调用 `file->f_op->read_iter(...)` 时，实际进入 `ext4_file_read_iter()`。对于本例的普通缓冲读取，ext4 再调用 `generic_file_read_iter()`，后者进入 `filemap_read()`：

```text
vfs_read → new_sync_read → ext4_file_read_iter
         → generic_file_read_iter → filemap_read
```

这解释了为什么 ext4 读取会先经过通用页缓存代码：ext4 选择复用它。直接 I/O 和 DAX 则有不同分支。参见 [Linux 6.8 ext4/file.c](https://github.com/torvalds/linux/blob/v6.8/fs/ext4/file.c)。

### 7.2 页缓存如何回调 ext4

`filemap_read()` 通过 `file->f_mapping` 查找对应文件偏移的缓存 folio（页缓存管理的数据单元）。如果所需数据不在缓存中，内核通常尝试预读，准备一个或多个 folio，并通过 `mapping->a_ops->readahead` 请求文件系统填充；如果所需 folio 仍未就绪，也可能通过 `mapping->a_ops->read_folio` 单独读取。

ext4 在初始化 inode 时，根据文件系统配置为 `inode->i_mapping->a_ops` 选择操作表。普通缓冲读取使用的表包含如下成员：

```c
// fs/ext4/inode.c，简化
static const struct address_space_operations ext4_aops = {
    .read_folio = ext4_read_folio,
    .readahead  = ext4_readahead,
    // ...
};
```

这里是进入 ext4 的第二处函数指针跳转：`f_op->read_iter` 选择文件读取入口；`mapping->a_ops` 选择**如何填充该文件的页缓存**。参见 [ext4 操作表](https://github.com/torvalds/linux/blob/v6.8/fs/ext4/inode.c)和 [页缓存读取说明](https://www.kernel.org/doc/html/v6.8/core-api/mm-api.html)。

### 7.3 ext4 如何找到磁盘上的数据

在 Linux 6.8 中，`ext4_readahead()` 和 `ext4_read_folio()` 最终可以进入 `ext4_mpage_readpages()`。后者根据 folio 的文件偏移计算文件内的逻辑块号，再调用 `ext4_map_blocks()` 查询映射，得到块设备上的物理块号和连续长度。ext4 通常使用 extent 树记录这些映射；对于旧格式文件，也可能使用间接块映射。映射元数据自身如果不在缓存，也可能需要额外 I/O。

例如，假设文件系统块大小为 4096 字节，文件偏移 8192 对应逻辑块 2。如果 ext4 的映射结果是物理块 1002，那么数据位于该块设备从 `1002 × 4096` 字节开始的位置。这只是示例；真实位置由文件系统的映射结果决定，而不是从文件偏移直接推算。

### 7.4 构造 BIO，等待数据并返回

对于可以合并读取的已映射数据，[`ext4_mpage_readpages()`](https://github.com/torvalds/linux/blob/v6.8/fs/ext4/readpage.c) 构造读取用的 `bio`，设置块设备扇区位置和完成回调，将目标页缓存 folio 加入 BIO，再调用 `submit_bio()`：

```c
// fs/ext4/readpage.c，流程示意，省略分配参数和边界处理
ext4_map_blocks(NULL, inode, &map, 0);  // 逻辑块 → 物理块
bio = bio_alloc(/* 块设备、段数、REQ_OP_READ 等 */);
bio->bi_iter.bi_sector = 物理块号 << (块大小的位数 - 9);
bio->bi_end_io = mpage_end_io;
bio_add_folio(bio, folio, length, 0);  // 数据读入页缓存 folio
submit_bio(bio);                       // 提交给块层
```

块层和设备驱动处理请求后，完成回调处理 I/O 结果；成功时 folio 被标记为数据就绪并解锁，等待它的读取者可以继续。`filemap_read()` 随后将数据从页缓存复制到 `iter` 描述的用户缓冲区，推进 `ki_pos`，最终将读取字节数逐层返回给 `read()`。启用加密或 verity 时，完成路径可能还要先解密或验证。

简化调用关系如下，分支和箭头表示可能经过的路径，并不都是直接调用：

```text
read → vfs_read → new_sync_read → ext4_file_read_iter
     → generic_file_read_iter → filemap_read
       → 页缓存未命中或未就绪
       → a_ops->readahead / a_ops->read_folio
       → ext4_readahead / ext4_read_folio
       → ext4_mpage_readpages → ext4_map_blocks
       → bio_alloc + bio_add_folio → submit_bio
       → 块层 / 设备驱动 / 存储设备 → BIO 完成回调
       → folio 就绪 → filemap_read 复制到用户缓冲区 → 返回
```

并非每次缓存未命中都会提交数据 BIO：稀疏空洞可以补零，inline data 可以从 inode 内联数据读取；块不连续等情况可能退回 `block_read_full_folio()`。预读和单 folio 读取也不是每次都会同时发生。

### 块 I/O 和磁盘读取

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

## 8. 文件描述符和 inode 结构

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
6. generic_file_read_iter()       [示例文件系统的通用读取]
  ↓
7. 检查页缓存
  ├─ 命中：复制到用户缓冲区后返回
  └─ 未命中：经过文件系统映射（以下为块设备示例）
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
