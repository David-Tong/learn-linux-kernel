# Linux 内核启动过程（kernel-boot）

本目录包含关于 Linux 内核启动的详细教程，涵盖从 BIOS 到内核完全初始化的整个过程。

## 📚 文章列表

### 1. [vmlinuz-assembly-code-part1.md](vmlinuz-assembly-code-part1.md) - 启动协议和实模式代码

**学习内容**：
- 🔹 Linux 启动协议规范（zImage vs bzImage）
- 🔹 实模式下的 CPU 状态和寻址方式
- 🔹 BIOS 引导参数结构
- 🔹 实模式初始化代码的第一步

**关键代码位置**：
- `Documentation/x86/boot.rst` - 启动协议文档
- `arch/x86/boot/header.S` - 实模式入口（第 1-200 行）
- `arch/x86/boot/main.c` - 设置代码主程序
- `arch/x86/include/uapi/asm/bootparam.h` - 参数结构定义

**预计时间**：2-3 小时

**难度**：⭐⭐（中等）

**先决条件**：
- 基本的 x86 汇编知识
- 理解 CPU 寄存器概念

---

### 2. [vmlinuz-assembly-code-part2.md](vmlinuz-assembly-code-part2.md) - 保护模式和长模式转换

**学习内容**：
- 🔹 全局描述符表（GDT）的设置和作用
- 🔹 进入保护模式的具体步骤
- 🔹 32 位和 64 位代码的转换
- 🔹 页表结构和初始化
- 🔹 进入长模式（64 位）的完整过程

**关键代码位置**：
- `arch/x86/boot/pm.c` - 保护模式转换代码
- `arch/x86/boot/compressed/head_32.S` - 32 位汇编代码
- `arch/x86/boot/compressed/head_64.S` - 64 位汇编和页表（第 1-300 行）
- `arch/x86/boot/cpucheck.c` - CPU 功能检查

**预计时间**：3-4 小时

**难度**：⭐⭐⭐（较难）

**先决条件**：
- 完成 part1 的学习
- 熟悉 x86 保护模式概念
- 理解分页机制基础

---

### 3. [vmlinuz-c-code-part1.md](vmlinuz-c-code-part1.md) - 解压缩和 C 代码初始化

**学习内容**：
- 🔹 多种压缩格式的支持和选择
- 🔹 内核解压缩的完整过程
- 🔹 E820 内存表的解析
- 🔹 start_kernel() 函数的初始化流程
- 🔹 内存分配器（buddy system）的初始化

**关键代码位置**：
- `arch/x86/boot/compressed/misc.c` - 解压缩主程序（第 1-100 行）
- `lib/decompress_gzip.c` - gzip 解压实现
- `arch/x86/kernel/head64.c` - C 代码入口（第 1-150 行）
- `init/main.c` - start_kernel() 函数（第 800-1100 行）
- `mm/page_alloc.c` - 内存分配器初始化（第 6700-6800 行）

**预计时间**：3-4 小时

**难度**：⭐⭐⭐（需要理解内存管理）

**先决条件**：
- 完成 part1 和 part2 的学习
- 基本的 C 语言知识
- 理解内存寻址概念

---

## 🎓 学习路径

### 推荐顺序
1. 先阅读本 README 获得整体概览
2. 按顺序学习 part1 → part2 → part3
3. 每部分学完后，查看对应的源代码
4. 编译内核时追踪这些代码的执行

### 时间规划
- 完整学习：8-12 小时
- 快速浏览：3-4 小时
- 深入研究（包括代码追踪）：15-20 小时

---

## 🔍 关键概念速记

### 实模式 vs 保护模式 vs 长模式

| 特性 | 实模式 | 保护模式 | 长模式 |
|------|--------|---------|--------|
| **位宽** | 16 位 | 32 位 | 64 位 |
| **寻址** | 段:偏移 | 线性地址 | 线性地址 |
| **最大地址** | 1MB | 4GB | 2^64 字节 |
| **段描述符** | 无 | GDT/LDT | LDT |
| **分页** | 否 | 可选 | 必需 |

### 启动流程概览

