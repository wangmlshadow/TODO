# Linux 0.11核心代码
> repo: https://github.com/sunym1993/flash-linux0.11-talk
> repo：https://github.com/mengchaobbbigrui/Linux-0.11code
## 最开始的两行代码
首先了解一下寄存器：
寄存器	原文	             解释	        说明
AX	    accumulator	        累加寄存器   	通常用来执行加法，函数调用的返回值一般也放在这里面
CX	    counter    	        计数寄存器	    通常用来作为计数器，比如for循环
DX	    data    	        数据寄存器	    数据存取
BX	    base    	        基址寄存器	    读写I/O端口时，edx用来存放端口号
SP	    stack pointer	    栈指针寄存器	栈顶指针，指向栈的顶部
BP	    base pointer	    基址指针寄存器	栈底指针，指向栈的底部，通常用ebp+偏移量的形式来定位函数存放在栈中的局部变量
SI	    source index	    源变址寄存器	字符串操作时，用于存放数据源的地址
DI	    destination index	目标变址寄存器	字符串操作时，用于存放目的地址的，和esi两个经常搭配一起使用，执行字符串的复制等操作
ES	    extra segment	    附加段寄存器
CS	    code segment	    代码段寄存器
SS	    stack segment	    栈段寄存器
DS	    data segment	    数据段寄存器
FS	    segment part 2  	无名寄存器
GS	    segment part 3	    无名寄存器

> git/Linux-0.11code/boot/bootsect.s
> 背景知识：
> - 当你按下开机键的那一刻，在主板上提前写死的固件程序 BIOS 会将硬盘中启动区的 512 字节的数据，原封不动复制到内存中的 **0x7c00** 这个位置，并跳转到那个位置进行执行。
> - ds 是一个 16 位的段寄存器，具体表示数据段寄存器，在内存寻址时充当段基址的作用。啥意思呢？就是当我们之后用汇编语言写一个内存地址时，实际上仅仅是写了偏移地址。
> - 这个 ds 被赋值为了 0x07c0，由于 x86 为了让自己在 16 位这个实模式下能访问到 20 位的地址线这个历史因素（不了解这个的就先别纠结为啥了），所以段基址要先左移四位。那 0x07c0 左移四位就是 0x7c00，那这就刚好和这段代码被 BIOS 加载到的内存地址 0x7c00 一样了。
```c
SETUPLEN = 4				! nr of setup-sectors
BOOTSEG  = 0x07c0			! original address of boot-sector
INITSEG  = 0x9000			! we move boot here - out of the way
SETUPSEG = 0x9020			! setup starts here
SYSSEG   = 0x1000			! system loaded at 0x10000 (65536).
ENDSEG   = SYSSEG + SYSSIZE		! where to stop loading

! ROOT_DEV:	0x000 - same type of floppy as boot.
!		0x301 - first partition on first drive etc
ROOT_DEV = 0x306

entry start
start:
	mov	ax,#BOOTSEG
	mov	ds,ax
```
**最开始的两行代码（mov）做的事其实是根据cpu的规定，设定内存寻址时的段基址 0x7c00**
## 自己给自己挪个地
首先接着上述继续看代码：
```c
start:
	mov	ax,#BOOTSEG
	mov	ds,ax
	mov	ax,#INITSEG
	mov	es,ax
	mov	cx,#256
	sub	si,si
	sub	di,di
	rep
	movw
	jmpi	go,INITSEG
```
> 这些指令的意思：
> - 如果 sub 后面的两个寄存器一模一样，就相当于把这个寄存器里的值清零
> - rep 表示重复执行后面的指令
> - movw 表示复制一个字（word 16位）
> - rep movw 不断重复地复制一个字
> - 重复复制 cx 次，也就是256次
> 总的来说就是将 ds:si 位置开始的512字节的数据复制到 es:di 处
> 将内存地址 0x7c00 处开始往后的 512 字节的数据，原封不动复制到 0x90000 处
> - jmpi 是一个段间跳转指令，表示跳转到 0x9000:go 处执行
> - 段基址 : 偏移地址 这种格式的内存地址要如何计算吧？段基址仍然要先左移四位，因此结论就是跳转到 0x90000 + go 这个内存地址处执行
> - go 就是一个标签，最终编译成机器码的时候会被翻译成一个值，这个值就是 go 这个标签在文件内的偏移地址。这个偏移地址再加上 0x90000，就刚好是 go 标签后面那段代码 mov ax,cs 此时所在的内存地址了。
>
**所以到此为止，其实就是一段 512 字节的代码和数据，从硬盘的启动区先是被移动到了内存 0x7c00 处，然后又立刻被移动到 0x90000 处，并且跳转到此处往后再稍稍偏移 go 这个标签所代表的偏移地址处，也就是 mov ax,cs 这行指令的位置。**
## 做好最最基础的准备工作
```c
go:	mov	ax,cs
	mov	ds,ax
	mov	es,ax
! put stack at 0x9ff00.
	mov	ss,ax
	mov	sp,#0xFF00		! arbitrary value >>512
```
> - cs 寄存器表示代码段寄存器，CPU 当前正在执行的代码在内存中的位置，就是由 cs:ip 这组寄存器配合指向的，其中 cs 是基址，ip 是偏移地址
> - 由于之前执行过一个段间跳转指令,所以现在 cs 寄存器里的值就是 0x9000，ip 寄存器里的值是 go 这个标签的偏移地址。那这三个 mov 指令就分别给 ds、es 和 ss 寄存器赋值为了 0x9000
> - 所以现在 cs 寄存器里的值就是 0x9000，ip 寄存器里的值是 go 这个标签的偏移地址。那这三个 mov 指令就分别给 ds、es 和 ss 寄存器赋值为了 0x9000
> - es 是扩展段寄存器，仅仅是个扩展
> - ss 为栈段寄存器，后面要配合栈基址寄存器 sp 来表示此时的栈顶地址。而此时 sp 寄存器被赋值为了 0xFF00 了，所以目前的栈顶地址就是 ss:sp 所指向的地址 0x9FF00 处。
> 
**这一部分其实就是把代码段寄存器 cs，数据段寄存器 ds，栈段寄存器 ss 和栈基址寄存器 sp 分别设置好了值，方便后续使用。
其实操作系统在做的事情，就是给如何访问代码，如何访问数据，如何访问栈进行了一下内存的初步规划。其中访问代码和访问数据的规划方式就是设置了一个基址而已，访问栈就是把栈顶指针指向了一个远离代码位置的地方而已。**
## 把自己在硬盘里的其他部分也放到内存来
```c
load_setup:
	mov	dx,#0x0000		! drive 0, head 0
	mov	cx,#0x0002		! sector 2, track 0
	mov	bx,#0x0200		! address = 512, in INITSEG
	mov	ax,#0x0200+SETUPLEN	! service 2, nr of sectors
	int	0x13			! read it
	jnc	ok_load_setup		! ok - continue
	mov	dx,#0x0000
	mov	ax,#0x0000		! reset the diskette
	int	0x13
	j	load_setup

ok_load_setup:

! Get disk drive parameters, specifically nr of sectors/track

	mov	dl,#0x00
	mov	ax,#0x0800		! AH=8 is get drive parameters
	int	0x13
	mov	ch,#0x00
	seg cs
	mov	sectors,cx
	mov	ax,#INITSEG
	mov	es,ax

! Print some inane message

	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#24
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg1
	mov	ax,#0x1301		! write string, move cursor
	int	0x10

! ok, we've written the message, now
! we want to load the system (at 0x10000)

	mov	ax,#SYSSEG
	mov	es,ax		! segment of 0x010000
	call	read_it
	call	kill_motor

! After that we check which root-device to use. If the device is
! defined (!= 0), nothing is done and the given device is used.
! Otherwise, either /dev/PS0 (2,28) or /dev/at0 (2,8), depending
! on the number of sectors that the BIOS reports currently.

	seg cs
	mov	ax,root_dev
	cmp	ax,#0
	jne	root_defined
	seg cs
	mov	bx,sectors
	mov	ax,#0x0208		! /dev/ps0 - 1.2Mb
	cmp	bx,#15
	je	root_defined
	mov	ax,#0x021c		! /dev/PS0 - 1.44Mb
	cmp	bx,#18
	je	root_defined
undef_root:
	jmp undef_root
root_defined:
	seg cs
	mov	root_dev,ax

! after that (everyting loaded), we jump to
! the setup-routine loaded directly after
! the bootblock:

	jmpi	0,SETUPSEG
```
> - int 0x13 表示发起 0x13 号中断，这条指令上面给 dx、cx、bx、ax 赋值都是作为这个中断程序的参数。CPU 会通过这个中断号，去寻找对应的中断处理程序的入口地址，并跳转过去执行，逻辑上就相当于执行了一个函数。而 0x13 号中断的处理程序是 BIOS 提前给我们写好的，是读取磁盘的相关功能的函数。
> - 这段代码的作用就是将硬盘的第 2 个扇区开始，把数据加载到内存 0x90200 处，共加载 4 个扇区
> - 如果复制成功，就跳转到 ok_load_setup 这个标签，如果失败，则会不断重复执行这段代码，也就是重试。
> - 这段代码的作用就是把从硬盘第 6 个扇区开始往后的 240 个扇区，加载到内存 0x10000 处

