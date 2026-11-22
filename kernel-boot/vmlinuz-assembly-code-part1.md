# vmlinuz 中的汇编代码详解（第一部分）- Linux 启动协议和内存布局

> 本文基于 Linux Kernel 7.0.0 版本，重点讲解 x86-64 架构上 vmlinuz 中的汇编代码部分。

## 概述

vmlinuz 文件的前半部分完全由汇编代码组成，负责从实模式启动到进入保护模式/长模式的完整过程。这部分代码是与硬件最紧密相关的，涉及 CPU 模式切换、内存管理、中断处理等低级细节。

## 1. Linux 启动协议

### 基本规范

Linux 启动协议定义了引导加载程序和内核之间的接口。对于 x86 架构，协议文档位于：

**源代码位置：**
```
kernel-7.0.0/Documentation/x86/boot.rst
```

### 关键要求

**内存布局：**
```
0x00000000 +--------------------------+
           | 实模式中断向量表 (IVT)   | 1 KB
0x00000400 +--------------------------+
           | BIOS 数据区              | 256 B
0x00000500 +--------------------------+
           | 保留                     |
0x00007C00 +--------------------------+
           | MBR 引导扇区             | 512 B
0x00007E00 +--------------------------+
           | 保留                     |
0x00010000 +--------------------------+
           | 内核实模式代码 (内核设置)|
           | kernel setup code        | 32 KB
0x00018000 +--------------------------+
           | 保留                     |
0x00020000 +--------------------------+
           | 压缩内核 + 解压缩器      |
           | kernel + decompressor   |
0x01000000 +--------------------------+
           | 解压缩后的内核 (64位模式)|
           | uncompressed kernel     | (可变大小)
0x???????? +--------------------------+
```

### 启动参数结构

引导加载程序必须在地址 `0x90000` 处设置启动参数结构。

**源代码位置：**
```
kernel-7.0.0/arch/x86/include/uapi/asm/bootparam.h

struct boot_params {
    // 在此文件的第 65-220 行
};
```

**主要结构体定义：**

```c
// 来自 arch/x86/include/uapi/asm/bootparam.h (第 65-120 行)
struct boot_params {
    struct screen_info screen_info;         // 屏幕信息（偏移 0x0）
    struct apm_bios_info apm_bios_info;     // APM BIOS 信息（偏移 0x40）
    __u8  _pad2[4];
    __u64 tboot_addr;                       // tboot 地址（偏移 0x50）
    struct ist_info ist_info;               // IST 信息（偏移 0x58）
    __u8  _pad3[8];
    __u64 acpi_rsdp_addr;                   // ACPI RSDP 地址（偏移 0x70）
    __u8  _pad4[8];
    __u8  edid_info[128];                   // EDID 信息（偏移 0x90）
    struct efi_info efi_info;               // EFI 信息（偏移 0x110）
    __u32 alt_mem_k;                        // 替代内存大小（KB）
    __u32 scratch;                          // 临时存储
    __u8  e820_entries;                     // E820 表项数
    __u8  eddbuf_entries;                   // EDD 表项数
    __u8  edd_mbr_sig_buf_entries;          // EDD MBR 签名表项数
    __u8  kbd_status;                       // 键盘状态
    struct setup_header hdr;                // 设置头（关键！在偏移 0x1f1）
    // ... 更多字段
};
```

### setup_header 结构 - 最关键的部分

**源代码位置：**
```
kernel-7.0.0/arch/x86/include/uapi/asm/bootparam.h

struct setup_header {
    // 在此文件的第 50-64 行
};
```

**详细的结构定义：**

