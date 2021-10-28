[TOC]

首先说明一下，在Linux编写多线程程序需要包含头文件

```c++
＃include <pthread.h> 
```

当然，进包含一个头文件是不能搞定线程的，还需要连接`libpthread.so`这个库，因此在程序链接阶段应该有类似 
`gcc program.o -o program -lpthread`

## 第一个例子

在Linux下创建的线程的API接口是`pthread_create()`，它的完整定义是：

```cpp
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void*), void *arg); 
```

> 函数参数： 
>
> 1. 线程句柄 thread：当一个新的线程调用成功之后，就会通过这个参数将线程的句柄返回给调用者，以便对这个线程进行管理。 
> 2. 入口函数 start_routine()： 当你的程序调用了这个接口之后，就会产生一个线程，而这个线程的入口函数就是start_routine()。**如果线程创建成功，这个接口会返回0**。 
> 3. 入口函数参数 *arg : `start_routine()`函数有一个参数，这个参数就是`pthread_create`的最后一个参数arg。这种设计可以在线程创建之前就帮它准备好一些专有数据，最典型的用法就是使用C++编程时的`this`指针。`start_routine()`有一个返回值，这个返回值可以通过`pthread_join()`接口获得。 
> 4. 线程属性 attr： `pthread_create()`接口的第二个参数用于设置线程的属性。这个参数是可选的，当不需要修改线程的默认属性时，给它传递NULL就行。具体线程有那些属性，我们后面再做介绍。

好，那么我们就利用这些接口，来完成在Linux上的第一个多线程程序，见代码1所示：

**代码1:** 第一个多线程编程范例

```c++
#include <stdio.h> 
#include <pthread.h>  

void* thread( void *arg )  
{  
    printf( "This is a thread and arg = %d.\n", *(int*)arg); 
    *(int*)arg = 0;  
    return arg;  
}  


int main( int argc, char *argv[] )
{  
    pthread_t th;  
    int ret;  
    int arg = 10;  
    int *thread_ret = NULL;  
    ret = pthread_create( &th, NULL, thread, &arg ); 
    if( ret != 0 ){  
        printf( "Create thread error!\n"); 
        return -1;  
    }  
    printf( "This is the main process.\n" );
    pthread_join( th, (void**)&thread_ret );
    printf( "thread_ret = %d.\n", *thread_ret ); 
    return 0;  
}  
```

将这段代码保存为`thread.c`文件，可以执行下面的命令来生成可执行文件： 
`$ gcc thread.c -o thread -lpthread` 
这段代码的执行结果可能是这样：

```erlang
$ ./thread
This is the main process.
This is a thread and arg = 10.
thread_ret = 0.
```

注意，我说的是可能有这样的结果，在不同的环境下可能会有出入。因为这是多线程程序，线程代码可能先于第24行代码被执行。

> 代码分析： 
> \- 在第18行调用`pthread_create()`接口创建了一个新的线程，这个线程的入口函数是`thread()`，并且给这个入口函数传递了一个参数，且参数值为10。 
> \- 这个新创建的线程要执行的任务非常简单，只是将显示“This is a thread and arg = 10”这个字符串，因为arg这个参数值已经定义好了，就是10。之后线程将arg参数的值修改为0，并将它作为线程的返回值返回给系统。
>
> `pthread_join()`这个接口的第一个参数就是新创建线程的句柄了，而第二个参数就会去接受线程的返回值。`pthread_join()`接口会**阻塞主进程的执行**，直到合并的线程执行结束。由于线程在结束之后会将0返回给系统，那么`pthread_join()`获得的线程返回值自然也就是0。输出结果“thread_ret = 0”也证实了这一点。

那么现在有一个问题，那就是`pthread_join()`接口干了什么？什么是线程合并呢？

## 线程的合并与分离

### 线程的合并

我们首先要明确的一个问题就是什么是线程的合并。从前面的叙述中读者们已经了解到了，`pthread_create()`接口负责创建了一个线程。那么线程也属于系统的资源，这跟内存没什么两样，而且线程本身也要占据一定的内存空间。

众所周知的一个问题就是C/C++编程中如果要通过`malloc()`或`new`分配了一块内存，就必须使用`free()`或`delete`来回收这块内存，否则就会产生著名的**内存泄漏问题**。

既然线程和内存没什么两样，那么有创建就必须得有回收，否则就会产生另外一个著名的**资源泄漏问题**，这同样也是一个严重的问题。那么线程的合并就是回收线程资源了。

