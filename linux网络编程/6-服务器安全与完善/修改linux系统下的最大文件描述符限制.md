## [修改Linux系统下的最大文件描述符限制](https://www.cnblogs.com/crzqj/p/7528500.html)

通常我们通过终端连接到linux系统后执行ulimit -n 命令可以看到本次登录的session其文件描述符的限制，如下：

```shell
$ulimit -n
1024
```

当然可以通过ulimit -SHn 102400 命令来修改该限制，但这个变更只对当前的session有效，当断开连接重新连接后更改就失效了。

如果想永久变更需要修改/etc/security/limits.conf 文件，如下：

```shell
vi /etc/security/limits.conf
\* hard nofile 102400
\* soft nofile 102400
```

保存退出后重新登录，其最大文件描述符已经被永久更改了。

这只是修改用户级的最大文件描述符限制，也就是说每一个用户登录后执行的程序占用文件描述符的总数不能超过这个限制。

## 系统级的限制

它是限制所有用户打开文件描述符的总和，可以通过修改内核参数来更改该限制：

```shell
sysctl -w fs.file-max=102400
```

使用sysctl命令更改也是临时的，如果想永久更改需要在/etc/sysctl.conf添加

```shell
fs.file-max=102400
```

保存退出后使用sysctl -p 命令使其生效。

与file-max参数相对应的还有file-nr，这个参数是只读的，可以查看当前文件描述符的使用情况。

直接修改内核参数，无须重启系统。
`sysctl -w fs.file-max 65536`
或者
`echo "65536" > /proc/sys/fs/file-max`
两者作用是相同的，前者改内核参数，后者直接作用于内核参数在虚拟文件系统（procfs, psuedo file system）上对应的文件而已。
可以用下面的命令查看新的限制
`sysctl -a | grep fs.file-max`
或者
`cat /proc/sys/fs/file-max`

修改内核参数
/etc/sysctl.conf
`echo "fs.file-max=65536" >> /etc/sysctl.confsysctl -p`

查看当前file handles使用情况：
`sysctl -a | grep fs.file-nr`
或者
`cat /proc/sys/fs/file-nr825 0 65536`

另外一个命令：
`lsof | wc -l`

 

下面是摘自kernel document中关于file-max和file-nr参数的说明

1. > file-max & file-nr: 

2. > The kernel allocates file handles dynamically, but **as yet it doesn't free them again.** 

3. > 内核可以动态的分配文件句柄，但到目前为止是不会释放它们的 

4. > The value **in file-max denotes the maximum number of file handles that the Linux kernel will allocate. When you \**get lots of error messages about running \*\*out of file handles, you might want to increase \*\*this limit.\*\*\*\*\**** 

5. > file-max的值是linux内核可以分配的最大文件句柄数。如果你看到了很多关于打开文件数已经达到了最大值的错误信息，你可以试着增加该值的限制 

6. > Historically, the three values **in file-nr denoted the number of allocated file handles, the number of allocated but unused file handles, and the maximum number of file handles. Linux 2.6 always reports 0 \**as the number of free file handles -- \*\*this \*\*is not an error, it just means that the number of allocated file handles exactly matches the number of used file handles.\*\*\*\*\**** 

7. > 在kernel 2.6之前的版本中，file-nr 中的值由三部分组成，分别为：1.已经分配的文件句柄数，2.已经分配单没有使用的文件句柄数，3.最大文件句柄数。但在kernel 2.6版本中第二项的值总为0，这并不是一个错误，它实际上意味着已经分配的文件句柄无一浪费的都已经被使用了 