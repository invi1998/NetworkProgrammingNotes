# 一：nginx简介

nginx(2002年开发,2004年10才出现第一个版本0.1.0):web服务器，市场份额，排在第二位，Apache(1995)第一位；
web服务器，反向代理，负载均衡，邮件代理；运行时需要的系统资源比较少，所以经常被称呼为轻量级服务器；
是一个俄罗斯人（Igor Sysoev)，C语言（不是c++）开发的，并且开源了；
nginx号称并发处理百万级别的TCP连接，非常稳定，热部署（运行的时候能升级），高度模块化设计，自由许可证。
很多人开发自己的模块来增强nginx，第三方业务模块（c++开发）； OpenResty；
linux epoll技术； windows IOCP

# 二：为什么选择nginx

单机10万并发，而且同时能够保持高效的服务，epoll这种高并发技术好处就是：高并发只是占用更多内存就能 做到；
内存池，进程池，线程池，事件驱动等等；
学习研究大师级的人写的代码，是一个程序开发人员能够急速进步的最佳途径；

### 配置固定IP地址

需要修改配置文件 需要vim编辑器
vim安装：sudo apt-get install vim-gtk
两台主机（windows, ubuntu)
IP地址不能相同，但是要在同一网段下
主动发起数据包的这一端叫做“客户端”，另一端叫做“服务器端”
windows下的网络信息用ipconfig来查看

```linux
windows
IPv4 地址 . . . . . . . . . . . . : 192.168.1.105
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : 192.168.1.1

ubuntu
IPv4 地址 . . . . . . . . . . . . : 192.168.1.166
子网掩码 . . . . . . . . . . . . : 255.255.255.0 /  192.168.1.0/24(linux子网掩码)
默认网关. . . . . . . . . . . . . : 192.168.1.1
```

linux查看网络信息（ifconfig)

```shell
ifconfig
```

看到网卡叫做 ens33

编辑ubuntu下的网络配置

```shell
cd /etc
cd network
# 编辑该文件夹下的 interfaces 文件
sudo vim interfaces

# 注释掉 DHCP
# iface ens33 inet dhcp

# 写入
inace ens33 inet static
address 192.168.1.166
geteway 192.168.1.1
netmask 192.168.1.0/24

```

vim小技巧

```shell
# vim分为命令状态和文本输入状态
# 切换为输入状态
i
# 切换为命令状态
esc
# 保存退出(命令状态下)
:wq
# 不保存退出
:q! 
```

### 修改DNS

```shell
# /etc/network目录下
sudo vim /etc/resolvconf.resolv.conf.d.base

# 写入
nameserver 8.8.8.8
# 存盘退出
# 重启机器
sudo rebot
```

_其实关于网络配置这里，如果是20以后的服务器server ubuntu，是在进行系统安装的时候就可以直接进行设置的（这里我是已经早就设置好了，所以以上流程没有进行实操）_

### 配置远程连接

#### 一：在ubuntu安装ssh服务

```shell
# 查看是否安装了ssh
ps -elgrep ssh
# 安装ssh
sudo apt-get install openssh-server

```

#### 二：windows安装远程连接软件xshell

### 安装编译工具gcc（编译C）,g++（编译cpp,c++）等

_Ubuntu缺省情况下，并没有提供C/C++的编译环境，因此还需要手动安装。但是如果单独安装gcc以及g++比较麻烦，幸运的是，Ubuntu提供了一个build-essential软件包。也就是说，安装了该软件包，编译c/c++所需要的软件包也都会被安装。因此如果想在Ubuntu中编译c/c++程序,只需要安装该软件包就可以了。_

```shell
# 安装编译必须的环境（库）
sudo apt-get install build-essential
```

### 共享一个操作目录

通过虚拟机把windows下的目录进行共享，让linux可以访问这个目录

# 三：安装nginx，搭建web服务器

_Linux pwd（英文全拼：print work directory） 命令用于显示工作目录。
执行 pwd 指令可立刻得知您目前所在的工作目录的绝对路径名称。
语法_

```shell

pwd [--help][--version]

# 参数说明:

# --help 在线帮助。
# --version 显示版本信息。
```

（3.1）安装前提
a)epoll,linux 内核版本为2.6或者以上；

