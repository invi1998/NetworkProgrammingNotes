# （1）信号集

情景回顾：
上一章我们写了一个信号处理函数，接收SIGUSR1和SIGUSR2信号，当这些信号来的时候，我们进行相应的处理函数
那么现在有这样一个疑问：
假设 SIGUSR1 这个信号来了，我们去执行这个对应的信号处理函数，在这个函数执行期间，突然又来了一个SIGUSR1信号
（也就是说，在我信号处理函数没有执行完的时候，又来一个相同的信号）那么系统会不会再次触发这个信号处理函数呢?

一般是不会的，也就是说当某个信号来得时候，导致系统去执行对应的信号处理函数，那么系统通常会屏蔽或者忽略这个相同的信号
将这个信号进行堵塞，一直堵到信号处理函数执行结束

那么这就要求，当前进程必须记住阻塞了那些信号

比如：我们一开始有一堆 0000000000000 ，这里每一个 0 带表一个信号，突然间来一个 SIGUSR1信号，那么我就需要把这个SIGUSR1的位置置为1，
即：0000010000000，置为1后那么我就要去调用SIGUSR1对应的信号处理函数，在函数执行期间，如果再来一个SIGUSR1信号，那么系统会去判断，这个信号
是不是1，如果是1那么说明我这个信号还没处理完，那么这个新来的SIGUSR1就得排队，不能打断我当前的信号处理函数，信号处理函数执行完之后，系统会把这个1置为0， 0000000000000 然后这个排队的SIGUSR1信号才能再次把个标志置为1,然后去执行信号处理函数

因为linux里大概有60多种信号，所以我们需要一个叫做 信号集 的数据类型（结构），这个信号集能够把这些信号来或者没来都装起来
0000000000,0000000000,0000000000,00,0000000000,0000000000,0000000000,00 (64个二进制位)
这里每一个0都表示一种信号，来了置为1，没来为0，置为1后处理相应的信号处理函数，处理完置为0
相同信号来得时候，这个信号已经被置为1了，那就说明该信号正在处理中，那么这个信号就得进行排队等待
同时，如果相同信号在信号处理函数期间来了多次，那么这些信号会被合并为一次

linux中是使用 sigset_t结构类型来表示信号集的

```c
typedef struct 
{
    unsigned long sig[2]; // 1表示来，0表示没来，所以正好两个无符号 long，64位二进制数
}sigset_t
```

信号集的定义：信号集表示一组信号的来 [1] 或者没来 [2]
信号集的相关数据类型：sigset_t;

# （2）信号相关函数

概念：

未决信号集：没有被当前进程处理的信号
阻塞信号集：将某个信号放到阻塞信号集，这个信号就不会被处理了。一般是人为的。阻塞解除，信号就变成递达状态，然后被处理
阻塞信号集和未决信号集存在于内核区的pcb中
阻塞信号集和未决信号集、自定义信号集的关系
信号被发送后，在处理之前要先查询阻塞信号的对应状态，如果为1，表示对应的信号应该阻塞，此时将不处理该信号，并将未决信号集中该信号对应的标志设置为1。如果该信号在阻塞信号集中的标志为0，表示对应的未决信号不应被阻塞，此时处理该信号集。阻塞信号一般是人为设置的，但是不能直接设置阻塞信号集，只能先设置自定义信号集，然后通过 接口函数将自定义信号集设置给阻塞信号。

一个进程，里面会有一个信号集，用来记录当前屏蔽（阻塞）了那些信号

将自定义信号集set设置为空,把信号集中所有信号都清零

```c
int sigemptyset(sigset_t* set);
```

将所有信号加入自定义信号集set（把信号集中所有信号都设置为1）

```c
int sigfillset(sigset_t* set);
```

将某个信号加入自定义信号集set

```c
int sigaddset(sigset_t* set,int signo);
```

将某个信号从自定义信号集中移除

```c
int sigdelset(sigset_t* set,int signo);
```

判断某个信号是否存在在自定义信号集中

```c
int sigismember(sigset_t* set,int signo);
```

将自定义信号集设置给阻塞信号集

