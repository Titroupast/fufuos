# bootpack.c 代码逐行解释

这是操作系统核心的C语言代码。它负责初始化屏幕、设置调色板，并绘制一个简单的图形用户界面。

---

### 函数声明

```c
void io_hlt(void);
void io_cli(void);
void io_out8(int port, int data);
int io_load_eflags(void);
void io_store_eflags(int eflags);
```
- 这部分是函数声明，它们告诉C编译器这些函数的存在、参数类型和返回值类型。
- 这些函数都是在汇编文件 `naskfunc.nas` 中实现的底层函数。
- `io_hlt`: 让CPU暂停，直到有中断发生。
- `io_cli`: 关闭中断。
- `io_out8`: 向指定的I/O端口写入8位数据。
- `io_load_eflags`: 读取EFLAGS寄存器的值。
- `io_store_eflags`: 向EFLAGS寄存器写入值。

```c
void init_palette(void);
void set_palette(int start, int end, unsigned char *rgb);
void boxfill8(unsigned char *vram, int xsize, unsigned char c, int x0, int y0, int x1, int y1);
```
- 这是在本文件中实现的三个函数的原型声明。
- `init_palette`: 初始化调色板。
- `set_palette`: 设置调色板中的颜色。
- `boxfill8`: 在屏幕上绘制一个填充矩形。

---

### 颜色定义

```c
#define COL8_000000		0
#define COL8_FF0000		1
// ... (其他颜色定义)
#define COL8_848484		15
```
- 使用 `#define` 预处理指令为16种颜色分别定义了别名。
- `COL8_` 前缀表示这是一个8位颜色。后面的六位十六进制数代表RGB（红绿蓝）值。
- 数字 `0` 到 `15` 是这些颜色在调色板中的索引。

---

### 主函数 `HariMain`

```c
void HariMain(void)
{
	char *vram;/* 声明变量vram、用于BYTE [...]地址 */
	int xsize, ysize;

	init_palette();/* 设定调色板 */
	vram = (char *) 0xa0000;/* 地址变量赋值 */
	xsize = 320;
	ysize = 200;

	/* 绘制背景和UI元素 */
	boxfill8(vram, xsize, COL8_008484,  0,         0,          xsize -  1, ysize - 29);
	// ... (其他绘制调用)
	boxfill8(vram, xsize, COL8_FFFFFF, xsize -  3, ysize - 24, xsize -  3, ysize -  3);

	for (;;) {
		io_hlt();
	}
}
```
- **`HariMain`**: 这是C代码的入口点，相当于标准C程序中的`main`函数。
- **`char *vram; int xsize, ysize;`**: 声明变量。`vram`是一个指向字符的指针，将用来指向显存地址。`xsize`和`ysize`用来存储屏幕的宽和高。
- **`init_palette();`**: 调用函数来初始化16色的调色板。
- **`vram = (char *) 0xa0000;`**: 将显存的起始地址 `0xa0000` 赋值给 `vram` 指针。在VGA的320x200 256色模式下，这是显存的地址。
- **`xsize = 320; ysize = 200;`**: 设置屏幕的尺寸。
- **`boxfill8(...)`**: 这一系列调用使用不同的颜色和坐标在屏幕上绘制矩形，构成了一个简单的桌面背景、任务栏和按钮的雏形。
- **`for (;;) { io_hlt(); }`**: 这是一个无限循环。循环体内只有 `io_hlt()`，这条指令会使CPU暂停运行，直到有中断信号（如键盘输入、鼠标移动或定时器）到来。这是一种在操作系统无事可做时降低CPU功耗的常用方法。

---

### 调色板初始化函数 `init_palette`

```c
void init_palette(void)
{
	static unsigned char table_rgb[16 * 3] = {
		0x00, 0x00, 0x00,	/*  0:黑 */
		// ... (RGB颜色值)
		0x84, 0x84, 0x84	/* 15:暗灰 */
	};
	set_palette(0, 15, table_rgb);
	return;
}
```
- **`static unsigned char table_rgb[16 * 3]`**: 定义一个静态数组，存储16种颜色的RGB值。每个颜色由3个字节（R, G, B）表示，所以数组大小为 `16 * 3 = 48` 字节。`static` 关键字确保这个数组在程序运行期间只被初始化一次。
- **`set_palette(0, 15, table_rgb);`**: 调用 `set_palette` 函数，将 `table_rgb` 数组中定义的从索引0到15的颜色信息写入到显卡的硬件调色板中。

---

### 设置调色板函数 `set_palette`