线程的合并是一种**主动回收线程资源**的方案。当一个进程或线程调用了针对其它线程的`pthread_join()`接口，就是线程合并了。这个接口**会阻塞调用进程或线程，直到被合并的线程结束为止**。当被合并线程结束，`pthread_join()`接口就会回收这个线程的资源，并将这个线程的返回值返回给合并者。

### 线程的分离

与线程合并相对应的另外一种线程资源回收机制是**线程分离**，调用接口是`pthread_detach()`。

线程分离是将线程资源的回收工作交由**系统自动来完成**，也就是说当被分离的线程结束之后，系统会自动回收它的资源。因为线程分离是启动系统的自动回收机制，那么程序也就**无法获得被分离线程的返回值**，这就使得`pthread_detach()`接口只要拥有一个参数就行了，那就是被分离线程句柄。

线程合并和线程分离都是用于**回收线程资源**的，可以根据不同的业务场景酌情使用。不管有什么理由，你都必须选择其中一种，否则就会引发资源泄漏的问题，这个问题与内存泄漏同样可怕。

## 线程的属性

前面还说到过线程是有属性的，这个属性由一个线程属性对象来描述。线程属性对象由`pthread_attr_init()`接口初始化，并由`pthread_attr_destory()`来销毁，它们的完整定义是：

```cpp
int pthread_attr_init(pthread_attr_t *attr); 

int pthread_attr_destory(pthread_attr_t *attr);  
```

> 线程拥有哪些属性呢？ 
> 一般地，Linux下的线程有：绑定属性、分离属性、调度属性、堆栈大小属性和满占警戒区大小属性。

下面我们就分别来介绍这些属性。

### 绑定属性

说到这个绑定属性，就不得不提起另外一个概念：轻进程（Light Weight Process，简称LWP）。

> 轻进程和Linux系统的内核线程拥有相同的概念，属于内核的调度实体。一个轻进程可以控制一个或多个线程。
>
> 在计算机操作系统中,轻量级进程（LWP）是一种实现多任务的方法。与普通进程相比，LWP与其他进程共享所有（或大部分）它的逻辑地址空间和系统资源；与线程相比，**LWP有它自己的进程标识符**，并和其他进程有着父子关系；这是和类Unix操作系统的系统调用vfork()生成的进程一样的。另外，线程既可由应用程序管理，又可由内核管理，而LWP只能由内核管理并像普通进程一样被调度。Linux内核是支持LWP的典型例子。

默认情况下，对于一个拥有n个线程的程序，启动多少轻进程，由哪些轻进程来控制哪些线程由操作系统来控制，这种状态被称为非绑定的。那么绑定的含义就很好理解了，只要指定了某个线程“绑”在某个轻进程上，就可以称之为绑定的了。

被绑定的线程具有**较高的响应速度**，因为操作系统的调度主体是轻进程，绑定线程可以保证在需要的时候它总有一个轻进程可用。绑定属性就是干这个用的。

设置绑定属性的接口是pthread_attr_setscope()，它的完整定义是：

```cpp
int pthread_attr_setscope(pthread_attr_t *attr, int scope);
```

它有两个参数，第一个就是线程属性对象的指针，第二个就是绑定类型，拥有两个取值：`PTHREAD_SCOPE_SYSTEM`（绑定的）和`PTHREAD_SCOPE_PROCESS`（非绑定的）。代码2演示了这个属性的使用。

**代码2 :** 设置线程绑定属性

```cpp
#include <stdio.h> 
#include <pthread.h> 
……  
int main( int argc, char *argv[] )
{
    pthread_attr_t attr;  
    pthread_t th;  
    ……  
    pthread_attr_init( &attr ); 
    pthread_attr_setscope( &attr, PTHREAD_SCOPE_SYSTEM );  
    pthread_create( &th, &attr, thread, NULL );  
    ……  
}  
```

不知道你是否在这里发现了本文的矛盾之处。就是这个绑定属性跟我们之前说的NPTL有矛盾之处。

在介绍NPTL的时候就说过业界有一种m:n的线程方案，就跟这个绑定属性有关。但是笔者还说过NPTL因为Linux的“蠢”没有采取这种方案，而是采用了“1:1”的方案。这也就是说，Linux的线程永远都是绑定。

对，Linux的线程永远都是绑定的，所以`PTHREAD_SCOPE_PROCESS`在Linux中不管用，而且会返回ENOTSUP错误。

