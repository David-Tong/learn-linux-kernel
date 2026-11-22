# Linux Kernel 7.0.0 编译和运行指南

## 概述

本指南详细讲解如何编译 Linux Kernel 7.0.0 并在虚拟机中运行测试。这是学习内核最实践的方式。

## 1. 系统要求和工具安装

### 1.1 硬件要求

- **CPU**：至少双核处理器（建议四核以上）
- **内存**：8GB 以上（编译时需要大量内存）
- **磁盘空间**：
  - 源代码：约 1.5GB
  - 编译输出：约 10GB
  - QEMU 虚拟磁盘：10-20GB
  - 总计：至少 30GB 可用空间

### 1.2 Ubuntu/Debian 系统安装工具

```bash
# 更新包管理器
sudo apt update
sudo apt upgrade -y

# 安装编译依赖
sudo apt install -y \
    build-essential \
    kernel-package \
    libncurses-dev \
    libssl-dev \
    libelf-dev \
    bison \
    flex \
    bc \
    libunwind-dev \
    libelf-dev \
    libdw-dev \
    binutils-dev \
    libiberty-dev \
    libzstd-dev

# 安装虚拟化工具（可选但推荐）
sudo apt install -y qemu qemu-system-x86 qemu-system-gui

# 安装版本控制工具
sudo apt install -y git
```

**源文件**：`kernel-7.0.0/Documentation/process/changes.rst` - 详细的系统要求

### 1.3 验证编译环境

```bash
# 验证编译工具版本
gcc --version          # 应该是 7.0 或更高
make --version         # 应该是 3.82 或更高
ld --version          # binutils
yacc --version        # bison（yacc 的 GNU 版本）
flex --version

# 验证内核编译依赖
grep -E "CONFIG_" /boot/config-$(uname -r) | head -5
```

## 2. 获取 Linux Kernel 7.0.0 源代码

### 2.1 从 kernel.org 克隆（推荐）

这是最推荐的方式，可以获得完整的 Git 历史：

```bash
# 克隆稳定版本仓库（包含所有版本）
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux

# 检查出 v7.0.0 版本
git checkout v7.0.0

# 验证版本
git log --oneline -1
# 输出应该是: v7.0.0 commit 信息
```

**优点**：
- 获得完整 Git 历史，可以查看任何文件的变化
- 可以轻松切换到其他版本
- 体积较大（约 2GB）但只需下载一次

### 2.2 从 kernel.org 下载压缩包

如果网络不稳定，可以下载压缩包：

```bash
# 下载 Linux 7.0.0 源代码（xz 压缩，约 150MB）
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.0.tar.xz

# 或使用 curl
curl -O https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.0.tar.xz

# 验证完整性（可选但推荐）
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.0.tar.xz.sha256
sha256sum -c linux-7.0.0.tar.xz.sha256

# 解压缩
tar -xf linux-7.0.0.tar.xz
cd linux-7.0.0

# 验证版本
head -3 Makefile
# 输出应该包含 VERSION = 7
```

**优点**：
- 文件体积小（约 150MB）
- 下载速度快
- 适合网络不稳定的环境

### 2.3 从 GitHub 克隆（备用）

```bash
# 克隆 Linus Torvalds 的主仓库
git clone https://github.com/torvalds/linux.git
cd linux

# 检查出 v7.0.0
git checkout v7.0.0

# 或克隆稳定版本仓库（推荐）
git clone https://github.com/gregkh/linux.git
cd linux
git checkout linux-7.0.0
```

## 3. 内核配置

### 3.1 配置选项

配置定义了哪些功能应该编译进内核。有多种方法可以创建配置文件：

**方法 1：使用现有系统的配置**

如果想快速开始，可以基于当前系统的配置：

```bash
# 复制当前系统配置
cp /boot/config-$(uname -r) .config

# 更新配置以适应新内核版本
make oldconfig
# 对于新选项，直接按 Enter 接受默认值
```

