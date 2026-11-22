# 系统调用详细参考

> 参考文档：深入讲解用户空间如何切换到内核空间的完整过程（汇编/硬件级别）

本文档提供系统调用机制的详细技术细节，包括汇编代码、寄存器操作、硬件功能等。

请先阅读主文档 [../read-syscall-kernel-internals.md](../read-syscall-kernel-internals.md) 的第 2 章获得基本理解。

## 1. 参数传递的完整细节

### x86-64 寄存器规则

Linux x86-64 系统调用参数传递规则（来自 System V AMD64 ABI）：

```
系统调用号  → RAX
参数 1     → RDI
参数 2     → RSI
参数 3     → RDX
参数 4     → RCX（注意：普通函数调用会用 RCX，但系统调用不用）
参数 5     → R8
参数 6     → R9
返回值     ← RAX
```

这个规则是固定的、全局的，所有系统调用都遵循。

### 为什么必须用寄存器？

**关键原因：syscall 时栈会切换**

```
用户程序
  ├─ 用户栈（User Stack）
  └─ 参数可能在栈上

syscall 执行
  ↓
栈切换

内核
  ├─ 内核栈（Kernel Stack）
  └─ 看不到用户栈了！
```

如果参数在用户栈上，内核无法访问。所以必须用 CPU 寄存器，因为寄存器是 CPU 内部的，不受栈切换影响。

### 具体例子

**例子 1：read(3, buf, 1024)**

```
RAX = 0
RDI = 3        （文件描述符）
RSI = &buf     （缓冲区地址）
RDX = 1024     （字节数）
syscall
```

**例子 2：write(1, "hello", 5)**

```
RAX = 1        （write 的系统调用号）
RDI = 1        （stdout）
RSI = &msg     （"hello" 的地址）
RDX = 5        （长度）
syscall
```

**例子 3：open("/file", O_RDONLY, 0)**

```
RAX = 2        （open 的系统调用号）
RDI = &path    （"/file"）
RSI = 0        （O_RDONLY）
RDX = 0        （mode）
syscall
```

**例子 4：mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, 3, 0) - 6 个参数**

```
RAX = 9        （mmap 的系统调用号）
RDI = 0        （addr）
RSI = 4096     （length）
RDX = 1        （prot = PROT_READ）
RCX = 2        （flags = MAP_PRIVATE）
R8 = 3         （fd）
R9 = 0         （offset）
syscall
```

### 系统调用号表

位置：Linux Kernel 7.0.0 源码中 `arch/x86/entry/syscalls/syscall_64.tbl`

常用系统调用号（x86-64）：

```
0    read
1    write
2    open
3    close
4    stat
5    fstat
6    lstat
9    link
10   unlink
21   access
23   select
32   dup2
33   pause
37   alarm
39   getpid
41   dup
50   gethostname
54   fcntl
57   setpgid
60   umask
...
```

## 2. CPU 硬件层面的操作

### syscall 指令做了什么

当 CPU 执行 `syscall` 时，硬件自动完成以下操作：

**权限切换：**
```
当前：Ring 3（用户态）
     ↓
     syscall 指令执行
     ↓
结果：Ring 0（内核态）
```

**保存返回信息：**
```
CPU 自动保存到特殊寄存器：
  RCX = RIP  （用户程序中 syscall 的下一条指令地址）
  R11 = RFLAGS  （CPU 标志寄存器）
这样 sysret 时才知道回到哪里
```

**栈切换：**
```
用户栈在内存的某个位置
内核栈在内存的另一个位置
RSP 自动指向内核栈
（由 MSR_KERNEL_GS_BASE 和其他 MSR 控制）
```

**跳转到内核入口：**
```
CPU 查看 MSR_LSTAR 寄存器
这个寄存器保存系统调用入口地址
CPU 自动跳转到这个地址
x86-64 Linux 中这个地址是 entry_SYSCALL_64
```

### sysret 指令做了什么

sysret 是 syscall 的反向操作：

```
当前：Ring 0（内核态）
     ↓
     sysret 指令执行
     ↓
结果：Ring 3（用户态）

同时恢复：
  RIP ← RCX （从 syscall 时保存的值）
  RFLAGS ← R11 （从 syscall 时保存的值）
  权限级别 ← Ring 3
```

### MSR（Model-Specific Register）

MSR 是 CPU 的特殊寄存器，用于系统配置：

| MSR 名字 | 寄存器号 | 用途 |
|---------|---------|------|
| MSR_LSTAR | 0xc0000082 | 存储 syscall 入口地址（64位长模式） |
| MSR_CSTAR | 0xc0000083 | 存储 compat 模式 syscall 入口地址 |
| MSR_SYSCALL_MASK | 0xc0000084 | syscall 时要清除的 RFLAGS 位 |
| MSR_GS_BASE | 0xc0000101 | GS 段寄存器的基地址 |
| MSR_KERNEL_GS_BASE | 0xc0000102 | 内核态 GS 段寄存器的基地址 |

Linux 启动时会设置这些 MSR：

```c
wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
wrmsrl(MSR_SYSCALL_MASK, SYSCALL_MASK);
```

## 3. 内核侧的汇编处理

### entry_SYSCALL_64 的流程

位置：`arch/x86/entry/entry_64.S`

```asm
ENTRY(entry_SYSCALL_64)
    swapgs
    
    mov %rsp, %gs:CPU_TSS_SP0
    mov %gs:CPU_TSS_RSP0, %rsp
    
    push %r15
    push %r14
    push %r13
    push %r12
    push %rbp
    push %rbx
    push %r11
    push %r10
    push %r9
    push %r8
    push %rax
    push %rcx
    push %rdx
    push %rsi
    push %rdi
    
    cmp $NR_syscalls, %rax
    jae bad_syscall
    
    mov %rsp, %rdi
    call do_syscall_64
    
    pop %rdi
    pop %rsi
    pop %rdx
    pop %rcx
    pop %rax
    pop %r8
    pop %r9
    pop %r10
    pop %r11
    pop %rbx
    pop %rbp
    pop %r12
    pop %r13
    pop %r14
    pop %r15
    
    mov %gs:CPU_TSS_SP0, %rsp
    swapgs
    
    sysret
ENDPROC(entry_SYSCALL_64)
```

## 4. 内核侧的 C 代码处理

### do_syscall_64 函数

位置：`arch/x86/entry/common.c`

```c
__visible noinstr void do_syscall_64(struct pt_regs *regs) {
    int syscall_nr = regs->ax;
    
    if (syscall_nr >= NR_syscalls) {
        regs->ax = -ENOSYS;
        return;
    }
    
    long result = sys_call_table[syscall_nr](regs);
    regs->ax = result;
}
```

## 5. 完整的调用链

```
用户程序 → glibc read() → syscall 指令
  ↓
CPU 切换到 Ring 0
  ↓
entry_SYSCALL_64（汇编）→ 保存寄存器
  ↓
do_syscall_64（C 语言）→ 查表调用
  ↓
sys_read() → 实际读取操作
  ↓
copy_to_user → 返回数据
  ↓
sysret 指令 → 返回用户态
  ↓
用户程序获得数据
```

## 参考资源

- Intel 官方文档：Intel® 64 and IA-32 Architectures Software Developer's Manual
- System V AMD64 ABI：https://refspecs.linuxbase.org/elf/x86-64-abi-0.99.pdf
- Linux 内核代码：https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/
- glibc 源码：https://sourceware.org/git/glibc.git