既然Linux并不支持线程的非绑定，为什么还要提供这个接口呢？答案就是兼容！因为Linux的NTPL是号称POSIX标准兼容的，而绑定属性正是POSIX标准所要求的，所以提供了这个接口。如果读者们只是在Linux下编写多线程程序，可以完全忽略这个属性。如果哪天你遇到了支持这种特性的系统，别忘了我曾经跟你说起过这玩意儿：）

### 分离属性

前面说过线程能够被合并和分离，分离属性就是让线程在创建之前就决定它应该是分离的。如果设置了这个属性，就没有必要调用`pthread_join()`或`pthread_detach()`来回收线程资源了。 
设置分离属性的接口是`pthread_attr_setdetachstate()`，它的完整定义是：

```cpp
pthread_attr_setdetachstat(pthread_attr_t *attr, int detachstate); 
```

它的第二个参数有两个取值：`PTHREAD_CREATE_DETACHED`（分离的）和`PTHREAD_CREATE_JOINABLE`（可合并的，也是默认属性）。代码3演示了这个属性的使用。

**代码3 :** 设置线程分离属性

```cpp
#include <stdio.h>
#include <pthread.h>  
……  
int main( int argc, char *argv[] )
{
    pthread_attr_t attr;  
    pthread_t th; 
    ……  
    pthread_attr_init( &attr );  
    pthread_attr_setscope( &attr, PTHREAD_SCOPE_SYSTEM );  
    pthread_create( &th, &attr, thread, NULL );
    ……  
} 
```

### 调度属性

> 线程的调度属性有三个，分别是：算法、优先级和继承权。

算法

> Linux提供的线程调度算法有三个：轮询、先进先出和其它。

其中轮询和先进先出调度算法是POSIX标准所规定，而其他则代表采用Linux自己认为更合适的调度算法，所以默认的调度算法也就是其它了。 
轮询和先进先出调度算法都属于**实时调度算法**。

- 轮询指的是时间片轮转，当线程的时间片用完，系统将重新分配时间片，并将它放置在就绪队列尾部，这样可以保证具有相同优先级的轮询任务获得公平的CPU占用时间；
- 先进先出就是先到先服务，一旦线程占用了CPU则一直运行，直到有更高优先级的线程出现或自己放弃。

设置线程调度算法的接口是`pthread_attr_setschedpolicy()`，它的完整定义是：

```cpp
pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy);
```

它的第二个参数有三个取值：`SCHED_RR`（轮询）、`SCHED_FIFO`（先进先出）和`SCHED_OTHER`（其它）。

优先级

Linux的线程优先级与进程的优先级不一样，进程优先级我们后面再说。

> Linux的线程优先级是从1到99的数值，数值越大代表优先级越高。 
> 而且要注意的是，只有采用SHCED_RR或SCHED_FIFO调度算法时，优先级才有效。对于采用SCHED_OTHER调度算法的线程，其优先级恒为0。

设置线程优先级的接口是`pthread_attr_setschedparam()`，它的完整定义是：

```cpp
struct sched_param {
    int sched_priority;
}  
int pthread_attr_setschedparam(pthread_attr_t *attr, struct sched_param *param);  
```

`sched_param`结构体的`sched_priority`字段就是线程的优先级了。

此外，即便采用`SCHED_RR`或`SCHED_FIFO`调度算法，线程优先级也不是随便就能设置的。首先，进程必须是以root账号运行的；其次，还需要放弃线程的继承权。什么是继承权呢？

继承权

继承权就是当创建新的线程时，新线程要继承父线程（创建者线程）的调度属性。如果不希望新线程继承父线程的调度属性，就要放弃继承权。 
设置线程继承权的接口是`pthread_attr_setinheritsched()`，它的完整定义是：

```cpp
int pthread_attr_setinheritsched(pthread_attr_t *attr, int inheritsched);  
```

它的第二个参数有两个取值：`PTHREAD_INHERIT_SCHED`（拥有继承权）和`PTHREAD_EXPLICIT_SCHED`（放弃继承权）。新线程在默认情况下是拥有继承权。 
代码4能够演示不同调度算法和不同优先级下各线程的行为，同时也展示如何修改线程的调度属性。

**代码4：**设置线程调度属性

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h> 
#include <pthread.h>  
#define THREAD_COUNT 12  

