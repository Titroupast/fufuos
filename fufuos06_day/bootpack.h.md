# `bootpack.h` 文件解析

`bootpack.h` 是操作系统的核心头文件。它像一个中央仓库，集中管理了项目所需的数据结构定义、函数声明和重要的常量。这确保了不同C文件中的代码可以无缝地协同工作。

## 主要内容

### 1. `struct BOOTINFO`
这个结构体至关重要，它是汇编引导程序向C语言内核传递信息的唯一途径。
```c
struct BOOTINFO {
	char cyls;
	char leds;
	char vmode;
	char reserve;
	short scrnx, scrny; // 屏幕分辨率
	char *vram;         // 指向显存的指针
};
#define ADR_BOOTINFO	0x00000ff0
```
它包含了屏幕分辨率 (`scrnx`, `scrny`) 和一个指向显存的指针 (`vram`) 等关键信息，这些信息对于图形绘制至关重要。

### 2. 汇编函数原型 (`naskfunc.nas`)
头文件为那些用汇编语言实现的底层函数声明了C语言原型。这使得我们可以直接在C代码中调用这些与硬件交互的底层函数。
- **CPU 控制**: `io_hlt`, `io_cli`, `io_sti` (停机, 关/开中断)。
- **I/O 端口**: `io_out8` (向硬件端口写入数据)。
- **系统寄存器**: `load_gdtr`, `load_idtr` (加载 GDT 和 IDT 的地址到CPU寄存器)。
- **中断处理程序**: `asm_inthandler21`, `asm_inthandler27`, `asm_inthandler2c` (中断的汇编入口点)。

### 3. C 函数原型
它声明了来自其他C文件的所有主要函数，使它们在整个项目中可见。
- **`graphic.c`**: 包含初始化调色板、绘制矩形 (`boxfill8`)、渲染文本 (`putfonts8_asc`) 和管理鼠标指针的函数。
- **`dsctbl.c`**: 初始化 GDT 和 IDT 的函数 (`init_gdtidt`)。
- **`int.c`**: 初始化 PIC 的函数 (`init_pic`) 和C语言级别的中断处理函数。

### 4. 数据结构定义
头文件定义了设置 GDT 和 IDT 所需的内存描述符和中断门描述符的C结构体。
- **`struct SEGMENT_DESCRIPTOR`**: 定义了 GDT 中一个段描述符的布局。
- **`struct GATE_DESCRIPTOR`**: 定义了 IDT 中一个门描述符的布局。

### 5. 常量和宏
文件中充满了 `#define` 定义的常量，这些常量为重要的数值赋予了有意义的名称。
- **内存地址**: `ADR_GDT`, `ADR_IDT` 等。这些定义了关键系统表的内存地址。
- **颜色常量**: `COL8_FFFFFF` (白色), `COL8_008484` (暗青色) 等。这些使得图形代码更具可读性。
- **PIC 端口**: `PIC0_IMR`, `PIC1_ICW1` 等。这些定义了可编程中断控制器的硬件端口地址。
- **访问权限**: `AR_DATA32_RW`, `AR_INTGATE32`。这些是用于设置GDT和IDT条目权限和类型的位掩码。
