---
title: 8-bit的绝唱——GameBoy探索笔记
date: 2020-03-11 18:50:04
tags: "pwn"
cover: /img/IMG_20200311_173638_076.jpg
---
作为一个五十三分之九的任豚，在家赋闲就会想找点事做，正巧想研究一下古代时期的程序员是怎么耕作的，于是有了以下这些东西。  
# 

## #SM83指令集
这就是GameBoy用的板子了，硬件方面不熟，仅截取指令部分说说。  

### LD系列

#### LD r, r’

将寄存器r'中的数据读取到寄存器r中
#### LD r, n
将立即数n读取到寄存器r中
#### LD r, (HL)
将16位寄存器HL所指的地址中的数据读取到寄存器r中 
#### LD (HL), r
将寄存器r中的数据读取到16位寄存器HL所指的地址中
#### LD (HL), n
将立即数n读取到16位寄存器HL所指的地址中

#### LD A, (nn)

将绝对地址nn指向的数据读取到寄存器A中

#### LD (nn), A

将寄存器A中的数据读取到绝对地址nn指向的内存中

#### PUSH rr

#### POP rr

### 控制流系列

#### JP nn

#### JP cc, nn

#### JR e

#### JR cc, e

#### CALL nn

#### CALL cc, nn

#### RET

#### RET cc

#### RETI

剩下的不列了，有点多。看表好了

![](/img/image-20200311213525350.png)

# Game Boy芯片外围设备与特性
## Boot ROM
GB芯片有一个嵌入的Boot ROM，可用空间0x0000-0x00FF，这点内存打pwn随便就malloc一个，属实少的可怜。当系统离开重置状态时，CPU从0x0000开始执行。这个boot干了些什么呢？最重要的部分是打出Logo，你任还是你任。接着会检查是否有卡带插着。卡带代码会从0x0100开始映射，在执行前会将Boot映射的区域清空，以防止被读取出来，这大概就是原始时代的防破解方法了吧。  
然后Boot ROM还有蛮多版本的。DMG boot ROM是最常见的一个，有空去研究一下。还蛮好奇那个Logo怎么打出来的。

更新，这是反编译下来的结果