```c
// 来自 arch/x86/include/uapi/asm/bootparam.h (第 50-64 行)
struct setup_header {
    __u8    setup_sects;                    // 设置代码占用的扇区数（偏移 1f1h）
    __u16   root_flags;                     // 根文件系统标志（偏移 1f2h）
    __u32   syssize;                        // 内核大小（以 16 字节为单位）（偏移 1f4h）
    __u16   ram_size;                       // RAM 大小（已弃用）（偏移 1f8h）
    __u16   vid_mode;                       // 视频模式（偏移 1fah）
    __u16   root_dev;                       // 根设备（偏移 1fch）
    __u16   boot_flag;                      // 启动标志（0xAA55）（偏移 1feh）
    __u16   jump;                           // 跳转指令（偏移 200h）
    __u32   header;                         // "HdrS" 魔数（偏移 202h）
    __u16   version;                        // 版本号（偏移 206h）
    __u32   realmode_swtch;                 // 实模式切换地址（偏移 208h）
    __u16   start_sys_seg;                  // 启动系统段（偏移 20ch）
    __u16   kernel_version;                 // 内核版本（偏移 20eh）
    __u8    type_of_loader;                 // 加载器类型（偏移 210h）
    __u8    loadflags;                      // 加载标志（偏移 211h）
    __u16   setup_move_size;                // 设置代码移动大小（偏移 212h）
    __u32   code32_start;                   // 32 位代码开始地址（偏移 214h）
    __u32   ramdisk_image;                  // RAM 磁盘镜像地址（偏移 218h）
    __u32   ramdisk_size;                   // RAM 磁盘大小（偏移 21ch）
    __u32   bootsect_kludge;                // 启动扇区 kludge 地址（偏移 220h）
    __u16   heap_end_ptr;                   // 堆结束指针（偏移 224h）
    __u8    ext_loader_ver;                 // 扩展加载器版本（偏移 226h）
    __u8    ext_loader_type;                // 扩展加载器类型（偏移 227h）
    __u32   cmd_line_ptr;                   // 命令行指针（偏移 228h）
    __u32   initrd_addr_max;                // initrd 最大地址（偏移 22ch）
    __u32   kernel_alignment;               // 内核对齐（偏移 230h）
    __u8    relocatable_kernel;             // 可重定位内核（偏移 234h）
    __u8    min_alignment;                  // 最小对齐（偏移 235h）
    __u16   xloadflags;                     // 扩展加载标志（偏移 236h）
    __u32   cmdline_size;                   // 命令行大小（偏移 238h）
    __u32   hardware_subarch;               // 硬件子架构（偏移 23ch）
    __u64   hardware_subarch_data;          // 硬件子架构数据（偏移 240h）
    __u32   payload_offset;                 // 有效负载偏移（偏移 248h）
    __u32   payload_length;                 // 有效负载长度（偏移 24ch）
    __u64   setup_data;                     // 设置数据（偏移 250h）
    __u64   pref_address;                   // 首选地址（偏移 258h）
    __u32   init_size;                      // 初始大小（偏移 260h）
    __u32   handover_offset;                // 握手偏移（偏移 264h）
    __u32   kernel_info_offset;             // 内核信息偏移（偏移 268h）
};
```

## 2. 实模式代码入口

### 位置：arch/x86/boot/header.S

这是 vmlinuz 文件最开始的代码，CPU 在启动时首先执行这里。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/header.S

关键行：
- 第 1-50 行：文件头和初始设置
- 第 51-100 行：MZ 兼容头
- 第 200 行：实模式代码真正开始位置
- 第 200-400 行：引导代码和参数验证
```

**代码片段和详细注释：**

```asm
; arch/x86/boot/header.S
; 这个文件定义了 vmlinuz 的开头
; 源代码位置：kernel-7.0.0/arch/x86/boot/header.S 第 1 行开始

    .code16                         ; 16 位代码（实模式）
    .section ".bstext", "ax"
    
    .globl _start
_start:
    # 这是 vmlinuz 中的第一条指令
    # 位于文件偏移 0x0
    # 源代码：kernel-7.0.0/arch/x86/boot/header.S:46-60
    
    # 由于 DOS 兼容性，前面有 MZ 头
    .byte 0x4d, 0x5a                ; MZ 头（"MZ"）
    
    # ... DOS 存根代码（大约 200+ 字节）...
    
    # 在偏移 0x200 处，真正的内核启动代码开始
    # 源代码：kernel-7.0.0/arch/x86/boot/header.S:200-250
    
    .org 0x200
    
