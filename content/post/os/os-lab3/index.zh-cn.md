---
title: "操作系统实验学习记录（3）"
description: 哈工大操作系统实验
date: 2020-07-30
image: 
categories:
    - 操作系统
musicid:
---

又到了写学习记录的时候了\~\
今天的主题是关于进程的，主要探讨的是进程在不同状态之间的切换问题。
通过在进程切换状态的相应代码的关键部分添加输出语句，将进程切换状态的相关信息写入文件中，以便后续的观察和数据分析。

------------------------------------------------------------------------

**基础知识**

首先我们需要掌握一些基础的知识。\
\
在linux系统中，进程的相关信息是用结构体来保存的，包括一个进程的进程号(pid)，状态(state)，时间片(counter)等等一众信息，我们通常将这个结构体成为PCB(Process
Control Block，进程控制块)。\
其中我们当前最关心的就是state了。\
\
state可以取0\~4这5个值，含义分别如下：

-   TASK_RUNNING --- 0 **\*\*** 运行或就绪状态
-   TASK_INTERRUPTIBLE --- 1 **\*\*** 可中断等待
-   TASK_UNINTERRUPTIBLE --- 2 **\*\*** 不可中断等待
-   TASK_ZOMBIE --- 3 **\*\*** 僵死状态
-   TASK_STOPPED --- 4 **\*\*** 停止状态

具体含义可参见《内核》p178，此处不赘述了。\
\
在内核代码中，有一些函数涉及到进程状态的转换，例如：fork() --
创建进程，schedual() -- 进程的切换，sys_sleep()进程的休眠等等。\
总结来看，我们需要做出修改的文件有fork.c, sche.c,
exit.c。这3个文件都在kernel文件夹下。

------------------------------------------------------------------------

**具体步骤**

具体流程可以参见实验楼，简单概括如下：\

1.  编写process.c文件，其作用在于创建若干进程，并分配其占用cpu以及i/o的时间。
2.  修改内核代码，在适当位置加上输出语句，输出所需信息至/var/process.log文件。
3.  运行python脚本程序，根据process.log统计创建的进程具体的时间占用。

------------------------------------------------------------------------

**编写process.c文件**

```c++
#include &lt;unistd.h&gt;
#include &lt;time.h&gt;
#include &lt;sys/times.h&gt;

#define HZ   100

void cpuio_bound(int last, int cpu_time, int io_time);

int main(int argc, char * argv[])
{
  int pid, stat_addr;
  if(!(pid = fork()))    /*条件，pid = 0，即在子进程中，下同*/
 {
        cpuio_bound(10,1,1);
  }
    else
 {
        printf(&quot;fork %d\n&quot;, pid);
       if(!(pid = fork()))
      {
            cpuio_bound(10,1,0);
      }
        else
     {
            printf(&quot;fork %d\n&quot;, pid);
           if(!(pid = fork()))
          {
                cpuio_bound(10,0,1);
          }
            else
         {
                printf(&quot;fork %d\n&quot;, pid);
               if(!(pid = fork()))
        {
          cpuio_bound(10,1,9);
        }
        else
        {
          printf(&quot;fork %d\n&quot;, pid);
          }
          }
        }
    }
    wait(&amp;stat_addr);
 return 0;
}

/*
 * 此函数按照参数占用CPU和I/O时间
 * last: 函数实际占用CPU和I/O的总时间，不含在就绪队列中的时间，&gt;=0是必须的
 * cpu_time: 一次连续占用CPU的时间，&gt;=0是必须的
 * io_time: 一次I/O消耗的时间，&gt;=0是必须的
 * 如果last &gt; cpu_time + io_time，则往复多次占用CPU和I/O
 * 所有时间的单位为秒
 */
void cpuio_bound(int last, int cpu_time, int io_time)
{
    struct tms start_time, current_time;
 clock_t utime, stime;
    int sleep_time;

    while (last &gt; 0)
  {
        /* CPU Burst */
      times(&amp;start_time);
       /* 其实只有t.tms_utime才是真正的CPU时间。但我们是在模拟一个
      * 只在用户状态运行的CPU大户，就像“for(;;);”。所以把t.tms_stime
         * 加上很合理。*/
        do
       {
            times(&amp;current_time);
         utime = current_time.tms_utime - start_time.tms_utime;
            stime = current_time.tms_stime - start_time.tms_stime;
        } while ( ( (utime + stime) / HZ )  &lt; cpu_time );
        last -= cpu_time;

       if (last &lt;= 0 )
           break;

     /* IO Burst */
       /* 用sleep(1)模拟1秒钟的I/O操作 */
       sleep_time=0;
     while (sleep_time &lt; io_time)
      {
            sleep(1);
         sleep_time++;
     }
        last -= sleep_time;
   }
}
```

