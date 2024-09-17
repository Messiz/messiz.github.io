---
title: "操作系统实验学习记录（2）"
description: 哈工大操作系统实验
date: 2020-07-22
image: 
categories:
    - OS
musicid:
---

操作系统实验2，和第一次实验时间隔得稍微久了那么亿点点。。不要在意这些细节，赶紧进入正题（掩盖自己懒的事实）\

------------------------------------------------------------------------

首先阐明本实验的目的：实现系统调用\
\
要实现一个系统调用，首先得明白从用户输入命令，到系统调用相应的函数，最终完成功能，中间到底经历了怎样的过程。

真正的系统调用实现在内核中，操作系统向用户提供了API（Application
Programming
Interface），这些API并不实现系统调用的功能，而是去调用真正的系统调用，以实现相应的功能。

> -   用户程序
> -   API
> -   系统调用

大致上是从上到下的这样一种顺序，即：用户程序中调用操作系统提供的API，而API进一步调用内核中的系统调用来实现功能。\
\
下面以lib/close.c为例，来具体看一看其代码实现：

```c++
/*
 *  linux/lib/close.c
 *
 *  (C) 1991  Linus Torvalds
 */

#define __LIBRARY__
#include &lt;unistd.h&gt;

_syscall1(int,close,int,fd)
```

除去注释，宏定义和头文件包含，核心语句就是

```c++
_syscall1(int,close,int,fd)
```

显然这是一个宏，打开close.c中唯一包含的头文件(include/unistd.h)，可以找到

```c++
#define _syscall1(type,name,atype,a) \
type name(atype a) \
{ \
long __res; \
__asm__ volatile (&quot;int $0x80&quot; \
        : &quot;=a&quot; (__res) \
        : &quot;0&quot; (__NR_##name),&quot;b&quot; ((long)(a))); \
if (__res &gt;= 0) \
        return (type) __res; \
errno = -__res; \
return -1; \
}
```

那么将close.c中的宏展开就是这样

```c++
int close(int fd)
{
    long __res;
    __asm__ volatile (&quot;int $0x80&quot;
        : &quot;=a&quot; (__res)
        : &quot;0&quot; (__NR_close),&quot;b&quot; ((long)(fd)));
    if (__res &gt;= 0)
        return (int) __res;
    errno = -__res;
    return -1;
}
```

这是一个c语言的函数，在函数体中有一段内嵌汇编，可以看出，**NR_close传给eax，fd强转为long后传给ebx，这二者作为输入；核心的汇编语句是int
\$0x80，即调用0x80号中断，由此可见，api本质上是通过中断的方式去调用系统调用的。\
\
又注意到**NR_close并不在函数体中定义，那么在unistd.h这个文件中往前寻找，果然：

```c++
...
#define __NR_open       5
#define __NR_close      6
#define __NR_waitpid    7
#define __NR_creat      8
...
```

原来\_\_NR_close也是宏，定义为6，之前说过它被传递给eax，这代表这close在中断向量表中的位置为6。\
\
至此，我们理清了api做的事情：根据相应的系统调用号，发出中断请求int
0x80。

下一步就要研究中断指令int 0x80究竟干了一件什么事。\
\
在内核初始化时，调用了这样一个函数

```c++
void sched_init(void)
{
//    ……
    set_system_gate(0x80,&amp;system_call);
}
```

实际上就是将system_call的地址写入0x80的中断描述符表（Interrupt
Descriptor
Table，IDT），使得中断0x80发生时，会自动调用system_call函数，此函数在kernel/system_call.s中，是一个汇编文件

```x86asm
!……
! # 这是系统调用总数。如果增删了系统调用，必须做相应修改
nr_system_calls = 72
!……

.globl system_call
.align 2
system_call:

! # 检查系统调用编号是否在合法范围内
    cmpl \$nr_system_calls-1,%eax
    ja bad_sys_call
    push %ds
    push %es
    push %fs
    pushl %edx
    pushl %ecx

! # push %ebx,%ecx,%edx，是传递给系统调用的参数
    pushl %ebx

! # 让ds, es指向GDT，内核地址空间
    movl $0x10,%edx
    mov %dx,%ds
    mov %dx,%es
    movl $0x17,%edx
! # 让fs指向LDT，用户地址空间
    mov %dx,%fs
    call sys_call_table(,%eax,4)
    pushl %eax
    movl current,%eax
    cmpl $0,state(%eax)
    jne reschedule
    cmpl $0,counter(%eax)
    je reschedule
```

其中call sys_call_table(,%eax,4)是关键，这个寻址方式实际上表达了call
sys_call_table + 4 \*
%eax，而eax中实际放的是系统调用号，也就是说这句话是根据调用号，在sys_call_table表中找到对应的系统调用函数，然后调用它。至于为什么是4
\* %eax呢，因为32为的系统，内存地址是32位（4 Byte）的。