**方法 2：交互式配置菜单**

图形化配置工具，易于定制：

```bash
# 基于菜单的配置（推荐）
make menuconfig
# 或使用 Qt 图形界面
make xconfig
# 或使用 GTK+ 图形界面
make gconfig
```

**源文件**：`kernel-7.0.0/scripts/kconfig/Makefile` - 配置工具

**方法 3：最小配置（用于测试）**

创建最小配置文件，只包含必需的功能：

```bash
# 生成最小配置
make tinyconfig
# 这会生成一个非常小的内核配置

# 或从零开始，创建基本配置
cat > .config << 'EOF'
CONFIG_64BIT=y
CONFIG_X86=y
CONFIG_PRINTK=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_CONSOLE_LOGLEVEL_DEFAULT=7
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
EOF

# 生成依赖的配置选项
make olddefconfig
```

### 3.2 关键配置选项详解

**源文件**：`kernel-7.0.0/arch/x86/Kconfig:1-100` - x86 特定配置

编译内核时的重要配置选项：

```
# 编译类型
CONFIG_64BIT=y                    # 编译 64 位内核
CONFIG_X86=y                      # x86 架构
CONFIG_X86_64=y                   # x86-64 架构

# 处理器特性
CONFIG_HAVE_CPUID_INSTRUCTION=y   # 支持 CPUID 指令
CONFIG_HAVE_CMPXCHG_LOCAL=y       # 支持本地原子操作
CONFIG_X86_USE_3DNOW=y            # 支持 3DNow 指令集（AMD）
CONFIG_X86_USE_CMOV=y             # 支持条件移动指令

# 内存管理
CONFIG_HAVE_MEMBLOCK=y            # 使用 memblock 内存分配器
CONFIG_HAVE_PAGE_SIZE_4K=y        # 4KB 页大小
CONFIG_ZONE_DMA=y                 # 支持 DMA 区域
CONFIG_ZONE_NORMAL=y              # 支持普通区域

# 文件系统
CONFIG_EXT4_FS=y                  # EXT4 文件系统（内置）
CONFIG_DEVTMPFS=y                 # 动态 /dev 文件系统
CONFIG_DEVTMPFS_MOUNT=y           # 自动挂载 devtmpfs

# 控制台和日志
CONFIG_PRINTK=y                   # 内核日志打印
CONFIG_CONSOLE_LOGLEVEL_DEFAULT=7 # 控制台日志级别（0-8）
CONFIG_DEBUG_INFO=y               # 调试信息
CONFIG_DEBUG_KERNEL=y             # 内核调试

# 中断和定时器
CONFIG_HAVE_IRQ_TIME_ACCOUNTING=y # IRQ 时间统计
CONFIG_GENERIC_CLOCKEVENTS=y      # 通用时钟事件

# 安全和保护
CONFIG_HAVE_FUNCTION_TRACER=y     # 函数追踪支持
CONFIG_STACK_VALIDATION=y         # 栈验证
```

### 3.3 验证配置

```bash
# 检查配置是否有效
make help | grep -E "config|menuconfig"

# 显示当前配置
cat .config | grep "^CONFIG" | wc -l
# 输出编译进内核的配置选项数量

# 查看特定配置选项
grep CONFIG_EXT4_FS .config
grep CONFIG_DEBUG_KERNEL .config
```

## 4. 编译内核

### 4.1 完整编译过程

**源文件**：`kernel-7.0.0/Makefile:1-100` - 顶级 Makefile

```bash
# 进入内核源代码目录
cd linux-7.0.0

# （可选）清除之前的编译结果
make clean

# 验证配置文件
make prepare

# 编译内核（这会耗费大量时间，通常 15-60 分钟）
# -j N：使用 N 个并行编译进程（通常设置为 CPU 核心数）
make -j$(nproc)

# 编译完成后，生成的内核文件为：
# - arch/x86/boot/bzImage：可启动的内核镜像
# - vmlinux：未压缩的 ELF 格式内核（用于调试）
# - System.map：符号映射表

# 查看编译后的文件
ls -lh arch/x86/boot/bzImage
ls -lh vmlinux
file arch/x86/boot/bzImage
file vmlinux
```