其中cpuio_bound函数是用来占用cpu的其中三个参数分别是总持续时间，cpu占用时间以及io占用时间，具体实现过程就不看了（其实是我看不懂\^\_\^）。至于主程序，就是fork了4个新的进程，并且分别对他们调用cpuio_bound函数，以及在终端打印各自的pid号，方便观察。

------------------------------------------------------------------------

**修改内核代码**

这一步需要在fork.c，sche.c，以及exit.c中添加相应的打印代码。而我们知道，在内核中是无法调用用户态函数fprintf的，因此，我们需要仿照内核态函数printk写一个用于输出到文件的函数fprintk。所幸，有现成的代码（我当然没这个能力写\~）。代码可以和printk放在一起在kernel/printk.c中。
```c++
int fprintk(int fd, const char *fmt, ...)
{
    va_list args;
    int count;
    struct file * file;
    struct m_inode * inode;

    va_start(args, fmt);
    count=vsprintf(logbuf, fmt, args);
    va_end(args);
/* 如果输出到stdout或stderr，直接调用sys_write即可 */
    if (fd &lt; 3)
    {
        __asm__(&quot;push %%fs\n\t&quot;
            &quot;push %%ds\n\t&quot;
            &quot;pop %%fs\n\t&quot;
            &quot;pushl %0\n\t&quot;
        /* 注意对于Windows环境来说，是_logbuf,下同 */
            &quot;pushl $logbuf\n\t&quot;
            &quot;pushl %1\n\t&quot;
        /* 注意对于Windows环境来说，是_sys_write,下同 */
            &quot;call sys_write\n\t&quot;
            &quot;addl $8,%%esp\n\t&quot;
            &quot;popl %0\n\t&quot;
            &quot;pop %%fs&quot;
            ::&quot;r&quot; (count),&quot;r&quot; (fd):&quot;ax&quot;,&quot;cx&quot;,&quot;dx&quot;);
    }
    else
/* 假定&gt;=3的描述符都与文件关联。事实上，还存在很多其它情况，这里并没有考虑。*/
    {
    /* 从进程0的文件描述符表中得到文件句柄 */
        if (!(file=task[0]-&gt;filp[fd]))
            return 0;
        inode=file-&gt;f_inode;

        __asm__(&quot;push %%fs\n\t&quot;
            &quot;push %%ds\n\t&quot;
            &quot;pop %%fs\n\t&quot;
            &quot;pushl %0\n\t&quot;
            &quot;pushl $logbuf\n\t&quot;
            &quot;pushl %1\n\t&quot;
            &quot;pushl %2\n\t&quot;
            &quot;call file_write\n\t&quot;
            &quot;addl $12,%%esp\n\t&quot;
            &quot;popl %0\n\t&quot;
            &quot;pop %%fs&quot;
            ::&quot;r&quot; (count),&quot;r&quot; (file),&quot;r&quot; (inode):&quot;ax&quot;,&quot;cx&quot;,&quot;dx&quot;);
    }
    return count;
}
```
同样细节不深究，我们赶紧干正事。

-   进程的创建

