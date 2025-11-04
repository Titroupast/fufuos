### 📄 **`dsctbl.c` - GDT 与 IDT 设置**

### 🌟 0. 概览与定位

*   **文件类型:** C 语言源文件
*   **文件作用:** 负责初始化全局段描述符表（GDT）和中断描述符表（IDT），这是 CPU 在保护模式下进行内存管理和中断处理的基础。
*   **本次更新亮点:** 针对 Day 7 的目标，此文件**新增了对键盘（`0x21`）和鼠标（`0x2c`）中断描述符的设置**，将硬件中断与相应的处理程序关联起来。

### 💖 1. 核心数据结构

此文件中使用的核心数据结构均定义在 `bootpack.h` 中。

*   **结构体名称:** `struct SEGMENT_DESCRIPTOR`
*   **作用:** GDT 中的一个条目，用于定义一个内存段的基地址、上限、访问权限等属性。
*   **成员列表:**
    *   `limit_low`, `base_low`: 段限长和基地址的低位。
    *   `base_mid`, `base_high`: 基地址的中位和高位。
    *   `access_right`: 定义段的类型、权限等属性。
    *   `limit_high`: 段限长的高位，以及一些扩展属性。

*   **结构体名称:** `struct GATE_DESCRIPTOR`
*   **作用:** IDT 中的一个条目，定义了一个中断或异常的处理程序入口地址、所属段及属性。
*   **成员列表:**
    *   `offset_low`, `offset_high`: 中断处理程序地址的低位和高位。
    *   `selector`: 中断处理程序所在的代码段的选择子。
    *   `dw_count`: 固定为 0，高 4 位是 DPL (Descriptor Privilege Level)。
    *   `access_right`: 定义门的类型（如中断门、陷阱门）和属性。

### ✨ 2. 关键函数分析

#### A. `init_gdtidt`

*   **功能描述:** 初始化 GDT 和 IDT。
*   **参数:** 无
*   **核心逻辑:**
    1.  **GDT 初始化:**
        *   将整个 GDT 表清零。
        *   使用 `set_segmdesc` 设置两个关键段：一个覆盖所有内存的数据段和一个用于 `bootpack.c` 的代码段。
        *   调用 `load_gdtr` 指令将 GDT 的地址和大小加载到 CPU 的 GDTR 寄存器中。
    2.  **IDT 初始化:**
        *   将整个 IDT 表清零。
        *   调用 `load_idtr` 指令将 IDT 的地址和大小加载到 CPU 的 IDTR 寄存器中。
    3.  **IDT 设置:**
        *   调用 `set_gatedesc` 为键盘中断（`0x21`）、IRQ7 伪中断（`0x27`）和鼠标中断（`0x2c`）分别注册对应的汇编中断处理函数（`asm_inthandler21` 等）。

#### B. `set_segmdesc`

*   **功能描述:** 设置 GDT 中的一个段描述符。
*   **参数:**
    *   `sd` (`struct SEGMENT_DESCRIPTOR *`): 指向要设置的段描述符的指针。
    *   `limit` (`unsigned int`): 段的字节数上限。
    *   `base` (`int`): 段的起始地址。
    *   `ar` (`int`): 段的访问权限属性。
*   **核心逻辑:**
    *   根据 x86 描述符的格式，将 `limit`、`base` 和 `ar` 的值拆分并填充到 `SEGMENT_DESCRIPTOR` 结构体的各个成员中。
    *   如果 `limit` 超过 `0xfffff`，会自动设置 G-bit 并将 `limit` 的单位从字节（B）调整为 4KB。

#### C. `set_gatedesc`

*   **功能描述:** 设置 IDT 中的一个门描述符。
*   **参数:**
    *   `gd` (`struct GATE_DESCRIPTOR *`): 指向要设置的门描述符的指针。
    *   `offset` (`int`): 中断处理程序的入口地址。
    *   `selector` (`int`): 中断处理程序所在的代码段选择子。
    *   `ar` (`int`): 门的访问权限和属性。
*   **核心逻辑:**
    *   按照 x86 门描述符的格式，将 `offset`、`selector` 和 `ar` 的值填充到 `GATE_DESCRIPTOR` 结构体的相应成员中。

### 📝 3. 变量与常量定义

*   **全局变量:**
    *   无
*   **宏定义/常量:**
    *   本文件没有定义新的常量，但使用了在 `bootpack.h` 中定义的地址常量（如 `ADR_GDT`, `ADR_IDT`）和访问权限常量（如 `AR_DATA32_RW`, `AR_INTGATE32`）。

### 💡 4. 与 Day 7 核心内容的关联

*   **加快中断处理 & 激活鼠标:** 此文件是实现中断处理的**基础前提**。通过在 `init_gdtidt` 中调用 `set_gatedesc`，它将来自 PIC 的键盘中断信号（IRQ1，对应中断号 `0x21`）和鼠标中断信号（IRQ12，对应中断号 `0x2c`）与操作系统中具体的处理函数（`asm_inthandler21` 和 `asm_inthandler2c`）绑定。没有这个绑定，CPU 就无法找到并执行中断服务程序，也就无法接收键盘和鼠标的数据，更谈不上后续的“加快处理”或“激活鼠标”。
