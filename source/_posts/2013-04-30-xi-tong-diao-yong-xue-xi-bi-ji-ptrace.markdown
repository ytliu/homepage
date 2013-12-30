---
layout: post
title: "系统调用学习笔记 - ptrace和wait"
date: 2013-04-30 09:46
comments: true
categories: Linux
keywords: ptrace wait
---

在系统安全这门课上讲到ptrace这个系统调用，我马上想到当年做CFIMon里面用到的ptrace:

{% codeblock lang:c %}
int child(char **arg) {
    /*
     * force the task to stop before executing the first
     * user level instruction
     */
    ptrace(PTRACE_TRACEME, 0, NULL, NULL);

    ......

    execvp(arg[0], arg);
    /* not reached */
    return -1;
}
{% endcodeblock %}

然后在parent的代码里面：

{% codeblock lang:c %}

    /*
     * wait for the child to exec
     */
    ret = waitpid(pid, &status, WUNTRACED);
    
    if (WIFEXITED(status))
        errx(1, "task %s [%d] exited already status %d\n", arg[0], pid, WEXITSTATUS(status));
    
    ......

    fds[0].fd = perf_event_open(&fds[0].hw, pid, -1, -1, 0);
    fds[0].buf = mmap(NULL, map_size, PROT_READ|PROT_WRITE, MAP_SHARED, fds[0].fd, 0);

    ......

    /*
     * effectively activate monitoring
     */
    ptrace(PTRACE_CONT, pid, NULL, NULL);

    ......
{% endcodeblock %}

先是waitpid，等待child调用execv，之后设置一系列参数（在这里是打开perf_event，并mmap一段内存区域），然后调用ptrace让child继续执行。