进程创建自然是fork函数来工作，深入之后发现，在sys_fork中调用的copy_process函数是创建新进程的核心步骤。该函数在kernel/fork.c文件中：

```c++
......
p-&gt;state = TASK_UNINTERRUPTIBLE;
p-&gt;pid = last_pid;
p-&gt;father = current-&gt;pid;
p-&gt;counter = p-&gt;priority;
p-&gt;signal = 0;
p-&gt;alarm = 0;
p-&gt;leader = 0;      /* process leadership doesn&#39;t inherit */
p-&gt;utime = p-&gt;stime = 0;
p-&gt;cutime = p-&gt;cstime = 0;
p-&gt;start_time = jiffies;

/*向log文件输出新建进程*/
       fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, p-&gt;pid, &#39;N&#39;, jiffies);

p-&gt;tss.back_link = 0;
p-&gt;tss.esp0 = PAGE_SIZE + (long) p;
p-&gt;tss.ss0 = 0x10;
p-&gt;tss.eip = eip;
p-&gt;tss.eflags = eflags;
```

这里将p-\>state设置为TASK_UNINTERRUPTIBLE并且接下来就对p进行一系列的赋值工作，由于fork创建的子进程除了pid之外与父进程的PCB是一样的，因此这里就可以输出一条创建进程的信息，用'N'来表示新建进程。

-   进程的就绪态

首先还是copy_process函数中，刚刚新建的进程，创建完成后应该变为就绪态

```c++
......
  set_tss_desc(gdt+(nr&lt;&lt;1)+FIRST_TSS_ENTRY,&amp;(p-&gt;tss));
  set_ldt_desc(gdt+(nr&lt;&lt;1)+FIRST_LDT_ENTRY,&amp;(p-&gt;ldt));
  p-&gt;state = TASK_RUNNING;    /* do this last, just in case */
  /*向log文件输出就绪进程*/
  fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, p-&gt;pid, &#39;J&#39;, jiffies);
  return last_pid;
......
```

这里将p-\>state改为了TASK_RUNNING，因此我们相应地输出转为就绪态的信息，用'J'来表示就绪态。

其次是这里，在sche函数中的schedual函数。在schedule函数中，首先会检测进程的报警定时值（alarm），并将处于可中断状态且接收到信号的进程唤醒。

```c++
if (*p) {
   if ((*p)-&gt;alarm &amp;&amp; (*p)-&gt;alarm &lt; jiffies) {
            (*p)-&gt;signal |= (1&lt;&lt;(SIGALRM-1));
            (*p)-&gt;alarm = 0;
       }
    if (((*p)-&gt;signal &amp; ~(_BLOCKABLE &amp; (*p)-&gt;blocked)) &amp;&amp;
  (*p)-&gt;state==TASK_INTERRUPTIBLE)
   {
        (*p)-&gt;state=TASK_RUNNING;
      /*修改*/
       fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, (*p)-&gt;pid, &#39;J&#39;, jiffies);
 }
}
```

schedual的核心功能在于进程调度，故一定有将当前进程由'R'变为'J'，以及将下一进程由'J'变为'R'的过程，添加代码如下:

```c++
if(task[next]-&gt;pid != current-&gt;pid)  /*队列中下一个非本进程*/
{
   if(current-&gt;state == TASK_RUNNING)
    {
        fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, current-&gt;pid, &#39;J&#39;, jiffies);
  }
    fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, task[next]-&gt;pid, &#39;R&#39;, jiffies);
}

switch_to(next);
```

最后是wake_up函数，用于将进程从"不可中断睡眠"中唤醒，从阻塞态'W'变为就绪态'J'：

```c++
void wake_up(struct task_struct **p)
{
  if (p &amp;&amp; *p) {
      (**p).state=0;
        /*wake_up函数直接将队列中的第一个进程唤醒*/
      fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, (*p)-&gt;pid, &#39;J&#39;, jiffies);
     *p=NULL;
 }
}
```

-   进程的阻塞态

