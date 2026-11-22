# 编译和运行内核（kernel-build）

本目录提供了完整的 Linux Kernel 7.0.0 编译和测试指南。从工具安装到在 QEMU 中运行，以及如何使用 GDB 进行调试。

## 📚 文章列表

### [building-and-running-kernel.md](building-and-running-kernel.md) - 完整的内核编译指南

**学习内容**：
- 🔹 系统要求和编译环境配置
- 🔹 获取 Linux 7.0.0 源代码的多种方式
- 🔹 内核配置选项详解
- 🔹 完整编译过程和优化
- 🔹 模块编译和安装
- 🔹 安装内核和更新引导加载程序
- 🔹 使用 QEMU 进行虚拟测试
- 🔹 交叉编译（ARM/ARM64）
- 🔹 GDB/KGDB 内核调试
- 🔹 常见问题排查

**关键代码位置**：
- `Makefile` - 顶级构建文件（第 1-100 行）
- `scripts/Makefile.*` - 构建规则
- `arch/x86/Kconfig` - x86 配置选项
- `Documentation/process/changes.rst` - 系统要求
- `Documentation/admin-guide/README.rst` - 编译指南

**预计时间**：
- 学习时间：2-3 小时
- 实际编译：15-60 分钟（取决于硬件）
- 完整实践：3-5 小时

**难度**：⭐⭐（实践性强但相对简单）

**先决条件**：
- Linux 系统基本操作知识
- 命令行工具使用
- 理解文件系统和磁盘概念

---

## 🎯 快速开始（5 分钟）

```bash
# 1. 获取源代码
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.0.tar.xz
tar -xf linux-7.0.0.tar.xz
cd linux-7.0.0

# 2. 配置内核
cp /boot/config-$(uname -r) .config
make oldconfig

# 3. 编译
make -j$(nproc)

# 4. 在 QEMU 中测试
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -nographic \
    -append "console=ttyS0"
```

---

## 🛠️ 编译工具链详解

### 必需工具

| 工具 | 用途 | Linux 命令 |
|------|------|-----------|
| GCC | C 编译器 | `apt install build-essential` |
| Make | 构建工具 | 包含在 `build-essential` |
| Binutils | 链接器、汇编器 | 包含在 `build-essential` |
| Bison | 语法分析 | `apt install bison` |
| Flex | 词法分析 | `apt install flex` |
| ncurses-dev | 菜单工具 | `apt install libncurses-dev` |
| OpenSSL | 密码库 | `apt install libssl-dev` |
| libelf | ELF 解析库 | `apt install libelf-dev` |
| bc | 计算器（内核使用） | `apt install bc` |

### 验证工具版本

```bash
#!/bin/bash
# 验证编译环境脚本

echo "=== 编译工具检查 ==="

echo "GCC 版本:"
gcc --version | head -1

echo "Make 版本:"
make --version | head -1

echo "Binutils 版本:"
ld --version | head -1

echo "Bison 版本:"
bison --version

echo "Flex 版本:"
flex --version

echo "Perl 版本:"
perl --version | head -2

echo "OpenSSL 版本:"
openssl version

echo "=== 完成 ==="
```

---

## 🔧 编译配置深入讲解

### 配置方法对比

| 方法 | 命令 | 适用场景 | 配置文件 |
|------|------|---------|---------|
| **menuconfig** | `make menuconfig` | 交互式定制 | 生成 .config |
| **xconfig** | `make xconfig` | Qt 图形界面 | 生成 .config |
| **gconfig** | `make gconfig` | GTK 图形界面 | 生成 .config |
| **oldconfig** | `make oldconfig` | 更新现有配置 | 更新 .config |
| **olddefconfig** | `make olddefconfig` | 自动更新 | 自动处理 |
| **defconfig** | `make defconfig` | 默认配置 | 生成 .config |
| **tinyconfig** | `make tinyconfig` | 最小配置 | 生成 .config |

### 关键配置选项

```bash
# CONFIG_* 选项的含义：
# =y     ： 编译进内核
# =m     ： 编译为可加载模块
# 未定义 ： 不编译

# 重要选项示例
CONFIG_64BIT=y                      # 64 位内核
CONFIG_X86=y                        # x86 架构
CONFIG_MODULES=y                    # 支持可加载模块
CONFIG_DEBUG_KERNEL=y               # 内核调试
CONFIG_DEBUG_INFO=y                 # 调试符号
CONFIG_KGDB=y                       # KGDB 调试支持
CONFIG_GDB_SCRIPTS=y                # GDB 脚本支持
CONFIG_HAVE_FUNCTION_TRACER=y       # 函数追踪
CONFIG_FTRACE=y                     # 函数追踪框架
```

---

## 📊 编译性能优化

### 并行编译

```bash
# 使用 CPU 核心数
make -j$(nproc)

# 或手动指定
make -j4

# 对于大型编译，可以调整：
# 核心数 + 1（通常最优）
make -j5
```

### 增量编译

```bash
# 只重新编译改变的文件
make -j4

# 完全清除后重新编译
make mrproper      # 删除所有编译文件和 .config
make clean         # 只删除编译文件，保留 .config
```

### 编译时间估算

| 硬件配置 | 编译时间 | 配置 |
|---------|--------|------|
| 2 核 2GB RAM | 60-90 分钟 | 完整配置 |
| 4 核 8GB RAM | 30-45 分钟 | 完整配置 |
| 8 核 16GB RAM | 15-25 分钟 | 完整配置 |
| 8 核 16GB RAM | 5-10 分钟 | 最小配置 |