**数据加载完毕后，继续执行**
> - jmpi 0,0x9020（jmpi	0,SETUPSEG），跳转到 0x90200 处，就是硬盘第二个扇区开始处的内容。第二个扇区的最开始处，那也就是 setup.s 文件的第一行代码。

## 进入保护模式前最后一次折腾内存
```c
INITSEG  = 0x9000	! we move boot here - out of the way
SYSSEG   = 0x1000	! system loaded at 0x10000 (65536).
SETUPSEG = 0x9020	! this is the current segment

.globl begtext, begdata, begbss, endtext, enddata, endbss
.text
begtext:
.data
begdata:
.bss
begbss:
.text

entry start
start:

! ok, the read went well so we get current cursor position and save it for
! posterity.

	mov	ax,#INITSEG	! this is done in bootsect already, but...
	mov	ds,ax
	mov	ah,#0x03	! read cursor pos
	xor	bh,bh
	int	0x10		! save it in known place, con_init fetches
	mov	[0],dx		! it from 0x90000.

! Get memory size (extended mem, kB)

	mov	ah,#0x88
	int	0x15
	mov	[2],ax

! Get video-card data:

	mov	ah,#0x0f
	int	0x10
	mov	[4],bx		! bh = display page
	mov	[6],ax		! al = video mode, ah = window width

! check for EGA/VGA and some config parameters

	mov	ah,#0x12
	mov	bl,#0x10
	int	0x10
	mov	[8],ax
	mov	[10],bx
	mov	[12],cx

! Get hd0 data

	mov	ax,#0x0000
	mov	ds,ax
	lds	si,[4*0x41]
	mov	ax,#INITSEG
	mov	es,ax
	mov	di,#0x0080
	mov	cx,#0x10
	rep
	movsb

! Get hd1 data

	mov	ax,#0x0000
	mov	ds,ax
	lds	si,[4*0x46]
	mov	ax,#INITSEG
	mov	es,ax
	mov	di,#0x0090
	mov	cx,#0x10
	rep
	movsb
    
! Check that there IS a hd1 :-)

	mov	ax,#0x01500
	mov	dl,#0x81
	int	0x13
	jc	no_disk1
	cmp	ah,#3
	je	is_disk1
no_disk1:
	mov	ax,#INITSEG
	mov	es,ax
	mov	di,#0x0090
	mov	cx,#0x10
	mov	ax,#0x00
	rep
	stosb
is_disk1:

! now we want to move to protected mode ...

	cli			! no interrupts allowed !

! first we move the system to it's rightful place

	mov	ax,#0x0000
	cld			! 'direction'=0, movs moves forward
do_move:
	mov	es,ax		! destination segment
	add	ax,#0x1000
	cmp	ax,#0x9000
	jz	end_move
	mov	ds,ax		! source segment
	sub	di,di
	sub	si,si
	mov 	cx,#0x8000
	rep
	movsw
	jmp	do_move

! then we load the segment descriptors

end_move:
```
> - int 0x10 是触发 BIOS 提供的显示服务中断处理程序，而 ah 寄存器被赋值为 0x03 表示显示服务里具体的读取光标位置功能。
> - int 0x10 中断程序执行完毕并返回时，dx 寄存器里的值表示光标的位置，具体说来其高八位 dh 存储了行号，低八位 dl 存储了列号。
> - mov [0],dx 就是把这个光标位置存储在 [0] 这个内存地址处。注意，前面我们说过，这个内存地址仅仅是偏移地址，还需要加上 ds 这个寄存器里存储的段基址，最终的内存地址是在 0x90000 处，这里存放着光标的位置，以便之后在初始化控制台的时候用到。
> - 再接下来的几行代码，都是和刚刚一样的逻辑，调用一个 BIOS 中断获取点什么信息，然后存储在内存中某个位置
> - cli，表示关闭中断的意思
> - rep movsw 把内存地址 0x10000 处开始往后一直到 0x90000 的内容，统统复制到内存的最开始的 0 位置