```
	LD SP,$fffe		; $0000  Setup Stack
	XOR A			; $0003  Zero the memory from $8000-$9FFF (VRAM)
	LD HL,$9fff		; $0004
Addr_0007:
	LD (HL-),A		; $0007
	BIT 7,H		    ; $0008
	JR NZ, Addr_0007; $000a

	LD HL,$ff26		; $000c  Setup Audio
	LD C,$11		; $000f
	LD A,$80		; $0011 
	LD (HL-),A		; $0013
	LD ($FF00+C),A	; $0014
	INC C			; $0015
	LD A,$f3		; $0016
	LD ($FF00+C),A	; $0018
	LD (HL-),A		; $0019
	LD A,$77		; $001a
	LD (HL),A		; $001c

	LD A,$fc		; $001d  Setup BG palette
	LD ($FF00+$47),A; $001f

	LD DE,$0104		; $0021  Convert and load logo data from cart into Video RAM
	LD HL,$8010		; $0024
Addr_0027:
	LD A,(DE)		; $0027
	CALL $0095		; $0028
	CALL $0096		; $002b
	INC DE		; $002e
	LD A,E		; $002f
	CP $34		; $0030
	JR NZ, Addr_0027	; $0032

	LD DE,$00d8		; $0034  Load 8 additional bytes into Video RAM
	LD B,$08		; $0037
Addr_0039:
	LD A,(DE)		; $0039
	INC DE		; $003a
	LD (HL+),A		; $003b
	INC HL		; $003c
	DEC B			; $003d
	JR NZ, Addr_0039	; $003e

	LD A,$19		; $0040  Setup background tilemap
	LD ($9910),A	; $0042
	LD HL,$992f		; $0045
Addr_0048:
	LD C,$0c		; $0048
Addr_004A:
	DEC A			; $004a
	JR Z, Addr_0055	; $004b
	LD (HL-),A		; $004d
	DEC C			; $004e
	JR NZ, Addr_004A	; $004f
	LD L,$0f		; $0051
	JR Addr_0048	; $0053

	; === Scroll logo on screen, and play logo sound===

Addr_0055:
	LD H,A		; $0055  Initialize scroll count, H=0
	LD A,$64		; $0056
	LD D,A		; $0058  set loop count, D=$64
	LD ($FF00+$42),A	; $0059  Set vertical scroll register
	LD A,$91		; $005b
	LD ($FF00+$40),A	; $005d  Turn on LCD, showing Background
	INC B			; $005f  Set B=1
Addr_0060:
	LD E,$02		; $0060
Addr_0062:
	LD C,$0c		; $0062
Addr_0064:
	LD A,($FF00+$44)	; $0064  wait for screen frame
	CP $90		; $0066
	JR NZ, Addr_0064	; $0068
	DEC C			; $006a
	JR NZ, Addr_0064	; $006b
	DEC E			; $006d
	JR NZ, Addr_0062	; $006e

	LD C,$13		; $0070
	INC H			; $0072  increment scroll count
	LD A,H		; $0073
	LD E,$83		; $0074
	CP $62		; $0076  $62 counts in, play sound #1
	JR Z, Addr_0080	; $0078
	LD E,$c1		; $007a
	CP $64		; $007c
	JR NZ, Addr_0086	; $007e  $64 counts in, play sound #2
Addr_0080:
	LD A,E		; $0080  play sound
	LD ($FF00+C),A	; $0081
	INC C			; $0082
	LD A,$87		; $0083
	LD ($FF00+C),A	; $0085
Addr_0086:
	LD A,($FF00+$42)	; $0086
	SUB B			; $0088
	LD ($FF00+$42),A	; $0089  scroll logo up if B=1
	DEC D			; $008b  
	JR NZ, Addr_0060	; $008c

	DEC B			; $008e  set B=0 first time
	JR NZ, Addr_00E0	; $008f    ... next time, cause jump to "Nintendo Logo check"

	LD D,$20		; $0091  use scrolling loop to pause
	JR Addr_0060	; $0093

	; ==== Graphic routine ====

	LD C,A		; $0095  "Double up" all the bits of the graphics data
	LD B,$04		; $0096     and store in Video RAM
Addr_0098:
	PUSH BC		; $0098
	RL C			; $0099
	RLA			; $009b
	POP BC		; $009c
	RL C			; $009d
	RLA			; $009f
	DEC B			; $00a0
	JR NZ, Addr_0098	; $00a1
	LD (HL+),A		; $00a3
	INC HL		; $00a4
	LD (HL+),A		; $00a5
	INC HL		; $00a6
	RET			; $00a7

Addr_00A8:
	;Nintendo Logo
	.DB $CE,$ED,$66,$66,$CC,$0D,$00,$0B,$03,$73,$00,$83,$00,$0C,$00,$0D 
	.DB $00,$08,$11,$1F,$88,$89,$00,$0E,$DC,$CC,$6E,$E6,$DD,$DD,$D9,$99 
	.DB $BB,$BB,$67,$63,$6E,$0E,$EC,$CC,$DD,$DC,$99,$9F,$BB,$B9,$33,$3E 

Addr_00D8:
	;More video data
	.DB $3C,$42,$B9,$A5,$B9,$A5,$42,$3C

	; ===== Nintendo logo comparison routine =====

Addr_00E0:	
	LD HL,$0104		; $00e0	; point HL to Nintendo logo in cart
	LD DE,$00a8		; $00e3	; point DE to Nintendo logo in DMG rom

Addr_00E6:
	LD A,(DE)		; $00e6
	INC DE		; $00e7
	CP (HL)		; $00e8	;compare logo data in cart to DMG rom
	JR NZ,$fe		; $00e9	;if not a match, lock up here
	INC HL		; $00eb
	LD A,L		; $00ec
	CP $34		; $00ed	;do this for $30 bytes
	JR NZ, Addr_00E6	; $00ef

	LD B,$19		; $00f1
	LD A,B		; $00f3
Addr_00F4:
	ADD (HL)		; $00f4
	INC HL		; $00f5
	DEC B			; $00f6
	JR NZ, Addr_00F4	; $00f7
	ADD (HL)		; $00f9
	JR NZ,$fe		; $00fa	; if $19 + bytes from $0134-$014D  don't add to $00
						;  ... lock up

	LD A,$01		; $00fc
	LD ($FF00+$50),A	; $00fe	;turn off DMG rom
```

0x100字节的代码就干了这么些事，挺紧凑的。那个卡带和里面的logo对比是要干什么？防伪吗？确实不太明白上个世纪都在用什么技术。

而且也没摸透到底是怎么做到的图像渲染。

## DMA
### Object Attribute Memory (OAM) DMA
好眼熟的东西，还是计组里学的。这是个高吞吐量的机制，能以1字节/CPU时钟周期读取数据而不影响CPU，甚至比依靠自带指令集写的memcpy更快。但是每次必定移动160字节而无法打断。
OAM DMA使用了一种简化版的内存解码策略，导致部分内存无法作为源地址。

## 图像处理单元（PPU）

### LCDC PPU控制寄存器

### LCDC PPU状态寄存器

###  SCY - Vertical scroll register

###  SCX - Horizontal scroll register

### LY - Scanline register

### LYC - Scanline compare register

以上都看不明白是什么，此前从未了解过gpu相关的东西，或许该找个机会学学图形学，也没有几行代码能看，先放着

# GameBoy游戏卡带

## Header

卡带有头，0x100-0x150这一部分。

主要是这几个部分组成

### 0100-0103 程序执行入口

```
nop    	;$0100
jp		;$0150
```

为啥要nop成了一个未解之谜。

### 0104-0133 任天堂的 LOGO

是我Nintendo哒！会与$00A8的logo做个比较。

### 0134-0143 标题

游戏名，中古游戏都不兴那种轻小说命名吗

TODO