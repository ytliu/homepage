---
layout: post
title: "fork in multi-threaded program"
date: 2013-09-11 16:53
comments: true
categories: System Linux
keywords: multi-thread fork
---

Today I've been asked a interesting problem: 

> If there's a multi-threaded program, when one of the threads execute `fork()`, is the child process still multi-threaded?

In the beginning, I thought that when `fork`, all of the memory space of current process is COW (Copy-On-Write) to the new process, and the thread-related information is no exception. So the child process should still be multi-threaded.

However, since I'm not sure about that, I execute a simple test code:

<!-- more -->

{% codeblock lang:c %}
#include <pthread.h>
#include <stdio.h>
#include <sys/time.h>
#include <string.h>
#define MAX 5
int pid;
pthread_t thread[2];
pthread_mutex_t mut;
int number=0, i;
void *thread1()
{
    printf ("thread1 : I'm thread 1\n");
    for (i = 0; i < MAX; i++)
    {
        printf("thread1 : number = %d\n",number);
        pthread_mutex_lock(&mut);
        number++;
        pthread_mutex_unlock(&mut);
        sleep(2);
    }
    printf("pid : %d, thread1 : done\n", pid);
    pthread_exit(NULL);
}
void *thread2()
{
    printf("thread2 : I'm thread 2\n");
    for (i = 0; i < MAX; i++)
    {
        printf("thread2 : number = %d\n",number);
        pthread_mutex_lock(&mut);
        number++;
        pthread_mutex_unlock(&mut);
        sleep(3);
    }
    printf("pid : %d, thread2 : done\n", pid);
    pthread_exit(NULL);
}
void thread_create(void)
{
    int temp;
    memset(&thread, 0, sizeof(thread));          //comment1
    /* Create threads */
    if((temp = pthread_create(&thread[0], NULL, thread1, NULL)) != 0)       //comment2
        printf("Thread 1 create fail\n");
    else
        printf("Thread 1 create successfully\n");
    if((temp = pthread_create(&thread[1], NULL, thread2, NULL)) != 0)  //comment3
        printf("Thread 2 create fail");
    else
        printf("Thread 2 create successfully\n");
}
void thread_wait(void)
{
    /* Wait for threads*/
    if(thread[0] !=0) {                   //comment4
        pthread_join(thread[0],NULL);
        printf("pid - %d Thread 1 Finished\n", pid);
    }
    if(thread[1] !=0) {                //comment5
        pthread_join(thread[1],NULL);
        printf("pid - %d Thread 2 Finished\n", pid);
    }
}
int main()
{
    pthread_mutex_init(&mut,NULL);
    printf("Main thread create threads\n");
    thread_create();
    pid = fork();
    printf("pid-%d Main thread wait for threads\n", pid);
    thread_wait();
    return 0;
}
{% endcodeblock %}

	$ gcc -lpthread -o thread_example thread_example.c
	$ ./thread_example

Then the result shows as follows:

	Main thread create threads
	Thread 1 create successfully
	thread1 : I'm thread 1
	Thread 2 create successfully
	thread1 : number = 0
	thread2 : I'm thread 2
	thread2 : number = 1
	pid-99995 Main thread wait for threads
	pid-0 Main thread wait for threads
	pid - 0 Thread 1 Finished
	pid - 0 Thread 2 Finished
	thread1 : number = 2
	thread2 : number = 3
	thread1 : number = 4
	thread2 : number = 5
	pid : 99995, thread1 : done
	pid - 99995 Thread 1 Finished
	pid : 99995, thread2 : done
	pid - 99995 Thread 2 Finished

Which means, the child process does not have other 2 threads created in the parent process before!	