boot_sector:
    # 检查引导加载程序（GRUB2 或其他）
    cmpw $0xaa55, %bx               ; 检查 BX 是否为 0xaa55
    jne bad_loader                   ; 如果不是，启动失败
    
    # 启用 A20 地址线（实模式只能访问 1MB）
    # 源代码：kernel-7.0.0/arch/x86/boot/a20.c
    call enable_a20
    
    # 保存引导加载程序传递的值到段寄存器
    # 源代码：kernel-7.0.0/arch/x86/boot/header.S:260-280
    mov %ax, %ds
    mov %ax, %es
    mov %ax, %ss
    mov %ax, %fs
    mov %ax, %gs
    
    # 设置堆栈指针
    # 实模式下最多只能有 64KB 的堆栈
    mov $0x4000, %sp
    
    # 跳转到设置代码
    # 源代码：kernel-7.0.0/arch/x86/boot/main.c 中的 main() 函数
    jmp setup_code
```

## 3. 设置代码（Setup Code）

### 位置：arch/x86/boot/main.c 和 arch/x86/boot/pm.c

在汇编头部之后，是用 C 语言和汇编混合写的设置代码。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/main.c
- 第 1-100 行：头文件和全局变量
- 第 150-200 行：main() 函数
- 第 200-400 行：CPU 检查和内存检测

kernel-7.0.0/arch/x86/boot/pm.c
- 第 1-50 行：保护模式切换
- 第 50-150 行：GDT 和 IDT 设置
```

**代码示例：**

```c
// arch/x86/boot/main.c
// 源代码位置：kernel-7.0.0/arch/x86/boot/main.c:170-250

void main(void)
{
    /* 首先，尽早初始化控制台以便输出调试信息 */
    console_init();                 /* 源代码：kernel-7.0.0/arch/x86/boot/printf.c */
    
    /* 验证启动参数 */
    if (boot_params.hdr.header != HdrS) {
        puts("Invalid kernel setup header\n");
        die("Invalid kernel");    /* 源代码：kernel-7.0.0/arch/x86/boot/reboot.c */
    }
    
    /* 检查引导加载程序版本 */
    if (boot_params.hdr.version < 0x020c) {
        puts("Boot loader version is too old\n");
        die("Boot loader version too old");
    }
    
    /* 初始化内存管理 */
    detect_memory();                /* 源代码：kernel-7.0.0/arch/x86/boot/memory.c:140-180 */
    
    /* 获取 CPU 信息 */
    get_cpu_vendor_id();            /* 源代码：kernel-7.0.0/arch/x86/boot/cpucheck.c:60-100 */
    get_cpu_name();                 /* 源代码：kernel-7.0.0/arch/x86/boot/cpucheck.c:100-150 */
    get_cpu_cap_flags();            /* 源代码：kernel-7.0.0/arch/x86/boot/cpucheck.c:150-200 */
    
    /* 检查必需的 CPU 功能 */
    check_cpu();                    /* 源代码：kernel-7.0.0/arch/x86/boot/cpucheck.c:300-350 */
    
    /* 获取 BIOS 提供的信息 */
    get_bios_information();         /* 源代码：kernel-7.0.0/arch/x86/boot/main.c:300-350 */
    get_efi_information();          /* 源代码：kernel-7.0.0/arch/x86/boot/efi.c */
    get_acpi_information();         /* 源代码：kernel-7.0.0/arch/x86/boot/acpi.c */
    
    /* 验证启动参数的完整性 */
    verify_boot_params();           /* 源代码：kernel-7.0.0/arch/x86/boot/main.c:450-500 */
    
    /* 进入 32 位保护模式 */
    enable_tsc();                   /* 源代码：kernel-7.0.0/arch/x86/boot/main.c:550-560 */
    setup_gdt();                    /* 源代码：kernel-7.0.0/arch/x86/boot/pm.c:50-120 */
    setup_idt();                    /* 源代码：kernel-7.0.0/arch/x86/boot/pm.c:120-180 */
    
    /* 最后一步：跳转到 32 位代码 */
    protected_mode_jump();          /* 源代码：kernel-7.0.0/arch/x86/boot/pm.c:200-250 */
}
```