**此时的内存分布：**
1. 栈顶地址仍然是 0x9FF00 没有改变
2. 0x90000 开始往上的位置，原来是 bootsect 和 setup 程序的代码，现 bootsect 的一部分代码在已经被操作系统为了记录内存、硬盘、显卡等一些临时存放的数据给覆盖了一部分。
3. 内存最开始的 0 到 0x80000 这 512K 被 system 模块给占用了，之前讲过，这个 system 模块就是除了 bootsect 和 setup 之外的全部程序链接在一起的结果，可以理解为操作系统的全部。

## 先解决寄存器的历史包袱问题
> 这是 x86 的历史包袱问题，现在的 CPU 几乎都是支持 32 位模式甚至 64 位模式了，很少有还仅仅停留在 16 位的实模式下的 CPU。所以我们要为了这个历史包袱，写一段模式转换的代码，如果 Intel CPU 被重新设计而不用考虑兼容性，那么今天的代码将会减少很多甚至不复存在。

```c
end_move:
	mov	ax,#SETUPSEG	! right, forgot this at first. didn't work :-)
	mov	ds,ax
	lidt	idt_48		! load idt with 0,0
	lgdt	gdt_48		! load gdt with whatever appropriate
```
**实模式和保护模式**
- 实模式：这个模式的 CPU 计算物理地址的方式，就是段基址左移四位，再加上偏移地址。
- 保护模式： ds 寄存器里存储的值，在实模式下叫做段基址，在保护模式下叫段选择子。段选择子里存储着段描述符的索引。通过段描述符索引，可以从全局描述符表 gdt 中找到一个段描述符，段描述符里存储着段基址。段基址取出来，再和偏移地址相加，就得到了物理地址。
> lgdt	gdt_48: 全局描述符表（gdt）在内存中的位置存储在一个叫 gdtr 的寄存器中。

## 六行代码就进入了保护模式
```c
! that was painless, now we enable A20

	call	empty_8042
	mov	al,#0xD1		! command write
	out	#0x64,al
	call	empty_8042
	mov	al,#0xDF		! A20 on
	out	#0x60,al
	call	empty_8042
```
上述这段代码打开A20地址线
> 简单理解，这一步就是为了突破地址信号线 20 位的宽度，变成 32 位可用。这是由于 8086 CPU 只有 20 位的地址线，所以如果程序给出 21 位的内存地址数据，那多出的一位就被忽略了，比如如果经过计算得出一个内存地址为：1 0000 00000000 00000000，那实际上内存地址相当于 0，因为高位的那个 1 被忽略了，地方不够。 当 CPU 到了 32 位时代之后，由于要考虑兼容性，还必须保持一个只能用 20 位地址线的模式，所以如果你不手动开启的话，即使地址线已经有 32 位了，仍然会限制只能使用其中的 20 位。