### 4.2 编译内核模块

如果配置中启用了模块（CONFIG_MODULES=y），需要编译和安装模块：

```bash
# 编译模块
make modules

# 或与内核一起编译（推荐）
make -j$(nproc) all modules

# 安装模块到系统（需要 root 权限）
sudo make modules_install
# 这会将模块安装到 /lib/modules/7.0.0/ 目录

# 查看已安装的模块
ls /lib/modules/7.0.0/
```

### 4.3 编译过程详解

编译过程包含多个阶段：

**源文件**：`kernel-7.0.0/scripts/Makefile.build:1-50` - 编译规则

```makefile
# 1. 生成版本头文件
include/generated/compile.h    # 包含编译时间、编译器版本等

# 2. 编译内核代码
# 对每个 .c 文件进行预处理、编译、汇编

gcc -Iinclude -Iarch/x86/include -c -o init/main.o init/main.c

# 3. 链接目标文件
# 将所有 .o 文件链接成一个完整的内核

ld -r -o vmlinux.o arch/x86/entry/entry_64.o arch/x86/kernel/setup.o ...

# 4. 生成 vmlinux（未压缩内核）
ld -T arch/x86/kernel/vmlinux.lds.S vmlinux.o -o vmlinux

# 5. 生成 System.map（符号表）
nm vmlinux > System.map

# 6. 生成 bzImage（可启动的压缩内核）
arch/x86/boot/tools/build \
    arch/x86/boot/setup.o \
    arch/x86/boot/vmlinux.bin \
    -o arch/x86/boot/bzImage
```

### 4.4 解决常见编译错误

**错误：缺少开发工具**

```bash
# 解决方法
sudo apt install -y build-essential
```

**错误：缺少头文件**

```bash
# 解决方法
sudo apt install -y libncurses-dev libssl-dev libelf-dev
```

**错误：编译过程中内存不足**

```bash
# 减少并行编译进程数
make -j2

# 或清除之前的编译并重新开始
make mrproper
make menuconfig
make -j2
```

## 5. 安装内核

### 5.1 安装到系统（需要 root）

```bash
# 安装内核和模块到 /boot
sudo make install

# 这会：
# 1. 复制 bzImage 到 /boot/vmlinuz-7.0.0
# 2. 复制 System.map 到 /boot/System.map-7.0.0
# 3. 复制 .config 到 /boot/config-7.0.0
# 4. 更新 GRUB 引导器

# 验证安装
ls -l /boot/ | grep 7.0.0

# 重新启动系统测试新内核
sudo reboot
```

### 5.2 使用 GRUB 启动新内核

```bash
# 查看 GRUB 菜单
grep -E "^menuentry" /boot/grub/grub.cfg

# GRUB 会自动检测新内核并添加菜单项
# 重启后在 GRUB 菜单选择新内核

# 查看当前运行的内核
uname -r

# 查看已安装的内核
ls /boot/vmlinuz-*
```

## 6. 使用 QEMU 进行内核测试

QEMU 是开源虚拟机软件，可以在不影响系统的情况下测试新内核。

### 6.1 创建虚拟磁盘镜像

```bash
# 创建 10GB 虚拟磁盘
qemu-img create -f qcow2 rootfs.qcow2 10G

# 格式化为 ext4
# 注意：这需要 root 权限或特殊的 qemu 工具
# 更简单的方法是使用预制的 rootfs 镜像

# 下载预制的 rootfs 镜像（包含基础系统文件）
wget https://buildroot.uclibc.org/downloads/binaries/2023.05-rc1/qemu_x86_64/rootfs.ext2

# 或使用 Buildroot 自己生成 rootfs
# （需要额外的编译过程）
```