这个表的位置在include/linux/sys.h

```c++
...
extern int sys_open();
extern int sys_close();
extern int sys_waitpid();
extern int sys_creat();
...

fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
sys_setreuid,sys_setregid, sys_iam, sys_whoami };
```

前面是系统调用函数的extern声明，最后则是这个表啦！

最为核心的，也就是close系统调用本体，sys_close函数，位于fs/open.c文件中

```c++
int sys_close(unsigned int fd)
{
        struct file * filp;

        if (fd &gt;= NR_OPEN)
                return -EINVAL;
        current-&gt;close_on_exec &amp;= ~(1&lt;&lt;fd);
        if (!(filp = current-&gt;filp[fd]))
                return -EINVAL;
        current-&gt;filp[fd] = NULL;
        if (filp-&gt;f_count == 0)
                panic(&quot;Close: file count is 0&quot;);
        if (--filp-&gt;f_count)
                return (0);
        iput(filp-&gt;f_inode);
        return (0);
}
```

细节就不分析了，总之就是在做close该做的事情。

好了！到此为止，我以close函数为例，由表面的api一步步深挖到最核心的系统调用实现。这样应该理清了一个系统调用的具体实现和调用过程。下一步就是关键的自己动手做啦！

------------------------------------------------------------------------

(分割线好像不太明显。。)\
\
我们的目标是实现iam和whoami\
\
首先列举一下需要做的事情：

1.  写sys_iam()和sys_whoami()两个系统调用函数。（废话）
2.  定义iam和whoami的API接口。
3.  修改一些文件，比如添加相应的调用号的宏定义，更改调用总数等。
4.  写应用程序测试系统调用是否成功。

------------------------------------------------------------------------

**定义API**

在kernel文件夹内建立who.c文件，并编写以下内容：

```c++
#include &lt;unistd.h&gt;
#include &lt;errno.h&gt;
#include &lt;asm/segment.h&gt;

char a[100];    /*用来保存name*/

int sys_iam(const char * name)
{
        int i = 0;
        /*循环存放到字符串尾*/
        while((get_fs_byte(name + i)) != &#39;\0&#39;)
        {
                i++;
        }

        if(i &gt;= 24)
        {
                return -EINVAL;
        }

        i = 0;
        while((get_fs_byte(name + i)) != &#39;\0&#39;)
        {
                a[i] = get_fs_byte(name + i);
                i++;
        }

        printk(&quot;len(a):%d&quot;, i);
        return i;

}

int sys_whoami(char * name, unsigned int size)
{
        int i = 0;
        while(a[i] != &#39;\0&#39;)
        {
                i++;
        }

        if(i &gt; size)
        {
                return -1;
        }   

        i = 0;
        while(a[i] != &#39;\0&#39;)
        {
                put_fs_byte(a[i], name + i);
                i++;
        }
        return i;
}
```

简单分析一下

> char a\[100\]; /*用来保存name*/

采用char\[\]数组来存放用户的name\
\
在sys_iam函数中，先检查name的字符数，超过规定的个数则返回错误代码，符合要求则再次循环，将其存入a。

而在sys_whoami函数中，同样检查字符个数，符合要求后调用put_fs_byte函数，将a\[\]中的字符串复制到name中，利用指针返回。

------------------------------------------------------------------------

**用户测试程序**

```c++
#define __LIBRARY__
#include&lt;unistd.h&gt;
#include&lt;stdio.h&gt;   
#include&lt;errno.h&gt;

/* 功能:用户态文件,测试系统调用iam() */  

/* 系统调用: */
/* 第一个是函数返回值类型，第二个为函数名，第三个是参数1的数据类型，第四个是参数1的名称，依次类推 */
_syscall1(int,iam,const char*,myname)

#define NAMELEN 50    /* 定义最大长度为50 */
char myname[NAMELEN] = {0}; /* 用户态下，存储名字,并且要初始化 */

/* argc是命令行参数的个数，argv存储了所有的命令行输入的所有参数的值 */
/* 比如./iam warm-like-spring  :得到argc=2,argv[0]=./iam,argv[1]=warm-like-spring */
int main(int argc, char* argv[])
{
   int res = 0;    /* 用于接受系统调用后的返回值 */
 unsigned int namelen = 0;  /* 名字的实际长度 */

 if(argc &lt; 2){
        printf(&quot;Input arguments is less!\n&quot;);
       return -1;
   }
    else {
      while(argv[1][namelen] != &#39;\0&#39;){       /* 读取并拷贝main参数接收的输入名字字符串 */
         myname[namelen] = argv[1][namelen];
           ++namelen;
        }
        printf(&quot;before systemcall, myname: %s, namelen: %d in iam.c\n&quot;,myname,namelen);
     res = iam(myname);  /* 传递拷贝的参数进行调用 */
    }

  if(res == -1) 
       printf(&quot;systemcall have bug, res:%d, errno:%d\n&quot;,res,errno);  /* 打印对应的错误信息码 */
 else
     printf(&quot;systemcall success, res:%d\n&quot;,res);
 return 0;
}
————————————————
版权声明：本文为CSDN博主「嘻嘻作者哈哈」的原创文章
原文链接：https://blog.csdn.net/weixin_43971252/article/details/89714559</code></pre></td>
```