void show_thread_policy( int threadno ) 
{
    int policy;  
    struct sched_param param;
    pthread_getschedparam( pthread_self(), &policy, param ); 
    switch( policy ){  
        case SCHED_OTHER: 
            printf( "SCHED_OTHER %d\n", threadno ); 
            break; 
        case SCHED_RR: 
            printf( "SCHDE_RR %d\n", threadno ); 
            break; 
        case SCHED_FIFO: 
            printf( "SCHED_FIFO %d\n", threadno ); 
            break; 
        default:
            printf( "UNKNOWN\n");
    }
}  

void* thread( void *arg )
{
    int i, j; 
    long threadno = (long)arg; 
    printf( "thread %d start\n", threadno ); 
    sleep(1);  

    show_thread_policy( threadno ); 
    for( i = 0; i < 10; ++i ) {
        for( j = 0; j < 100000000; ++j ){} 
        printf( "thread %d\n", threadno );  
    }  
    printf( "thread %d exit\n", threadno );
    return NULL;  
}  

int main( int argc, char *argv[] )
{
    long i;  
    pthread_attr_t attr[THREAD_COUNT]; 
    pthread_t pth[THREAD_COUNT];  

    struct sched_param param;  

    for( i = 0; i < THREAD_COUNT; ++i )  
        pthread_attr_init( &attr[i] );  
        for( i = 0; i < THREAD_COUNT / 2; ++i ) {
            param.sched_priority = 10;
            pthread_attr_setschedpolicy( &attr[i], SCHED_FIFO ); 
            pthread_attr_setschedparam( &attr[i], param );  
            pthread_attr_setinheritsched( &attr[i], PTHREAD_EXPLICIT_SCHED );  
        }  

        for( i = THREAD_COUNT / 2; i < THREAD_COUNT; ++i ) { 
            param.sched_priority = 20;   
            pthread_attr_setschedpolicy( &attr[i], SCHED_FIFO );  
            pthread_attr_setschedparam( &attr[i], param );  
            pthread_attr_setinheritsched( &attr[i], PTHREAD_EXPLICIT_SCHED );  
        }  

        for( i = 0; i < THREAD_COUNT; ++i )  
            pthread_create( &pth[i], &attr[i], thread, (void*)i );
    
        for( i = 0; i < THREAD_COUNT; ++i )
            pthread_join( pth[i], NULL );    
    
        for( i = 0; i < THREAD_COUNT; ++i )   
            pthread_attr_destroy( &attr[i] );
    return 0;   
}  
```

这段代码中含有一些没有介绍过的接口，读者们可以使用Linux的联机帮助来查看它们的具体用法和作用。

### 堆栈大小属性

从前面的这些例子中可以了解到，线程的主函数与程序的主函数main()有一个很相似的特性，那就是可以拥有局部变量。虽然同一个进程的线程之间是共享内存空间的，但是它的局部变量确并不共享。原因就是局部变量存储在堆栈中，而不同的线程拥有不同的堆栈。Linux系统为每个线程默认分配了**8MB的堆栈空间**，如果觉得这个空间不够用，可以通过修改线程的堆栈大小属性进行扩容。 
修改线程堆栈大小属性的接口是`pthread_attr_setstacksize()`，它的完整定义为：

```cpp
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);  
```

它的第二个参数就是堆栈大小了，以字节为单位。需要注意的是，线程堆栈不能小于16KB，而且尽量按**4KB(32位系统)或2MB（64位系统）**的整数倍分配，也就是内存页面大小的整数倍。此外，修改线程堆栈大小是有风险的，如果你不清楚你在做什么，最好别动它（其实我很后悔把这么危险的东西告诉了你:）。

### 满栈警戒区属性

既然线程是有堆栈的，而且还有大小限制，那么就一定会出现将堆栈用满的情况。线程的堆栈用满是非常危险的事情，因为这可能会导致对内核空间的破坏，一旦被有心人士所利用，后果也不堪设想。为了防治这类事情的发生，Linux为线程堆栈设置了一个满栈警戒区。这个区域一般就是一个页面，属于线程堆栈的一个扩展区域。一旦有代码访问了这个区域，就会发出`SIGSEGV`信号进行通知。

虽然满栈警戒区可以起到安全作用，但是也有弊病，就是会白白浪费掉内存空间，对于内存紧张的系统会使系统变得很慢。所有就有了关闭这个警戒区的需求。同时，如果我们修改了线程堆栈的大小，那么系统会认为我们会自己管理堆栈，也会将警戒区取消掉，如果有需要就要开启它。 
修改满栈警戒区属性的接口是`pthread_attr_setguardsize()`，它的完整定义为：

```cpp
int pthread_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);  
```

它的第二个参数就是警戒区大小了，以字节为单位。与设置线程堆栈大小属性相仿，应该尽量按照4KB或2MB的整数倍来分配。当设置警戒区大小为0时，就关闭了这个警戒区。 
虽然栈满警戒区需要浪费掉一点内存，但是能够极大的提高安全性，所以这点损失是值得的。而且一旦修改了线程堆栈的大小，一定要记得同时设置这个警戒区。

## 线程本地存储

内线程之间可以共享内存地址空间，线程之间的数据交换可以非常快捷，这是线程最显著的优点。但是多个线程访问共享数据，需要昂贵的同步开销，也容易造成与同步相关的BUG，更麻烦的是有些数据根本就不希望被共享，这又是缺点。可谓：“成也萧何，败也萧何”，说的就是这个道理。

C程序库中的errno是个最典型的一个例子。errno是一个全局变量，会保存最后一个系统调用的错误代码。在单线程环境并不会出现什么问题。但是在多线程环境，由于所有线程都会有可能修改errno，这就很难确定errno代表的到底是哪个系统调用的错误代码了。这就是有名的“非线程安全（Non Thread-Safe）”的。

此外，从现代技术角度看，在很多时候使用多线程的目的并不是为了对共享数据进行并行处理（在Linux下有更好的方案，后面会介绍）。更多是由于多核心CPU技术的引入，为了充分利用CPU资源而进行并行运算（不互相干扰）。换句话说，大多数情况下每个线程只会关心自己的数据而不需要与别人同步。

为了解决这些问题，可以有很多种方案。比如使用不同名称的全局变量。但是像errno这种名称已经固定了的全局变量就没办法了。在前面的内容中提到在线程堆栈中分配局部变量是不在线程间共享的。但是它有一个弊病，就是线程内部的其它函数很难访问到。

目前解决这个问题的简便易行的方案是线程本地存储，即**Thread Local Storage**，简称**TLS**。利用TLS，errno所反映的就是本线程内最后一个系统调用的错误代码了，也就是线程安全的了。 
Linux提供了对TLS的完整支持，通过下面这些接口来实现：

```cpp
int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));  
int pthread_key_delete(pthread_key_t key);  
void* pthread_getspecific(pthread_key_t key);  
int pthread_setspecific(pthread_key_t key, const void *value); 
```

> pthread_key_create()接口用于创建一个线程本地存储区。 
> \- key : 第一个参数用来返回这个存储区的句柄，需要使用一个全局变量保存，以便所有线程都能访问到。 
> \- *destructor : 第二个参数是线程本地数据的回收函数指针，如果希望自己控制线程本地数据的生命周期，这个参数可以传递NULL。
>
> pthread_key_delete()接口用于回收线程本地存储区。其唯一的参数就要回收的存储区的句柄。
>
> pthread_getspecific()和pthread_setspecific()这个两个接口分别用于获取和设置线程本地存储区的数据。这两个接口在不同的线程下会有不同的结果不同（相同的线程下就会有相同的结果），这也就是线程本地存储的关键所在。

代码5展示了如何在Linux使用线程本地存储，注意执行结果，分析一下线程本地存储的一些特性,以及内存回收的时机。

**代码5 :**使用线程本地存储

```cpp
#include <stdio.h>  
#include <stdlib.h>  
#include <pthread.h>  
#define THREAD_COUNT 10  

