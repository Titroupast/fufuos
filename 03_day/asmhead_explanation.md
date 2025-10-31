# asmhead.nas 逐行详解

`asmhead.nas` 是 Haribote OS 的头部汇编程序。它紧接着 `ipl10.nas`（Initial Program Loader）之后被加载和执行。其核心任务是完成从 CPU 的16位实模式到32位保护模式的切换，为后续 C 语言编写的操作系统内核 `bootpack.c` 准备好运行环境。

下面我们将对该文件进行详细的逐行解释。

---

### 1. 常量定义

```assembly
; haribote-os boot asm
; TAB=4

BOTPAK	EQU		0x00280000		; 加载bootpack
DSKCAC	EQU		0x00100000		; 磁盘缓存的位置
DSKCAC0	EQU		0x00008000		; 磁盘缓存的位置（实模式）
```

- `EQU` 是汇编语言中的伪指令，用于定义常量。
- **`BOTPAK EQU 0x00280000`**: 定义了 `bootpack`（操作系统内核）最终被加载到的目标内存地址。这是一个32位保护模式下的地址。
- **`DSKCAC EQU 0x00100000`**: 定义了用于缓存磁盘内容的内存地址。`DSKCAC` 是 "Disk Cache" 的缩写。这也是一个32位保护模式下的地址。
- **`DSKCAC0 EQU 0x00008000`**: 定义了在实模式下可以访问的磁盘缓存区。实模式下，CPU只能访问1MB以下的内存，所以需要一个临时的、低地址的缓存区。`ipl10.nas` 会把软盘上除了自身以外的数据先读到这里。

---

### 2. BOOT_INFO 相关信息

```assembly
; BOOT_INFO相关
CYLS	EQU		0x0ff0			; 引导扇区设置
LEDS	EQU		0x0ff1
VMODE	EQU		0x0ff2			; 关于颜色的信息
SCRNX	EQU		0x0ff4			; 分辨率X
SCRNY	EQU		0x0ff6			; 分辨率Y
VRAM	EQU		0x0ff8			; 图像缓冲区的起始地址
```

这部分定义了一块内存区域（`0x0ff0` 到 `0x0fff`），用于存储从引导程序（`ipl10.nas` 和 `asmhead.nas`）传递给操作系统内核（`bootpack.c`）的信息。C代码可以直接读取这些固定地址的数据，从而了解当前的硬件状态。

- **`CYLS EQU 0x0ff0`**: 存储软盘的柱面数，这个值由 `ipl10.nas` 写入。
- **`LEDS EQU 0x0ff1`**: 存储键盘上 LED 灯（如 CapsLock、NumLock）的状态。
- **`VMODE EQU 0x0ff2`**: 存储屏幕的颜色模式（如8位色、16位色）。
- **`SCRNX EQU 0x0ff4`**: 存储屏幕的水平分辨率（Screen X）。
- **`SCRNY EQU 0x0ff6`**: 存储屏幕的垂直分辨率（Screen Y）。
- **`VRAM EQU 0x0ff8`**: 存储显存（Video RAM）的起始地址。

---

### 3. 程序加载地址

```assembly
		ORG		0xc200			;  这个的程序要被装载的内存地址
```

- `ORG` (Origin) 是一个伪指令，它告诉汇编器，这段代码将被加载到内存的 `0xc200` 地址。`ipl10.nas` 会负责把 `asmhead.nas` 编译后的二进制码加载到这个位置。

---

### 4. 设置屏幕模式

```assembly
; 画面モードを設定

		MOV		AL,0x13			; VGA显卡，320x200x8bit
		MOV		AH,0x00
		INT		0x10
		MOV		BYTE [VMODE],8	; 屏幕的模式（参考C语言的引用）
		MOV		WORD [SCRNX],320
		MOV		WORD [SCRNY],200
		MOV		DWORD [VRAM],0x000a0000
```

这段代码通过 BIOS 中断来设置屏幕的显示模式。

