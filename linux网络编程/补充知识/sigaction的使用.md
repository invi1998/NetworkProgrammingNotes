# sigaction的使用

## sigaction结构体定义

```c
struct sigaction
{
    void (*sa_handler)(int);
    void (*sa_sigaction)(int, siginfo_t*, void*);
    sigset_t sa_mask;
    int sa_flags;
};
```

* sa_handler：信号处理器函数的地址，亦或是常量SIG_IGN、SIG_DFL之一。仅当sa_handler是信号处理程序的地址时，亦即sa_handler的取值在SIG_IGN和SIG_DFL之外，才会对sa_mask和sa_flags字段加以处理。
* sa_sigaction：如果设置了SA_SIGINFO标志位，则会使用sa_sigaction处理函数，否则使用sa_handler处理函数。
* sa_mask：定义一组信号，在调用由sa_handler所定义的处理器程序时将阻塞该组信号，不允许它们中断此处理器程序的执行。
* sa_flags：位掩码，指定用于控制信号处理过程的各种选项。
* SA_NODEFER：捕获该信号时，不会在执行处理器程序时将该信号自动添加到进程掩码中。
* SA_ONSTACK：针对此信号调用处理器函数时，使用了由sigaltstack()安装的备选栈。
* SA_RESETHAND：当捕获该信号时，会在调用处理器函数之前将信号处置重置为默认值(即SIG_IGN)。
* SA_SIGINFO：调用信号处理器程序时携带了额外参数，其中提供了关于信号的深入信息


## 使用示例一(使用sa_handler)

```cpp
void setupSignalHandlers(void)
{
    struct sigaction act;

    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_ONSTACK | SA_RESETHAND;
    act.sa_handler = sigtermHandler;
    sigaction(SIGTERM, &act, NULL);

    return;
}

static void sigtermHandler(int sig)
{
    // TODO
}

```

## 使用示例二(使用sa_sigaction)

```cpp
// 设置信号处理
void setupSignalHandlers(void)
{
    struct sigaction act;

    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_ONSTACK | SA_RESETHAND | SA_SIGINFO;
    act.sa_sigaction = sigsegvHandler;
    sigaction(SIGSEGV, &act, NULL);

    return;
}

static void sigsegvHandler(int sig, siginfo_t *info, void *secret)
{
    // TODO
}
```

## 对段错误等致命信号的处理

当接收到段错误等致命信号时，可以先捕获该信号，做一些处理，比如保存调用堆栈信息等，再向进程发送该信号，确保程序能够以正常方式结束，比如设置生成dump文件等。

```cpp
struct sigaction act;
// TODO

sigemptyset (&act.sa_mask);
act.sa_flags = SA_NODEFER | SA_ONSTACK | SA_RESETHAND;
act.sa_handler = SIG_DFL;
sigaction (sig, &act, NULL);
kill(getpid(),sig);
```