### 6.2 配置引导参数

编译内核时，需要为 QEMU 准备内核和 initrd：

```bash
# 1. 复制编译好的 bzImage
cp arch/x86/boot/bzImage ../qemu-bzimage

# 2. 创建简单的 initramfs（初始内存文件系统）
# initramfs 包含启动所需的最小程序

mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev}

# 创建最小的 init 脚本
cat > initramfs/init << 'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t devtmpfs dev /dev
mount -t sysfs sys /sys

# 现在内核已经完全启动
# 执行一个 shell
/bin/sh
EOF

chmod +x initramfs/init

# 生成 initramfs 镜像
cd initramfs
find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz
cd ..
```

### 6.3 在 QEMU 中启动内核

```bash
# 最简单的启动方式
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd ../initramfs.cpio.gz \
    -nographic \
    -append "console=ttyS0 root=/dev/ram"

# 更完整的示例（使用虚拟磁盘）
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -hda rootfs.qcow2 \
    -nographic \
    -append "console=ttyS0 root=/dev/sda1" \
    -m 2G \
    -smp 4

# 命令行参数说明：
# -kernel：指定内核镜像
# -initrd：指定初始 ramdisk
# -hda：硬盘 A
# -nographic：不使用图形界面（通过串口输出）
# -append：传递给内核的引导参数
# -m：分配给虚拟机的内存
# -smp：CPU 核心数
```

### 6.4 在 QEMU 中进行调试

**使用 GDB 进行内核调试**：

```bash
# 1. 编译内核时启用调试信息
# 确保 .config 中有：
# CONFIG_DEBUG_INFO=y
# CONFIG_DEBUG_KERNEL=y
# CONFIG_GDB_SCRIPTS=y

# 2. 启动 QEMU 时启用 GDB 服务器
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd ../initramfs.cpio.gz \
    -nographic \
    -append "console=ttyS0" \
    -s -S  # -s：启用 GDB 服务器（默认监听 1234 端口）
           # -S：启动时暂停

# 3. 在另一个终端启动 GDB
gdb vmlinux

# 4. 在 GDB 中连接到 QEMU
(gdb) target remote localhost:1234

# 5. 设置断点并调试
(gdb) break start_kernel
(gdb) continue
(gdb) next
(gdb) step
(gdb) print variable_name
```

## 7. 交叉编译

用于编译不同架构的内核（如 ARM、ARM64）。

### 7.1 安装交叉编译工具链

```bash
# 对于 ARM 32 位
sudo apt install -y gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

# 对于 ARM 64 位（ARMv8）
sudo apt install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# 对于 RISC-V
sudo apt install -y gcc-riscv64-linux-gnu g++-riscv64-linux-gnu

# 验证安装
arm-linux-gnueabihf-gcc --version
aarch64-linux-gnu-gcc --version
```

### 7.2 编译 ARM 内核

```bash
cd linux-7.0.0

# 配置 ARM 内核
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

# 编译
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)

# 编译结果
# - arch/arm/boot/zImage：压缩内核
# - arch/arm/boot/dts/*.dtb：设备树文件
```

### 7.3 编译 ARM64 内核

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# 在 QEMU 中运行 ARM64 内核
qemu-system-aarch64 \
    -kernel arch/arm64/boot/Image \
    -nographic \
    -append "console=ttyAMA0" \
    -m 2G
```

## 8. 内核调试和分析

### 8.1 使用 printk 进行日志输出

**源文件**：`kernel-7.0.0/include/linux/printk.h:1-100`

```c
// 在内核代码中添加调试信息