- **`MOV AL,0x13`, `MOV AH,0x00`, `INT 0x10`**: 这是调用 BIOS 的 `0x10` 号中断。当 `AH=0x00` 时，这个中断用于设置显示模式。`AL=0x13` 代表设置屏幕为 **320x200 分辨率、8位色（256色）** 的图形模式。
- **`MOV BYTE [VMODE],8`**: 将颜色深度（8位）存入 `BOOT_INFO` 的 `VMODE` 位置。
- **`MOV WORD [SCRNX],320`**: 将屏幕宽度320存入 `SCRNX`。`WORD` 指明操作数是16位的。
- **`MOV WORD [SCRNY],200`**: 将屏幕高度200存入 `SCRNY`。
- **`MOV DWORD [VRAM],0x000a0000`**: 将该模式下显存的起始地址 `0xa0000` 存入 `VRAM`。`DWORD` 指明操作数是32位的。

---

### 5. 获取键盘LED状态

```assembly
; 通过BIOS获取指示灯状态

		MOV		AH,0x02
		INT		0x16 			; keyboard BIOS
		MOV		[LEDS],AL
```

- **`MOV AH,0x02`, `INT 0x16`**: 调用 BIOS 的 `0x16` 号键盘中断。当 `AH=0x02` 时，这个中断的功能是获取键盘 LED 灯的状态。
- **`MOV [LEDS],AL`**: 中断调用结束后，状态信息会保存在 `AL` 寄存器中。这行代码将 `AL` 的值存入 `BOOT_INFO` 的 `LEDS` 位置。

---

### 6. 屏蔽所有中断 (PIC)

```assembly
; 防止PIC接受所有中断
;	AT兼容机的规范、PIC初始化
;	然后之前在CLI不做任何事就挂起
;	PIC在同意后初始化

		MOV		AL,0xff
		OUT		0x21,AL
		NOP						; 不断执行OUT指令
		OUT		0xa1,AL

		CLI						; 进一步中断CPU
```

在进行模式切换这种底层、关键的操作时，必须防止被外部硬件中断（如键盘、鼠标、时钟）打扰。

- **PIC (Programmable Interrupt Controller)**: 可编程中断控制器，是管理硬件中断的芯片。PC上有两片PIC（主片和从片）。
- **`OUT 0x21,AL`**: `OUT` 指令用于向硬件端口写入数据。`0x21` 是主PIC的 **IMR（中断屏蔽寄存器）** 端口。向其写入 `0xff` (二进制 `11111111`) 表示屏蔽所有8个中断。
- **`OUT 0xa1,AL`**: `0xa1` 是从PIC的 IMR 端口，同样写入 `0xff` 来屏蔽其管理的所有中断。
- **`NOP`**: (No Operation) 空操作指令。在某些旧CPU上，连续的 `OUT` 指令可能会执行过快，插入一个 `NOP` 可以确保硬件有足够的时间响应。
- **`CLI`**: (Clear Interrupt Flag) 这是CPU指令，用于清除CPU内部的中断标志位（IF）。即使PIC放行了中断，`CLI` 也会让CPU忽略它。这提供了一个双重保险。

---

### 7. 开启A20 Gate

```assembly
; 让CPU支持1M以上内存、设置A20GATE

		CALL	waitkbdout
		MOV		AL,0xd1
		OUT		0x64,AL
		CALL	waitkbdout
		MOV		AL,0xdf			; enable A20
		OUT		0x60,AL
		CALL	waitkbdout
```

这是历史遗留问题。为了兼容古老的8086 CPU，早期PC的地址线 A20 是默认关闭的。这导致内存地址超过1MB时会发生回卷（wrap-around）。要访问1MB以上的内存（进入保护模式的必要条件），必须手动打开 A20 Gate。