```c
! well, that went ok, I hope. Now we have to reprogram the interrupts :-(
! we put them right after the intel-reserved hardware interrupts, at
! int 0x20-0x2F. There they won't mess up anything. Sadly IBM really
! messed this up with the original PC, and they haven't been able to
! rectify it afterwards. Thus the bios puts interrupts at 0x08-0x0f,
! which is used for the internal hardware interrupts as well. We just
! have to reprogram the 8259's, and it isn't fun.

	mov	al,#0x11		! initialization sequence
	out	#0x20,al		! send it to 8259A-1
	.word	0x00eb,0x00eb		! jmp $+2, jmp $+2
	out	#0xA0,al		! and to 8259A-2
	.word	0x00eb,0x00eb
	mov	al,#0x20		! start of hardware int's (0x20)
	out	#0x21,al
	.word	0x00eb,0x00eb
	mov	al,#0x28		! start of hardware int's 2 (0x28)
	out	#0xA1,al
	.word	0x00eb,0x00eb
	mov	al,#0x04		! 8259-1 is master
	out	#0x21,al
	.word	0x00eb,0x00eb
	mov	al,#0x02		! 8259-2 is slave
	out	#0xA1,al
	.word	0x00eb,0x00eb
	mov	al,#0x01		! 8086 mode for both
	out	#0x21,al
	.word	0x00eb,0x00eb
	out	#0xA1,al
	.word	0x00eb,0x00eb
	mov	al,#0xFF		! mask off all interrupts for now
	out	#0x21,al
	.word	0x00eb,0x00eb
	out	#0xA1,al
```
这一段代码是对可编程中断控制器 8259 芯片进行的编程。重新编程后8259 这个芯片的引脚与中断号的对应关系如下
PIC请求号	中断号	用途
IRQ0	0x20	时钟中断
IRQ1	0x21	键盘中断
IRQ2	0x22	接连从芯片
IRQ3	0x23	串口2
IRQ4	0x24	串口1
IRQ5	0x25	并口2
IRQ6	0x26	软盘驱动器
IRQ7	0x27	并口1
IRQ8	0x28	实时钟中断
IRQ9	0x29	保留
IRQ10	0x2a	保留
IRQ11	0x2b	保留
IRQ12	0x2c	鼠标中断
IRQ13	0x2d	数学协处理器
IRQ14	0x2e	硬盘中断
IRQ15	0x2f	保留
```c
! well, that certainly wasn't fun :-(. Hopefully it works, and we don't
! need no steenking BIOS anyway (except for the initial loading :-).
! The BIOS-routine wants lots of unnecessary data, and it's less
! "interesting" anyway. This is how REAL programmers do it.
!
! Well, now's the time to actually move into protected mode. To make
! things as simple as possible, we do no register set-up or anything,
! we let the gnu-compiled 32-bit programs do that. We just jump to
! absolute address 0x00000, in 32-bit protected mode.

	mov	ax,#0x0001	! protected mode (PE) bit
	lmsw	ax		! This is it!
	jmpi	0,8		! jmp offset 0 of segment 8 (cs)
```
> mov	ax,#0x0001 和 lmsw	ax将 cr0 这个寄存器的位 0 置 1，模式就从实模式切换到保护模式，cr0寄存器用位置0的值标记是否开启保护模式
> 段间跳转指令 jmpi，后面的 8 表示 cs（代码段寄存器）的值，0 表示偏移地址。请注意，此时已经是保护模式了，之前也说过，保护模式下内存寻址方式变了，段寄存器里的值被当做段选择子。
> 8 用二进制表示就是 00000,0000,0000,1000
> 第 0 项是空值，第一项被表示为代码段描述符，是个可读可执行的段，第二项为数据段描述符，是个可读可写段，不过他们的段基址都是 0。所以，这里取的就是这个代码段描述符，段基址是 0，偏移也是 0，那加一块就还是 0 咯，所以最终这个跳转指令，就是跳转到内存地址的 0 地址处，开始执行。

```s
! This routine checks that the keyboard command queue is empty
! No timeout is used - if this hangs there is something wrong with
! the machine, and we probably couldn't proceed anyway.
empty_8042:
	.word	0x00eb,0x00eb
	in	al,#0x64	! 8042 status port
	test	al,#2		! is input buffer full?
	jnz	empty_8042	! yes - loop
	ret

gdt:
	.word	0,0,0,0		! dummy

	.word	0x07FF		! 8Mb - limit=2047 (2048*4096=8Mb)
	.word	0x0000		! base address=0
	.word	0x9A00		! code read/exec
	.word	0x00C0		! granularity=4096, 386

	.word	0x07FF		! 8Mb - limit=2047 (2048*4096=8Mb)
	.word	0x0000		! base address=0
	.word	0x9200		! data read/write
	.word	0x00C0		! granularity=4096, 386

idt_48:
	.word	0			! idt limit=0
	.word	0,0			! idt base=0L

gdt_48:
	.word	0x800		! gdt limit=2048, 256 GDT entries
	.word	512+gdt,0x9	! gdt base = 0X9xxxx
	
.text
endtext:
.data
enddata:
.bss
endbss:
```
> 操作系统全部代码的 system 这个大模块，system 模块怎么生成的呢？由 Makefile 文件可知，是由 head.s 和 main.c 以及其余各模块的操作系统代码合并来的，可以理解为操作系统的全部核心代码编译后的结果。

## 又要重新设置一遍idt和gdt
> git/Linux-0.11code/boot/head.s

```c
_pg_dir:
startup_32:
	movl $0x10,%eax
	mov %ax,%ds
	mov %ax,%es
	mov %ax,%fs
	mov %ax,%gs
	lss _stack_start,%esp
```
> 连续五个 mov 操作，分别给 ds、es、fs、gs 这几个段寄存器赋值为 0x10，根据段描述符结构解析，表示这几个段寄存器的值为指向全局描述符表中的第二个段描述符，也就是数据段描述符。
> lss 指令相当于让 ss:esp 这个栈顶指针指向了 _stack_start 这个标号的位置。
> _stack_start: 
```c
long user_stack[4096 >> 2];

struct
{
  long *a;
  short b;
}
stack_start = {&user_stack[4096 >> 2], 0x10};
```
> 首先，stack_start 结构中的高位 8 字节是 0x10，将会赋值给 ss 栈段寄存器，低位 16 字节是 user_stack 这个数组的最后一个元素的地址值，将其赋值给 esp 寄存器。赋值给 ss 的 0x10 仍然按照保护模式下的段选择子去解读，其指向的是全局描述符表中的第二个段描述符（数据段描述符），段基址是 0。赋值给 esp 寄存器的就是 user_stack 数组的最后一个元素的内存地址值，那最终的栈顶地址，也指向了这里（user_stack + 0），后面的压栈操作，就是往这个新的栈顶地址处压。

```c
	call setup_idt
	call setup_gdt
	movl $0x10,%eax		# reload all the segment registers
	mov %ax,%ds		# after changing gdt. CS was already
	mov %ax,%es		# reloaded in 'setup_gdt'
	mov %ax,%fs
	mov %ax,%gs
	lss _stack_start,%esp
```
> 先设置了 idt 和 gdt，然后又重新执行了一遍刚刚执行过的代码。为什么要重新设置这些段寄存器呢？因为上面修改了 gdt，所以要重新设置一遍以刷新才能生效。那我们接下来就把目光放到设置 idt 和 gdt 上。中断描述符表 idt 我们之前没设置过，所以这里设置具体的值，理所应当。

