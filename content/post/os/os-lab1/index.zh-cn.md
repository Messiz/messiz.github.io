---
title: "操作系统实验学习记录（1）"
description: 
date: 2020-04-28
image: 
categories:
    - 操作系统
musicid:
---

由于这学期开设了操作系统的课程，而现在又是疫情期间，学校也已经通知本学期没有线下课程了，而学校的操作一天老师基本都是让我们自学（还是外教），光看书实在难以进行（英文原版），故前些日子在网上找到了哈工大李治军老师的操作系统课程，讲述得算很细致了。而课程刚好有配套的实验，为了图方便，我就在自己的虚拟机上搭建了相关的环境（当然这过程中也走了不少弯路），开始实验之旅。\
本篇博客是相关记录的开篇，对应的是操作系统的引导部分。主要想记录一下实验过程中遇到的问题。

------------------------------------------------------------------------

**bootsect.s** (引导部分)

```x86asm
SETUPLEN=2
SETUPSEG=0x07e0
entry _start
_start:
! 首先读入光标位置
    mov ah,#0x03
    xor bh,bh
    int 0x10

! 显示字符串&quot;MessizOS is ready!&quot;
! 要显示的字符串长度
    mov cx,#24
    mov bx,#0x0007
    mov bp,#msg1
! es:bp 是显示字符串的地址
! 相比 linux-0.11 中的代码，需要增加对es的处理因为原来代码中输出之前已经处理了es
    mov ax,#0x07c0
    mov es,ax
    mov ax,#0x1301
    int 0x10

load_setup:
    mov dx,#0x0000
    mov cx,#0x0002
    mov bx,#0x0200
    mov ax,#0x0200+SETUPLEN
    int 0x13
    jnc ok_load_setup
    mov dx,#0x0000
    mov ax,#0x0000
    int 0x13
    jmp load_setup
ok_load_setup:
    jmpi 0,SETUPSEG

! msg1处放置字符串
msg1:
! 换行+回车
    .byte 13,10
    .ascii &quot;MessizOS is ready!&quot;
! 两对换行+回车
    .byte 13,10,13,10

! boot_flag必须在最后两个字节
.org 510

! 设置引导扇区标志0xAA55
! 必须有它才能引导
boot_flag:
    .word 0xAA55
```

------------------------------------------------------------------------

这段代码首先是要实现启动时在屏幕上打印出一行字符串。

```x86asm
! 首先读入光标位置
mov ah,#0x03
xor bh,bh
int 0x10

! 显示字符串&quot;MessizOS is ready!&quot;
! 要显示的字符串长度
mov cx,#24
mov bx,#0x0007
mov bp,#msg1
! es:bp 是显示字符串的地址
! 相比 linux-0.11 中的代码，需要增加对es的处理因为原来代码中输出之前已经处理了es
mov ax,#0x07c0
mov es,ax
mov ax,#0x1301
int 0x10
```

先通过10号中断读取当前光标位置，再通过13号中断将指定的字符串打印到光标位置。\
至于字符串部分则在下面：


```x86asm
msg1:
! 换行+回车
    .byte 13,10
    .ascii &quot;MessizOS is ready!&quot;
! 两对换行+回车
    .byte 13,10,13,10
```

由于有两对换行+回车，因此字符串长度需要+6。

------------------------------------------------------------------------

```x86asm
SETUPSEG=0x07e0
```

由于历史原因，操作系统启动之后执行的代码地址是0x7c00，故将SETUPSEG设置为0x07e0。段地址是要左移4位（也就是\*16）才得到实际地址的，所以0x07e0实际上是0x7e00，为什么呢？\
\
0x7e00 - 0x7c00 = 0x200。0x200是什么？？\
\
0x200转换成十进制恰好是512！再仔细思考一下，发现最后几行代码是：

```x86asm
! boot_flag必须在最后两个字节
.org 510

! 设置引导扇区标志0xAA55
! 必须有它才能引导
boot_flag:
    .word 0xAA55
```

实际上bootsect部分的代码恰好被设置成了512字节大小（.org 510
再加上最后语句.word的一个字，也就是两字节），后面紧跟的恰好是setup部分的代码。

```x86asm
load_setup:
    mov dx,#0x0000
    mov cx,#0x0002
    mov bx,#0x0200
    mov ax,#0x0200+SETUPLEN
    int 0x13
    jnc ok_load_setup
```

这部分代码则是通过13号中断从磁盘中读入数据到内存，具体的参数不细说了。\
总体来说，bootsect实现了显示字符串"MessizOS is
ready!"，以及读入setup部分代码并且跳转到setup部分继续执行。

------------------------------------------------------------------------

**setup.s**

