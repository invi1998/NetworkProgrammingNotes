# （1）nginx源码总述

# （2）nginx源码查看工具

visual studio,source Insight,visual stuido Code.
采用 Visual Studio Code来阅读nginx源码
Visual Studio Code:微软公司开发的一个跨平台的轻量级的编辑器（不要混淆vs2017:IDE集成开发环境，以编译器）；
Visual Studio Code在其中可以安装很多扩展模块；
.30.0版本，免费的,多平台；
官方地址：<https://code.visualstudio.com>
<https://code.visualstudio.com/download>
为支持语法高亮，跳转到函数等等，可能需要安装扩展包；
————————————————
版权声明：本文为CSDN博主「昔拉天使」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：<https://blog.csdn.net/qq_39885372/article/details/104666051>

# （3）nginx源码入口函数定位

core下nginx的main函数

# （4）创建一个自己的linux下的c语言程序

共享目录不见了，一般可能是虚拟机自带的工具 VMWare tools可能有问题；
VMWare-tools是VMware虚拟机自带的一系列的增强工具，文件共享功能就是WMWare-tools工具里边的
a)虚拟机->重新安装VMware tools
b)sudo mkdir /mnt/cdrom
c)sudo mount /dev/cdrom /mnt/cdrom
d)cd /mnt/cdrom
e)sudo cp WMwareTool…tar.gz …/
f)cd …
g)sudo tar -zxvf VMwareToo…tar.gz
h)cd wmware-tools-distrib
j)sudo ./vmware-install.pl
一路回车(安装最后几个步骤注意填写yes，（默认是no，需要写入yes）)。

gcc编译.c，g++编译 c++
c文件若很多，都需要编译，那么咱们就要写专门的MakeFile来编译了；
gcc -o:用于指定最终的可执行文件名