```c
/*
 *  setup_idt
 *
 *  sets up a idt with 256 entries pointing to
 *  ignore_int, interrupt gates. It then loads
 *  idt. Everything that wants to install itself
 *  in the idt-table may do so themselves. Interrupts
 *  are enabled elsewhere, when we can be relatively
 *  sure everything is ok. This routine will be over-
 *  written by the page tables.
 */
setup_idt:
	lea ignore_int,%edx
	movl $0x00080000,%eax
	movw %dx,%ax		/* selector = 0x0008 = cs */
	movw $0x8E00,%dx	/* interrupt gate - dpl=0, present */

	lea _idt,%edi
	mov $256,%ecx
rp_sidt:
	movl %eax,(%edi)
	movl %edx,4(%edi)
	addl $8,%edi
	dec %ecx
	jne rp_sidt
	lidt idt_descr
	ret

idt_descr:
	.word 256*8-1		# idt contains 256 entries
	.long _idt

_idt:	.fill 256,8,0		# idt is uninitialized
```
> 中断描述符表 idt 里面存储着一个个中断描述符，每一个中断号就对应着一个中断描述符，而中断描述符里面存储着主要是中断程序的地址，这样一个中断号过来后，CPU 就会自动寻找相应的中断程序，然后去执行它。
> 那这段程序的作用就是，设置了 256 个中断描述符，并且让每一个中断描述符中的中断程序例程都指向一个 ignore_int 的函数地址，这个是个默认的中断处理程序，之后会逐渐被各个具体的中断程序所覆盖。比如之后键盘模块会将自己的键盘中断处理程序，覆盖过去。
> 那现在，产生任何中断都会指向这个默认的函数 ignore_int，也就是说现在这个阶段你按键盘还不好使。

```c
_gdt:	.quad 0x0000000000000000	/* NULL descriptor */
	.quad 0x00c09a0000000fff	/* 16Mb */
	.quad 0x00c0920000000fff	/* 16Mb */
	.quad 0x0000000000000000	/* TEMPORARY - don't use */
	.fill 252,8,0			/* space for LDT's and TSS's etc */
```
> 其实和我们原先设置好的 gdt 一模一样。也是有代码段描述符和数据段描述符，然后第四项系统段描述符并没有用到，不用管。最后还留了 252 项的空间，这些空间后面会用来放置任务状态段描述符 TSS 和局部描述符 LDT。

**为什么原来已经设置过一遍了，这里又要重新设置一遍，你可千万别想有什么复杂的原因，就是因为原来设置的 gdt 是在 setup 程序中，之后这个地方要被缓冲区覆盖掉，所以这里重新设置在 head 程序中，这块内存区域之后就不会被其他程序用到并且覆盖了，就这么个事。**
```c
after_page_tables:
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $_main
	jmp setup_paging
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.
```
> 开启分页机制，并且跳转到 main 函数

## Linux内存管理：分段和分页
```c
/*
 * Setup_paging
 *
 * This routine sets up paging by setting the page bit
 * in cr0. The page tables are set up, identity-mapping
 * the first 16MB. The pager assumes that no illegal
 * addresses are produced (ie >4Mb on a 4Mb machine).
 *
 * NOTE! Although all physical memory should be identity
 * mapped by this routine, only the kernel page functions
 * use the >1Mb addresses directly. All "normal" functions
 * use just the lower 1Mb, or the local data space, which
 * will be mapped to some other place - mm keeps track of
 * that.
 *
 * For those with more memory than 16 Mb - tough luck. I've
 * not got it, why should you :-) The source is here. Change
 * it. (Seriously - it shouldn't be too difficult. Mostly
 * change some constants etc. I left it at 16Mb, as my machine
 * even cannot be extended past that (ok, but it was cheap :-)
 * I've tried to show which constants to change by having
 * some kind of marker at them (search for "16Mb"), but I
 * won't guarantee that's all :-( )
 */
.align 2
setup_paging:
	movl $1024*5,%ecx		/* 5 pages - pg_dir+4 page tables */
	xorl %eax,%eax
	xorl %edi,%edi			/* pg_dir is at 0x000 */
	cld;rep;stosl
	movl $pg0+7,_pg_dir		/* set present bit/user r/w */
	movl $pg1+7,_pg_dir+4		/*  --------- " " --------- */
	movl $pg2+7,_pg_dir+8		/*  --------- " " --------- */
	movl $pg3+7,_pg_dir+12		/*  --------- " " --------- */
	movl $pg3+4092,%edi
	movl $0xfff007,%eax		/*  16Mb - 4096 + 7 (r/w user,p) */
	std
1:	stosl			/* fill pages backwards - more efficient :-) */
	subl $0x1000,%eax
	jge 1b
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit */
	ret			/* this also flushes prefetch-queue */
```
> 启动分页机制
> 代码中给出一个内存地址，在保护模式下要先经过分段机制的转换，才能最终变成物理地址，这是在没有开启分页机制的时候，只需要经过这一步转换即可得到最终的物理地址了，但是在开启了分页机制后，又会多一步转换。也就是说，在没有开启分页机制时，由程序员给出的逻辑地址，需要先通过分段机制转换成物理地址。但在开启分页机制后，逻辑地址仍然要先通过分段机制进行转换，只不过转换后不再是最终的物理地址，而是线性地址，然后再通过一次分页机制转换，得到最终的物理地址。
> CPU 在看到我们给出的内存地址后，首先把线性地址被拆分成  高 10 位：中间 10 位：后 12 位  ，高 10 位负责在页目录表中找到一个页目录项，这个页目录项的值加上中间 10 位拼接后的地址去页表中去寻找一个页表项，这个页表项的值，再加上后 12 位偏移地址，就是最终的物理地址。
> 而这一切的操作，都由计算机的一个硬件叫 MMU，中文名字叫内存管理单元，有时也叫 PMMU，分页内存管理单元。由这个部件来负责将虚拟地址转换为物理地址。
> 所以整个过程我们不用操心，作为操作系统这个软件层，只需要提供好页目录表和页表即可，这种页表方案叫做二级页表，第一级叫页目录表 PDE，第二级叫页表 PTE。
> 之后再开启分页机制的开关。其实就是更改 cr0 寄存器中的一位即可（31 位），还记得我们开启保护模式么，也是改这个寄存器中的一位的值。
> **再看上述代码：**
> - 当时 linux-0.11 认为，总共可以使用的内存不会超过 16M，也即最大地址空间为 0xFFFFFF。
> - 而按照当前的页目录表和页表这种机制，1 个页目录表最多包含 1024 个页目录项（也就是 1024 个页表），1 个页表最多包含 1024 个页表项（也就是 1024 个页），1 页为 4KB（因为有 12 位偏移地址），因此，16M 的地址空间可以用 1 个页目录表 + 4 个页表搞定。
> - 所以，上面这段代码就是，将页目录表放在内存地址的最开头。

