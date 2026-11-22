# vmlinuz 中的汇编代码详解（第二部分）- 保护模式和长模式转换

> 本文基于 Linux Kernel 7.0.0 版本，重点讲解保护模式到长模式的转换过程。

## 1. 进入保护模式

### 位置：arch/x86/boot/pm.c

这是所有准备工作的关键点，CPU 从实模式转换到保护模式。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/pm.c
- 第 100-150 行：protected_mode_jump 函数
```

**代码示例：**

```asm
; arch/x86/boot/pm.c
; 源代码位置：kernel-7.0.0/arch/x86/boot/pm.c:100-150

protected_mode_jump:
    # 这是一个汇编函数，需要禁用中断并启用保护模式
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:105-120
    
    # 禁用所有中断（将在保护模式中重新设置）
    cli
    
    # 加载 GDT
    # lgdtl 指令加载全局描述符表
    # GDT 包含多个 8 字节的段描述符
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:110
    lgdtl gdt_desc                  ; Load GDT Descriptor
    
    # 设置 CR0 的 PE（保护启用）位
    # CR0 是控制寄存器 0，PE 位在第 0 位
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:115-120
    mov %cr0, %eax
    or $1, %al                      ; 设置 PE 位（bit 0）
    mov %eax, %cr0
    
    # 通过远跳转进入 32 位代码
    # 这会加载 CS（代码段选择子）并清除预取队列
    # 源代码：kernel-7.0.0/arch/x86/boot/pm.c:125-130
    .byte 0xea                       ; 远跳转操作码
    .long protected_mode_code        ; 目标地址
    .word 0x0010                     ; 代码段选择子（1 * 8 = 偏移量 8）
```

**关键解释：**

CR0 寄存器的结构：
```
位 0：PE（Protection Enable）- 保护模式启用
位 1：MP（Monitor Coprocessor）
位 2：EM（Emulation）
位 3：TS（Task Switched）
位 4：ET（Extension Type）
位 5：NE（Numeric Error）
位 16：WP（Write Protect）
位 18：AM（Alignment Mask）
位 29：NW（Not Writethrough）
位 30：CD（Cache Disable）
位 31：PG（Paging）
```

## 2. 32 位保护模式代码

### 位置：arch/x86/boot/compressed/head_32.S

在 32 位保护模式中运行的代码。这是过渡阶段，准备进入 64 位长模式。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/compressed/head_32.S
- 第 1-50 行：entry point
- 第 50-150 行：段和栈设置
- 第 150-200 行：页表准备
```

**代码示例：**

```asm
; arch/x86/boot/compressed/head_32.S
; 源代码位置：kernel-7.0.0/arch/x86/boot/compressed/head_32.S:30-100

.code32
.align 4

ENTRY(startup_32)
    # 进入这个函数后，CPU 在 32 位保护模式
    # 分段寻址仍然有效
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_32.S:35-50
    
    # 建立数据段
    movl $__KERNEL_DS, %eax
    movl %eax, %ds
    movl %eax, %es
    movl %eax, %ss
    
    # 建立栈
    # 栈应该足够大，用于解压过程中的函数调用
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_32.S:55-60
    leal -PTREGS_SIZE(%rcx), %rsp
    
    # 保存启动参数地址（在 RSI 中由引导加载程序传递）
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_32.S:65-70
    movl %esi, %eax
    
    # 刷新指令缓存
    cld
    
    # 检查 CPUID 是否可用（需要在 64 位模式中）
    # 这会被跳过，直到进入 64 位模式
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_32.S:80-100
    
    # 计算内核重定位
    # 内核可以加载到任何对齐的地址
    
    # 准备进入 64 位长模式
    # 首先启用 PAE 和其他必需的 MSR
    
    jmp startup_64
```

## 3. 长模式转换（x86-64）

### 位置：arch/x86/boot/compressed/head_64.S