```shell
# 查看linux内核
uname -a
```

```shell
# Linux uname（英文全拼：unix name）命令用于显示系统信息。

# uname 可显示电脑以及操作系统的相关信息。

# 语法
# uname [-amnrsv][--help][--version]
# 参数说明：

# -a或--all 　显示全部的信息。
# -m或--machine 　显示电脑类型。
# -n或--nodename 　显示在网络上的主机名称。
# -r或--release 　显示操作系统的发行编号。
# -s或--sysname 　显示操作系统名称。
# -v 　显示操作系统的版本。
# --help 　显示帮助。
# --version 　显示版本信息。
# 实例
# 显示系统信息：

uname -a
# Linux iZbp19byk2t6khuqj437q6Z 4.11.0-14-generic #20~16.04.1-Ubuntu SMP Wed Aug 9 09:06:22 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
# 显示计算机类型：

uname -m
# x86_64
# 显示计算机名：

uname -n
# runoob-linux
# 显示操作系统发行编号：

uname -r
# 4.11.0-14-generic
# 显示操作系统名称：

uname -s
# Linux
# 显示系统版本与时间：

uname -v
#20~16.04.1-Ubuntu SMP Wed Aug 9 09:06:22 UTC 2017

```

b)gcc编译器，g++编译器
c)pcre库：函数库；支持解析正则表达式；

```shell
sudo apt-get install libpcre3-dev
```

d)zlib库：压缩解压缩功能

```shell
sudo apt-get install libz-dev
```

e)openssl库：ssl功能相关库，用于网站加密通讯

```shell
sudo apt-get install libssl-dev
```

（3.2）nginx源码下载以及目录结构简单认识
nginx官网 <http://www.nginx.org>
nginx的几种版本
(1)mainline版本：版本号中间数字一般为奇数。更新快，一个月内就会发布一个新版本，最新功能，bug修复等，稳定性差一点；
(2)stable版本：稳定版，版本号中间数字一般为偶数。经过了长时间的测试，比较稳定，商业化环境中用这种版本；这种版本发布周期比较长，几个月；
(3)Legacy版本：遗产，遗留版本，以往的老版本；
安装，现在有这种二进制版本：通过命令行直接安装；
灵活：要通过编译 nginx源码手段才能把第三方模块弄进来；

<http://nginx.org/download/nginx-1.20.1.tar.gz>

```shell
# 创建一个目录用于存放nginx源码包
mkdir nginxsourcecode
# 进入文件夹sourcecode
cd nginxsourcecode
# 下载源码压缩包
wget http://nginx.org/download/nginx-1.20.1.tar.gz
# wget是一个下载文件的工具，它用在命令行下。对于Linux用户是必不可少的工具，我们经常要下载一些软件或从远程服务器恢复备份到本地服务器。

#   wget支持HTTP，HTTPS和FTP协议，可以使用HTTP代理。所谓的自动下载是指，wget可以在用户退出系统的之后在后台执行。这意味这你可以登录系统，启动一个wget下载任务，然后退出系统，wget将在后台执行直到任务完成

#    wget 可以跟踪HTML页面上的链接依次下载来创建远程服务器的本地版本，完全重建原始站点的目录结构。这又常被称作”递归下载”。

#    wget 非常稳定，它在带宽很窄的情况下和不稳定网络中有很强的适应性.如果是由于网络的原因下载失败，wget会不断的尝试，直到整个文件下载完毕。如果是服务器打断下载过程，它会再次联到服务器上从停止的地方继续下载。这对从那些限定了链接时间的服务器上下载大文件非常有用。

# 查看下载内容
ls -la
# invi@inviubuntu:~/sourcecode$ ls -la
# total 1048
# drwxrwxr-x 2 invi invi    4096 Oct  2 00:57 .
# drwxr-x--- 4 invi invi    4096 Oct  2 00:54 ..
# -rw-rw-r-- 1 invi invi 1061461 May 25 15:34 nginx-1.20.1.tar.gz

# 解压源码包
tar -xzvf nginx-1.20.1.tar.gz

# 进入解压目录
cd nginx-1.20.1/

invi@inviubuntu:~/sourcecode/nginx-1.20.1$ ls -la
total 820
drwxr-xr-x 8 invi invi   4096 May 25 12:35 .
drwxrwxr-x 3 invi invi   4096 Oct  2 01:13 ..
drwxr-xr-x 6 invi invi   4096 Oct  2 01:13 auto
-rw-r--r-- 1 invi invi 311503 May 25 12:35 CHANGES
-rw-r--r-- 1 invi invi 475396 May 25 12:35 CHANGES.ru
drwxr-xr-x 2 invi invi   4096 Oct  2 01:13 conf
-rwxr-xr-x 1 invi invi   2590 May 25 12:35 configure
drwxr-xr-x 4 invi invi   4096 Oct  2 01:13 contrib
drwxr-xr-x 2 invi invi   4096 Oct  2 01:13 html
-rw-r--r-- 1 invi invi   1397 May 25 12:35 LICENSE
drwxr-xr-x 2 invi invi   4096 Oct  2 01:13 man
-rw-r--r-- 1 invi invi     49 May 25 12:35 README
drwxr-xr-x 9 invi invi   4096 Oct  2 01:13 src

# 将代码高亮工具拷贝到vim
cp -r contrib/vim ~/.vim

```