```c
/*
 * I put the kernel page tables right after the page directory,
 * using 4 of them to span 16 Mb of physical memory. People with
 * more than 16MB will have to expand this.
 */
.org 0x1000
pg0:

.org 0x2000
pg1:

.org 0x3000
pg2:

.org 0x4000
pg3:

.org 0x5000
```
> 之后紧挨着这个页目录表，放置 4 个页表，代码里也有这四个页表的标签项。
> 最终将页目录表和页表填写好数值，来覆盖整个 16MB 的内存。随后，开启分页机制。此时内存中的页表相关的布局如下。
```c
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
```
> 通过一个寄存器告诉 CPU 我们把这些页表放在了哪里，就是这段代码。
> 相当于告诉 cr3 寄存器，0 地址处就是页目录表，再通过页目录表可以找到所有的页表，也就相当于 CPU 知道了分页机制的全貌了。
> 
## 进入main函数前最后一跃
```c
after_page_tables:
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $_main
	jmp setup_paging
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.
```
> 启用分页指令完成后会跳转到main函数的内存地址
> git/Linux-0.11code/init/main.c
```c
void main(void)		/* This really IS void, no error here. */
{			/* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
 	ROOT_DEV = ORIG_ROOT_DEV;
 	drive_info = DRIVE_INFO;
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024) 
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
	mem_init(main_memory_start,memory_end);
	trap_init();
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);
	hd_init();
	floppy_init();
	sti();
	move_to_user_mode();
	if (!fork()) {		/* we count on this going ok */
		init();
	}
/*
 *   NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
	for(;;) pause();
}
```
## 进入内核前的苦力活
> 在这里总结一下之前的内容，进入内核前做了什么
1. 当你按下开机键的那一刻，在主板上提前写死的固件程序 BIOS 会将硬盘中启动区的 512 字节的数据，原封不动复制到内存中的 0x7c00 这个位置，并跳转到那个位置进行执行。有了这个步骤之后，我们就可以把代码写在硬盘第一扇区，让 BIOS 帮我们加载到内存并由 CPU 去执行，我们不用操心这个过程。而这一个扇区的代码，就是操作系统源码中最最最开始的部分，它可以执行一些指令，也可以把硬盘的其他部分加载到内存，其实本质上也是执行一些指令。这样，整个计算机今后如何运作，就完全交到我们自己的手中，想怎么玩就怎么玩了。
2. 接下来将硬盘中的其他部分加载到内存中，4个扇区+240个扇区
3. 设置内存中临时存放的一些变量，调整内存布局，将每块内存复制到他应该在的位置
4. 进入保护模式，设置分段、分页机制，idtr（中断）、gdtr（分段）、cr3（标志位），其中涉及操作系统的中断处理，逻辑地址到线性地址到物理地址的转换。
5. 压栈，转到main函数执行的地方
> 开机 --> 加载启动区 --> 加载setup.s --> 加载内核 --> 设置GDT --> 进入保护模式 --> 分页机制 --> 跳转到内核

## 整个操作系统就20几行代码
**s首先看一下main函数的代码**
```c
void main(void)		/* This really IS void, no error here. */
{			/* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
 	ROOT_DEV = ORIG_ROOT_DEV;
 	drive_info = DRIVE_INFO;
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024) 
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
	mem_init(main_memory_start,memory_end); // 内存初始化
	trap_init();  // 中断初始化
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();  // 进程调度初始化
	buffer_init(buffer_memory_end);
	hd_init();
	floppy_init();

	sti();
	move_to_user_mode();
	if (!fork()) {		/* we count on this going ok */
		init();
	}
/*
 *   NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
	for(;;) pause();
}
```
> - main函数的第一部分是一些参数的取值和计算，包括根设备 ROOT_DEV，之前在汇编语言中获取的各个设备的参数信息 drive_info，以及通过计算得到的内存边界。其中的设备信息都是由 setup.s 这个汇编程序调用 BIOS 中断获取的各个设备的信息，并保存在约定好的内存地址 0x90000 处。
> - 接下来是各种初始化init操作。
> - 切换到用户态模式，并在一个新的进程中做一个最终的初始化 init，这个 init 函数里会创建出一个进程，设置终端的标准 IO，并且再创建出一个执行 shell 程序的进程用来接受用户的命令
> - 第四部分是个死循环，如果没有任何任务可以运行，操作系统会一直陷入这个死循环无法自拔。

## 管理内存前先划分出三个边界值
```c
void main(void)		/* This really IS void, no error here. */
{			/* The startup routine assumes (well, ...) this */
    ...
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024) 
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;
	...
}
```
> **针对不同的内存大小，设置不同的边界值**，即主内存和缓冲区的内存范围