这是最关键的部分，从 32 位保护模式转换到 64 位长模式。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/compressed/head_64.S
- 第 50-150 行：startup_64 函数
- 第 150-250 行：长模式相关设置
- 第 300-400 行：页表设置
```

**启用长模式的步骤：**

```asm
; arch/x86/boot/compressed/head_64.S
; 源代码位置：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:50-180

ENTRY(startup_64)
    # 现在我们在 32 位保护模式
    # 需要完成以下步骤进入 64 位长模式：
    # 1. 启用 PAE（物理地址扩展）
    # 2. 建立页表
    # 3. 加载 CR3
    # 4. 启用长模式（通过 EFER MSR）
    # 5. 启用分页（设置 CR0.PG）
    # 6. 进行远跳转加载 CS
    
    # 步骤 1：启用 PAE
    # PAE 允许 32 位模式寻址超过 4GB 内存
    # CR4.PAE 位在第 5 位
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:60-70
    movl %cr4, %eax
    orl $0x00000020, %eax           ; CR4.PAE = 1（第 5 位）
    movl %eax, %cr4
    
    # 步骤 2 & 3：建立并加载页表
    # 页表已经由启动代码建立在内存中
    # 加载页表基址到 CR3
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:75-85
    leal pgtable(%rbx), %eax
    movl %eax, %cr3
    
    # 步骤 4：启用长模式
    # EFER = Extended Feature Enable Register (MSR 0xc0000080)
    # EFER.LME = Bit 8 = Long Mode Enable
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:90-105
    movl $0xc0000080, %ecx          ; EFER MSR 号
    rdmsr                           ; 读取 MSR 到 EDX:EAX
    btsl $8, %eax                   ; 设置第 8 位（LME）
    wrmsr                           ; 写回 MSR
    
    # 步骤 5：启用分页
    # 这会激活长模式
    # CR0.PG = Bit 31 = Paging Enable
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:110-120
    movl %cr0, %eax
    btsl $31, %eax                  ; 设置 PG 位
    movl %eax, %cr0
    
    # 现在我们在 64 位长模式（但仍在兼容模式）
    # 需要进行远跳转进入真正的 64 位模式
    
    # 步骤 6：远跳转到 64 位代码
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:125-135
    .byte 0xea                       ; 远跳转操作码
    .long real_mode_switch_continuation
    .word 0x0010                     ; 64 位代码段选择子

.code64
real_mode_switch_continuation:
    # 现在在真正的 64 位模式
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:140-180
    
    # 建立 64 位段寄存器
    xorq %rax, %rax
    movq %rax, %ds
    movq %rax, %es
    movq %rax, %ss
    movq %rax, %fs
    movq %rax, %gs
    
    # 建立 64 位栈
    leaq stack_init(%rip), %rsp
    
    # 进行跳转到 C 代码
    jmp decompress_kernel
```

## 4. 页表结构

### 位置：arch/x86/boot/compressed/head_64.S

初始页表的设置涉及 4 级分页。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/compressed/head_64.S
- 第 450-550 行：页表定义
```

**页表结构详解：**

