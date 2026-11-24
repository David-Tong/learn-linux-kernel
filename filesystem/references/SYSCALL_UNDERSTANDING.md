# Linux 系统调用机制完整理解

## 核心问题：sys_read 和 ksys_read 怎么连接的？

这份文档回答你最初的问题。

## 答案总结

```
系统调用表 (syscall_64.tbl)
└─ 0  common  read  sys_read
   └─ 编译系统生成 __x64_sys_read() stub
      └─ SYSCALL_DEFINE3 宏为这个 stub 提供实现
         └─ __x64_sys_read() { return ksys_read(...); }
            └─ ksys_read() 执行真正工作
```

## 四个关键层级

### 1. 系统调用表（syscall_64.tbl）

**位置**: `arch/x86/entry/syscalls/syscall_64.tbl`

**格式**（来自文件头注释）:
```
# <number> <abi> <name> <entry point> [<compat entry point> [noreturn]]
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
```

**read 系统调用的定义**:
```
0	common	read			sys_read
```

**含义**:
- 系统调用号: 0
- 应用场景: common（通用）
- 文字名称: read
- 入口点前缀: sys_read（这是关键！）

**关键点**: 表中只指定 `sys_read`，不是 `__x64_sys_read`

### 2. 编译时自动生成的 Stub

编译系统读到 `sys_read` 这个前缀后，自动生成 `__x64_sys_read()` 函数。

**生成规则**:
```
sys_read  →编译系统→  __x64_sys_read()
sys_write →编译系统→  __x64_sys_write()
sys_open  →编译系统→  __x64_sys_open()
```

这样，不同架构可以有不同的 stub：
- x86-64: `__x64_sys_read()`
- ARM64: `__arm64_sys_read()`
- RISC-V: `__riscv_sys_read()`

### 3. SYSCALL_DEFINE3 宏

**宏定义**（位置: `include/linux/syscalls.h`）:
```c
#define SYSCALL_DEFINE3(name, ...) \
    SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
```

**使用**（位置: `fs/read_write.c`）:
```c
SYSCALL_DEFINE3(read, unsigned int, fd, 
                      char __user *, buf, 
                      size_t, count) {
    return ksys_read(fd, buf, count);
}
```

**宏的作用**:
1. 将 `read` 转换为 `_read`
2. 传给 `SYSCALL_DEFINEx(3, _read, ...)`
3. 这个宏为自动生成的 `__x64_sys_read()` 提供参数类型和函数体
4. 确保参数正确转换和检查

**关键认识**: SYSCALL_DEFINE3 不是生成 stub 名称，而是为已经自动生成的 stub 提供实现！

### 4. 真实实现

**ksys_read()**（位置: `fs/read_write.c`）:
```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count) {
    // 完成真正的文件读取工作
    struct fd f = fdget_pos(fd);
    // ... 获取 file 结构
    ret = vfs_read(f.file, buf, count, &pos);
    // ... 处理返回值
    return ret;
}
```

## 编译时和运行时的流程

### 编译时

```
1. 编译系统扫描 syscall_64.tbl
   ↓
2. 看到 "0  common  read  sys_read"
   ↓
3. 根据 sys_read 前缀生成 __x64_sys_read() 函数声明
   ↓
4. SYSCALL_DEFINE3 宏处理 fs/read_write.c 中的代码
   ↓
5. 生成 __x64_sys_read() 的函数体: { return ksys_read(fd, buf, count); }
   ↓
6. 编译器生成最终的二进制代码
```

### 运行时

```
用户程序调用 read(fd, buf, count)
   ↓
glibc 中的 read() 准备参数并执行 syscall 指令
   - RAX = 0（系统调用号）
   - RDI = fd
   - RSI = buf
   - RDX = count
   ↓
CPU 切换到内核态（Ring 3 → Ring 0）
   ↓
内核入口函数 entry_SYSCALL_64 读取 RAX = 0
   ↓
内核查询系统调用表 syscall_64.tbl
   ↓ 找到：0  common  read  sys_read  →  __x64_sys_read()
   ↓
跳转到 __x64_sys_read()
   ↓
__x64_sys_read() 执行: return ksys_read(fd, buf, count);
   ↓
ksys_read() 执行真正的读取逻辑
   ↓
返回字节数给用户程序
```

## 四方协作模式

这四个部分如何协作：

| 部分 | 位置 | 时机 | 职责 |
|------|------|------|------|
| **系统调用表** | `syscall_64.tbl` | 编译前 | 定义系统调用号和入口点名 |
| **编译系统** | Linux build | 编译时 | 根据入口点名生成 stub 函数 |
| **SYSCALL_DEFINE3** | `include/linux/syscalls.h` + `fs/read_write.c` | 编译时 | 为 stub 提供参数和实现 |
| **系统调用表查询** | 内核运行时 | 运行时 | 根据 RAX 查表找到 stub 并跳转 |

## 为什么这样设计？

### 1. 表驱动

- 一份表可支持多个架构
- 编译时为每个架构生成不同的 stub
- 只需维护一份表定义

### 2. 自动生成

- 减少手工编写 stub 函数的工作量
- 确保命名规范和参数一致
- 编译器自动生成，不易出错

### 3. 两层分离

**系统调用层** (`__x64_sys_read`):
- 被系统调用表指向
- 只转发参数
- 架构相关

**实现层** (`ksys_read`):
- 真正的功能实现
- 架构无关
- 内核代码可以直接调用（不经过系统调用）

### 4. 编译时检查

- SYSCALL_DEFINE3 宏进行参数类型检查
- 编译时发现问题，而不是运行时

## 关键术语澄清

| 术语 | 含义 | 位置 | 时机 |
|------|------|------|------|
| `sys_read` | 表中的入口点前缀名 | `syscall_64.tbl` | 编译前定义 |
| `__x64_sys_read()` | 编译系统生成的 stub 函数 | 编译生成 | 编译时生成 |
| `ksys_read()` | 真实的内核实现函数 | `fs/read_write.c` | 编译前定义 |
| `SYSCALL_DEFINE3` | C 预处理宏，为 stub 提供实现 | `include/linux/syscalls.h` | 编译时展开 |

## 一行表定义如何生成一个完整的系统调用

```
0	common	read			sys_read
  ↓
  编译系统提取 "sys_read" 前缀
  ↓
  为 x86-64 生成: __x64_sys_read()
  为 ARM64 生成: __arm64_sys_read()
  为 RISC-V 生成: __riscv_sys_read()
  ↓
  SYSCALL_DEFINE3(read, ...) 宏处理 fs/read_write.c 中的代码
  ↓
  为所有生成的 stub 提供参数类型和函数体:
  {
      return ksys_read(fd, buf, count);
  }
  ↓
  编译完成！一行表定义 + 一个宏 = 多个架构的系统调用 stub
```

## 总结

你最初的问题：
> `ksys_read` 怎么和 `0	common	read	sys_read` 连接在一起的？

**答案**：
1. 系统调用表定义了系统调用号和入口点名 `sys_read`
2. 编译时，编译系统根据 `sys_read` 自动生成 `__x64_sys_read()` 函数
3. SYSCALL_DEFINE3 宏为这个自动生成的 `__x64_sys_read()` 提供参数和实现
4. 这个实现就是调用真实的 `ksys_read()` 函数
5. 运行时，系统调用表被查询，找到 `__x64_sys_read()`，跳转执行

这就是完整的连接过程！
