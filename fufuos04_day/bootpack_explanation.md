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