```asm
; arch/x86/boot/compressed/head_64.S
; 源代码位置：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:460-550

# 页表结构（4 级分页）
# 在 64 位模式中，使用 4 级页表：
# 虚拟地址格式（64 位）：
# +-------+-------+-------+-------+------------+
# | PML4  | PDPT  |  PD   |  PT   | Page Offset|
# +-------+-------+-------+-------+------------+
# 48-63   39-47   30-38   21-29   0-20
#
# 物理地址从虚拟地址转换：
# 1. PML4 索引 = 虚拟地址[48-63] >> 16
# 2. PDPT 索引 = 虚拟地址[39-47] >> 9
# 3. PD 索引 = 虚拟地址[30-38] >> 9
# 4. PT 索引 = 虚拟地址[21-29] >> 9
# 5. 页内偏移 = 虚拟地址[0-20]

    .align PAGE_SIZE
pgtable:
    # PML4 表（级别 4）
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:470-480
    .quad 0x0000000000002007        ; 第一条 PML4 表项
                                    ; 指向 PDPT
                                    ; 0x2007 = 物理地址 0x2000 + 标志 0x007
                                    ; 标志含义：
                                    ;   bit 0 = 1（Present）
                                    ;   bit 1 = 1（Read/Write）
                                    ;   bit 2 = 1（User mode）
    
    # 其余 511 个 PML4 表项（都为 0，未使用）
    .fill 511, 8, 0
    
    # PDPT 表（级别 3）
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:485-495
    .quad 0x0000000000003007        ; 指向 PD
    .fill 511, 8, 0
    
    # PD 表（级别 2）
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:500-510
    .quad 0x0000000000004007        ; 指向 PT
    .fill 511, 8, 0
    
    # PT 表（级别 1）
    # 源代码：kernel-7.0.0/arch/x86/boot/compressed/head_64.S:515-540
    # 创建恒等映射（虚拟地址 = 物理地址）
    # 每个页表项覆盖 4KB 内存
    .quad 0x0000000000000007        ; 页 0：物理 0x0 + 标志 0x007
    .quad 0x0000000000001007        ; 页 1：物理 0x1000 + 标志 0x007
    .quad 0x0000000000002007        ; 页 2：物理 0x2000 + 标志 0x007
    .quad 0x0000000000003007        ; 页 3：物理 0x3000 + 标志 0x007
    .quad 0x0000000000004007        ; 页 4：物理 0x4000 + 标志 0x007
    # ... 更多页表项映射 2MB
    .fill (512 - 5), 8, 0           ; 其余页表项为 0
```

**页表项（Page Table Entry, PTE）的格式：**

```
位 63-52：保留（必须为 0）
位 51-12：物理基址（4KB 页）
位 11-9：可用（操作系统使用）
位 8：G（全局 - 防止 TLB 刷新）
位 7：PAT（页属性表）
位 6：D（脏位 - 页被写入过）
位 5：A（访问位 - 页被访问过）
位 4：PCD（缓存禁用）
位 3：PWT（写穿过）
位 2：U/S（用户/监督模式）
位 1：R/W（读/写）
位 0：P（存在）
```

## 5. 解压缩阶段

### 位置：arch/x86/boot/compressed/misc.c

在长模式中，解压缩内核镜像。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/compressed/misc.c
- 第 1-50 行：头文件和全局变量
- 第 300-400 行：extract_kernel 函数
```

**代码示例：**

```c
// arch/x86/boot/compressed/misc.c
// 源代码位置：kernel-7.0.0/arch/x86/boot/compressed/misc.c:300-400