```c
void set_palette(int start, int end, unsigned char *rgb)
{
	int i, eflags;
	eflags = io_load_eflags();	/* 记录中断许可标志的值 */
	io_cli(); 					/* 将中断许可标志置为0,禁止中断 */
	io_out8(0x03c8, start);
	for (i = start; i <= end; i++) {
		io_out8(0x03c9, rgb[0] / 4);
		io_out8(0x03c9, rgb[1] / 4);
		io_out8(0x03c9, rgb[2] / 4);
		rgb += 3;
	}
	io_store_eflags(eflags);	/* 复原中断许可标志 */
	return;
}
```
- 这个函数负责将RGB颜色数据写入VGA控制器的调色板寄存器。
- **`eflags = io_load_eflags();`**: 保存当前CPU的标志寄存器（EFLAGS）的状态，特别是中断允许标志（IF）。
- **`io_cli();`**: 关闭中断。在修改硬件状态（如调色板）时，需要禁止中断，以防止中断处理程序干扰这个过程。
- **`io_out8(0x03c8, start);`**: 向端口 `0x03c8` (VGA DAC写模式寄存器) 写入要开始设置的调色板索引号。
- **`for (i = start; i <= end; i++) { ... }`**: 循环遍历需要设置的每一种颜色。
- **`io_out8(0x03c9, rgb[0] / 4);`**: 依次向端口 `0x03c9` (VGA DAC数据寄存器) 写入当前颜色的R, G, B值。VGA调色板的每个颜色分量是6位的（0-63），而我们提供的`table_rgb`是8位的（0-255）。这里通过除以4将8位值转换为6位值。
- **`rgb += 3;`**: 将指针`rgb`移动3个字节，指向下一个颜色的RGB数据。
- **`io_store_eflags(eflags);`**: 恢复之前保存的EFLAGS寄存器的值，如果之前中断是开启的，现在就重新开启中断。

---

### 矩形填充函数 `boxfill8`

```c
void boxfill8(unsigned char *vram, int xsize, unsigned char c, int x0, int y0, int x1, int y1)
{
	int x, y;
	for (y = y0; y <= y1; y++) {
		for (x = x0; x <= x1; x++)
			vram[y * xsize + x] = c;
	}
	return;
}
```
- 这是一个简单的绘图函数，用于在屏幕上用指定的颜色`c`填充一个矩形区域。
- **参数**: `vram`是显存指针, `xsize`是屏幕宽度, `c`是颜色索引, `(x0, y0)`是矩形左上角坐标, `(x1, y1)`是矩形右下角坐标。
- **`for (y = y0; y <= y1; y++)`**: 外层循环遍历矩形区域的每一行。
- **`for (x = x0; x <= x1; x++)`**: 内层循环遍历当前行的每一个像素。
- **`vram[y * xsize + x] = c;`**: 这是核心操作。它计算出像素 `(x, y)` 在显存中的一维地址（`y * xsize + x`），然后将指定的颜色值 `c` 写入该地址。这样就在屏幕上绘制了一个点。两个循环结束后，整个矩形区域就被填充了。

---

### 新增数据结构

随着功能增加，代码引入了几个重要的数据结构，用于在启动加载程序和内核之间传递信息，以及设置CPU的关键数据结构（GDT和IDT）。

#### `struct BOOTINFO`

```c
struct BOOTINFO {
	char cyls, leds, vmode, reserve;
	short scrnx, scrny;
	char *vram;
};
```
- 这个结构体用于从启动扇区（`asmhead.nas`）向操作系统内核（`bootpack.c`）传递重要的系统信息。
- `cyls`: 启动时软盘的柱面数。
- `leds`: 启动时键盘LED灯的状态。
- `vmode`: 视频模式。
- `reserve`: 保留字节。
- `scrnx`, `scrny`: 屏幕的宽度和高度（分辨率），以像素为单位。
- `vram`: 指向显存起始地址的指针。

#### `struct SEGMENT_DESCRIPTOR`

```c
struct SEGMENT_DESCRIPTOR {
	short limit_low, base_low;
	char base_mid, access_right;
	char limit_high, base_high;
};
```
- 这是全局段描述符（Global Segment Descriptor）的结构。它用于定义内存段的起始地址、大小、访问权限等属性。这是保护模式内存管理的基础。
- `limit_low`, `limit_high`: 共同定义了段的界限（大小）。
- `base_low`, `base_mid`, `base_high`: 共同定义了32位的段基地址。
- `access_right`: 访问权限，定义了段的类型（代码段/数据段）、权限级别等。

#### `struct GATE_DESCRIPTOR`

```c
struct GATE_DESCRIPTOR {
	short offset_low, selector;
	char dw_count, access_right;
	short offset_high;
};
```
- 这是门描述符（Gate Descriptor）的结构，主要用于设置中断处理程序。当中断发生时，CPU会通过中断描述符表（IDT）中的门描述符找到并跳转到对应的处理函数。
- `offset_low`, `offset_high`: 共同定义了中断处理程序入口点的32位偏移地址。
- `selector`: 段选择子，指向GDT中的一个代码段。
- `access_right`: 定义了门的类型和属性。