printk(KERN_EMERG   "emergency message");   // 级别 0 - 紧急
printk(KERN_ALERT   "alert message");       // 级别 1 - 警报
printk(KERN_CRIT    "critical message");    // 级别 2 - 临界
printk(KERN_ERR     "error message");       // 级别 3 - 错误
printk(KERN_WARNING "warning message");     // 级别 4 - 警告
printk(KERN_NOTICE  "notice message");      // 级别 5 - 通知
printk(KERN_INFO    "information message"); // 级别 6 - 信息
printk(KERN_DEBUG   "debug message");       // 级别 7 - 调试

// pr_* 宏（更简洁的写法）
pr_emerg("emergency");
pr_err("error: %d", err_code);
pr_info("info: %s", message);
pr_debug("debug: 0x%x", value);
```

**查看内核日志**：

```bash
# 查看实时内核日志
tail -f /var/log/kern.log

# 或使用 dmesg
dmesg

# 显示最后 50 行
dmesg | tail -50

# 只显示错误和警告
dmesg | grep -E "ERROR|WARNING"

# 搜索特定内容
dmesg | grep "cpu"
```

### 8.2 使用 ftrace 进行函数追踪

**源文件**：`kernel-7.0.0/kernel/trace/ftrace.c:1-100`

ftrace 是内核函数追踪工具：

```bash
# 启用 ftrace
sudo bash
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 追踪所有函数
echo function > /sys/kernel/debug/tracing/current_tracer

# 只追踪特定函数（如 sys_read）
echo sys_read > /sys/kernel/debug/tracing/set_ftrace_filter

# 查看追踪结果
cat /sys/kernel/debug/tracing/trace | head -50

# 清除追踪
echo > /sys/kernel/debug/tracing/trace
```

### 8.3 性能分析（perf）

```bash
# 安装 perf
sudo apt install -y linux-tools-generic

# 记录 CPU 使用情况
perf record -F 99 -a -g sleep 10

# 查看报告
perf report

# 创建性能火焰图
perf script > out.perf
# 然后使用 FlameGraph 工具处理
```

## 9. 常见问题排查

### 9.1 编译错误

| 错误信息 | 原因 | 解决方案 |
|--------|------|--------|
| `make: command not found` | 未安装 make | `sudo apt install build-essential` |
| `gcc: command not found` | 未安装 gcc | `sudo apt install build-essential` |
| `No such file or directory` | 文件路径错误 | 检查源代码目录 |
| 内存不足 | 编译时内存溢出 | 减少并行数：`make -j2` |

### 9.2 启动错误

| 错误 | 原因 | 解决方案 |
|------|------|--------|
| `Kernel panic` | 内核崩溃 | 检查编译配置，查看 dmesg 日志 |
| `Unable to mount root fs` | 找不到根文件系统 | 检查引导参数和文件系统配置 |
| `No space left on device` | 磁盘满 | 清理编译文件：`make clean` |

## 10. 完整示例工作流

```bash
#!/bin/bash
# 完整的编译和测试工作流

set -e

# 1. 获取源代码
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout v7.0.0

# 2. 配置内核
cp /boot/config-$(uname -r) .config
make oldconfig

# 3. 编译
make -j$(nproc) all

# 4. 创建 QEMU 测试环境
mkdir -p qemu_test
cd qemu_test

# 5. 启动 QEMU
qemu-system-x86_64 \
    -kernel ../arch/x86/boot/bzImage \
    -nographic \
    -append "console=ttyS0" \
    -m 2G

# 6. 在 QEMU 中测试
# （按 Ctrl-A，然后按 X 退出 QEMU）
```

## 总结

编译和运行 Linux 内核的完整流程：

1. **安装工具**：编译器、调试工具、虚拟化工具
2. **获取源代码**：从 kernel.org 或 GitHub
3. **配置内核**：选择需要编译的功能模块
4. **编译**：生成 bzImage 和模块
5. **测试**：在 QEMU 中进行测试
6. **调试**：使用 GDB、ftrace、perf 等工具分析

通过这个过程，你可以深入理解内核的启动过程、内存管理、I/O 操作等核心概念。
