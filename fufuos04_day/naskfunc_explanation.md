# naskfunc.nas 代码逐行解释

这是一个用 NASM 汇编语言编写的文件，主要包含了一些用于和硬件进行底层交互的函数，这些函数通常会在C语言中被调用，以实现操作系统的一些核心功能，比如中断处理和端口I/O。

---

### 文件头指令

```assembly
; naskfunc
; TAB=4

[FORMAT "WCOFF"]				; 制作目标文件的模式	
[INSTRSET "i486p"]				; 使用到486为止的指令
[BITS 32]						; 3制作32位模式用的机器语言
[FILE "naskfunc.nas"]			; 文件名
```

- **`; naskfunc` 和 `; TAB=4`**: 这是注释。第一行是文件名，第二行可能是编辑器的设置，表示一个TAB键等于4个空格。
- **`[FORMAT "WCOFF"]`**: 指定输出的目标文件格式为 `WCOFF` (Windows Common Object File Format)。这是一种被 Windows 平台上的链接器所支持的格式。
- **`[INSTRSET "i486p"]`**: 指定汇编器使用 486 处理器的指令集。这意味着代码可以在 486 以及更高版本的CPU上运行。
- **`[BITS 32]`**: 指定生成的机器码是 32 位的。这是在保护模式下运行的操作系统所必需的。
- **`[FILE "naskfunc.nas"]`**: 这是一个指示，告诉汇编器源文件的名称是什么，有助于调试。

---

### 全局符号声明

```assembly
		GLOBAL	_io_hlt, _io_cli, _io_sti, _io_stihlt
		GLOBAL	_io_in8,  _io_in16,  _io_in32
		GLOBAL	_io_out8, _io_out16, _io_out32
		GLOBAL	_io_load_eflags, _io_store_eflags
```

- **`GLOBAL`**: 这个指令将标签（在这里是函数名）声明为全局的。这样一来，当这个汇编文件被编译成目标文件后，链接器就可以识别这些标签，并让其他文件（比如C语言代码）调用它们。
- `_io_hlt`: 让CPU暂停执行（Halt）。
- `_io_cli`: 关闭中断（Clear Interrupt Flag）。
- `_io_sti`: 开启中断（Set Interrupt Flag）。
- `_io_stihlt`: 先开启中断，然后让CPU暂停。
- `_io_in8`, `_io_in16`, `_io_in32`: 从硬件端口分别读取8位、16位、32位数据。
- `_io_out8`, `_io_out16`, `_io_out32`: 向硬件端口分别写入8位、16位、32位数据。
- `_io_load_eflags`: 读取EFLAGS寄存器的值。
- `_io_store_eflags`: 向EFLAGS寄存器写入值。

---

### 代码段

```assembly
[SECTION .text]
```

- **`[SECTION .text]`**: 声明接下来的内容属于代码段（`.text` section）。在目标文件中，代码通常被存放在这个段里。

---

### 函数实现

#### _io_hlt

```assembly
_io_hlt:	; void io_hlt(void);
		HLT
		RET
```
- **`_io_hlt:`**: 函数标签，函数的入口点。
- **`HLT`**: `Halt`指令，使CPU停止执行指令，直到接收到一个外部中断。这是一种节能的方式。
- **`RET`**: `Return`指令，从函数返回。

#### _io_cli

```assembly
_io_cli:	; void io_cli(void);
		CLI
		RET
```
- **`CLI`**: `Clear Interrupt Flag`指令，将EFLAGS寄存器中的中断标志位（IF）清零，从而禁用CPU对可屏蔽硬件中断的响应。这在进行一些关键操作（如修改中断处理程序）时非常重要。
- **`RET`**: 从函数返回。

#### _io_sti

```assembly
_io_sti:	; void io_sti(void);
		STI
		RET
```
- **`STI`**: `Set Interrupt Flag`指令，将EFLAGS寄存器中的中断标志位（IF）置1，允许CPU响应可屏蔽硬件中断。
- **`RET`**: 从函数返回。

#### _io_stihlt

