# 一、gdb简介

GDB是一个由GNU开源组织发布的、UNIX/LINUX操作系统下的、基于命令行的、功能强大的程序调试工具。 对于一名Linux下工作的c/c++程序员，gdb是必不可少的工具；

# 二、gdb使用流程

这里用c程序做基本演示，c++程序也是一样的；

## 1、启动gdb

编译一个测试程序，-g表示可以调试，命令如下：

```shell
gcc -g test.c -o test
```


启动gdb，命令如下：

```shell
gdb test 
gdb -q test //表示不打印gdb版本信息，界面较为干净；
```


测试如下：

```shell
root@ubuntu:/home/eit/c_test# gdb test
GNU gdb (Ubuntu 7.7-0ubuntu3) 7.7
Copyright (C) 2014 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from test...done.
(gdb) q
```

```shell
root@ubuntu:/home/eit/c_test# gdb -q test
Reading symbols from test...done.
(gdb) 
```


到此gdb启动完成！

## 2、查看源码

list(简写 l)： 查看源程序代码，默认显示10行，按回车键继续看余下的。
测试如下：

```shell
(gdb) list 
9	#define MAX_SIZE
10	
11	int main()
12	{
13	    int i,fd,size1 ,size2 ,len;
14	    char *buf = "helo!I'm liujiangyong ";
15	    char buf_r[15];
16	    len = strlen(buf);
17	    fd = open("/home/hello.txt",O_CREAT | O_TRUNC | O_RDWR,0666);
18	    if (fd<0)
(gdb) 
19	        {
20	            perror("open :");
21	            exit(1);
22	        }
23	    else
24	        {
25	        printf("open file:hello.txt %d\n",fd);
26	        }
27	    size1 = write(fd,buf,len);
28	    if (fd<0)
(gdb) 
29	    {
30	        printf("writre erro;");
31	
32	    }
33	    else
34	    {
35	        printf("写入的长度：%d\n写入文本内容：%s\n",size1,buf);
36	
37	    }
38	    lseek(fd,0,SEEK_SET);
(gdb) 
39	    size2 = read(fd,buf_r,12);
40	    if (size2 <0)
41	    {
42	        printf("read  erro\n");
43	    }
44	    else
45	    {
46	        printf("读取长度：%d\n 文本内容是：%s\n",size2,buf_r);
47	    }
48	    close(fd);    
(gdb) 
49	
50	
51	}
(gdb) 
Line number 52 out of range; write.c has 51 lines.
(gdb) 
```



## 3、运行程序

run(简写 r) ：运行程序直到遇到 结束或者遇到断点等待下一个命令；
测试如下：

```shell
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/eit/c_test/test 
open file:hello.txt 3
写入的长度：22
写入文本内容：helo!I'm liujiangyong 
读取长度：12
 文本内容是：helo!I'm liu
[Inferior 1 (process 19987) exited normally]
(gdb) 
```



## 4、设置断点

break(简写 b) ：格式 b 行号，在某行设置断点；
info breakpoints ：显示断点信息
Num： 断点编号
Disp：断点执行一次之后是否有效 kep：有效 dis：无效
Enb： 当前断点是否有效 y：有效 n：无效
Address：内存地址
What：位置

```shell
(gdb) b 5
Breakpoint 3 at 0x400836: file write.c, line 5.
(gdb) b 26 
Breakpoint 4 at 0x4008a6: file write.c, line 26.
(gdb) b 30
Breakpoint 5 at 0x4008c6: file write.c, line 30.
(gdb) info breakpoints 
Num     Type           Disp Enb Address            What
3       breakpoint     keep y   0x0000000000400836 in main at write.c:5
4       breakpoint     keep y   0x00000000004008a6 in main at write.c:26
5       breakpoint     keep y   0x00000000004008c6 in main at write.c:30
(gdb) 
```



## 5、单步执行

使用 continue、step、next命令
测试如下：

