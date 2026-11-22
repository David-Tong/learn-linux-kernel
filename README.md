# learn-linux-kernel

一份全面的 Linux Kernel 7.0.0 学习指南，涵盖内核启动、文件系统和构建等核心主题。

## 目录结构

### 1. kernel-boot - 内核启动过程
深入学习 Linux 内核如何启动和初始化。

#### 汇编代码部分
- **vmlinuz-assembly-code-part1.md** - Linux 启动协议和实模式代码
  - Linux 启动协议规范（`Documentation/x86/boot.rst`）
  - 启动参数结构（`arch/x86/include/uapi/asm/bootparam.h`）
  - 实模式代码入口（`arch/x86/boot/header.S`）
  - 设置代码（`arch/x86/boot/main.c`、`arch/x86/boot/pm.c`）
  - A20 地址线启用（`arch/x86/boot/a20.c`）
  - 全局描述符表设置（`arch/x86/boot/pm.c`）

- **vmlinuz-assembly-code-part2.md** - 保护模式和长模式转换
  - 进入保护模式（`arch/x86/boot/pm.c`）
  - 32 位保护模式代码（`arch/x86/boot/compressed/head_32.S`）
  - 长模式转换详解（`arch/x86/boot/compressed/head_64.S`）
  - 页表结构和设置（`arch/x86/boot/compressed/head_64.S`）
  - 解压缩阶段（`arch/x86/boot/compressed/misc.c`）
  - CPU 功能检查（`arch/x86/boot/cpucheck.c`）

#### C 代码部分
- **vmlinuz-c-code-part1.md** - 解压缩和内核初始化
  - 解压缩阶段（`arch/x86/boot/compressed/misc.c`）
  - 支持的压缩格式（gzip、bzip2、xz、lzo、zstd）
  - 进入 C 代码的准备（`arch/x86/kernel/head64.c`）
  - E820 内存表解析（`arch/x86/include/uapi/asm/bootparam.h`）
  - start_kernel() 函数详解（`init/main.c`）
  - 页分配器和 slab 分配器初始化（`mm/page_alloc.c`）

### 2. filesystem - 文件系统和 I/O 操作
理解文件系统工作原理和系统调用机制。

- **read-syscall-kernel-internals.md** - read() 系统调用内核实现
  - read() POSIX 接口（`include/uapi/unistd.h`）
  - 系统调用入口点（`arch/x86/entry/entry_64.S`）
  - sys_read() 处理器（`fs/read_write.c`）
  - 文件描述符和文件结构（`include/linux/fs.h`）
  - VFS 读取操作（`fs/read_write.c`）
  - 页缓存管理（`mm/filemap.c`）
  - 块 I/O 层（`block/blk-core.c`）
  - 设备驱动和中断处理（`drivers/ata/`、`drivers/scsi/`）
  - I/O 完成流程（`mm/filemap.c`）

### 3. kernel-build - 编译和运行内核
实践指南，学习如何编译和测试自定义内核。

- **building-and-running-kernel.md** - 完整的内核编译指南
  - 系统要求和工具安装（`scripts/`）
  - 获取内核源代码（kernel.org、GitHub）
  - 内核配置方法（`Kconfig`、`scripts/kconfig/`）
  - 完整编译过程（`Makefile`、`scripts/Makefile.*`）
  - 安装内核和模块（`scripts/`）
  - 使用 QEMU 测试（文档和示例）
  - 交叉编译（ARM、ARM64）
  - 内核调试（GDB、KGDB）
  - 构建系统参考（`scripts/`）

## 学习路径建议

### 初级
1. 从 **kernel-boot/vmlinuz-assembly-code-part1.md** 开始，理解基本的启动流程
2. 然后阅读 **kernel-boot/vmlinuz-assembly-code-part2.md**，深入理解保护模式转换
3. 参考 **kernel-build/building-and-running-kernel.md** 编译第一个自定义内核

### 中级
4. 学习 **filesystem/read-syscall-kernel-internals.md** 理解系统调用机制
5. 在 QEMU 中测试和调试内核

### 高级
6. 修改内核代码进行实验
7. 使用 GDB/KGDB 进行内核调试
8. 编写自定义设备驱动或文件系统

## 核心主题覆盖

| 主题 | 文件 | 关键代码位置 |
|------|------|-----------|
| 启动协议 | kernel-boot/vmlinuz-assembly-code-part1.md | `Documentation/x86/boot.rst` |
| 实模式代码 | kernel-boot/vmlinuz-assembly-code-part1.md | `arch/x86/boot/header.S` |
| 保护模式 | kernel-boot/vmlinuz-assembly-code-part2.md | `arch/x86/boot/pm.c` |
| 长模式转换 | kernel-boot/vmlinuz-assembly-code-part2.md | `arch/x86/boot/compressed/head_64.S` |
| 页表设置 | kernel-boot/vmlinuz-assembly-code-part2.md | `arch/x86/boot/compressed/head_64.S` |
| 解压缩阶段 | kernel-boot/vmlinuz-c-code-part1.md | `arch/x86/boot/compressed/misc.c` |
| 系统调用 | filesystem/read-syscall-kernel-internals.md | `arch/x86/entry/entry_64.S` |
| VFS 层 | filesystem/read-syscall-kernel-internals.md | `fs/read_write.c` |
| 页缓存 | filesystem/read-syscall-kernel-internals.md | `mm/filemap.c` |
| 块 I/O | filesystem/read-syscall-kernel-internals.md | `block/blk-core.c` |
| 编译系统 | kernel-build/building-and-running-kernel.md | `Makefile`、`scripts/` |