```assembly
_io_stihlt:	; void io_stihlt(void);
		STI
		HLT
		RET
```
- **`STI`**: 首先允许中断。
- **`HLT`**: 然后让CPU暂停。这个组合很常用，因为它保证在CPU暂停前中断是开启的，这样CPU可以被中断唤醒。
- **`RET`**: 从函数返回。

#### _io_in8

```assembly
_io_in8:	; int io_in8(int port);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,0
		IN		AL,DX
		RET
```
- C语言接口为 `int io_in8(int port)`。
- **`MOV EDX,[ESP+4]`**: 从栈中获取第一个参数`port`，并将其值存入`EDX`寄存器。函数参数通过栈传递，`[ESP+4]`指向第一个参数。
- **`MOV EAX,0`**: 将`EAX`寄存器清零，以确保高位是干净的。
- **`IN AL,DX`**: 从`DX`寄存器指定的I/O端口读取一个字节（8位）的数据，并存放到`AL`寄存器（`EAX`的低8位）。
- **`RET`**: 返回。函数的返回值（读取到的数据）存放在`EAX`中。

#### _io_in16 / _io_in32

```assembly
_io_in16:	; int io_in16(int port);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,0
		IN		AX,DX
		RET

_io_in32:	; int io_in32(int port);
		MOV		EDX,[ESP+4]		; port
		IN		EAX,DX
		RET
```
- 这两个函数与`_io_in8`类似，只是读取的数据宽度不同。
- `_io_in16`使用`IN AX,DX`读取16位数据到`AX`寄存器。
- `_io_in32`使用`IN EAX,DX`读取32位数据到`EAX`寄存器。

#### _io_out8

```assembly
_io_out8:	; void io_out8(int port, int data);
		MOV		EDX,[ESP+4]		; port
		MOV		AL,[ESP+8]		; data
		OUT		DX,AL
		RET
```
- C语言接口为 `void io_out8(int port, int data)`。
- **`MOV EDX,[ESP+4]`**: 获取第一个参数`port`到`EDX`。
- **`MOV AL,[ESP+8]`**: 获取第二个参数`data`到`AL`寄存器。`[ESP+8]`指向第二个参数。
- **`OUT DX,AL`**: 将`AL`寄存器中的8位数据写入到`DX`寄存器指定的I/O端口。
- **`RET`**: 返回。

#### _io_out16 / _io_out32

```assembly
_io_out16:	; void io_out16(int port, int data);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,[ESP+8]		; data
		OUT		DX,AX
		RET

_io_out32:	; void io_out32(int port, int data);
		MOV		EDX,[ESP+4]		; port
		MOV		EAX,[ESP+8]		; data
		OUT		DX,EAX
		RET
```
- 这两个函数与`_io_out8`类似，只是写入的数据宽度不同。
- `_io_out16`使用`OUT DX,AX`将`AX`中的16位数据写出。
- `_io_out32`使用`OUT DX,EAX`将`EAX`中的32位数据写出。

#### _io_load_eflags

```assembly
_io_load_eflags:	; int io_load_eflags(void);
		PUSHFD		; PUSH EFLAGS 
		POP		EAX
		RET
```
- **`PUSHFD`**: `Push EFLAGS Double-word`指令，将32位的EFLAGS寄存器的内容压入栈中。
- **`POP EAX`**: 将刚刚压入栈的EFLAGS值弹出到`EAX`寄存器。
- **`RET`**: 返回。此时EFLAGS的值作为函数的返回值存放在`EAX`中。

#### _io_store_eflags

```assembly
_io_store_eflags:	; void io_store_eflags(int eflags);
		MOV		EAX,[ESP+4]
		PUSH	EAX
		POPFD		; POP EFLAGS 
		RET
```
- C语言接口为 `void io_store_eflags(int eflags)`。
- **`MOV EAX,[ESP+4]`**: 从栈中获取参数`eflags`的值到`EAX`寄存器。
- **`PUSH EAX`**: 将`EAX`中的值（即要设置的eflags值）压入栈。
- **`POPFD`**: `Pop EFLAGS Double-word`指令，将栈顶的值弹出到EFLAGS寄存器，从而更新EFLAGS的状态。
- **`RET`**: 返回。