pthread_key_t g_key;  

typedef struct thread_data{  
    int thread_no;  
} thread_data_t;  

void show_thread_data()  
{  
    thread_data_t *data = pthread_getspecific( g_key ); 
    printf( "Thread %d \n", data->thread_no );  
}  



void* thread( void *arg )  
{  
    thread_data_t *data = (thread_data_t *)arg;  
    printf( "Start thread %d\n", data->thread_no ); 
    pthread_setspecific( g_key, data );  
    show_thread_data();  
    printf( "Thread %d exit\n", data->thread_no );  
}  

void free_thread_data( void *arg )  
{  
    thread_data_t *data = (thread_data_t*)arg;  
    printf( "Free thread %d data\n", data->thread_no );  
    free( data );  
}  

int main( int argc, char *argv[] ) 
{  
    int i;  
    pthread_t pth[THREAD_COUNT];  
    thread_data_t *data = NULL;  
    pthread_key_create( &g_key, free_thread_data );  
    
    for( i = 0; i < THREAD_COUNT; ++i ) {  
        data = malloc( sizeof( thread_data_t ) );  
        data->thread_no = i;  
        pthread_create( &pth[i], NULL, thread, data );  
    }  
    
    for( i = 0; i < THREAD_COUNT; ++i ) 
        pthread_join( pth[i], NULL );  
    pthread_key_delete( g_key );  
    return 0;  
} 
```

## 线程的同步

虽然线程本地存储可以避免线程访问共享数据，但是线程之间的大部分数据始终还是共享的。在涉及到对共享数据进行读写操作时，就必须使用同步机制，否则就会造成线程们哄抢共享数据的结果，这会把你的数据弄的七零八落理不清头绪。 
Linux提供的线程同步机制主要有**互斥锁**和**条件变量**。其它形式的线程同步机制用得并不多，本书也不准备详细讲解，有兴趣的读者可以参考相关文档。

### 互斥锁

首先我们看一下互斥锁。所谓的互斥就是线程之间互相排斥，获得资源的线程排斥其它没有获得资源的线程。Linux使用互斥锁来实现这种机制。

既然叫锁，就有加锁和解锁的概念。当线程获得了加锁的资格，那么它将独享这个锁，其它线程一旦试图去碰触这个锁就立即被系统“拍晕”。当加锁的线程解开并放弃了这个锁之后，那些被“拍晕”的线程会被系统唤醒，然后继续去争抢这个锁。至于谁能抢到，只有天知道。但是总有一个能抢到。于是其它来凑热闹的线程又被系统给“拍晕”了……如此反复。感觉线程的“头”很痛：）

从互斥锁的这种行为看，线程加锁和解锁之间的代码相当于一个独木桥，同意时刻只有一个线程能执行。从全局上看，**在这个地方，所有并行运行的线程都变成了排队运行了**。比较专业的叫法是同步执行，这段代码区域叫临界区。同步执行就破坏了线程并行性的初衷了，临界区越大破坏得越厉害。所以在实际应用中，应该尽量避免有临界区出现。**实在不行，临界区也要尽量的小**。如果连缩小临界区都做不到，那还使用多线程干嘛？

互斥锁在Linux中的名字是`mutex`。这个似乎优点眼熟。对，在前面介绍NPTL的时候提起过，但是那个叫futex，是系统底层机制。对于提供给用户使用的则是这个`mutex`。Linux初始化和销毁互斥锁的接口是`pthread_mutex_init()`和`pthead_mutex_destroy()`，

对于加锁和解锁则有`pthread_mutex_lock()`、`pthread_mutex_trylock()`和`pthread_mutex_unlock()`。 
这些接口的完整定义如下：

```cpp
int pthread_mutex_init(pthread_mutex_t *restrict mutex,const pthread_mutexattr_t *restrict attr);  
int pthread_mutex_destory(pthread_mutex_t *mutex );  
int pthread_mutex_lock(pthread_mutex_t *mutex);  
int pthread_mutex_trylock(pthread_mutex_t *mutex);  
int pthread_mutex_unlock(pthread_mutex_t *mutex);  
```

从这些定义中可以看到，互斥锁也是有属性的。只不过这个属性在绝大多数情况下都不需要改动，所以使用默认的属性就行。方法就是给它传递NULL。

`phtread_mutex_trylock()`比较特别，用它试图加锁的线程永远都不会被系统“拍晕”，只是通过返回EBUSY来告诉程序员这个锁已经有人用了。至于是否继续“强闯”临界区，则由程序员决定。系统提供这个接口的目的可不是让线程“强闯”临界区的。它的根本目的还是为了提高并行性，留着这个线程去干点其它有意义的事情。当然，如果很幸运恰巧这个时候还没有人拥有这把锁，那么自然也会取得临界区的使用权。

代码6演示了在Linux下如何使用互斥锁。

**代码6 :** 使用互斥锁

```cpp
#include <stdio.h>  
#include <stdlib.h> 
#include <errno.h>  
#include <unistd.h>  
#include <pthread.h> 