## 4. 保护模式转换

### A20 地址线启用

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/a20.c
- 第 1-50 行：INT 15 方法
- 第 50-150 行：键盘控制器方法
- 第 150-200 行：快速 A20 方法
```

**代码示例：**

```asm
; arch/x86/boot/a20.c
; 源代码位置：kernel-7.0.0/arch/x86/boot/a20.c:140-200

; 方法 1：通过 BIOS INT 15
enable_a20_int15:
    # 源代码：kernel-7.0.0/arch/x86/boot/a20.c:140-160
    mov $0x2401, %ax                ; INT 15，功能 2401h
    int $0x15
    
    # 检查返回状态
    jc a20_int15_failed
    cmpb $0, %ah
    jne a20_int15_failed
    
    ret
    
a20_int15_failed:
    # 方法 1 失败，尝试方法 2
    
; 方法 2：通过键盘控制器
; 源代码：kernel-7.0.0/arch/x86/boot/a20.c:160-220
enable_a20_kbc:
    # 禁用 IRQ1（键盘）
    call a20_io_delay
    mov $0xad, %al
    out %al, $0x64
    
    # 读取输出端口
    call a20_io_delay
    mov $0xd0, %al
    out %al, $0x64
    call a20_io_delay
    in $0x60, %al
    
    # 保存原值
    mov %al, %cl
    
    # 设置 A20 位
    or $2, %al
    
    # 写回输出端口
    call a20_io_delay
    mov $0xd1, %al
    out %al, $0x64
    call a20_io_delay
    mov %cl, %al
    out %al, $0x60
    
    # 重新启用 IRQ1
    call a20_io_delay
    mov $0xae, %al
    out %al, $0x64
    
    ret
```

## 5. 全局描述符表（GDT）设置

### 位置：arch/x86/boot/pm.c

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/pm.c
- 第 1-50 行：GDT 定义
- 第 50-120 行：setup_gdt() 函数
```

**代码示例：**

```asm
; arch/x86/boot/pm.c
; 源代码位置：kernel-7.0.0/arch/x86/boot/pm.c:1-80

setup_gdt:
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:50-80
    
    # GDT 表的结构
    .align 8
gdt:
    # 描述符 0：空描述符（必需）
    .quad 0x0000000000000000
    
    # 描述符 1：代码段（x86 实模式兼容）
    # 基址：0x0000，限制：0xFFFF，特权级：0
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:10-20
    .quad 0x00cf9a000000ffff
    
    # 描述符 2：数据段
    # 基址：0x0000，限制：0xFFFF，特权级：0
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:20-30
    .quad 0x00cf92000000ffff
    
    # 描述符 3：代码段（32 位）
    .quad 0x00cf9a000000ffff
    
    # 描述符 4：数据段（32 位）
    .quad 0x00cf92000000ffff

gdt_end:

# GDT 描述符（GDTR）
# 源代码：kernel-7.0.0/arch/x86/boot/pm.c:35-45
gdt_desc:
    .word gdt_end - gdt - 1          ; GDT 大小（字节数 - 1）
    .long gdt                        ; GDT 基址
```

继续阅读第二部分...

## 参考资源

- Linux 启动协议文档：`kernel-7.0.0/Documentation/x86/boot.rst`
- x86 启动代码：`kernel-7.0.0/arch/x86/boot/`
- 启动参数定义：`kernel-7.0.0/arch/x86/include/uapi/asm/bootparam.h`
