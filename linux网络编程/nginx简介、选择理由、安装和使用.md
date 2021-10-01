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

auto / :编译相关的脚本，可执行文件configure一会会用到这些脚本
cc / : 检查编译器的脚本
lib / : 检查依赖库的脚本
os / : 检查操作系统类型的脚本
type / : 检查平台类型的脚本
CHANGES : 修复的bug，新增加的功能说明
CHANGES.ru : 俄语版CHANGES
conf / : 默认的配置文件
configure : 编译nginx之前必须先执行本脚本以生成一些必要的中间文件
contrib / : 脚本和工具，典型的是vim高亮工具
vim / : vim高亮工具
html / : 欢迎界面和错误界面相关的html文件
man / : nginx帮助文件目录
src / : nginx源码目录
core : 核心代码
event : event(事件)模块相关代码
http : http(web服务)模块相关代码
mail : 邮件模块相关代码
os : 操作系统相关代码
stream : 流处理相关代码
objs/:执行了configure生成的中间文件目录
ngx_modules.c：内容决定了我们一会编译nginx的时候有哪些模块会被编译到nginx里边来。
Makefile:执行了configure脚本产生的编译规则文件，执行make命令时用到
*/

（3.3）nginx的编译和安装
a)编译的第一步：用configure来进行编译之前的配置工作
./configure
–prefix：指定最终安装到的目录：默认值 /usr/local/nginx
–sbin-path：用来指定可执行文件目录：默认的是 sbin/ nginx
–conf-path：用来指定配置文件目录：默认的是 conf/nginx.conf
b)用make来编译,生成了可执行文件 make
c)用make命令开始安装 sudo make install

# 四：nginx的启动和简单使用

启动: sudo ./nginx