```shell
(gdb) r
Starting program: /home/eit/c_test/test 

Breakpoint 3, main () at write.c:12
12	{
(gdb) n
14	    char *buf = "helo!I'm liujiangyong ";
(gdb) 
16	    len = strlen(buf);
(gdb) 
17	    fd = open("/home/hello.txt",O_CREAT | O_TRUNC | O_RDWR,0666);
(gdb) s
open64 () at ../sysdeps/unix/syscall-template.S:81
81	../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb) 
main () at write.c:18
18	    if (fd<0)
(gdb) 
25	        printf("open file:hello.txt %d\n",fd);
(gdb) 
__printf (format=0x400a26 "open file:hello.txt %d\n") at printf.c:28
28	printf.c: No such file or directory.
(gdb) c
Continuing.
open file:hello.txt 3

Breakpoint 4, main () at write.c:27
27	    size1 = write(fd,buf,len);
(gdb) 
Continuing.
写入的长度：22
写入文本内容：helo!I'm liujiangyong 
读取长度：12
 文本内容是：helo!I'm liu
[Inferior 1 (process 20737) exited normally]
(gdb) 
```



## 6、查看变量

使用print、whatis命令
测试如下：

```shell
main () at write.c:28
28	    if (fd<0)
(gdb) 
35	        printf("写入的长度：%d\n写入文本内容：%s\n",size1,buf);
(gdb) print fd
$10 = 3
(gdb) whatis fd
type = int
(gdb) 
```



## 7、退出gdb

用quit命令退出gdb：

```shell
(gdb) r
Starting program: /home/eit/c_test/test 
open file:hello.txt 3
写入的长度：22
写入文本内容：helo!I'm liujiangyong 
读取长度：12
 文本内容是：helo!I'm liu
[Inferior 1 (process 20815) exited normally]
(gdb) q
root@ubuntu:/home/eit/c_test# 
```

continue（简写 c)： 继续执行程序，直到下一个断点或者结束；
next（简写 n ）：单步执行程序，但是遇到函数时会直接跳过函数，不进入函数；
step(简写 s) ：单步执行程序，但是遇到函数会进入函数；
until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体；
until+行号： 运行至某行，不仅仅用来跳出循环；
finish： 运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息；
call 函数(参数)：调用程序中可见的函数，并传递“参数”，如：call gdb_test(55)；
quit：简记为 q ，退出gdb；

# 三、gdb基本使用命令

## 1、运行命令

run：简记为 r ，其作用是运行程序，当遇到断点后，程序会在断点处停止运行，等待用户输入下一步的命令。
continue （简写c ）：继续执行，到下一个断点处（或运行结束）
next：（简写 n），单步跟踪程序，当遇到函数调用时，也不进入此函数体；此命令同 step 的主要区别是，step 遇到用户自定义的函数，将步进到函数中去运行，而 next 则直接调用函数，不会进入到函数体内。
step （简写s）：单步调试如果有函数调用，则进入函数；与命令n不同，n是不进入调用的函数的
until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体。
until+行号： 运行至某行，不仅仅用来跳出循环
finish： 运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息。
call 函数(参数)：调用程序中可见的函数，并传递“参数”，如：call gdb_test(55)
quit：简记为 q ，退出gdb

## 2、设置断点

break n （简写b n）:在第n行处设置断点
（可以带上代码路径和代码名称： b OAGUPDATE.cpp:578）
b fn1 if a＞b：条件断点设置
break func（break缩写为b）：在函数func()的入口处设置断点，如：break cb_button
delete 断点号n：删除第n个断点
disable 断点号n：暂停第n个断点
enable 断点号n：开启第n个断点
clear 行号n：清除第n行的断点
info b （info breakpoints） ：显示当前程序的断点设置情况
delete breakpoints：清除所有断点：

## 3、查看源码

list ：简记为 l ，其作用就是列出程序的源代码，默认每次显示10行。
list 行号：将显示当前文件以“行号”为中心的前后10行代码，如：list 12
list 函数名：将显示“函数名”所在函数的源代码，如：list main
list ：不带参数，将接着上一次 list 命令的，输出下边的内容。

## 4、打印表达式