sleep_on函数和interruptible_sleep_on函数实现进程由运行态'R'转为阻塞态'W'，而其中interruptible_sleep_on是指可以被中断唤醒的。这两个函数依旧在sche.c中：

```c++
void sleep_on(struct task_struct **p)
{
 struct task_struct *tmp;

  if (!p)
      return;
  if (current == &amp;(init_task.task))
        panic(&quot;task[0] trying to sleep&quot;);
 tmp = *p;
 *p = current;
 current-&gt;state = TASK_UNINTERRUPTIBLE;
 /*修改*/
   fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, current-&gt;pid, &#39;W&#39;, jiffies);
  schedule();
   if (tmp)
 {
        tmp-&gt;state=0;    /*0也就是TASK_RUNNING*/
    /*修改*/
        fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, tmp-&gt;pid, &#39;J&#39;, jiffies);
  }  
}
```

```c++
void interruptible_sleep_on(struct task_struct **p)
{
   struct task_struct *tmp;

  if (!p)
      return;
  if (current == &amp;(init_task.task))
        panic(&quot;task[0] trying to sleep&quot;);
 tmp=*p;
   *p=current;
repeat:    current-&gt;state = TASK_INTERRUPTIBLE;
   /*修改*/
   fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, current-&gt;pid, &#39;W&#39;, jiffies);
  schedule();
   if (*p &amp;&amp; *p != current) {
      (**p).state=0;
        /*修改*/
       fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, (*p)-&gt;pid, &#39;J&#39;, jiffies);
     goto repeat;
 }
    *p=NULL;
 if (tmp)
 {
        tmp-&gt;state=0;
      /*修改*/
       fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, tmp-&gt;pid, &#39;J&#39;, jiffies);
  }
        
}
```

sys_pause函数同样会将进程置为阻塞态'W'。当schedule函数发现系统中没有可执行的任务时，就会切换到0号任务，而0号任务就会循环执行sys_pause，将自己设置为可中断的阻塞态。要注意排除0号进程，因为实际上0号可以看作一直运行，所以这里将其排除在外。

```c++
int sys_pause(void) {
  current-&gt;state = TASK_INTERRUPTIBLE;
   /*修改，当进程号为0时，会一直调用sys_pause，但实际上0号可以看作一直运行，所以这里将其排除在外*/
    if(current-&gt;pid != 0) {
        fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, current-&gt;pid, &#39;W&#39;, jiffies);
    }
    schedule();
    return 0;
}
```

最后是exit.c文件中的sys_waitpid函数，exit.c也在kernel文件夹中：

```c++
if (flag) {
    if (options &amp; WNOHANG)
        return 0;
    current-&gt;state=TASK_INTERRUPTIBLE;
    /*修改*/
    fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, current-&gt;pid, &#39;W&#39;, jiffies);
    schedule();
    if (!(current-&gt;signal &amp;= ~(1&lt;&lt;(SIGCHLD-1))))
        goto repeat;
    else
        return -EINTR;
}
return -ECHILD;
```

-   进程的退出

一个进程最终的归宿就是退出，当然退出函数一定在exit.c中，也就是do_exit函数：

```c++
current-&gt;state = TASK_ZOMBIE;
current-&gt;exit_code = code;
/*子程序先退出，父程序再醒来，因此在tell_father之前先fprink*/
fprintk(3, &quot;%ld\t%c\t%ld\n&quot;, current-&gt;pid, &#39;E&#39;, jiffies);
tell_father(current-&gt;father);
schedule();
return (-1);   /* just to suppress warnings */
```

需要在call_father之前退出子进程。

------------------------------------------------------------------------

**运行process**

将linux0.11文件夹重新编译，将process.c文件通过挂载磁盘传到linux0.11中，./run运行linux0.11，完成编译工作，运行process，神奇的一幕出现了（并不）------

![](/post/2edb89bc/1.png)

数据太多，截取一段process.log文件示意：

