UNIX系统的信号机制最简单的接口是signal函数。signal函数的功能：为指定的信号安装一个新的信号处理函数。

```c
# include <signal.h>
void (*signal(int signo, void (*func)(int)))(int);
```

复杂原型分开看：

```c
void (*signal( int signo, void (*func)(int) )  )(int);
```

函数名      ：signal

函数参数   ：int signo, void (*func)(int)

返回值类型：void (*)(int);

signo参数是信号名（参见：<http://www.cnblogs.com/nufangrensheng/p/3514157.html中UNIX系统信号Signal栏下的信号名）。func的值是常量SIG_IGN、常量SIG_DFL或当接到此信号后要调用的函数的地址。如果指定SIG_IGN，则向内核表示忽略此信号（记住有两个信号SIGKILL和SIGSTOP不能忽略）。如果指定SIG_DFL，则表示接到此信号后的动作是系统默认动作。当指定函数地址时，则在信号发生时，调用该函数，我们称这种处理为“捕捉”该信号。称此函数为信号处理程序（signal> handler）或信号捕捉函数（signal-catching function）。

signal的返回值是指向之前的信号处理程序的指针。（之前的信号处理程序，也就是在执行signal（signo，func）之前，对信号signo的信号处理程序）

开头所示的signal函数原型太复杂了，如果使用下面的typedef，则可使其简单一些：

typedef void Sigfunc（int）；

然后，可将signal函数原型写成：

Sigfunc *signal（int，Sigfunc*）；

如果查看系统的头文件<signal.h>，则很可能会找到下列形式的声明：

```c
# define    SIG_ERR        ( void (*) () )-1
# define    SIG_DFL        ( void (*) () )0
# define    SIG_IGN        ( void (*) () )1
```

这些常量可用于代替“指向函数的指针，该函数需要一个整型参数，而且无返回值”。signal的第二个参数及其返回值就可用它们表示。这些常量所使用的三个值不一定是-1，0和1。但大多数UNIX系统都使用上面所示的值。