## 操作系统就用一张大表管理内存
> git/Linux-0.11code/mm/memory.c
```c
/* these are not to be changed without changing head.s etc */
#define LOW_MEM 0x100000
#define PAGING_MEMORY (15*1024*1024)
#define PAGING_PAGES (PAGING_MEMORY>>12)
#define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
#define USED 100

#define CODE_SPACE(addr) ((((addr)+4095)&~4095) < \
current->start_code + current->end_code)

static long HIGH_MEMORY = 0;

#define copy_page(from,to) \
__asm__("cld ; rep ; movsl"::"S" (from),"D" (to),"c" (1024):"cx","di","si")

static unsigned char mem_map [ PAGING_PAGES ] = {0,};
// mem_init(main_memory_start,memory_end); // 内存初始化
void mem_init(long start_mem, long end_mem)
{
	int i;

	HIGH_MEMORY = end_mem;
	for (i=0 ; i<PAGING_PAGES ; i++)
		mem_map[i] = USED;
	i = MAP_NR(start_mem);
	end_mem -= start_mem;
	end_mem >>= 12;
	while (end_mem-->0)
		mem_map[i++]=0;
}
```
> mem_init做的事就是准备了一个表，记录了哪些内存被占用了，哪些内存没被占用。
> mem_map 这个数组的每个元素都代表一个 4K 内存是否空闲（准确说是使用次数）。4K 内存通常叫做 1 页内存，而这种管理方式叫分页管理，就是把内存分成一页一页（4K）的单位去管理。
> 1M 以下的内存这个数组干脆没有记录，这里的内存是无需管理的，或者换个说法是无权管理的，也就是没有权利申请和释放，因为这个区域是内核代码所在的地方，不能被“污染”。
> 1M 到 2M 这个区间是缓冲区，2M 是缓冲区的末端，缓冲区的开始在哪里之后再说，这些地方不是主内存区域，因此直接标记为 USED，产生的效果就是无法再被分配了。
> 2M 以上的空间是主内存区域，而主内存目前没有任何程序申请，所以初始化时统统都是零，未来等着应用程序去申请和释放这里的内存资源。

**程序申请内存**
```c
/*
 * Get physical address of first (actually last :-) free page, and mark it
 * used. If no free pages left, return 0.
 */
unsigned long get_free_page(void)
{
register unsigned long __res asm("ax");

__asm__("std ; repne ; scasb\n\t"
	"jne 1f\n\t"
	"movb $1,1(%%edi)\n\t"
	"sall $12,%%ecx\n\t"
	"addl %2,%%ecx\n\t"
	"movl %%ecx,%%edx\n\t"
	"movl $1024,%%ecx\n\t"
	"leal 4092(%%edx),%%edi\n\t"
	"rep ; stosl\n\t"
	"movl %%edx,%%eax\n"
	"1:"
	:"=a" (__res)
	:"0" (0),"i" (LOW_MEM),"c" (PAGING_PAGES),
	"D" (mem_map+PAGING_PAGES-1)
	:"di","cx","dx");
return __res;
}
```
> 选择 mem_map 中首个空闲页面，并标记为已使用。

## 你的键盘什么时候生效
> git/Linux-0.11code/kernel/traps.c
```c
// trap_init();  // 中断初始化
void trap_init(void)
{
	int i;

	set_trap_gate(0,&divide_error);
	set_trap_gate(1,&debug);
	set_trap_gate(2,&nmi);
	set_system_gate(3,&int3);	/* int3-5 can be called from all */
	set_system_gate(4,&overflow);
	set_system_gate(5,&bounds);
	set_trap_gate(6,&invalid_op);
	set_trap_gate(7,&device_not_available);
	set_trap_gate(8,&double_fault);
	set_trap_gate(9,&coprocessor_segment_overrun);
	set_trap_gate(10,&invalid_TSS);
	set_trap_gate(11,&segment_not_present);
	set_trap_gate(12,&stack_segment);
	set_trap_gate(13,&general_protection);
	set_trap_gate(14,&page_fault);
	set_trap_gate(15,&reserved);
	set_trap_gate(16,&coprocessor_error);
	for (i=17;i<48;i++)
		set_trap_gate(i,&reserved);
	set_trap_gate(45,&irq13);
	outb_p(inb_p(0x21)&0xfb,0x21);
	outb(inb_p(0xA1)&0xdf,0xA1);
	set_trap_gate(39,&parallel_interrupt);
}
```
> git/Linux-0.11/include/asm/system.h
```c
// 效果就是在中断描述符表中插入了一个中断描述符。
#define _set_gate(gate_addr,type,dpl,addr) \
__asm__ ("movw %%dx,%%ax\n\t" \
	"movw %0,%%dx\n\t" \
	"movl %%eax,%1\n\t" \
	"movl %%edx,%2" \
	: \
	: "i" ((short) (0x8000+(dpl<<13)+(type<<8))), \
	"o" (*((char *) (gate_addr))), \
	"o" (*(4+(char *) (gate_addr))), \
	"d" ((char *) (addr)),"a" (0x00080000))

#define set_intr_gate(n,addr) \
	_set_gate(&idt[n],14,0,addr)

#define set_trap_gate(n,addr) \
	_set_gate(&idt[n],15,0,addr)

#define set_system_gate(n,addr) \
	_set_gate(&idt[n],15,3,addr)
```
> 这段代码就是往这个 idt 表里一项一项地写东西，其对应的中断号就是第一个参数，中断处理程序就是第二个参数。产生的效果就是，之后如果来一个中断后，CPU 根据其中断号，就可以到这个中断描述符表 idt 中找到对应的中断处理程序了。
> 这个 system 与 trap 的区别仅仅在于，设置的中断描述符的特权级不同，前者是 0（内核态），后者是 3（用户态），这块展开将会是非常严谨的、绕口的、复杂的特权级相关的知识，不明白的话先不用管，就理解为都是设置一个中断号和中断处理程序的对应关系就好了。
> 17 到 48 号中断都批量设置为了 reserved 函数，这是暂时的，后面各个硬件初始化时要重新设置好这些中断，把暂时的这个给覆盖掉。