```x86asm
INITSEG = 0x9000

entry _start
_start:
! 首先读入光标位置
    mov ah,#0x03
    xor bh,bh
    int 0x10

    mov cx,#25
    mov bx,#0x0007
    mov bp,#msg2

    mov ax,cs
    mov es,ax
    mov ax,#0x1301
    int 0x10

    mov ax,cs
    mov es,ax
! 初始化ss:sp
    mov ax,#INITSEG
    mov ss,ax
    mov sp,#0xFF00

! 读入硬件信息至0x90000(INITSEG)处
    mov ax,#INITSEG
    mov ds,ax
    mov ah,#0x03
    xor bh,bh
    int 0x10
! 读出光标位置放入0x90000
    mov [0],dx

! 读入内存大小
    mov ah,#0x88
    int 0x15
    mov [2],ax

! 从0x41处拷贝16个字节(磁盘参数表)
    mov ax,#0x0000
    mov ds,ax
    lds si,[4*0x41]
    mov ax,#INITSEG
    mov es,ax
    mov di,#0x0004
    mov cx,#0x10
! 重复16(0x10)次
    rep
    movsb
! 移动 ds:si  ---&gt;  es:di (move string byte)

! 打印前准备
    mov ax,cs
    mov es,ax
    mov ax,#INITSEG
    mov ds,ax

! Cursor Position
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#13
    mov bx,#0x0007
    mov bp,#msg_cursor
    mov ax,#0x1301
    int 0x10
    mov dx,[0]
    call print_hex

! Memory Size
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#14
    mov bx,#0x0007
    mov bp,#msg_memory
    mov ax,#0x1301
    int 0x10
    mov dx,[2]
    call print_hex

! Add KB
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#2
    mov bx,#0x0007
    mov bp,#msg_kb
    mov ax,#0x1301
    int 0x10

! Cyles
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#8
    mov bx,#0x0007
    mov bp,#msg_cyles
    mov ax,#0x1301
    int 0x10
    mov dx,[4]
    call print_hex

! Heads
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#8
    mov bx,#0x0007
    mov bp,#msg_heads
    mov ax,#0x1301
    int 0x10
    mov dx,[6]
    call print_hex

! Sectors
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#10
    mov bx,#0x0007
    mov bp,#msg_sectors
    mov ax,#0x1301
    int 0x10
    mov dx,[12]
    call print_hex

inf_loop:
    jmp inf_loop


! --------------------------函数--------------------------
! 打印信息
print_hex:    
! 4个16进制数字
    mov cx,#4
! bp指向栈顶，将栈顶的值放入dx
!   mov dx,(bp)


print_digit:
! 循环，将dx的高4为移动到低4位
    rol dx,#4
    mov ax,#0xe0f
! ah 为中断请求值，al的低4位为掩码
    and al,dl
    add al,#0x30
    cmp al,#0x3a
! 不大于十
    jl outp

! 大于十，再加7
    add al,#0x07

outp:
    int 0x10
    loop print_digit
    ret

! 打印回车换行
print_nl:
! 0xd是13，CR
    mov ax,#0xe0d
    int 0x10
! 0xa是10，LF
    mov al,#0xa
    int 0x10
    ret
! --------------------------函数--------------------------


! --------------------------文本数据--------------------------
msg2:
! 换行+回车
    .byte 13,10
    .ascii &quot;Now we are in SETUP&quot;
! 两对换行+回车
    .byte 13,10,13,10

msg_cursor:
    .byte 13,10
    .ascii &quot;Cursor Pos:&quot;
msg_memory:
    .byte 13,10
    .ascii &quot;Memory size:&quot;
msg_cyles:
    .byte 13,10
    .ascii &quot;Cyles:&quot;
msg_heads:
    .byte 13,10
    .ascii &quot;Heads:&quot;
msg_sectors:
    .byte 13,10
    .ascii &quot;Sectors:&quot;
msg_kb:
    .ascii &quot;KB&quot;
! --------------------------文本数据--------------------------


! boot_flag必须在最后两个字节
.org 510

! 设置引导扇区标志0xAA55
! 必须有它才能引导
boot_flag:
    .word 0xAA55

``` 
***
setup程序首先复制了bootsect的部分代码，打印出了字符串&quot;Now we are in SETUP&quot;。而由于setup部分也是两个扇区，所以最后也是 .org 510。

setup的另一项功能是将一些硬件参数读出来，并显示在屏幕上。

```x86asm
print_hex:    
! 4个16进制数字
    mov cx,#4
! bp指向栈顶，将栈顶的值放入dx
!   mov dx,(bp)


print_digit:
! 循环，将dx的高4为移动到低4位
    rol dx,#4
    mov ax,#0xe0f
! ah 为中断请求值，al的低4位为掩码
    and al,dl
    add al,#0x30
    cmp al,#0x3a
! 不大于十
    jl outp

! 大于十，再加7
    add al,#0x07

outp:
    int 0x10
    loop print_digit
    ret
```

这部分代码是打印函数，dx中放的是需要打印的参数，存放的是二进制数值，所以要转化成对应的十六进制ascii码需要加上0x30或者是0x37（'0'的ascii码是30h，大写字母则需再加7）。

注意！！！此处我犯了个错误导致打印不出来，

```x86asm
!   mov dx,(bp)
```

就是这一句，之前是没有注释掉的，bp所指向的并不是数据区，故不能这么用，若是bp指向数据区则没问题。

```x86asm
! Cursor Position
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#13
    mov bx,#0x0007
    mov bp,#msg_cursor
    mov ax,#0x1301
    int 0x10
    mov dx,[0]
    call print_hex
```

最后 mov dx,\[0\]
已经将0x9000:0位置的两个字节数据放进了dx，所以上面那句是不需要的。

------------------------------------------------------------------------

而上述代码中我犯了另一个错误

```x86asm
mov ax,#0x1301
```

本来这里的0x没有加，这就变成了十进制数，这与我们预期的并不同，所以打印也就失败了。（这里真的是用调试器单步了好久才看出来）

------------------------------------------------------------------------

```x86asm
! 读入内存大小
    mov ah,#0x88
    int 0x15
    mov [2],ax

! 从0x41处拷贝16个字节(磁盘参数表)
    mov ax,#0x0000
    mov ds,ax
    lds si,[4*0x41]
    mov ax,#INITSEG
    mov es,ax
    mov di,#0x0004
    mov cx,#0x10
! 重复16(0x10)次
    rep
    movsb
! 移动 ds:si  ---&gt;  es:di (move string byte)
```

至于此处的读入就不细说了。

------------------------------------------------------------------------

**总结**

bootsect打印字符串 --\> 读入setup --\>
打印字符串，读入硬件信息，打印硬件信息。

ps：博客是不是写得太长了点。。。我好累（哭）\