```
BIOS/UEFI
   ↓
Bootloader (GRUB)
   ↓
kernel/arch/x86/boot/header.S (实模式)
   ↓
kernel/arch/x86/boot/pm.c (进入保护模式)
   ↓
kernel/arch/x86/boot/compressed/head_32.S (32位)
   ↓
kernel/arch/x86/boot/compressed/head_64.S (进入长模式+页表)
   ↓
kernel/arch/x86/boot/compressed/misc.c (解压缩)
   ↓
kernel/arch/x86/kernel/head64.c (C代码入口)
   ↓
kernel/init/main.c:start_kernel() (正式初始化)
```

---

## 🛠️ 实践操作

### 查看启动代码

```bash
# 克隆内核源代码
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout v7.0.0

# 查看启动协议文档
head -100 Documentation/x86/boot.rst

# 查看实模式入口代码
head -100 arch/x86/boot/header.S

# 查看保护模式转换代码
cat arch/x86/boot/pm.c | head -50

# 查看 start_kernel 函数
grep -n "asmlinkage __visible void start_kernel" init/main.c
sed -n '800,850p' init/main.c
```

### 追踪代码执行

在编译内核时可以添加 `printk()` 调试语句来观察执行流程：

```bash
# 在 init/main.c 的 start_kernel() 中添加
printk(KERN_INFO "Starting kernel initialization...\n");

# 重新编译
make -j$(nproc)

# 在 QEMU 中运行并查看日志
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -nographic
# 或使用 dmesg 查看启动日志
dmesg | head -50
```

---

## 📌 重要文件速查

| 功能 | 文件名 | 主要内容 |
|------|--------|---------|
| 启动协议 | `Documentation/x86/boot.rst` | 完整的启动协议规范 |
| 实模式入口 | `arch/x86/boot/header.S` | CPU 第一条指令 |
| 保护模式 | `arch/x86/boot/pm.c` | CR0 设置、GDT 加载 |
| 页表建立 | `arch/x86/boot/compressed/head_64.S` | 长模式页表 |
| 解压缩 | `arch/x86/boot/compressed/misc.c` | 多格式解压缩 |
| C 入口 | `arch/x86/kernel/head64.c` | C 代码的起点 |
| 核心初始化 | `init/main.c` | start_kernel() 函数 |
| 内存管理 | `mm/page_alloc.c` | buddy 分配器 |

---

## 💡 学习建议

1. **按顺序学习**：不要跳过任何部分，每个部分都是后续的基础

2. **边读边想**：在阅读代码时，思考"为什么这样做"而不仅仅是"做了什么"

3. **动手编译**：理论学习后必须进行实际编译，这样才能真正理解

4. **使用符号**：在讨论和笔记中使用统一的术语和缩写：
   - GDT：全局描述符表
   - IDT：中断描述符表
   - CR0、CR4：控制寄存器
   - EFER：扩展功能使能寄存器

5. **记录问题**：遇到不理解的地方，记下来然后深入研究源代码

---

## 🔗 相关资源

- **Linux 启动协议**：`Documentation/x86/boot.rst`
- **内核源代码在线浏览**：https://elixir.bootlin.com/linux/v7.0.0/source/
- **官方内核文档**：https://www.kernel.org/doc/html/latest/
- **LWN.net 文章**：https://lwn.net/（关于内核最新动态）

---

## ❓ 常见问题

**Q: 我需要理解所有汇编指令吗？**
A: 不需要。重要的是理解逻辑流程和每个阶段的目的。每个指令旁都有详细注释。

**Q: vmlinuz 是如何压缩的？**
A: vmlinuz 通常使用 gzip、xz 或 bzip2 压缩。内核在启动时自动解压缩。

**Q: 为什么需要分段转换（实→保→长）？**
A: 这是 x86 架构的历史遗留。需要逐步转换来维持向后兼容性。

**Q: 我可以跳过长模式直接在保护模式下运行吗？**
A: 现代 Linux 7.0.0 要求 64 位，所以必须进入长模式。

---

**下一步**：准备好了吗？[开始阅读第一部分](vmlinuz-assembly-code-part1.md)！