```c
void tty_init(void)
{
	rs_init();
	con_init();
}

/*
 *  void con_init(void);
 *
 * This routine initalizes console interrupts, and does nothing
 * else. If you want the screen to clear, call tty_write with
 * the appropriate escape-sequece.
 *
 * Reads the information preserved by setup.s to determine the current display
 * type and sets everything accordingly.
 */
void con_init(void)
{
	...
	set_trap_gate(0x21,&keyboard_interrupt);
	...
}
/* from bsd-net-2: */
```
> 设置键盘相关的中断
> 虽然设置了键盘中断，但是到目前为止这些中断还是处于禁用状态

```c
// sti(); // main
#define sti() __asm__ ("sti"::)
```
> sti 最终会对应一个同名的汇编指令 sti，表示允许中断。所以这行代码之后，键盘才真正开始生效！

## 读取硬盘前的准备工作有哪些
> git/Linux-0.11code/kernel/blk_drv/ll_rw_blk.c
```c
// blk_dev_init() // main()
void blk_dev_init(void)
{
	int i;

	for (i=0 ; i<NR_REQUEST ; i++) {
		request[i].dev = -1;
		request[i].next = NULL;
	}
}
```
> git/Linux-0.11code/kernel/blk_drv/blk.h
```c
/*
 * Ok, this is an expanded form so that we can use the same
 * request for paging requests when that is implemented. In
 * paging, 'bh' is NULL, and 'waiting' is used to wait for
 * read/write completion.
 */
struct request {
	int dev;		/* -1 if no request 表示设备号，-1 就表示空闲。 */
	int cmd;		/* READ or WRITE 表示命令，其实就是 READ 还是 WRITE，也就表示本次操作是读还是写。 */
	int errors;     /* 表示操作时产生的错误次数。 */
	unsigned long sector;          // 表示起始扇区。
	unsigned long nr_sectors;      // 表示扇区数。
	char * buffer;                 // 表示数据缓冲区，也就是读盘之后的数据放在内存中的什么位置。
	struct task_struct * waiting;  // 是个 task_struct 结构，这可以表示一个进程，也就表示是哪个进程发起了这个请求。
	struct buffer_head * bh;       // 是缓冲区头指针
	struct request * next;         // 指向了下一个请求项。
};
```
> 比如读请求时，cmd 就是 READ，sector 和 nr_sectors 这俩就定位了所要读取的块设备（可以简单先理解为硬盘）的哪几个扇区，buffer 就定位了这些数据读完之后放在内存的什么位置。
>  request 结构可以完整描述一个读盘操作。然后那个 request 数组就是把它们都放在一起，并且它们又通过 next 指针串成链表。

**读盘操作**
> 读操作的系统调用函数是 sys_read
> 简化版函数如下：
```c
int sys_read(unsigned int fd,char * buf,int count) {
    struct file * file = current->filp[fd];  // 在进程文件描述符数组filp中找到一个空闲项  在系统文件表file_table中找到一个空闲项
    struct m_inode * inode = file->f_inode;  // 根据文件名从文件系统中查找inode
    // 校验 buf 区域的内存限制
    verify_area(buf,count);
    // 仅关注目录文件或普通文件
    return file_read(inode,file,buf,count);
}
```
> 入参 fd 是文件描述符，通过它可以找到一个文件的 inode，进而找到这个文件在硬盘中的位置。另两个入参 buf 就是要复制到的内存中的位置，count 就是要复制多少个字节。
```c
int file_read(struct m_inode * inode, struct file * filp, char * buf, int count) {
    int left,chars,nr;
    struct buffer_head * bh;
    left = count;
    while (left) {  // 每次读入一个块的数据，直到入参所要求的大小全部读完为止。
        if (nr = bmap(inode,(filp->f_pos)/BLOCK_SIZE)) {
            if (!(bh=bread(inode->i_dev,nr)))  // 这个函数就是去读某一个设备的某一个数据块号的内容
                break;
        } else
            bh = NULL;
        nr = filp->f_pos % BLOCK_SIZE;
        chars = MIN( BLOCK_SIZE-nr , left );
        filp->f_pos += chars;
        left -= chars;
        if (bh) {
            char * p = nr + bh->b_data;
            while (chars-->0)
                put_fs_byte(*(p++),buf++);
            brelse(bh);
        } else {
            while (chars-->0)
                put_fs_byte(0,buf++);
        }
    }
    inode->i_atime = CURRENT_TIME;
    return (count-left)?(count-left):-ERROR;
}

struct buffer_head * bread(int dev,int block) {
    struct buffer_head * bh = getblk(dev,block);  // 申请了一个内存中的缓冲块
    if (bh->b_uptodate)
        return bh;
    ll_rw_block(READ,bh);  //  ll_rw_block 负责把数据读入这个缓冲块
    wait_on_buffer(bh);
    if (bh->b_uptodate)
        return bh;
    brelse(bh);
    return NULL;
}

void ll_rw_block(int rw, struct buffer_head * bh) {
    ...
    make_request(major,rw,bh);
}

static void make_request(int major,int rw, struct buffer_head * bh) { // 该函数会往刚刚的设备的请求项链表 request[32] 中添加一个请求项，只要 request[32] 中有未处理的请求项存在，都会陆续地被处理，直到设备的请求项链表是空为止。
    ...
if (rw == READ)
        req = request+NR_REQUEST;
    else
        req = request+((NR_REQUEST*2)/3);
/* find an empty request */
    while (--req >= request)
        if (req->dev<0)
            break;
    ...
/* fill up the request-info, and add it to the queue */
    req->dev = bh->b_dev;
    req->cmd = rw;
    req->errors=0;
    req->sector = bh->b_blocknr<<1;
    req->nr_sectors = 2;
    req->buffer = bh->b_data;
    req->waiting = NULL;
    req->bh = bh;
    req->next = NULL;
    add_request(major+blk_dev,req);
}
```
