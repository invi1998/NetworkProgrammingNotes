# （1）信号高级认识范例

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <errno.h>


//信号处理函数
void sig_usr(int signo)
{         
    if(signo == SIGUSR1)
    {
        printf("收到了SIGUSR1信号，我休息10秒......!\n");
        sleep(10);
        printf("收到了SIGUSR1信号，我休息10秒完毕，苏醒了......!\n");
    }
    else if(signo == SIGUSR2)
    {
        printf("收到了SIGUSR2信号，我休息10秒......!\n");
        sleep(10);
        printf("收到了SIGUSR2信号，我休息10秒完毕，苏醒了......!\n");
    }
    else
    {
        printf("收到了未捕捉的信号%d!\n",signo);
    }
}

int main(int argc, char *const *argv)
{
    if(signal(SIGUSR1,sig_usr) == SIG_ERR)  //系统函数，参数1：是个信号，参数2：是个函数指针，代表一个针对该信号的捕捉处理函数
    {
        printf("无法捕捉SIGUSR1信号!\n");
    }
    if(signal(SIGUSR2,sig_usr) == SIG_ERR) 
    {
        printf("无法捕捉SIGUSR2信号!\n");
    }
    
    for(;;)
    {
        sleep(1); //休息1秒    
        printf("休息1秒~~~~!\n");
    }
    printf("再见!\n");
    return 0;
}

```

主要观察这里信号处理函数里，sleep（10）这10秒钟，可能发生那些事，那些事是发生不了的

ps -eo pid,ppid,sid,tty,pgrp,comm,stat,cmd | grep -E ‘bash|PID|nginx’
用kill 发送 USR1信号给进程
（1）执行信号处理函数被卡住了10秒，这个时候因为流程回不到main()，所以main中的语句无法得到执行；
（2）在触发SIGUSR1信号并因此sleep了10秒种期间，就算你多次触发SIGUSR1信号，也不会重新执行SIGUSR1信号对应的信号处理函数,而是会等待上一个SIGUSR1信号处理函数执行完毕才 第二次执行SIGUSR1信号处理函数；换句话说：在信号处理函数被调用时，操作系统建立的新信号屏蔽字（sigprocmask()）,自动包括了正在被递送的信号，因此，保证了在处理一个给定信号的时候，如果这个信号再次发生，那么它会阻塞到对前一个信号处理结束为止；
（3）不管你发送了多少次kill -usr1信号，在该信号处理函数执行期间，后续所有的SIGUSR1信号统统被归结为一次。

比如当前正在执行SIGUSR1信号的处理程序但没有执行完毕，这个时候，你又发送来了5次SIGUSR1信号，那么当SIGUSR1信号处理程序执行完毕（解除阻塞），SIGUSR1信号的处理程序也只会被调用一次（而不会分别调用5次SIGUSR1信号的处理程序）。

kill -usr1,kill -usr2
（1）执行usr1信号处理程序，但是没执行完时，是可以继续进入到usr2信号处理程序里边去执行的，这个时候，相当于usr2信号处理程序没执行完毕
usr1信号处理程序也没执行完毕，此时再发送usr1和usr2都不会有响应
（2）既然是在执行usr1信号处理程序的时候来得usr2信号，导致又去执行了usr2信号处理程序，这就意味着：
只有usr2信号处理程序执行完毕，才会跳到usr1信号处理程序中。然后只有usr1信号处理程序执行完毕了，才会返回main主函数中继续去执行

思考：
如果我希望在我处理SIGUSR1信号，执行usr1信号处理函数的时候，如果来了SIGUSR2信号，不要跳转到usr2信号处理程序中去，（堵住/屏蔽 SIGUSR2）。可以做到，使用sigprocmask以及其他方法。

```shell
invi@inviubuntu:~$ kill -usr2 1812
invi@inviubuntu:~$ kill -usr1 1812

# 休息1秒~~~~!
# 休息1秒~~~~!
# 收到了SIGUSR2信号，我休息10秒......!
# 收到了SIGUSR1信号，我休息10秒......!
# 收到了SIGUSR1信号，我休息10秒完毕，苏醒了......!
# 收到了SIGUSR2信号，我休息10秒完毕，苏醒了......!
# 休息1秒~~~~!
# 休息1秒~~~~!
# 休息1秒~~~~!
```

# （2）服务器架构初步

通讯架构源代码

## （2.1）目录结构规划

nginx
 ├─app
 ├─misc
 ├─net
 ├─proc
 ├─signal
 └─_include

_include：专门存放各种头文件
app：放主应用程序的，.c（main函数所在文件）以及一些比较核心的文件
    link_obj：临时目录。会存放临时的.o文件，这个目录不会手工创建，后续使用makefile脚本创建
    deep：临时目录，会存放临时的.d开头的一些依赖文件

## （2.2）编译工具make的使用概述

## （2.3）makefile脚本用法介绍

## （2.4）makefile脚本具体实现讲解