---

### 主函数 `HariMain` (已更新)

`HariMain`函数已经更新，不再使用硬编码的屏幕尺寸和显存地址，而是通过`BOOTINFO`结构体来获取这些信息。同时，它现在还增加了显示字符和鼠标指针的功能。

```c
void HariMain(void)
{
	struct BOOTINFO *binfo = (struct BOOTINFO *) 0x0ff0;
	char s[40], mcursor[256];
	int mx, my;

	init_palette();
	init_screen(binfo->vram, binfo->scrnx, binfo->scrny);

	/* 显示鼠标 */
	mx = (binfo->scrnx - 16) / 2; /* 计算画面的中心坐标*/
	my = (binfo->scrny - 28 - 16) / 2;
	init_mouse_cursor8(mcursor, COL8_008484);
	putblock8_8(binfo->vram, binfo->scrnx, 16, 16, mx, my, mcursor, 16);

	putfonts8_asc(binfo->vram, binfo->scrnx,  8,  8, COL8_FFFFFF, "ABC 123");
	putfonts8_asc(binfo->vram, binfo->scrnx, 31, 31, COL8_000000, "Haribote OS.");
	putfonts8_asc(binfo->vram, binfo->scrnx, 30, 30, COL8_FFFFFF, "Haribote OS.");

	sprintf(s, "scrnx = %d", binfo->scrnx);
	putfonts8_asc(binfo->vram, binfo->scrnx, 16, 64, COL8_FFFFFF, s);

	for (;;) {
		io_hlt();
	}
}
```
- **`struct BOOTINFO *binfo = (struct BOOTINFO *) 0x0ff0;`**: `asmhead.nas`将`BOOTINFO`结构体保存在了内存地址`0x0ff0`处。这里我们创建一个指向该地址的指针 `binfo`，以便访问这些信息。
- **`char s[40], mcursor[256];`**: `s` 是一个用于格式化字符串的字符数组。`mcursor` 是一个256字节（16x16）的数组，用于存储鼠标指针的图像数据。
- **`init_screen(binfo->vram, binfo->scrnx, binfo->scrny);`**: 使用从`binfo`中获取的显存地址和屏幕分辨率来初始化屏幕背景，而不是使用固定的320x200。
- **`mx = ...; my = ...;`**: 计算屏幕中央的坐标，用于显示鼠标指针。`-28`是为了避开下方的任务栏。
- **`init_mouse_cursor8(mcursor, COL8_008484);`**: 初始化鼠标指针的图像数据。`mcursor` 数组会被填充，背景色(`bc`)设为桌面的颜色。
- **`putblock8_8(...)`**: 将 `mcursor` 中的鼠标图像数据绘制到屏幕中央。
- **`putfonts8_asc(...)`**: 在屏幕上不同的位置显示几行文字，通过控制坐标实现阴影效果。
- **`sprintf(s, "scrnx = %d", binfo->scrnx);`**: 使用 `sprintf` 函数（需要链接C标准库）将屏幕宽度格式化成一个字符串，并显示在屏幕上。

---

### 屏幕初始化函数 `init_screen`

```c
void init_screen(char *vram, int x, int y)
{
	boxfill8(vram, x, COL8_008484,  0,     0,      x -  1, y - 29);
	// ... (其他 boxfill8 调用)
	boxfill8(vram, x, COL8_FFFFFF, x -  3, y - 24, x -  3, y -  3);
	return;
}
```
- 这个函数使用`boxfill8`绘制了整个屏幕的背景和UI元素，如任务栏和按钮的边框。
- 它接收屏幕的宽度`x`和高度`y`作为参数，使得UI布局可以适应不同的屏幕分辨率。

---

### 字符与字符串绘制函数

#### `putfont8`
```c
void putfont8(char *vram, int xsize, int x, int y, char c, char *font)
{
	int i;
	char *p, d /* data */;
	for (i = 0; i < 16; i++) {
		p = vram + (y + i) * xsize + x;
		d = font[i];
		if ((d & 0x80) != 0) { p[0] = c; }
		// ... (检查其他位)
		if ((d & 0x01) != 0) { p[7] = c; }
	}
}
```
- 此函数用于在屏幕上绘制一个8x16像素的字符。
- `font`参数是一个指向16字节字符点阵数据的指针。每个字节代表一行像素，一个字节有8位，每一位对应一个像素的开关状态。
- 函数逐行扫描字体数据，如果某一位是1，就在对应位置`p[0]`到`p[7]`上绘制颜色`c`。