其实到现在为止我也还不是很清楚ptrace的用法和waitpid那几个参数的意思，于是想好好学习下，在google上搜到了一篇翻译的[玩转ptrace1](http://www.kgdb.info/gdb/playing_with_ptrace_part_i/)和[2](http://www.kgdb.info/gdb/playing_with_ptrace_part_ii/)，这里归纳整理下，另外，在IBM的[developerWorks](https://www.ibm.com/developerworks/cn/)上找到一篇介绍进程相关，以及waitpid的[博文](http://www.ibm.com/developerworks/cn/linux/kernel/syscall/part3/)，也一起学习了。

<!-- more -->
------

### 僵尸进程和wait

在linux中，当一个进程退出（如调用exit等）后，并不是马上完全消失掉了，它还会留下一些踪迹，成为一个僵尸进程（Zombie）。作为一个僵尸进程来说，它已经放弃了几乎所有内存空间，没有任何可执行代码，也不能被调度，仅仅在进程列表中保留一个位置，记载该进程的退出状态等信息供其他进程收集，除此之外，僵尸进程不再占有任何内存空间。在僵尸进程记录了这个进程是怎么死亡的（是正常退出呢，还是出现了错误，还是被其它进程强迫退出的？），以及它占用的总系统CPU时间和总用户CPU时间分别是多少？还有发生页错误的数目和收到信号的数目等。

而wait和waitpid这两个系统调用就是用来收集这些信息，并使得这个僵尸进程永远消失的。

这两个系统调用的原型是这样子的：

{% codeblock lang:c %}
  #include <sys/wait.h>  
    
  pid_t wait(int *status);  

  pit_t wait(pid_t pid,int *status,int options);  
{% endcodeblock %}

对于wait来说，它是等待所有的子进程的退出， 而对比来看，waitpid增加了两个参数`pid`和`option`。

对于pid来说，有四种情况：

pid  		| Description
:---------- | :-----------
pid == -1   | 等待任一个子进程（与wait等效）；
pid > 0  	| 则等待其进程ID与pid相等的子进程。
pid == 0 	| 等待其组ID等于调用进程组ID的任一个子进程。
pid < -1 	| 等待其组ID等于pid绝对值的任一子进程。

     
另外，如果参数status的值不是NULL，wait就会把子进程退出时的状态取出并存入其中，这是一个整数值，指出了子进程是正常退出还是被非正常结束的，以及正常结束时的返回值，或被哪一个信号结束的等信息。有一套专门的宏（macro）来对其进行操作：

macro  				| Description
:------------------ | :----------------
WIFEXITED(status)	| 若子进程是正常退出的，则为真，此时可以调用WEXITSTATUS(status)获得退出值
WEXITSTATUS(status) | 若子进程是被异常终止的，则为真，此时可以调用WTERMSIG(status)获得使其终止的信号编号
WIFSTOPPED(status)  | 若子进程是暂停状态，则为真，此时可以调用WTERMSIG(status)获得使其暂停的信号编号

    
到现在为止，还有一个参数叫option，它提供了一些额外的选项来控制waitpid，目前在Linux中只支持WNOHANG和WUNTRACED两个选项：

Option      | Description
:---------- | :-----------
WNOHANG		| waitpid在调用时发现没有已退出的子进程可收集，则返回0
WUNTRACED	| 在所有符合条件的pid中，如果其中有已经stopped的进程，则立即返回（而对于traced的进程，即使没有该选项，如果其stopped了，也会立即返回）

   
------

### ptrace笔记

ptrace可以做很多事，比如可以在用户层拦截和修改系统调用，在每次系统调用的时候改变子进程中的寄存器和内核映像，实现断点调试和系统调用的跟踪等等，具体的可以看玩转ptrace[1](http://www.kgdb.info/gdb/playing_with_ptrace_part_i/)和[2](http://www.kgdb.info/gdb/playing_with_ptrace_part_ii/)。这里对ptrace进行一个总结：

ptrace这个系统调用的作用是允许一个进程（the tracing process）来跟踪和控制另外一个进程（the traced process）。它的原型是这样的：

{% codeblock lang:c %}
#include <sys/ptrace.h>
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
{% endcodeblock %}

第一个参数`request`决定了ptrace的行为与其它参数的使用方法，它可以有好多值，以下列出了几种比较常见的取值：

request 			| Description
:------------------ | :-----------------
PTRACE_TRACEME		| 由子进程调用，让父进程跟踪自己（pid, addr, and data are ignored）
PTRACE_PEEKTEXT		| 读取内存地址addr的值（data is ignored）
PTRACE_PEEKDATA		| 同PTRACE_PEEKTEXT	
PTRACE_PEEKUSER		| 读取tracee's USER area的addr位移的数据（data is ignored）
PTRACE_POKETEXT		| 将data中的值写入addr内存地址中
PTRACE_POKEDATA		| 同PTRACE_POKETEXT
PTRACE_POKEUSER		| 将data中的值写入tracee's USER area的addr位移地址中
PTRACE_GETREGS		| 将general-purpose寄存器写入data（addr is ignored）
PTRACE_GETFPREGS	| 将 floating-point寄存器写入data（addr is ignored）
PTRACE_SETREGS		| 修改tracee的general-purpose寄存器（addr is ignored)
PTRACE_SETFPREGS	| 修改tracee的floating-point寄存器（addr is ignored)
PTRACE_CONT			| 重启之前暂停的进程，如果data不为零，则代表发给进程的信号值（addr is ignored）
PTRACE_SYSCALL,		| 执行PTRACE_CONT，并且使进程进入syscall-enter-stop和syscall-exit-stop模式
PTRACE_SINGLESTEP 	| 执行PTRACE_CONT，并且进行单点跟踪
PTRACE_ATTACH		| 发送一个SIGSTOP信号到pid进程，并开始进行跟踪（addr and data are ignored）
PTRACE_DETACH		| 先detach，然后执行PTRACE_CONT。

  
除此之外，还有很多可用的取值，在linux的[manpage](http://linux.die.net/man/2/ptrace)里面有很详细的描述。

需要注意的是，这里所说的attach和之后的一系列操作都是针对**thread**而言的，对于多线程来说，每一个thread都可以单独地被attach到一个跟踪者（tracer）。

在[玩转ptrace1](http://www.kgdb.info/gdb/playing_with_ptrace_part_i/)中举了一个很有趣的例子，可以通过这个例子来理解这些request都是怎么用的。

{% codeblock lang:c %}
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>
#include <sys/syscall.h>
const int long_size = sizeof(long);
void reverse(char *str)
{   int i, j;
    char temp;
    for(i = 0, j = strlen(str) - 2;
        i <= j; ++i, --j) {
        temp = str[i];
        str[i] = str[j];
        str[j] = temp;
    }
}
void getdata(pid_t child, long addr,
             char *str, int len)
{   char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 4,
                          NULL);
        memcpy(laddr, data.chars, long_size);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        data.val = ptrace(PTRACE_PEEKDATA,
                          child, addr + i * 4,
                          NULL);
        memcpy(laddr, data.chars, j);
    }
    str[len] = '\0';
}
void putdata(pid_t child, long addr,
             char *str, int len)
{   char *laddr;
    int i, j;
    union u {
            long val;
            char chars[long_size];
    }data;
    i = 0;
    j = len / long_size;
    laddr = str;
    while(i < j) {
        memcpy(data.chars, laddr, long_size);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * 4, data.val);
        ++i;
        laddr += long_size;
    }
    j = len % long_size;
    if(j != 0) {
        memcpy(data.chars, laddr, j);
        ptrace(PTRACE_POKEDATA, child,
               addr + i * 4, data.val);
    }
}
int main()
{
   pid_t child;
   child = fork();
   if(child == 0) {
      ptrace(PTRACE_TRACEME, 0, NULL, NULL);
      execl("/bin/ls", "ls", NULL);
   }
   else {
      long orig_eax;
      long params[3];
      int status;
      char *str, *laddr;
      int toggle = 0;
      while(1) {
         wait(&status);
         if(WIFEXITED(status))
             break;
         orig_eax = ptrace(PTRACE_PEEKUSER,
                           child, 4 * ORIG_EAX,
                           NULL);
         if(orig_eax == SYS_write) {
            if(toggle == 0) {
               toggle = 1;
               params[0] = ptrace(PTRACE_PEEKUSER,
                                  child, 4 * EBX,
                                  NULL);
               params[1] = ptrace(PTRACE_PEEKUSER,
                                  child, 4 * ECX,
                                  NULL);
               params[2] = ptrace(PTRACE_PEEKUSER,
                                  child, 4 * EDX,
                                  NULL);
               str = (char *)calloc((params[2]+1)
                                 * sizeof(char));
               getdata(child, params[1], str,
                       params[2]);
               reverse(str);
               putdata(child, params[1], str,
                       params[2]);
            }
            else {
               toggle = 0;
            }
         }
      ptrace(PTRACE_SYSCALL, child, NULL, NULL);
      }
   }
   return 0;
}
{% endcodeblock %}

