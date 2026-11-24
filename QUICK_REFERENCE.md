# Linux Kernel 7.0.0 学习快速参考卡

## 🚀 5 分钟快速开始

```bash
# 1. 获取源代码
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux && git checkout v7.0.0

# 2. 编译
make menuconfig    # 配置
make -j$(nproc)    # 编译

# 3. 测试
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -nographic
```

## 📂 文件导航

| 想要学习 | 阅读 | 耗时 |
|---------|------|------|
| 什么是 vmlinuz | kernel-boot/vmlinuz-assembly-code-part1.md | 2-3h |
| CPU 保护模式 | kernel-boot/vmlinuz-assembly-code-part2.md | 3-4h |
| 内核初始化 | kernel-boot/vmlinuz-c-code-part1.md | 3-4h |
| read() 系统调用 | filesystem/vfs-filesystem-read-dispatch.md | 5-6h |
| 如何编译内核 | kernel-build/building-and-running-kernel.md | 2-3h |
| 学习路线 | LEARNING_GUIDE.md | 1-2h |
| **如何阅读源代码** | **READING_SOURCE_CODE.md** | **30min** |

## 🔑 核心概念（30 秒版本）

### 启动流程
```
BIOS/UEFI → Bootloader → 实模式 → 保护模式 → 长模式 → 解压缩 → start_kernel() → Linux 运行
```

### 系统调用流程
```
用户空间 read() → 系统调用 → VFS → 页缓存 → 块 I/O → 存储设备 → 中断 → 返回数据
```

## 📍 关键代码位置

| 阶段 | 文件 | 位置 |
|------|------|------|
| 启动协议 | Documentation/x86/boot.rst | 完整 |
| 实模式入口 | arch/x86/boot/header.S | 1-100 |
| 保护模式 | arch/x86/boot/pm.c | 50-150 |
| 长模式 | arch/x86/boot/compressed/head_64.S | 1-200 |
| 解压缩 | arch/x86/boot/compressed/misc.c | 1-100 |
| C 初始化 | init/main.c | 800-1100 |
| 读文件 | fs/read_write.c | 400-500 |
| 页缓存 | mm/filemap.c | 1-100 |
| 块 I/O | block/blk-core.c | 1-100 |

## 💻 常用命令速查

### 编译相关
```bash
# 配置菜单
make menuconfig

# 编译
make -j$(nproc)

# 编译模块
make modules -j$(nproc)

# 安装
sudo make install
sudo make modules_install

# 清理
make clean      # 删除编译产物
make mrproper   # 完全清理
```

### 查看源代码
```bash
# 在线浏览
https://elixir.bootlin.com/linux/v7.0.0/source/

# 本地查看
grep -n "function_name" file.c          # 搜索
sed -n '100,150p' file.c                # 查看行数
wc -l file.c                            # 计数
```

### 在 QEMU 中运行
```bash
# 简单启动
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -nographic

# 调试模式（-s：GDB 服务器，-S：启动时暂停）
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -s -S -nographic
```

### GDB 调试
```bash
gdb vmlinux
(gdb) target remote localhost:1234
(gdb) break start_kernel
(gdb) continue
(gdb) next
(gdb) print x
(gdb) backtrace
```

### 查看日志
```bash
# 实时内核日志
tail -f /var/log/kern.log

# 或使用 dmesg
dmesg
dmesg | tail -50
dmesg | grep -i "error"
```

## 🧠 学习三步走

### 第 1 步：理论（2 周）
1. 阅读启动过程文章（vmlinuz 三部分）
2. 理解 CPU 模式转换
3. 学习内存管理基础

### 第 2 步：实践（1 周）
1. 编译 Linux 7.0.0 内核
2. 在 QEMU 中运行
3. 添加 printk() 调试信息
4. 观察启动输出

### 第 3 步：深入（1-2 周）
1. 学习系统调用原理
2. 使用 GDB 单步调试
3. 修改内核代码
4. 性能分析和优化

## 🐛 常见问题速解

| 问题 | 答案 |
|------|------|
| 编译失败 | `apt install build-essential libncurses-dev libssl-dev libelf-dev` |
| 内存不足 | `make -j2` 或 `make -j1` |
| QEMU 无响应 | 等待更长时间，或用 `-m 2G` 增加内存 |
| GDB 连接不了 | 确保 QEMU 以 `-s` 启动 |
| 找不到源代码 | 访问 https://elixir.bootlin.com/linux/v7.0.0/source/ |

## 📚 推荐阅读顺序

```
START → README.md（项目简介）
  ↓
LEARNING_GUIDE.md（学习路线）
  ↓
kernel-boot/README.md（启动概览）
  ↓
vmlinuz-assembly-code-part1.md（实模式）
  ↓
vmlinuz-assembly-code-part2.md（模式转换）
  ↓
vmlinuz-c-code-part1.md（初始化）
  ↓
kernel-build/README.md（编译指南）
  ↓
building-and-running-kernel.md（详细步骤）
  ↓
（实际编译和测试）
  ↓
filesystem/README.md（系统调用概览）
  ↓
vfs-filesystem-read-dispatch.md（深入分析）
  ↓
END → 成为 Linux 内核专家！
```

## 🌐 在线资源

- 官方内核文档：https://www.kernel.org/doc/
- 在线代码浏览：https://elixir.bootlin.com/
- Kernel Newbies：https://kernelnewbies.org/
- LWN.net：https://lwn.net/

## 📝 笔记模板

学习时可以使用这个模板记录笔记：

```markdown
# [主题名称]

## 要点
- 要点 1
- 要点 2
- 要点 3

## 源代码位置
- 文件1：路径
- 文件2：路径

## 关键代码片段
```c
// 代码
```

## 问题
- 问题 1：?
- 问题 2：?

## 下一步
- [ ] 阅读源代码
- [ ] 修改代码测试
- [ ] 编译并运行
```

## ⏱️ 时间规划建议

| 目标 | 时间 | 路径 |
|------|------|------|
| 理解启动流程 | 1 周 | Part1 → Part2 |
| 完整学习 | 4 周 | 全部文章 + 编译 |
| 精通内核 | 3 个月 | 全文 + 修改代码 + 调试 |
| 内核贡献者 | 1 年+ | 深入研究 + 参与开发 |

---

**记住**：Linux 内核是世界上最复杂的开源项目之一。不懂没关系，坚持下去，逐步深入！

**最后一句**：源代码是最好的文档。遇到疑惑时，直接看代码！