---

## 🧪 在 QEMU 中进行测试

### 安装 QEMU

```bash
# Ubuntu/Debian
sudo apt install -y qemu qemu-system-x86 qemu-system-gui

# 验证安装
qemu-system-x86_64 --version
```

### QEMU 命令行参数详解

```bash
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \     # 内核镜像
    -initrd initramfs.cpio.gz \         # 初始 ramdisk
    -hda rootfs.qcow2 \                 # 硬盘镜像
    -nographic \                        # 无图形界面
    -append "console=ttyS0" \           # 内核参数
    -m 2G \                             # 内存大小
    -smp 4 \                            # CPU 核心数
    -net none \                         # 禁用网络
    -s -S                               # GDB 调试端口
```

### 创建最小 initramfs

```bash
# 创建目录结构
mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev}

# 创建最小 init 脚本
cat > initramfs/init << 'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t devtmpfs dev /dev
mount -t sysfs sys /sys
echo "Kernel boot complete!"
exec /bin/sh
EOF

chmod +x initramfs/init

# 生成 initramfs 镜像
cd initramfs
find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz
cd ..
```

---

## 🐛 调试方法

### 方法 1：使用 printk() 添加日志

```c
// 在源代码中添加
printk(KERN_INFO "Debug: variable_x = %d\n", x);

// KERN_* 日志级别：
// KERN_EMERG   - 紧急
// KERN_ALERT   - 警报
// KERN_CRIT    - 临界
// KERN_ERR     - 错误
// KERN_WARNING - 警告
// KERN_NOTICE  - 通知
// KERN_INFO    - 信息
// KERN_DEBUG   - 调试
```

### 方法 2：使用 GDB 进行单步调试

```bash
# 终端 1：启动 QEMU
qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd initramfs.cpio.gz \
    -s -S \                     # -s：启用 GDB 服务器
    -nographic                  #  -S：启动时暂停

# 终端 2：启动 GDB
gdb vmlinux

# GDB 命令
(gdb) target remote localhost:1234     # 连接 QEMU
(gdb) break start_kernel               # 设置断点
(gdb) continue                         # 继续执行
(gdb) next                             # 单步（不进入函数）
(gdb) step                             # 单步（进入函数）
(gdb) print x                          # 打印变量
(gdb) backtrace                        # 显示调用栈
(gdb) quit                             # 退出 GDB
```

### 方法 3：使用 ftrace 进行函数追踪

```bash
# 在运行的系统中追踪函数
sudo bash
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer
echo start_kernel > /sys/kernel/debug/tracing/set_ftrace_filter

# 查看追踪结果
cat /sys/kernel/debug/tracing/trace | head -50

# 清除追踪
echo > /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

---

## 🌍 交叉编译

### 编译 ARM 内核

```bash
# 安装 ARM 工具链
sudo apt install -y gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

# 配置
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

# 编译
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j$(nproc)

# 结果：arch/arm/boot/zImage
```

### 编译 ARM64 内核

```bash
# 安装 ARM64 工具链
sudo apt install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# 配置
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig

# 编译
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# 结果：arch/arm64/boot/Image
```

---

## 📌 Makefile 目标速查

| 目标 | 功能 |
|------|------|
| `make help` | 显示所有可用目标 |
| `make menuconfig` | 交互式配置菜单 |
| `make oldconfig` | 更新配置 |
| `make all` | 编译内核和模块 |
| `make modules` | 只编译模块 |
| `make modules_install` | 安装模块 |
| `make install` | 安装内核 |
| `make clean` | 删除编译文件 |
| `make mrproper` | 彻底清除 |
| `make distclean` | 分发级别清除 |
| `make tags` | 生成标签文件（用于编辑器） |
| `make cscope` | 生成 cscope 数据库 |

---

## 🔗 相关资源

- **内核编译指南**：`Documentation/admin-guide/README.rst`
- **系统要求**：`Documentation/process/changes.rst`
- **内核配置**：`Documentation/kbuild/kconfig-language.rst`
- **GDB 调试指南**：`Documentation/dev-tools/gdb.rst`
- **KGDB 文档**：`Documentation/dev-tools/kgdb.rst`

---

## ❓ 常见问题

**Q: 编译失败，提示"command not found"？**
A: 安装缺失的工具。运行检查脚本确认所有依赖都已安装。

**Q: 编译过程中内存不足？**
A: 减少并行编译数：`make -j2` 或 `make -j1`

**Q: 编译成功但 QEMU 无法启动？**
A: 检查内核配置。尝试使用现有系统的配置：`cp /boot/config-$(uname -r) .config`

**Q: GDB 连接 QEMU 失败？**
A: 确保 QEMU 以 `-s` 参数启动，并等待足够时间直到完全启动。

**Q: 如何加快编译速度？**
A: 增加并行进程数（`-j8` 或 `-j16`）、关闭调试信息（`CONFIG_DEBUG_INFO=n`）或使用 ccache。

**Q: 交叉编译时找不到 libssl？**
A: 安装交叉编译版本的 libssl：`apt install libssl-dev:arm64` 等。

---

**下一步**：准备好编译你的第一个内核了吗？[开始阅读文章](building-and-running-kernel.md)！