- **`CALL waitkbdout`**: 调用一个等待函数，确保键盘控制器（8042芯片）处于可接受命令的状态。
- **`OUT 0x64,AL` with `AL=0xd1`**: `0x64` 是键盘控制器的状态/命令端口。发送 `0xd1` 命令表示“准备向键盘控制器输出端口（`0x60`）写入数据”。
- **`OUT 0x60,AL` with `AL=0xdf`**: `0x60` 是键盘控制器的数据端口。发送 `0xdf` 命令，这个命令的特定位会控制 A20 Gate 的开关，`0xdf` 的效果是 **开启A20 Gate**。

---

### 8. 切换到保护模式

这是整个文件最核心的部分。

```assembly
[INSTRSET "i486p"]				; 说明使用486指令

		LGDT	[GDTR0]			; 设置临时GDT
		MOV		EAX,CR0
		AND		EAX,0x7fffffff	; 使用bit31（禁用分页）
		OR		EAX,0x00000001	; bit0到1转换（保护模式过渡）
		MOV		CR0,EAX
		JMP		pipelineflush
pipelineflush:
		MOV		AX,1*8			;  写32bit的段
		MOV		DS,AX
		MOV		ES,AX
		MOV		FS,AX
		MOV		GS,AX
		MOV		SS,AX
```

- **`[INSTRSET "i486p"]`**: 告诉编译器 NASK，从这里开始可以使用486处理器的指令。
- **`LGDT [GDTR0]`**: (Load Global Descriptor Table) 加载GDT。GDT是保护模式下内存分段管理的核心。`GDTR0` 是一个内存地址，它存放了GDT的基地址和大小（界限）。这条指令把GDT的信息加载到CPU的 `GDTR` 寄存器中。
- **`MOV EAX,CR0`**: `CR0` 是CPU的一个控制寄存器。它的第0位（PE, Protection Enable）控制着CPU是处于实模式还是保护模式。
- **`OR EAX,0x00000001`**: 将 `CR0` 值的第0位置为1。
- **`MOV CR0,EAX`**: 将修改后的值写回 `CR0` 寄存器。**执行完这一句，CPU就正式从实模式切换到了32位保护模式！**
- **`JMP pipelineflush`**: 这是一条非常关键的跳转指令。CPU为了提高效率有指令预取流水线。在执行 `MOV CR0,EAX` 时，流水线里可能已经预取了下一条实模式的指令。切换到保护模式后，这些指令的解释方式完全不同了，必须清空流水线。一个长跳转（`JMP`）可以有效地刷新流水线。
- **`pipelineflush:`**: 跳转的目标标签。
- **`MOV AX,1*8`**: 在保护模式下，段寄存器（如`DS`, `SS`）存放的不再是段地址，而是 **段选择子(Selector)**。`1*8` 的结果是 `8`。`1` 是指GDT中的第1个段描述符（第0个是空描述符），`*8` 是因为每个描述符占8个字节。所以 `1*8` 就是定位到GDT中我们定义的第一个有效段。这个段在 `GDT0` 中被定义为一个可读写的32位数据段。
- **`MOV DS,AX` ... `MOV SS,AX`**: 将所有的数据段、附加段和堆栈段寄存器都设置为指向这个新的32位段。从此，程序就可以在32位的内存空间中运行了。

---

### 9. 移动内核和磁盘数据

进入保护模式后，需要将内核代码和之前加载的磁盘数据从临时位置移动到最终的目标位置。

#### 移动 `bootpack` (内核)
```assembly
; bootpack传递
		MOV		ESI,bootpack	; 源
		MOV		EDI,BOTPAK		; 目标
		MOV		ECX,512*1024/4
		CALL	memcpy
```
- **`MOV ESI, bootpack`**: `ESI` 是源地址寄存器。`bootpack` 是一个标签，指向 `asmhead.nas` 后面附加的 `bootpack.hrb` 文件的起始位置（由 `Makefile` 和链接过程保证）。
- **`MOV EDI, BOTPAK`**: `EDI` 是目标地址寄存器，设置为我们之前定义的常量 `0x00280000`。
- **`MOV ECX, 512*1024/4`**: `ECX` 是计数器。这里要复制512KB的数据。因为 `memcpy` 函数每次复制4个字节（一个 `DWORD`），所以总数要除以4。
- **`CALL memcpy`**: 调用内存复制函数。