```c
int sigprocmask(int how,const sigset_t* set,sigset_t* oldset);
// 执行成功返回0，执行失败返回-1
// 参数how的取值不同，带来的操作行为也不同，该参数可选值如下：
// 1．SIG_BLOCK:　该值代表的功能是将newset所指向的信号集中所包含的信号加到当前的信号掩码中，作为新的信号屏蔽字。
// 2．SIG_UNBLOCK:将参数newset所指向的信号集中的信号从当前的信号掩码中移除。
// 3．SIG_SETMASK:设置当前信号掩码参数为newset所指向的信号集中所包含的信号。
// 注意事项：sigprocmask()函数只为单线程的进程定义的，在多线程中要使用pthread_sigmask变量，在使用之前需要声明和初始化。



// 一个进程，里边会有一个信号集，用来记录当前屏蔽（阻塞)了哪些信号；
// 如果我们把这个信号集中的某个信号位设置为1，就表示屏蔽了同类信号，此时再来个同类信号，那么同类信号会被屏蔽，不能传递给进程；
// 如果这个信号集中有很多个信号位都被设置为1，那么所有这些被设置为1的信号都是属于当前被阻塞的而不能传递到该进程的信号；
// sigprocmask()函数，就能够设置该进程所对应的信号集中的内容；
```

how参数

```c
   SIG_BLOCK//将指定信号设置到阻塞信号集中mask=mask|set
          The set of blocked signals is the union of the current set and the set argument.

   SIG_UNBLOCK//将指定信号从阻塞信号集中清除mask=mask&~set
          The signals in set are removed from the current set of blocked signals.  It  is  permissible  to  attempt  to
          unblock a signal which is not blocked.

   SIG_SETMASK//相当于用自定义信号集替换阻塞信号集mask=set
          The set of blocked signals is set to the argument set.
```

set参数

```c
set参数为自定义信号集
```

oldset参数

```c
为传出参数，会将设置自定已信号集之前的阻塞信号集状态传出，如果对之前的阻塞信号集状态不感兴趣可以设置为 NULL
```

读取当前未决信号集

```c
int sigpending(sigset_t* set);
```

# （3）sigprocmask等信号函数范例演示

```c
#include <stdio.h>
#include <stdlib.h>  //malloc
#include <unistd.h>
#include <signal.h>

//信号处理函数
void sig_quit(int signo)
{   
    printf("收到了SIGQUIT信号!\n");
    if(signal(SIGQUIT,SIG_DFL) == SIG_ERR)  // 这里是把收到的SIGQUIT信号设置为系统的缺省动作（这个信号缺省是退出）
//     注意：这里是第一次进来，执行往printf后，才设置为缺省,也就是后续的SIGQUIT信号将执行信号的缺省动作
// == SIG_ERR 表示判断设置缺省处理是否失败
    {
        printf("无法为SIGQUIT信号设置缺省处理(终止进程)!\n");
        exit(1);
    }
}

int main(int argc, char *const *argv)
{
    sigset_t newmask,oldmask,pendmask; //信号集，新的信号集，原有的信号集，挂起的信号集
    if(signal(SIGQUIT,sig_quit) == SIG_ERR)  //注册信号对应的信号处理函数,"ctrl+\" 
    {        
        printf("无法捕捉SIGQUIT信号!\n");
        exit(1);   //退出程序，参数是错误代码，0表示正常退出，非0表示错误，但具体什么错误，没有特别规定，这个错误代码一般也用不到，先不管他；
    }

    sigemptyset(&newmask); //newmask信号集中所有信号都清0（表示这些信号都没有来）；
    sigaddset(&newmask,SIGQUIT); //设置newmask信号集中的SIGQUIT信号位为1，说白了，再来SIGQUIT信号时，进程就收不到，设置为1就是该信号被阻塞掉呗

    //sigprocmask()：设置该进程所对应的信号集
    if(sigprocmask(SIG_BLOCK,&newmask,&oldmask) < 0)  //第一个参数用了SIG_BLOCK表明设置 进程 新的信号屏蔽字 为 “当前信号屏蔽字 和 第二个参数指向的信号集的并集
    {                                                 //一个 ”进程“ 的当前信号屏蔽字，刚开始全部都是0的；所以相当于把当前 "进程"的信号屏蔽字设置成 newmask（屏蔽了SIGQUIT)；
                                                      //第三个参数不为空，则进程老的(调用本sigprocmask()之前的)信号集会保存到第三个参数里，用于后续，这样后续可以恢复老的信号集给线程
        printf("sigprocmask(SIG_BLOCK)失败!\n");
        exit(1);
    }
//     所以这段代码就相当于我把当前进程的信号集设置为了newmask
    printf("我要开始休息10秒了--------begin--，此时我无法接收SIGQUIT信号!\n");
    sleep(10);   //这个期间无法收到SIGQUIT信号的；
    printf("我已经休息了10秒了--------end----!\n");
    if(sigismember(&newmask,SIGQUIT))  //测试一个指定的信号位是否被置位(为1)，测试的是newmask
    {
        printf("SIGQUIT信号被屏蔽了!\n");
    }
    else
    {
        printf("SIGQUIT信号没有被屏蔽!!!!!!\n");
    }
    if(sigismember(&newmask,SIGHUP))  //测试另外一个指定的信号位是否被置位,测试的是newmask
    {
        printf("SIGHUP信号被屏蔽了!\n");
    }
    else
    {
        printf("SIGHUP信号没有被屏蔽!!!!!!\n");
    }

    //现在我要取消对SIGQUIT信号的屏蔽(阻塞)--把信号集还原回去
    if(sigprocmask(SIG_SETMASK,&oldmask,NULL) < 0) //第一个参数用了SIGSETMASK表明设置 进程  新的信号屏蔽字为 第二个参数 指向的信号集，第三个参数没用
    {
        printf("sigprocmask(SIG_SETMASK)失败!\n");
        exit(1);
    }

    printf("sigprocmask(SIG_SETMASK)成功!\n");
    
    if(sigismember(&oldmask,SIGQUIT))  //测试一个指定的信号位是否被置位,这里测试的当然是oldmask
    {
        printf("SIGQUIT信号被屏蔽了!\n");
    }
    else
    {
        printf("SIGQUIT信号没有被屏蔽，您可以发送SIGQUIT信号了，我要sleep(10)秒钟!!!!!!\n");
        int mysl = sleep(10);
        if(mysl > 0)
        {
            printf("sleep还没睡够，剩余%d秒\n",mysl);
        }
    }
    printf("再见了!\n");
    return 0;
}
```

