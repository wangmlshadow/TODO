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