#### 移动磁盘数据
```assembly
; 从引导区开始
		MOV		ESI,0x7c00		; 源
		MOV		EDI,DSKCAC		; 目标
		MOV		ECX,512/4
		CALL	memcpy
; 剩余的全部
		MOV		ESI,DSKCAC0+512	; 源
		MOV		EDI,DSKCAC+512	; 目标
		MOV		ECX,0
		MOV		CL,BYTE [CYLS]
		IMUL	ECX,512*18*2/4	; 除以4得到字节数
		SUB		ECX,512/4		; IPL偏移量
		CALL	memcpy
```
这部分代码将 `ipl10.nas` 读取到内存 `0x8000` 处（即 `DSKCAC0`）的磁盘数据，以及 `ipl10.nas` 自身（位于 `0x7c00`），一起复制到保护模式下的磁盘缓存区 `DSKCAC` (`0x00100000`)。
- **第一段 `memcpy`**: 复制 `ipl10.nas` 自身（512字节）到 `DSKCAC` 的开头。
- **第二段 `memcpy`**: 计算需要复制的剩余扇区大小，然后从 `DSKCAC0+512` 复制到 `DSKCAC+512`。
  - `MOV CL,BYTE [CYLS]`: 从 `BOOT_INFO` 中读取柱面数。
  - `IMUL ECX,512*18*2/4`: 计算总字节数。软盘每柱面有18个扇区，2个磁头，每个扇区512字节。最后除以4是因为 `memcpy` 每次复制4字节。
  - `SUB ECX,512/4`: 减去已经复制的 `ipl10.nas` 的大小。

---

### 10. 启动内核 `bootpack`

```assembly
; bootpack启动

		MOV		EBX,BOTPAK
		MOV		ECX,[EBX+16]
		ADD		ECX,3			; ECX += 3;
		SHR		ECX,2			; ECX /= 4;
		JZ		skip			; 传输完成
		MOV		ESI,[EBX+20]	; 源
		ADD		ESI,EBX
		MOV		EDI,[EBX+12]	; 目标
		CALL	memcpy
skip:
		MOV		ESP,[EBX+12]	; 堆栈的初始化
		JMP		DWORD 2*8:0x0000001b
```
这段代码是为 C 内核准备运行环境并跳转执行。`bootpack.hrb` 文件有一个头部，包含了内核运行所需的信息。

- **`MOV EBX, BOTPAK`**: `EBX` 指向 `bootpack` 的基地址。
- **`MOV ECX,[EBX+16]`**: 从 `bootpack` 头部的16字节偏移处读取数据段的大小。
- **`ADD ECX,3` / `SHR ECX,2`**: 这是一个向上取整的除以4的运算，计算需要复制多少个 `DWORD`。
- **`JZ skip`**: 如果大小为0，则跳过复制。
- **`MOV ESI,[EBX+20]` / `MOV EDI,[EBX+12]`**: 从头部读取数据段的源地址和目标地址。
- **`CALL memcpy`**: 将内核的数据段复制到指定位置。
- **`skip:`**: 标签。
- **`MOV ESP,[EBX+12]`**: 从 `bootpack` 头部的12字节偏移处读取堆栈顶部的地址，并设置 `ESP` (堆栈指针寄存器)。
- **`JMP DWORD 2*8:0x0000001b`**: 这是最后的 **远跳转** 指令，将CPU的控制权交给 `bootpack` 内核。
  - **`2*8`**: `16`。这是代码段的选择子，指向 GDT 中的第2个描述符（在 `GDT0` 中被定义为可执行的代码段）。
  - **`0x0000001b`**: 这是 `bootpack` 内核的入口点地址（即 `HariMain` 函数的地址）。这个地址是在编译链接时确定的。

---

### 11. 工具函数