## Linux Kernel 7.0.0 源代码获取

### 方法 1：从 kernel.org 克隆（推荐）
```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout v7.0.0
```

### 方法 2：下载压缩包
```bash
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.0.tar.xz
tar -xf linux-7.0.0.tar.xz
cd linux-7.0.0
```

### 方法 3：从 GitHub 克隆（备用）
```bash
git clone https://github.com/torvalds/linux.git
cd linux
git checkout v7.0.0
```

## 源代码文件导航

### 启动相关文件
```
arch/x86/boot/
├── header.S              # vmlinuz 头部和实模式代码
├── main.c               # 设置代码主程序
├── pm.c                 # 保护模式转换
├── a20.c                # A20 地址线启用
├── cpucheck.c           # CPU 功能检查
├── efi.c                # EFI 支持
├── acpi.c               # ACPI 支持
└── compressed/
    ├── head_32.S        # 32 位保护模式代码
    ├── head_64.S        # 64 位长模式转换和页表
    ├── misc.c           # 解压缩主程序
    └── ...
```

### 系统调用相关文件
```
arch/x86/entry/
├── entry_64.S           # 系统调用入口
├── syscall_64.c         # syscall 表
└── common.c             # 通用 syscall 处理

fs/
├── read_write.c         # read()、write() 实现
├── open.c               # open() 实现
├── close.c              # close() 实现
└── ...

include/linux/
├── fs.h                 # 文件系统结构
└── fdtable.h            # 文件描述符表

mm/
└── filemap.c            # 页缓存实现
```

### 块 I/O 相关文件
```
block/
├── blk-core.c           # 块 I/O 核心
├── blk-mq.c             # 多队列 I/O
└── ...

drivers/
├── ata/                 # ATA/SATA 驱动
├── scsi/                # SCSI 驱动
└── nvme/                # NVMe 驱动
```

### 编译相关文件
```
./
├── Makefile             # 主 Makefile
├── Kconfig              # 配置顶级文件
└── scripts/
    ├── Makefile.*       # 子系统 Makefile
    ├── kconfig/         # 配置工具
    └── ...
```

## 有用的命令

### 查找源代码
```bash
# 在内核源代码树中搜索
grep -r "function_name" arch/x86/boot/

# 使用 find 定位文件
find . -name "*.c" -path "*/fs/*" | head

# 查看文件行数
wc -l arch/x86/boot/compressed/head_64.S
```

### 查看特定行数的代码
```bash
# 查看 header.S 的 1-100 行
sed -n '1,100p' arch/x86/boot/header.S

# 显示包含关键词的行及行号
grep -n "startup_64" arch/x86/boot/compressed/head_64.S
```

## 参考资源

- **官方内核文档**：https://www.kernel.org/doc/html/latest/
- **Linux 启动协议**：`Documentation/x86/boot.rst`
- **内核编译指南**：`Documentation/admin-guide/README.rst`
- **内核源代码**：https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/
- **QEMU 文档**：https://www.qemu.org/documentation/
- **内核调试指南**：`Documentation/dev-tools/kgdb.rst`

## 阅读源代码快速入门

想要直接阅读 Linux Kernel 7.0.0 源代码？参考 [**READING_SOURCE_CODE.md**](READING_SOURCE_CODE.md)：

- 🚀 **快速开始**：5 分钟内定位任意函数
- 🔍 **查找技巧**：grep、find、cscope、tags 等工具
- 📝 **阅读方法**：系统化的三种阅读策略
- 🛠️ **工具配置**：VS Code、Vim 等编辑器配置
- 📚 **实例教学**：以 read() 系统调用为例追踪代码
- 💡 **常见问题**：宏定义、条件编译等理解技巧
- 🎯 **学习计划**：4 周内掌握源代码阅读

**推荐顺序**：先阅读本文档的相关章节，然后参考 READING_SOURCE_CODE.md 在源代码中找到具体实现。

## 许可证

详见 LICENSE 文件。
cp /boot/config-$(uname -r) .config && make olddefconfig
make -j$(nproc) && sudo make install modules_install
sudo reboot
```

### Verify Running Kernel
```bash
uname -a       # Full kernel info
uname -r       # Kernel release version
dmesg          # Kernel boot messages
lsmod          # Loaded modules
cat /proc/version  # Kernel version details
```

### View Kernel Source
- Main entry: `arch/x86/kernel/head_64.S` (x86-64)
- Syscall handler: `fs/read_write.c` (read syscall)
- VFS operations: `fs/open.c`, `fs/close.c`
- Page cache: `mm/filemap.c`
- Block I/O: `block/blk-core.c`

## References

- Linux Kernel Documentation: https://www.kernel.org/doc/html/latest/
- Bootloader Specification: https://www.gnu.org/software/grub/manual/grub/
- Kernel Source Repository: https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/
- QEMU Documentation: https://www.qemu.org/documentation/

## License

See LICENSE file for details.