```
1 N   48
1   J   48
0   J   48
1   R   48
2   N   49
2   J   49
1   W   49
2   R   49
3   N   63
3   J   64
2   J   64
3   R   64
3   W   68
2   R   68
2   E   73
1   J   73
1   R   73
4   N   74
4   J   74
1   W   74
4   R   74
5   N   106
5  J   106
4  W   107
5  R   107
4  J   109
5  E   109
4  R   109
4  W   115
0  R   115
4  J   538
4  R   538
4  W   538
0  R   538
4  J   604
4  R   604
4  W   604
0  R   604
4  J   647
4  R   647
4  W   647
0  R   647
4  J   723
4  R   723
4  W   723
0  R   723
4  J   748
4  R   748
4  W   748
0  R   748
......
......
```

------------------------------------------------------------------------

**分析数据**

python脚本是提供的，这里就贴上：

```python
#!/usr/bin/python
import sys
import copy

P_NULL = 0
P_NEW = 1
P_READY = 2
P_RUNNING = 4
P_WAITING = 8
P_EXIT = 16

S_STATE = 0
S_TIME = 1

HZ = 100

graph_title = r&quot;&quot;&quot;
-----===&lt; COOL GRAPHIC OF SCHEDULER &gt;===-----

             [Symbol]   [Meaning]
         ~~~~~~~~~~~~~~~~~~~~~~~~~~~
             number   PID or tick
              &quot;-&quot;     New or Exit 
              &quot;#&quot;       Running
              &quot;|&quot;        Ready
              &quot;:&quot;       Waiting
                    / Running with 
              &quot;+&quot; -|     Ready 
                    \and/or Waiting

-----===&lt; !!!!!!!!!!!!!!!!!!!!!!!!! &gt;===-----
&quot;&quot;&quot;

usage = &quot;&quot;&quot;
Usage: 
%s /path/to/process.log [PID1] [PID2] ... [-x PID1 [PID2] ... ] [-m] [-g]

Example:
# Include process 6, 7, 8 and 9 in statistics only. (Unit: tick)
%s /path/to/process.log 6 7 8 9

# Exclude process 0 and 1 from statistics. (Unit: tick)
%s /path/to/process.log -x 0 1

# Include process 6 and 7 only and print a COOL &quot;graphic&quot;! (Unit: millisecond)
%s /path/to/process.log 6 7 -m -g

# Include all processes and print a COOL &quot;graphic&quot;! (Unit: tick)
%s /path/to/process.log -g
&quot;&quot;&quot;

class MyError(Exception):
    pass

class DuplicateNew(MyError):
    def __init__(self, pid):
        args = &quot;More than one &#39;N&#39; for process %d.&quot; % pid
      MyError.__init__(self, args)

class UnknownState(MyError):
    def __init__(self, state):
     args = &quot;Unknown state &#39;%s&#39; found.&quot; % state
        MyError.__init__(self, args)

class BadTime(MyError):
    def __init__(self, time):
       args = &quot;The time &#39;%d&#39; is bad. It should &gt;= previous line&#39;s time.&quot; % time
       MyError.__init__(self, args)

class TaskHasExited(MyError):
    def __init__(self, state):
        args = &quot;The process has exited. Why it enter &#39;%s&#39; state again?&quot; % state
       MyError.__init__(self, args)

class BadFormat(MyError):
    def __init__(self):
       args = &quot;Bad log format&quot;
       MyError.__init__(self, args)

class RepeatState(MyError):
    def __init__(self, pid):
        args = &quot;Previous state of process %d is identical with this line.&quot; % (pid)
        MyError.__init__(self, args)

class SameLine(MyError):
   def __init__(self):
     args = &quot;It is a clone of previous line.&quot;
      MyError.__init__(self, args)

class NoNew(MyError):
  def __init__(self, pid, state):
     args = &quot;The first state of process %d is &#39;%s&#39;. Why not &#39;N&#39;?&quot; % (pid, state)
       MyError.__init__(self, args)

class statistics:
  def __init__(self, pool, include, exclude):
     if include:
          self.pool = process_pool()
            for process in pool:
                if process.getpid() in include:
                 self.pool.add(process)
        else:
            self.pool = copy.copy(pool)

     if exclude:
          for pid in exclude:
             if self.pool.get_process(pid):
                   self.pool.remove(pid)
 
  def list_pid(self):
     l = []
        for process in self.pool:
           l.append(process.getpid())
        return l

   def average_turnaround(self):
       if len(self.pool) == 0:
          return 0
     sum = 0
       for process in self.pool:
           sum += process.turnaround_time()
      return float(sum) / len(self.pool)

 def average_waiting(self):
      if len(self.pool) == 0:
          return 0
     sum = 0
       for process in self.pool:
           sum += process.waiting_time()
     return float(sum) / len(self.pool)
   
  def begin_time(self):
       begin = 0xEFFFFF
      for p in self.pool:
         if p.begin_time() &lt; begin:
                begin = p.begin_time()
        return begin

   def end_time(self):
     end = 0
       for p in self.pool:
         if p.end_time() &gt; end:
                end = p.end_time()
        return end

 def throughput(self):
       return len(self.pool) * HZ / float(self.end_time() - self.begin_time())

    def print_graphic(self):
        begin = self.begin_time()
     end = self.end_time()

       print graph_title

      for i in range(begin, end+1):
           line = &quot;%5d &quot; % i
         for p in self.pool:
             state = p.get_state(i)
                if state &amp; P_NEW:
                    line += &quot;-&quot;
               elif state == P_READY or state == P_READY | P_WAITING:
                  line += &quot;|&quot;
               elif state == P_RUNNING:
                 line += &quot;#&quot;
               elif state == P_WAITING:
                 line += &quot;:&quot;
               elif state &amp; P_EXIT:
                 line += &quot;-&quot;
               elif state == P_NULL:
                    line += &quot; &quot;
               elif state &amp; P_RUNNING:
                  line += &quot;+&quot;
               else:
                    assert False
                if p.get_state(i-1) != state and state != P_NULL:
                   line += &quot;%-3d&quot; % p.getpid()
               else:
                    line += &quot;   &quot;
         print line

class process_pool:
 def __init__(self):
     self.list = []
    
  def get_process(self, pid):
     for process in self.list:
           if process.getpid() == pid:
              return process
       return None

   def remove(self, pid):
      for process in self.list:
           if process.getpid() == pid:
              self.list.remove(process)

   def new(self, pid, time):
       p = self.get_process(pid)
     if p:
            if pid != 0:
             raise DuplicateNew(pid)
          else:
                p.states=[(P_NEW, time)]
      else:
            p = process(pid, time)
            self.list.append(p)
       return p

   def add(self, p):
       self.list.append(p)

 def __len__(self):
      return len(self.list)
    
  def __iter__(self):
     return iter(self.list)

class process:
  def __init__(self, pid, time):
      self.pid = pid
        self.states = [(P_NEW, time)]
 
  def getpid(self):
       return self.pid

    def change_state(self, state, time):
        last_state, last_time = self.states[-1]
       if state == P_NEW:
           raise DuplicateNew(pid)
      if time &lt; last_time:
          raise BadTime(time)
      if last_state == P_EXIT:
         raise TaskHasExited(state)
       if last_state == state and self.pid != 0: # task 0 can have duplicate state
            raise RepeatState(self.pid)

        self.states.append((state, time))

   def get_state(self, time):
      rval = P_NULL
     combo = P_NULL
        if self.begin_time() &lt;= time &lt;= self.end_time():
           for state, s_time in self.states:
               if s_time &lt; time:
                 rval = state
              elif s_time == time:
                 combo |= state
                else:
                    break
            if combo:
                rval = combo
      return rval

    def turnaround_time(self):
      return self.states[-1][S_TIME] - self.states[0][S_TIME]

    def waiting_time(self):
     return self.state_last_time(P_READY)

   def cpu_time(self):
     return self.state_last_time(P_RUNNING)

 def io_time(self):
      return self.state_last_time(P_WAITING)

 def state_last_time(self, state):
       time = 0
      state_begin = 0
       for s,t in self.states:
         if s == state:
               state_begin = t
           elif state_begin != 0:
               assert state_begin &lt;= t
               time += t - state_begin
               state_begin = 0
       return time


  def begin_time(self):
       return self.states[0][S_TIME]

  def end_time(self):
     return self.states[-1][S_TIME]
       
# Enter point
if len(sys.argv) &lt; 2:
   print usage.replace(&quot;%s&quot;, sys.argv[0])
   sys.exit(0)

# parse arguments
include = []
exclude = []
unit_ms = False
graphic = False
ex_mark = False

try:
  for arg in sys.argv[2:]:
        if arg == &#39;-m&#39;:
          unit_ms = True
           continue
     if arg == &#39;-g&#39;:
          graphic = True
           continue
     if not ex_mark:
         if arg == &#39;-x&#39;:
              ex_mark = True
           else:
                include.append(int(arg))
      else:
            exclude.append(int(arg))
except ValueError:
 print &quot;Bad argument &#39;%s&#39;&quot; % arg
  sys.exit(-1)

# parse log file and construct processes
processes = process_pool()

f = open(sys.argv[1], &quot;r&quot;)

# Patch process 0&#39;s New &amp; Run state
processes.new(0, 40).change_state(P_RUNNING, 40)

try:
    prev_time = 0
 prev_line = &quot;&quot;
    for lineno, line in enumerate(f):

     if line == prev_line:
            raise SameLine
       prev_line = line

        fields = line.split(&quot;\t&quot;)
     if len(fields) != 3:
         raise BadFormat

        pid = int(fields[0])
      s = fields[1].upper()

       time = int(fields[2])
     if time &lt; prev_time:
          raise BadTime(time)
      prev_time = time

        p = processes.get_process(pid)

      state = P_NULL
        if s == &#39;N&#39;:
         processes.new(pid, time)
      elif s == &#39;J&#39;:
           state = P_READY
       elif s == &#39;R&#39;:
           state = P_RUNNING
     elif s == &#39;W&#39;:
           state = P_WAITING
     elif s == &#39;E&#39;:
           state = P_EXIT
        else:
            raise UnknownState(s)
        if state != P_NULL:
          if not p:
               raise NoNew(pid, s)
          p.change_state(state, time)
except MyError, err:
    print &quot;Error at line %d: %s&quot; % (lineno+1, err)
   sys.exit(0)

# Stats
stats = statistics(processes, include, exclude)
att = stats.average_turnaround()
awt = stats.average_waiting()
if unit_ms:
   unit = &quot;ms&quot;
   att *= 1000/HZ
    awt *= 1000/HZ
else:
    unit = &quot;tick&quot;
print &quot;(Unit: %s)&quot; % unit
print &quot;Process   Turnaround   Waiting   CPU Burst   I/O Burst&quot;
for pid in stats.list_pid():
    p = processes.get_process(pid)
    tt = p.turnaround_time()
  wt = p.waiting_time()
 cpu = p.cpu_time()
    io = p.io_time()

    if unit_ms:
      print &quot;%7d   %10d   %7d   %9d   %9d&quot; % (pid, tt*1000/HZ, wt*1000/HZ, cpu*1000/HZ, io*1000/HZ)
    else:
        print &quot;%7d   %10d   %7d   %9d   %9d&quot; % (pid, tt, wt, cpu, io)
print &quot;Average:  %10.2f   %7.2f&quot; % (att, awt)
print &quot;Throughout: %.2f/s&quot; % (stats.throughput())

if graphic:
    stats.print_graphic()
```

最后运行结果是这样滴\~

![](/post/2edb89bc/2.png)

拜拜了您嘞\~（累瘫）\