pthread_mutex_t g_mutex;  

int g_lock_var = 0; 
void* thread1( void *arg )  
{  
	int i, ret;  
    time_t end_time;  
    end_time = time(NULL) + 10;  
    while( time(NULL) < end_time ) {  
        ret = pthread_mutex_trylock( &g_mutex );  
        if( EBUSY == ret ) {  
            printf( "thread1: the varible is locked by thread2.\n" );  
        } else {  
            printf( "thread1: lock the variable!\n" );  
            ++g_lock_var;  
            pthread_mutex_unlock( &g_mutex );  
        }  
        sleep(1);  
    }  
    return NULL;  
}  

void* thread2( void *arg )  
{  
    int i;  
    time_t end_time; 
    end_time = time(NULL) + 10; 
    while( time(NULL) < end_time ) {  
        pthread_mutex_lock( &g_mutex ); 
        printf( "thread2: lock the variable!\n" );  
        ++g_lock_var;  
        sleep(1);  
        pthread_mutex_unlock( &g_mutex );
    }  
    return NULL;  
}  



int main( int argc, char *argv[] )  
{  
    int i;  
    pthread_t pth1,pth2;  
    pthread_mutex_init( &g_mutex, NULL ); 
    pthread_create( &pth1, NULL, thread1, NULL ); 
    pthread_create( &pth2, NULL, thread2, NULL );  
    pthread_join( pth1, NULL );  
    pthread_join( pth2, NULL );  
    pthread_mutex_destroy( &g_mutex );  
    printf( "g_lock_var = %d\n", g_lock_var );  
    return 0;   
}  
```

> 最后需要补充一点，互斥锁在同一个线程内，没有互斥的特性。也就是说，线程不能利用互斥锁让系统将自己“拍晕”。解释这个现象的一个很好的理由就是，拥有锁的线程把自己“拍晕”了，谁还能再拥有这把锁呢？但是另外情况需要避免，就是两个线程已经各自拥有一把锁了，但是还想得到对方的锁，这个时候两个线程都会被“拍晕”。一旦这种情况发生，就谁都不能获得这个锁了，这种情况还有一个著名的名字——**死锁**。死锁是永远都要避免的事情，因为这是严重损人不利己的行为。

### 条件变量

条件变量关键点在“变量”上。与锁的不同之处就是，当线程遇到这个“变量”，并不是类似锁那样的被系统给“拍晕”，而是根据“条件”来选择是否在那里等待。等待什么呢？等待允许通过的“信号”。这个“信号”是系统控制的吗？显然不是！它是由另外一个线程来控制的。

如果说互斥锁可以比作独木桥，那么条件变量这就好比是马路上的红绿灯。车辆遇到红绿灯肯定会根据“灯”的颜色来判断是否通行，毕竟红灯停绿灯行这个道理在幼儿园的时候老师就教了。那么谁来控制“灯”的颜色呢？一定是交警啊，至少你我都不敢动它（有人会说那是自动的，可是间隔多少时间变换也是交警设置不是？）。那么“车辆”和“交警”就是马路上的两类线程，大多数情况下都是“车”多“交警”少。

更深一步理解，条件变量是一种事件机制。**由一类线程来控制“事件”的发生，另外一类线程等待“事件”的发生**。为了实现这种机制，条件变量必须是共享于线程之间的全局变量。而且，**条件变量也需要与互斥锁同时使用**。

初始化和销毁条件变量的接口是`pthread_cond_init()`和`pthread_cond_destory()`; 
控制“事件”发生的接口是`pthread_cond_signal()`或`pthread_cond_broadcast()`； 
等待“事件”发生的接口是`pthead_cond_wait()`或`pthread_cond_timedwait()`。 
它们的完整定义如下：

```cpp
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr);  

