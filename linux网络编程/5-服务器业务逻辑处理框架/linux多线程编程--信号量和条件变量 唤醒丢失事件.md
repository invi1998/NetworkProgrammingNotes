​       关于linux下信号量和条件变量的使用，在很多地方都可以找到相关文章，信号量、条件变量、互斥锁都是线程同步原语，在平时多线程编程中只要知道一两种就可以轻松搞定，我也是这么认为的，但是今天发现，有时还是有区别的。
​      在实现多线程编程中，其中有些东西是可以互相转换的，比如使用信号量可以实现条件变量，关于这三者的基本用法不在累述，我的博客中也有相关介绍，这里介绍条件变量丢失唤醒事件的事情。

在线程未获得相应的互斥锁时调用pthread_cond_signal或pthread_cond_broadcast函数可能会引起唤醒丢失问题。




唤醒丢失往往会在下面的情况下发生：

 

1、一个线程调用pthread_cond_signal或pthread_cond_broadcast函数；

 

2、另一个线程正处在测试条件变量和调用pthread_cond_wait函数之间；

3、没有线程正在处在阻塞等待的状态下。

 

下面是使用函数pthread_cond_wait（）和函数pthread_cond_signal（）的一个简单的例子：

 

```c++
pthread_mutex_t count_lock;

pthread_cond_t count_nonzero;
unsigned count;
decrement_count　() {
pthread_mutex_lock (&count_lock);
while(count==0)
pthread_cond_wait( &count_nonzero,&count_lock);
count=count -1;
pthread_mutex_unlock (&count_lock);
}
increment_count(){
pthread_mutex_lock(&count_lock);
if(count==0)
pthread_cond_signal(&count_nonzero);
count=count+1;
pthread_mutex_unlock(&count_lock);
}
```

​        count值为0时，decrement函数在pthread_cond_wait处被阻塞，并打开互斥锁count_lock。此时，当调用到函数increment_count时，pthread_cond_signal（）函数改变条件变量，告知decrement_count（）停止阻塞。读者可以试着让两个线程分别运行这两个函数，看看会出现什么样的结果。
​        函数pthread_cond_broadcast（pthread_cond_t *cond）用来唤醒所有被阻塞在条件变量cond上的线程。这些线程被唤醒后将再次竞争相应的互斥锁，所以必须小心使用这个函数。

在自己的测试过程也会发现丢失事件，即使按照非常严格的条件变量的使用方法。但是使用信号量就不会出现这种情况，因为信号量在内核中有相应的队列记录了信号的存在，不会将信号丢失。参考下面的代码：



```c++
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <string.h>
#include <semaphore.h>

void *read_func(void* args);
void *write_func(void* args);
pthread_mutex_t mutex;
pthread_cond_t cond;
sem_t sem_write;
sem_t sem_read;
int signal =0;
int count_read=0,count_write = 0;
int main()
{
    pthread_t pid1,pid2,pid3;
    pthread_mutex_init(&mutex,NULL);
    pthread_cond_init(&cond,NULL);

    sem_init(&sem_write,0,1);
    sem_init(&sem_read,0,0);
    int ret;
     
    ret=pthread_create(&pid2,NULL,write_func,NULL);
    if(ret!=0)
    {
        printf("can not create write_funx!\r\n");
    }
    usleep(10000);
    ret=pthread_create(&pid1,NULL,read_func,NULL);
    if(ret!=0)
    {
        printf("can not create read_funx!\r\n");
    }
    pthread_join(pid1,NULL);
    pthread_join(pid2,NULL);
    return 0;

}
void *read_func(void* args)
{
    for(;;)
    {
#if 0
        pthread_mutex_lock(&mutex);
        count_read++;
        signal =1;
        pthread_cond_signal(&cond);    
        pthread_mutex_unlock(&mutex);
        usleep(2);
#endif
#if 1
//        if(count_read%10000 == 0)
    //        usleep(6);
        sem_wait(&sem_write);
        count_read++;
        sem_post(&sem_read);
//        usleep(1);
    #endif
        if(count_read >= 900000)
        {
            sleep(1);
            printf("count_read is %d,count_write is %d\r\n",count_read,count_write);
//            sleep(1);
            exit(0);
        }
    }
}
void *write_func(void* args)
{
    for(;;)
    {
#if 0
        pthread_mutex_lock(&mutex);
        while(!signal)
        {

            pthread_cond_wait(&cond,&mutex);
        }
        count_write++;
        signal =0;
     
        pthread_mutex_unlock(&mutex);

#endif
#if 1
        sem_wait(&sem_read);
        count_write++;
        sem_post(&sem_write);
#endif
#if 0
    if(count_write != count_read)
        {
            printf("error count_read is %d count  write %d\r\n",count_read,count_write);
            exit(0);
        }
#endif
    }
}
```

这里是为了测试唤醒的效果，所以线程同步的情况并不是很严格，在read_func线程中是将count_read加1，然后唤醒write_func线程，这个线程中是将count_write加1.如果每次唤醒都存在的话，那么在若干次唤醒后，count_write和count_read是相等的，如果在write_func线程中发现两者不相等就直接退出，条件编译分别对应使用条件变量和信号量的两种情况。在使用条件变量是，即使在pthread_cond_signal后sleep一段时间后，仍然会出现丢失事件，在每次的测试结果都是不一样的，有的不会出现唤醒丢失，有的在刚开始就会出现唤醒丢失的出现，有的在最后才出现唤醒丢失情况！
    
但是使用信号量就不会出现这种情况，其实可以使用信号量实现条件变量，只不过是在初始化的时候稍微做一点手脚而已！！
    
其实应该还要申请一个信号量，让这个变量初始化为1，作为二进制变量！这样更合适，这里没有申请的原因是在两个线程中使用的是不同变量，如果这两个线程中都是对同一个全局变量操作，那么一个当做条件变量的信号量是不可缺少的！
————————————————
版权声明：本文为CSDN博主「鱼思故渊」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/yusiguyuan/article/details/20215591