asmlinkage __visible void *extract_kernel(void *rmode, 
                                          memptr heap,
                                          unsigned char *input_data, 
                                          unsigned long input_len,
                                          unsigned char *output, 
                                          unsigned long output_len)
{
    // 保存实模式参数
    // 源代码：kernel-7.0.0/arch/x86/boot/compressed/misc.c:310-320
    boot_params = *rmode;
    
    // 初始化控制台
    // 源代码：kernel-7.0.0/arch/x86/boot/compressed/misc.c:325-330
    console_init();
    debug_putstr("Early console initialized\n");
    
    // 准备解压缩环境
    // 堆用于动态分配内存
    real_mode = rmode;
    
    // 检查内核签名
    // 源代码：kernel-7.0.0/arch/x86/boot/compressed/misc.c:340-350
    if (memcmp((void *)input_data, "LINUX_KERNEL", 12) != 0) {
        error("Invalid kernel signature");
        return NULL;
    }
    
    // 自解压：内核可能与其自身重叠
    // 需要小心处理内存
    // 源代码：kernel-7.0.0/arch/x86/boot/compressed/misc.c:355-370
    if ((unsigned long)output + output_len > (unsigned long)input_data) {
        // 内核和压缩镜像重叠，需要先将压缩镜像移出
        memmove(output + output_len, input_data, input_len);
        input_data = output + output_len;
    }
    
    // 解压缩内核
    // 检查压缩格式
    // 源代码：kernel-7.0.0/arch/x86/boot/compressed/misc.c:375-410
    if (input_data[0] == 0x1f && input_data[1] == 0x8b) {
        // gzip 格式
        // 源代码：kernel-7.0.0/lib/decompress_gzip.c
        debug_putstr("Decompressing GZIP kernel...\n");
        gunzip(input_data, input_len, output, &output_len);
    } else if (input_data[0] == 0xfd && input_data[1] == 0x37) {
        // xz 格式
        // 源代码：kernel-7.0.0/lib/decompress_unxz.c
        debug_putstr("Decompressing XZ kernel...\n");
        unxz(input_data, input_len, output, &output_len);
    } else if (input_data[0] == 0x42 && input_data[1] == 0x5a) {
        // bzip2 格式
        // 源代码：kernel-7.0.0/lib/decompress_bunzip2.c
        debug_putstr("Decompressing BZIP2 kernel...\n");
        bunzip2(input_data, input_len, output, &output_len);
    } else {
        // 未压缩
        debug_putstr("Kernel not compressed\n");
        memmove(output, input_data, input_len);
    }
    
    debug_putstr("Kernel extracted\n");
    
    // 返回解压缩内核的入口点
    // 下一步会跳转到这个地址调用 start_kernel()
    return output;
}
```

## 6. CPU 功能检查

### 位置：arch/x86/boot/cpucheck.c

检查 CPU 是否支持长模式等必需功能。

**源代码位置：**
```
kernel-7.0.0/arch/x86/boot/cpucheck.c
- 第 1-50 行：check_cpu 函数
- 第 50-150 行：CPUID 检查
```

**代码示例：**

```asm
; arch/x86/boot/cpucheck.c 中的汇编部分
; 源代码位置：kernel-7.0.0/arch/x86/boot/cpucheck.c:60-150

check_cpu:
    # 检查 CPU 是否支持 CPUID 指令
    # 源代码：kernel-7.0.0/arch/x86/boot/cpucheck.c:65-85
    
    # 推送 EFLAGS
    pushfl
    
    # 尝试翻转 ID 位（bit 21）
    popl %eax
    movl %eax, %ecx
    xorl $0x200000, %eax
    pushl %eax
    popfl
    
    # 读回 EFLAGS
    pushfl
    popl %eax
    
    # 比较
    cmp %ecx, %eax
    jz no_cpuid                     ; CPUID 不可用
    
    # CPUID 可用，继续
    # 源代码：kernel-7.0.0/arch/x86/boot/cpucheck.c:90-120
    
    # 检查长模式支持
    # CPUID.80000001h:EDX.LM（bit 29）
    movl $0x80000001, %eax
    cpuid
    testl $0x20000000, %edx         ; LM 位（第 29 位）
    jz no_long_mode
    
    ret
    
no_long_mode:
    hlt                             ; CPU 不支持长模式，停止

no_cpuid:
    hlt                             ; CPUID 不可用，停止
```

## 总结

vmlinuz 中的汇编部分完成了以下关键转换：

1. **实模式基础设置** - 验证启动参数、启用 A20
2. **设置基础设施** - 建立 GDT、堆栈
3. **进入保护模式** - 通过设置 CR0.PE 和远跳转
4. **32 位代码准备** - 建立 32 位寄存器和栈
5. **启用 PAE** - 允许寻址超过 4GB
6. **建立页表** - 创建恒等映射
7. **启用长模式** - 通过 EFER.LME MSR
8. **启用分页** - 通过设置 CR0.PG
9. **进入 64 位代码** - 远跳转加载 64 位 CS
10. **解压缩内核** - 使用 gzip/xz/bzip2
11. **跳转到 C 代码** - 调用 start_kernel()

## 参考资源

- x86 启动代码：`kernel-7.0.0/arch/x86/boot/`
- 压缩代码：`kernel-7.0.0/arch/x86/boot/compressed/`
- 解压缩库：`kernel-7.0.0/lib/decompress_*.c`
- CPU 检查：`kernel-7.0.0/arch/x86/boot/cpucheck.c`