这里偷个懒，原博主写得很详细，就直接粘了他的代码过来。核心部分一个是上面通过宏，定义了iam的API，其次就是下面的测试main函数，用来调用API以及产生测试输出。

下面同样给出whoami.c的代码

```c++
#define __LIBRARY__
#include&lt;unistd.h&gt;
#include&lt;stdio.h&gt;
#include&lt;errno.h&gt;

/* 功能:用户态文件,测试系统调用whoami() */

_syscall2(int, whoami, char*, myname, unsigned int, size)

#define SIZE 23

int main(int argc, char** argv)
{
   char myname[SIZE+1] = {0}; /* 定义初始化 */
    unsigned int res = 0;

 res = whoami(myname, SIZE+1);
 if(res == -1)
        printf(&quot;systemcall have bug, res:%d, errno:%d\n&quot;,res,errno); 
   else{
       printf(&quot;systemcall success.\n&quot;); /*如果要执行testlab2.sh来测试,要把这一行注释掉*/
      printf(&quot;%s\n&quot;,myname);
  }
    return 0;
}
————————————————
版权声明：本文为CSDN博主「嘻嘻作者哈哈」的原创文章
原文链接：https://blog.csdn.net/weixin_43971252/article/details/89714559</code></pre></td>
```

值得注意的是，这两个文件是用户的测试程序，所以需要先挂载，以便宿主机和虚拟机之间交换文件

> sudo ./mount-hdc

将这两个文件放在hdc/usr/root文件夹下，这也就是linux0.11启动后用户所在的根目录，方便直接进行编译操作。

------------------------------------------------------------------------

**中间文件修改**

最外和最内的文件都写好了，下面需要修改一些中间文件，比如添加系统调用号，以及函数声明等来使得系统调用可以顺利进行。\

-   添加系统调用号的宏定义\

值得注意的是，不能直接修改include/unistd.h文件，因为在linux0.11系统中编译c文件，默认的头文件都在usr/include文件夹下，所以仍需要在挂载状态下，进入hdc/usr/include/unistd.h进行修改。

```c++
#define __NR_iam        72
#define __NR_whoami     73</code></pre></td>
```

由于本来有72个系统调用，编号到71，故在最后补上两个我们自己写的。

-   修改系统调用总数

打开kernel/system_call.s文件

```x86asm
! # 这是系统调用总数。如果增删了系统调用，必须做相应修改
nr_system_calls = 72
```

找到这一行，由于我们添加了两个系统调用，故将72改为74.

-   修改sys_call_table函数表 + 添加函数声明

打开include/linux/sys.h

```c++
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
sys_setreuid,sys_setregid, sys_iam, sys_whoami };
```

在最后添加上我们写的系统调用的名字。同时需要在前面也补上它们的声明

```c++
extern int sys_whoami();
extern int sys_iam();
```

------------------------------------------------------------------------

**修改Makefile文件**

修改Makefile文件，Makefile文件在各个文件夹下都有出现，主要是用来指导编译工作的，具体我就不太了解啦\~\
\
除去用户测试文件，我们只添加了一个who.c文件，路径是kernel/who.c，所以只需要修改kernel文件夹下的Makefile文件。

```makefile
OBJS  = sched.o system_call.o traps.o asm.o fork.o \
        panic.o printk.o vsprintf.o sys.o exit.o \
        signal.o mktime.o who.o
```

在最后添加who.o

```makefile
### Dependencies:
who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h
exit.s exit.o: exit.c ../include/errno.h ../include/signal.h \
  ../include/sys/types.h ../include/sys/wait.h ../include/linux/sched.h \
  ../include/linux/head.h ../include/linux/fs.h ../include/linux/mm.h \
  ../include/linux/kernel.h ../include/linux/tty.h ../include/termios.h \
  ../include/asm/segment.h
```

添加

> who.s who.o: who.c ../include/linux/kernel.h ../include/unistd.h

------------------------------------------------------------------------

**测试**\
终于到了检验成果的时候了！！！

> ./run

> gcc -o iam iam.c -Wall\
> \
> gcc -o whoami whoami.c -Wall

当当当当\~\
![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/9.png)

顺利出现了预期的结果\~\