int pthread_cond_destory(pthread_cond_t *cond);  

int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);  

int pthread_cond_timedwait(pthread_cond_t *cond,pthread_mutex_t *mutex, const timespec *abstime); 

int pthread_cond_signal(pthread_cond_t *cond);  

int pthread_cond_broadcast(pthread_cond_t *cond); 
```

对于等待“事件”的接口从其名称中可以看出，一种是无限期等待，一种是限时等待。后者与互斥锁的`pthread_mutex_trylock()`有些类似，即当等待的“事件”经过一段时间之后依然没有发生，那就去干点别的有意义的事情去。

而对于控制“事件”发生的接口则有“单播”和“广播”之说。所谓单播就是只有一个线程会得到“事件”已经发生了的“通知”，而广播就是所有线程都会得到“通知”。**对于广播情况，所有被“通知”到的线程也要经过由互斥锁控制的独木桥。**

对于条件变量的使用，可以参考代码7，它实现了一种生产者与消费者的线程同步方案。

**代码7 :** 使用条件变量

```cpp
#include <stdio.h>  

#include <stdlib.h>  

#include <pthread.h>  

#define BUFFER_SIZE 5  

pthread_mutex_t g_mutex;  

pthread_cond_t g_cond;  

typedef struct {  
    char buf[BUFFER_SIZE]; 
    int count;  
} buffer_t;  