首先子进程调用

{% codeblock lang:c %}
ptrace(PTRACE_TRACEME, 0, NULL, NULL);
{% endcodeblock %}

来让父进程跟踪自己，当然，这个也可以通过在父进程中调用

{% codeblock lang:c %}
ptrace(PTRACE_ATTACH, traced_process_id, NULL, NULL);
{% endcodeblock %}

来实现，之后子进程运行了`bin/ls`。当子进程发生系统调用的时候会将控制权转交入父进程，父进程通过

{% codeblock lang:c %}
orig_eax = ptrace(PTRACE_PEEKUSER, child, 4 * ORIG_EAX, NULL);
{% endcodeblock %}

来获得eax寄存器的值（从而判断调用的是哪个系统调用），之后继续通过`PTRACE_PEEKUSER`这个request来获得SYS_write系统调用的其它参数(ebx, ecx, edx)，当然，这个步骤还可以用

{% codeblock lang:c %}
#include <linux/user.h>
struct user_regs_struct regs;
ptrace(PTRACE_GETREGS, child, NULL, &regs);
{% endcodeblock %}

来替代，从而得到所有的寄存器的值regs。

之后，父进程在getdata函数中通过

{% codeblock lang:c %}
data.val = ptrace(PTRACE_PEEKDATA, child, addr + i * 4, NULL);
{% endcodeblock %}

将文件名从ecx中获得，并在putdata函数中通过

{% codeblock lang:c %}
ptrace(PTRACE_POKEDATA, child, addr + i * 4, data.val);
{% endcodeblock %}

将反转后的字符串写入寄存器ecx中，从而使得打出来的文件名是反转了的。

最后，通过调用

{% codeblock lang:c %}
ptrace(PTRACE_SYSCALL, child, NULL, NULL);
{% endcodeblock %}

将控制权返还给子进程，同时让其进入syscall-enter-stop和syscall-exit-stop模式，即在进入和退出system call的时候都会stop，并将控制权交给父进程。

这里有一个问题，就是当子进程调用PTRACE_TRACEME或者父进程调用PTRACE_ATTACH之后，在什么情况下会将子进程stop（从而将控制权交给父进程）呢？

要回答这个问题，首先要知道当我们使用ptrace的时候，内核中发生了什么？这里有一段简要的说明：当一个进程调用了 ptrace(PTRACE_TRACEME, …)之后，内核为该进程设置了一个标记，注明该进程将被跟踪。内核中的相关原代码（位于`arch/i386/kernel/ptrace.c`）如下：

{% codeblock lang:c %}
if (request == PTRACE_TRACEME) {
    /* are we already being traced? */
    if (current->ptrace & PT_PTRACED)
        goto out;
    /* set the ptrace bit in the process flags. */
    current->ptrace |= PT_PTRACED;
    ret = 0;
    goto out;
}
{% endcodeblock %}

一次系统调用完成之后，内核察看那个标记，然后执行trace系统调用（如果这个进程正处于被跟踪状态的话）。其汇编的细节可以在 `arch/i386/kernel/entry.S`中找到。

现在让我们来看看这个sys_trace()函数（位于`arch/i386/kernel/ptrace.c`）。它停止子进程，然后发送一个信号给父进程，告诉它子进程已经停滞，这个信号会激活正处于等待状态的父进程，让父进程进行相关处理。父进程在完成相关操作以后就调用ptrace(PTRACE_CONT, …)或者 ptrace(PTRACE_SYSCALL, …)，这将唤醒子进程，内核此时所作的是调用一个叫`wake_up_process()`的进程调度函数。其他的一些系统架构可能会通过发送SIGCHLD给子进程来达到这个目的。

------

由此可以看出，通过ptrace和wait(waitpid)等系统调用，可以使得父进程在系统调用的级别做子进程的跟踪和检查，由此来做很多的事情，但是有一个问题是，由于ptrace并不能指定哪些系统调用被跟踪，因此所有系统调用都会被stop并且转移控制权，由此会产生比较大的overhead。

总的来说，ptrace是一个非常强大的系统调用，这里只是介绍了其中几个比较常用的参数，更多的信息可以参照[ptrace的manpage](http://linux.die.net/man/2/ptrace)