在不同阶段输入 ctrl \ 进行测试

```shell
# 什么也不输入，程序完整运行
invi@inviubuntu:/mnt/hgfs/nginxWeb$ ./nginx3_5_1
我要开始休息10秒了--------begin--，此时我无法接收SIGQUIT信号!
我已经休息了10秒了--------end----!
SIGQUIT信号被屏蔽了!
SIGHUP信号没有被屏蔽!!!!!!
sigprocmask(SIG_SETMASK)成功!
SIGQUIT信号没有被屏蔽，您可以发送SIGQUIT信号了，我要sleep(10)秒钟!!!!!!
再见了!

# 在他休息屏蔽 ctrl \ 信号的10秒内连续发送 ctrl \ 信号
invi@inviubuntu:/mnt/hgfs/nginxWeb$ ./nginx3_5_1
我要开始休息10秒了--------begin--，此时我无法接收SIGQUIT信号!
^\^\^\^\^\^\我已经休息了10秒了--------end----!
SIGQUIT信号被屏蔽了!
SIGHUP信号没有被屏蔽!!!!!!
收到了SIGQUIT信号!
sigprocmask(SIG_SETMASK)成功!
SIGQUIT信号没有被屏蔽，您可以发送SIGQUIT信号了，我要sleep(10)秒钟!!!!!!
再见了!

# 在信号屏蔽期间不发送信号，解除屏蔽后 发送信号
invi@inviubuntu:/mnt/hgfs/nginxWeb$ ./nginx3_5_1
我要开始休息10秒了--------begin--，此时我无法接收SIGQUIT信号!
我已经休息了10秒了--------end----!
SIGQUIT信号被屏蔽了!
SIGHUP信号没有被屏蔽!!!!!!
sigprocmask(SIG_SETMASK)成功!
SIGQUIT信号没有被屏蔽，您可以发送SIGQUIT信号了，我要sleep(10)秒钟!!!!!!
^\收到了SIGQUIT信号!
sleep还没睡够，剩余9秒
再见了!
invi@inviubuntu:/mnt/hgfs/nginxWeb$ 

```

除此之外，我们还看见，sleep函数是可以被打断的
1）时间到达了，
2）来了一个信号使得sleep信号提前结束，那么此时sleep会返回一个值，这个值就是未休眠够的时间