buffer_t g_share = {"", 0};  

char g_ch = 'A';  

void* producer( void *arg )  
{  
    printf( "Producer starting.\n" );  

    while( g_ch != 'Z' ) {  

        pthread_mutex_lock( &g_mutex );  

        if( g_share.count < BUFFER_SIZE ) {  

            g_share.buf[g_share.count++] = g_ch++;  

            printf( "Prodcuer got char[%c]\n", g_ch - 1 );  

            if( BUFFER_SIZE == g_share.count ) {  

                printf( "Producer signaling full.\n" );  

                pthread_cond_signal( &g_cond );  

            }  
        }  

        pthread_mutex_unlock( &g_mutex );  

    }  

    printf( "Producer exit.\n" );  

    return NULL;  

}  



void* consumer( void *arg )  
{  

    int i;  
    printf( "Consumer starting.\n" );  
    while( g_ch != 'Z' ) {  

        pthread_mutex_lock( &g_mutex );  

        printf( "Consumer waiting\n" );  

        pthread_cond_wait( &g_cond, &g_mutex );  

        printf( "Consumer writing buffer\n" );  

        for( i = 0; g_share.buf[i] && g_share.count; ++i ) {  

            putchar( g_share.buf[i] );  

            --g_share.count;  

        }  

        putchar('\n');  
        pthread_mutex_unlock( &g_mutex );  
    }  



    printf( "Consumer exit.\n" );  

    return NULL;  
}  

int main( int argc, char *argv[] )  
{  
    pthread_t ppth, cpth;  

    pthread_mutex_init( &g_mutex, NULL );  

    pthread_cond_init( &g_cond, NULL );  

    pthread_create( &cpth, NULL, consumer, NULL );  

    pthread_create( &ppth, NULL, producer, NULL );  

    pthread_join( ppth, NULL );  

    pthread_join( cpth, NULL );  

    pthread_mutex_destroy( &g_mutex );  

    pthread_cond_destroy( &g_cond );  

    return 0;  

}  
```

> 这段代码存在一个潜在的问题： 
> 如果producer线程并行执行的比consumer快，producer线程会先获取锁，之后向consumer发出信号，但此时consumer没办法获取锁，也就执行不到`pthead_cond_wait()` 处，那么程序就陷入尴尬的境地，发生死锁。
>
> 简单的，可以在 `pthread_create( &cpth, NULL, consumer, NULL );` 和`pthread_create( &ppth, NULL, producer, NULL );` 之间加入一个长的延时函数usleep(100)，确保consumer线程先行执行到`pthead_cond_wait()` 处。

从代码中会发现，等待“事件”发生的接口都需要传递一个互斥锁给它。而实际上这个互斥锁还要在调用它们之前加锁，调用之后解锁。不单如此，在调用操作“事件”发生的接口之前也要加锁，调用之后解锁。这就面临一个问题，按照这种方式，等于“发生事件”和“等待事件”是互为临界区的。也就是说，如果“事件”还没有发生，那么有线程要等待这个“事件”就会阻止“事件”的发生。更干脆一点，就是这个“生产者”和“消费者”是在来回的走独木桥。但是实际的情况是，“消费者”在缓冲区满的时候会得到这个“事件”的“通知”，然后将字符逐个打印出来，并清理缓冲区。直到缓冲区的所有字符都被打印出来之后，“生产者”才开始继续工作。

为什么会有这样的结果呢？这就要说明一下`pthread_cond_wait()`接口对互斥锁做什么。答案是：**解锁**。`pthread_cond_wait()`首先会**解锁互斥锁，然后进入等待**。这个时候“生产者”就能够进入临界区，然后在条件满足的时候向“消费者”发出信号。

当`pthead_cond_wait()`获得“通知”之后，它还要对**互斥锁加锁**，这样可以防止“生产者”继续工作而“撑坏”缓冲区。另外，“生产者”在缓冲区不满的情况下才能工作的这个限定条件是很有必要的。因为在`pthread_cond_wait()`获得通知之后，在没有对互斥锁加锁之前，“生产者”可能已经重新进入临界区了，这样“消费者”又被堵住了。也就是因为条件变量这种工作性质，导致它必须与互斥锁联合使用。

此外，利用条件变量和互斥锁，可以模拟出很多其它类型的线程同步机制，比如：event、semaphore等。本书不再多讲，有兴趣的读者可以参考其它著作。或自己根据它们的行为自己来模拟实现。