print 表达式：简记为 p ，其中“表达式”可以是任何当前正在被测试程序的有效表达式，比如当前正在调试C语言的程序，那么“表达式”可以是任何C语言的有效表达式，包括数字，变量甚至是函数调用。
print a：将显示整数 a 的值
print ++a：将把 a 中的值加1,并显示出来
print name：将显示字符串 name 的值
print gdb_test(22)：将以整数22作为参数调用 gdb_test() 函数
print gdb_test(a)：将以变量 a 作为参数调用 gdb_test() 函数
display 表达式：在单步运行时将非常有用，使用display命令设置一个表达式后，它将在每次单步进行指令后，紧接着输出被设置的表达式及值。如： display a
watch 表达式：设置一个监视点，一旦被监视的“表达式”的值改变，gdb将强行终止正在被调试的程序。如： watch a
whatis ：查询变量或函数
info function： 查询函数
扩展info locals： 显示当前堆栈页的所有变量

## 5、查看运行信息

where/bt ：当前运行的堆栈列表；
bt backtrace 显示当前调用堆栈
up/down 改变堆栈显示的深度
set args 参数:指定运行时的参数
show args：查看设置好的参数
info program： 来查看程序的是否在运行，进程号，被暂停的原因。

## 6、分割窗口

layout：用于分割窗口，可以一边查看代码，一边测试：
layout src：显示源代码窗口
layout asm：显示反汇编窗口
layout regs：显示源代码/反汇编和CPU寄存器窗口
layout split：显示源代码和反汇编窗口
Ctrl + L：刷新窗口

## 7、cgdb强大工具

cgdb主要功能是在调试时进行代码的同步显示，这无疑增加了调试的方便性，提高了调试效率。界面类似vi，符合unix/linux下开发人员习惯;如果熟悉gdb和vi，几乎可以立即使用cgdb。

## 8、常用gdb调试命令汇总

# 四、总结

总的来说在Linux下开发程序gdb/cgdb是必须学会使用的，他的强大之处远不止于此，在程序的调试中用它会提高的我们的调试效率，当然gdb的功能与使用技巧还不止于此，多多探索，多多学习使用。

------

# gdb调试子进程

gdb默认情况下，父进程fork一个子进程，gdb只会继续调试父进程而不会管子进程的运行。

在一部分系统中(基于2.6内核的CentOS，支持follow-fork和detach-on-fork模式)，比如HP-UX11.x之后的版本，Linux2.5.60之后的版本，可以使用以下的方法来达到方便的进行多进程调试功能。

## **1.** 跟踪子进程进行调试，可以使用set follow-fork-mode mode来设置fork跟随模式。

**1.1 show follow-fork-mode**
   进入gdb以后，我们可以使用show follow-fork-mode来查看目前的跟踪模式。

**1.2 set follow-fork-mode parent**
   gdb只跟踪父进程，不跟踪子进程，这是默认的模式。

**1.3 set follow-fork-mode child**
   gdb在子进程产生以后只跟踪子进程，放弃对父进程的跟踪。

## **2.** 想同时调试父进程和子进程，以上的方法就不能满足了。Linux提供了set detach-on-fork mode命令来供我们使用

**2.1 show detach-on-fork**
   show detach-on-fork显示了目前是的detach-on-fork模式

**2.2 set detach-on-fork on**
   只调试父进程或子进程的其中一个(根据follow-fork-mode来决定)，这是默认的模式。

**2.3 set detach-on-fork off**
    父子进程都在gdb的控制之下，其中一个进程正常调试(根据follow-fork-mode来决定),另一个进程会被设置为暂停状态。

## 3.具体示例

  show follow-fork-mode
  **set follow-fork-mode child**
  show detach-on-fork
  **set detach-on-fork off**

 

## 4.其他方式

  使用gdb调试多进程时，如果想要在进程间进行切换，那么就需要在fork调用前设置： set detach-on-fork off ，
然后使用 info inferiors 来查看进程信息，得到的信息可以看到最前面有一个进程编号，使用 inferior num 来进行进程切换。
那么为什么要使用 set detache-on-fork off 呢？它的意思是在调用fork后相关进程的运行行为是怎么样的，是detache on/off ?
也就是说分离出去独立运行，不受gdb控制还是不分离，被阻塞住。这里还涉及到一个设置 set follow-fork-mode [parents/child] ,
就是fork之后，gdb的控制落在谁身上，如果是父进程，那么分离的就是子进程，反之亦然。如果detache-on-fork被off了，
那么未受控的那个进程就会被阻塞住，进程状态为T，即处于调试状态。