I then search this problem in google, and finally find this [post](http://www.linuxprogrammingblog.com/threads-and-fork-think-twice-before-using-them), I find that this is a quite interesting problem to use `fork` in multi-threaded program, this blog explain a little about why `fork` not support multi-thread, as well as some problems caused by this implementation, the information can also be found [here](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_atfork.html). I will excerpt some of the key points:

#### What happens when `fork()`

The fork(2) function creates a copy of the process, all memory pages are copied, open file descriptors are copied etc. One important thing that differs the child process from the parent is that *the child has only one thread*. 

#### Why?

Cloning the whole process with all threads would be problematic and in most cases not what the programmer wants. Just think about it: what to do with threads that are suspended executing a system call? 

It means, when there's thread waiting for system call return from kernel thread, if the `fork` causes multi-threaded child process, since there's only one return from kernel thread, other threads may not have corresponding kernel threads return back, which may lead to unexpected problem.

#### What are the problems

There are at least two serious problems with the semantics of fork() in a multi-threaded program. 

###### Critical sections, mutexes

One problem has to do with state (for example, memory) covered by mutexes. Consider the case where one thread has a mutex locked and the state covered by that mutex is inconsistent while another thread calls fork(). In the child, the mutex is in the locked state (locked by a nonexistent thread and thus can never be unlocked). Having the child simply reinitialize the mutex is unsatisfactory since this approach does not resolve the question about how to correct or otherwise deal with the inconsistent state in the child.

It is suggested that programs that use fork() call an exec function very soon afterwards in the child process, thus resetting all states. In the meantime, only a short list of async-signal-safe library routines are promised to be available.

###### Library functions

Unfortunately, this solution does not address the needs of multi-threaded libraries. Application programs may not be aware that a multi-threaded library is in use, and they feel free to call any number of library routines between the fork() and exec calls, just as they always have. Indeed, they may be extant single-threaded programs and cannot, therefore, be expected to obey new restrictions imposed by the threads library.

On the other hand, the multi-threaded library needs a way to protect its internal state during fork() in case it is re-entered later in the child process. The problem arises especially in multi-threaded I/O libraries, which are almost sure to be invoked between the fork() and exec calls to effect I/O redirection. The solution may require locking mutex variables during fork(), or it may entail simply resetting the state in the child after the fork() processing completes.

#### pthread_atfork()

The pthread_atfork() function was intended to provide multi-threaded libraries with a means to protect themselves from innocent application programs that call fork(), and to provide multi-threaded application programs with a standard mechanism for protecting themselves from fork() calls in a library routine or the application itself.

The expected usage was that the prepare handler would acquire all mutex locks and the other two fork handlers would release them.

For example, an application could have supplied a prepare routine that acquires the necessary mutexes the library maintains and supplied child and parent routines that release those mutexes, thus ensuring that the child would have got a consistent snapshot of the state of the library (and that no mutexes would have been left stranded). This is good in theory, but in reality not practical. Each and every mutex and lock in the process must be located and locked. Every component of a program including third-party components must participate and they must agree who is responsible for which mutex or lock. This is especially problematic for mutexes and locks in dynamically allocated memory. All mutexes and locks internal to the implementation must be locked, too. This possibly delays the thread calling fork() for a long time or even indefinitely since uses of these synchronization objects may not be under control of the application. A final problem to mention here is the problem of locking streams. At least the streams under control of the system (like stdin, stdout, stderr) must be protected by locking the stream with flockfile(). But the application itself could have done that, possibly in the same thread calling fork(). In this case, the process will deadlock.

Alternatively, some libraries might have been able to supply just a child routine that reinitializes the mutexes in the library and all associated states to some known value (for example, what it was when the image was originally executed). This approach is not possible, though, because implementations are allowed to fail *_init() and *_destroy() calls for mutexes and locks if the mutex or lock is still locked. In this case, the child routine is not able to reinitialize the mutexes and locks.

------

As explained, there is no suitable solution for functionality which requires non-atomic operations to be protected through mutexes and locks. This is why the POSIX.1 standard since the 1996 release requires that *the child process after fork() in a multi-threaded process only calls async-signal-safe interfaces*.
