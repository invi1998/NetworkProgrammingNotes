linux中signal.h中对对signal的定义是：

```c
void (*signal(int signo,void (*func)(int)))(int);
```

通过typedef可以转换成这样：

```c
typedef void Sigfunc(int);
Sigfunc *signal(int,Sigfunc *);
```

也就是说，signal有两个参数，一个是int,一个是Sigfunc ,返回值也是Sigfunc ，该指针指向一个参数为int，无返回值的函数，然而，SIG_ERR的定义是这样的：

```c
#define SIG_ERR (void (*)())-1
#define SIG_DEL (void (*)())0
#define SIG_IGN (void (*)())1
```

为什么不是这样定义的呢？？

```c
#define SIG_ERR (void (*)(int))-1
#define SIG_DEL (void (*)(int))0
#define SIG_IGN (void (*)(int))1
```

在网上搜索之后找到答案，C语言中是可以这样定义的：

```c
void fun(); 
int main()
{
       fun(1,2);
} 
void fun(int i, int j)
{
      printf("%d\n",i+j);
}
```

只是将-1强制转换为一个指针，通过编译。就像#define NULL (void *)0。可以将SIG_ERR跟其他的信号理解的一样，是一个整数。