### nginx源码包的目录结构

>auto / :编译相关的脚本，可执行文件configure一会会用到这些脚本
>>cc / : 检查编译器的脚本
>>lib / : 检查依赖库的脚本
>>os / : 检查操作系统类型的脚本
>>type / : 检查平台类型的脚本
>CHANGES : 修复的bug，新增加的功能说明
>CHANGES.ru : 俄语版CHANGES
>conf / : 默认的配置文件
>configure : 编译nginx之前必须先执行本脚本以生成一些必要的中间文件
>contrib / : 脚本和工具，典型的是vim高亮工具
>>vim / : vim高亮工具
>html / : 欢迎界面和错误界面相关的html文件
>man / : nginx帮助文件目录
>src / : nginx源码目录
>>core : 核心代码
>>event : event(事件)模块相关代码
>>http : http(web服务)模块相关代码
>>mail : 邮件模块相关代码
>>os : 操作系统相关代码
>>stream : 流处理相关代码
>objs/:执行了configure生成的中间文件目录
>ngx_modules.c：内容决定了我们一会编译nginx的时候有哪些模块会被编译到nginx里边来。
>Makefile:执行了configure脚本产生的编译规则文件，执行make命令时用到

（3.3）nginx的编译和安装
a)编译的第一步：用configure来进行编译之前的配置工作

```shell
# 进入nginx源码包目录下，执行
./configure
# –prefix：指定最终安装到的目录：默认值 /usr/local/nginx
# –sbin-path：用来指定可执行文件目录：默认的是 sbin/ nginx
# –conf-path：用来指定配置文件目录：默认的是 conf/nginx.conf
```

b)用make来编译,生成了可执行文件 ~/sourcecode/nginx-1.20.1$

```shell
make
```

c)用make命令开始安装 ~/sourcecode/nginx-1.20.1$

```shell
sudo make install
# 安装到了根路径下的 /usr/local/nginx里了
```

# 四：nginx的启动和简单使用

启动:

```shell
# 查看进程
ps -ef | grep nginx

# invi       20018    1211  0 02:04 pts/0    00:00:00 grep --color=auto nginx
# 可以看出来，nginx进程还没有

# e 查看所有进程
# f 以一种很宽泛的格式显示
# ps -ef就是把所有进程罗列出来
# | 管道符，
# grep 查找，它要在一个指定的文本中查找指定的内容nginx
# 然后这里的这个管道符就是要把ps -ef 列出的所有进程 作为 grep 的搜索内容，来在这些所有进程中搜索nginx
# 这句命令 就是查找 nginx进程

# 进入nginx的sbin（可执行二进制目录下）
cd /sbin
# /usr/local/nginx/sbin$ 

# 启动nginx
sudo ./nginx

# 查看是否启动
ps -ef | grep nginx

# root       20230       1  0 02:13 ?        00:00:00 nginx: master process ./nginx
# nobody     20231   20230  0 02:13 ?        00:00:00 nginx: worker process
# invi       20391   20364  0 02:15 pts/1    00:00:00 grep --color=auto nginx

```