#### `putfonts8_asc`
```c
void putfonts8_asc(char *vram, int xsize, int x, int y, char c, unsigned char *s)
{
	extern char hankaku[4096];
	for (; *s != 0x00; s++) {
		putfont8(vram, xsize, x, y, c, hankaku + *s * 16);
		x += 8;
	}
}
```
- 该函数用于绘制一个以`\0`结尾的ASCII字符串。
- **`extern char hankaku[4096];`**: 声明一个外部的字符点阵数据数组，这个数组定义在别的文件中（这里是`hankaku.c`，由`objconv`工具从字体文件生成）。
- 它遍历字符串`s`中的每一个字符，计算该字符在`hankaku`数组中的位置（`*s * 16`），然后调用`putfont8`来绘制它。
- 每绘制一个字符后，将横坐标`x`增加8个像素。

---

### 鼠标指针函数

#### `init_mouse_cursor8`
```c
void init_mouse_cursor8(char *mouse, char bc)
{
	static char cursor[16][16] = {
		"**************..",
		"*OOOOOOOOOOO*...",
		// ... (光标形状)
		".............***"
	};
	// ... (循环转换)
}
```
- 这个函数用于准备一个16x16像素的鼠标指针图像。
- 它定义了一个静态的二维字符数组`cursor`来描述鼠标的形状。`*`代表黑色，`O`代表白色，`.`代表背景色（透明）。
- 函数遍历`cursor`数组，并将字符形状转换为实际的颜色索引值，存入`mouse`指向的内存中。

#### `putblock8_8`
```c
void putblock8_8(char *vram, int vxsize, int pxsize,
	int pysize, int px0, int py0, char *buf, int bxsize)
{
	int x, y;
	for (y = 0; y < pysize; y++) {
		for (x = 0; x < pxsize; x++) {
			vram[(py0 + y) * vxsize + (px0 + x)] = buf[y * bxsize + x];
		}
	}
}
```
- 这是一个通用的矩形数据块传输函数。它可以将一块内存（`buf`）中的数据，复制到显存（`vram`）的指定位置。
- `(pxsize, pysize)`是数据块的尺寸，`(px0, py0)`是绘制到屏幕上的左上角坐标。
- `vxsize`是屏幕的宽度，`bxsize`是源数据块每一行的字节数。
- `HariMain`中用它来将`init_mouse_cursor8`准备好的鼠标图像数据绘制到屏幕上。

---

### GDT与IDT初始化

这部分代码用于初始化全局描述符表（GDT）和中断描述符表（IDT），这是CPU进入保护模式后正常工作所必需的关键步骤。在当前的`HariMain`中，`init_gdtidt()`尚未被调用，但函数已经备好。

#### `init_gdtidt`
```c
void init_gdtidt(void)
{
	struct SEGMENT_DESCRIPTOR *gdt = (struct SEGMENT_DESCRIPTOR *) 0x00270000;
	struct GATE_DESCRIPTOR    *idt = (struct GATE_DESCRIPTOR    *) 0x0026f800;
	int i;

	/* GDT初始化 */
	for (i = 0; i < 8192; i++) {
		set_segmdesc(gdt + i, 0, 0, 0);
	}
	set_segmdesc(gdt + 1, 0xffffffff, 0x00000000, 0x4092);
	set_segmdesc(gdt + 2, 0x0007ffff, 0x00280000, 0x409a);
	load_gdtr(0xffff, 0x00270000);

	/* IDT初始化 */
	for (i = 0; i < 256; i++) {
		set_gatedesc(idt + i, 0, 0, 0);
	}
	load_idtr(0x7ff, 0x0026f800);
}
```
- **GDT初始化**:
  - 将GDT的所有8192个条目清零。
  - 设置`gdt[1]`为一个覆盖整个4GB内存空间的数据段。
  - 设置`gdt[2]`为一个512KB的段，作为我们操作系统的代码段，基地址为`0x00280000`。
  - `load_gdtr`是一个汇编函数，它告诉CPU GDT的位置和大小。
- **IDT初始化**:
  - 将IDT的所有256个条目清零（此时尚未注册任何中断处理函数）。
  - `load_idtr`是一个汇编函数，它告诉CPU IDT的位置和大小。

#### `set_segmdesc`
```c
void set_segmdesc(struct SEGMENT_DESCRIPTOR *sd, unsigned int limit, int base, int ar)
{
    // ... 函数体 ...
}
```
- 这个辅助函数用于方便地设置`SEGMENT_DESCRIPTOR`结构体的各个字段。它会将32位的`limit`（段限长）、`base`（基地址）和`ar`（访问权限）正确地拆分并填充到结构体的各个成员中。

#### `set_gatedesc`
```c
void set_gatedesc(struct GATE_DESCRIPTOR *gd, int offset, int selector, int ar)
{
    // ... 函数体 ...
}
```
- 类似的，这个辅助函数用于设置`GATE_DESCRIPTOR`结构体的字段，主要用于注册中断处理函数的地址（`offset`）和段选择子（`selector`）。

喵~