#### `waitkbdout`
```assembly
waitkbdout:
		IN		 AL,0x64
		AND		 AL,0x02
		JNZ		waitkbdout		; AND结果不为0跳转到waitkbdout
		RET
```
- **`IN AL,0x64`**: 从键盘控制器（`0x64` 端口）读取状态字节到 `AL`。
- **`AND AL,0x02`**: 检查状态字节的第1位。如果这一位是1，表示控制器正忙，无法接收新命令。
- **`JNZ waitkbdout`**: (Jump if Not Zero) 如果 `AND` 运算结果不为0（即控制器忙），则循环跳转回 `waitkbdout`，继续等待。
- **`RET`**: 当控制器空闲时（第1位为0），`AND` 结果为0，循环结束，函数返回。

#### `memcpy`
```assembly
memcpy:
		MOV		EAX,[ESI]
		ADD		ESI,4
		MOV		[EDI],EAX
		ADD		EDI,4
		SUB		ECX,1
		JNZ		memcpy			; 运算结果不为0跳转到memcpy
		RET
```
一个简单的32位内存复制函数。
- **`MOV EAX,[ESI]`**: 从源地址 `ESI` 读取4个字节到 `EAX`。
- **`ADD ESI,4`**: 源地址指针向后移动4字节。
- **`MOV [EDI],EAX`**: 将 `EAX` 的内容写入目标地址 `EDI`。
- **`ADD EDI,4`**: 目标地址指针向后移动4字节。
- **`SUB ECX,1`**: 计数器减1。
- **`JNZ memcpy`**: 如果计数器不为0，则循环。
- **`RET`**: 复制完成，函数返回。

---

### 12. GDT (全局描述符表) 定义

```assembly
		ALIGNB	16
GDT0:
		RESB	8				; 初始值
		DW		0xffff,0x0000,0x9200,0x00cf	; 写32bit位段寄存器
		DW		0xffff,0x0000,0x9a28,0x0047	; 可执行的文件的32bit寄存器（bootpack用）

		DW		0
GDTR0:
		DW		8*3-1
		DD		GDT0
```

- **`ALIGNB 16`**: 确保下面的数据结构按16字节对齐。
- **`GDT0:`**: GDT 的起始地址标签。
- **`RESB 8`**: (Reserve Byte) 保留8个字节。GDT的第0个描述符必须是全0的空描述符，这是CPU规范。
- **`DW 0xffff,0x0000,0x9200,0x00cf`**: 这是 **第1个段描述符**（选择子为8）。它定义了一个覆盖整个4GB内存空间（段限长`0xfffff`，G位为1）的可读写数据段。
  - `段限长`: `0xffff` (低16位) + `0x0f` (来自 `0x00cf` 的低4位) -> `0xfffff`
  - `段基址`: `0x0000` (低16位) + `0x00` (来自 `0x9200` 的高8位) + `0x00` (来自 `0x00cf` 的高8位) -> `0x00000000`
  - `访问权限`: `0x92` -> Present=1, DPL=0, Type=数据, 可写
  - `标志位`: `c` (来自`0x00cf`) -> G=1 (4KB页), DB=1 (32位)
- **`DW 0xffff,0x0000,0x9a28,0x0047`**: 这是 **第2个段描述符**（选择子为16）。它定义了一个段基址为 `0x00280000`（`bootpack`的加载地址）、大小为512KB的可执行代码段。
- **`GDTR0:`**: GDT 寄存器需要加载的数据。
  - **`DW 8*3-1`**: GDT的界限（大小减1）。这里定义了3个描述符，每个8字节，所以总大小是 `24` 字节，界限就是 `23`。
  - **`DD GDT0`**: GDT的32位线性基地址。

---

### 13. bootpack 标签

```assembly
		ALIGNB	16
bootpack:
```

- `bootpack:` 只是一个标签，标记了 `bootpack.hrb` 文件内容附加在此处的起始位置。实际的文件内容是在链接阶段由 `make.bat` 中的 `copy` 命令拼接进来的。
