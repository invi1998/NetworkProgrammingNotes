# 从EMFILE和ENFILE说起，fd limit的问题

下面的描述，统一用“fd”来表示通常所说的“文件句柄”。UNIX/Linux系，称为“文件描述符（file descriptor）”，也因此才有“fd”这个缩写。“文件句柄”，貌似是Windows系的说法。

从问题入手。

与“Too many open files”这个错误相关的errno有EMFILE和ENFILE，查看open、socket、accept、socketpair、pipe等系统调用的手册，可以看到对EMFILE和ENFILE的解释：

- **EMFILE** The process already has the maximum number of files open.
- **EMFILE** Process file table overflow.
- **EMFILE** The per-process limit of open file descriptors has been reached.
- **EMFILE** Too many descriptors are in use by this process.
- **EMFILE** Too many file descriptors are in use by the process.

- **ENFILE** The system limit on the total number of open files has been reached.

其中，对于EMFILE的解释有几种，虽然说法不同，但都表达了一样的意思，就是进程的fd已用尽。

为了解决这个问题，最straightfoward(直截了当)的办法，就是用ulimit命令或者setrlimit系统调用。

ulimit的作用是，显示或修改“当前shell”的resource limits，或者在当前shell中启动的进程的resource limits。

对于ulimit，有两点需要说明：

1. 一是，ulimit对使用者的价值，是显示当前的各种limits，在“修改”方面，它几乎没什么用。当前shell的resource limits，在用户login shell时，就已经确定了，特别是hard limit。具体说，对于soft limit，ulimit做增加操作时，只能增加到和hard limit一样多；对于hard limit，ulimit只能做减少操作，不能做增加操作，且是不可逆的。
2. 二是，ulimit只能显示或修改“当前shell”的resource limits，它对system-wide resource limits（联想一下开头提到的EMFILE和ENFILE的区别），没有任何感知，也不能做任何操作。

在用户login shell的过程中，Linux PAM的一些modules会执行，其中pam_limits module会设置当前user session的resource limits，根据/etc/security/limits.conf（默认路径）文件和/etc/security/limits.d目录下的文件（如果有的话）。所以，如果想真正修改“当前shell”的resource limits，就直接修改/etc/security/limits.conf文件，再重新login。

setrlimit的使用，则更加tricky(棘手)。因为setrlimit是个系统调用（ulimit是一个shell built-in command），它由一个进程的实现代码调用，而一个进程有多重身份标识（real user/groud ID, effective user/groud ID），同时，如果有privilege(特权)，进程还可以动态的切换身份（setuid/setgid，seteuid/setegid），而最终，这个进程的resource limits，又取决于进程的身份。同时，无论进程是什么身份，它的resource limits都由/etc/security/limits.conf决定了。

**综目前所述，ulimit和setrlimit都不能很好的解决“Too many open files”的问题。那怎么更好的解决这个问题呢？**

ENFILE这个errno的存在，表明一定存在system-wide的resource limits，而不仅仅有process-specific的resource limits。按照常识，process-specific的resource limits，一定受限于system-wide的resource limits。

有一个与/etc/security/limits.conf类似的pseudo file(伪文件)，可以用来控制system-wide fd limit：/proc/sys/fs/file-max。这个文件是可读写的，在我们系统中，是644，owner是root。那是不是说，只要取得superuser权限，是不是就可以任意修改这个文件呢？显然不可能。且不说一个fd占一个sizeof(int)的空间，在kernel中，每个打开的文件，都关联一个struct file，通常在定义在kernel source的include/linux/fs.h中：

```c++
struct file {
        union {
                struct llist_node       fu_llist;
                struct rcu_head         fu_rcuhead;
        } f_u;
        struct path             f_path;
#define f_dentry        f_path.dentry
        struct inode            *f_inode;       /* cached value */
        const struct file_operations    *f_op;


        /*
         * Protects f_ep_links, f_flags.
         * Must not be taken from IRQ context.
         */
        spinlock_t              f_lock;
        atomic_long_t           f_count;
        unsigned int            f_flags;
        fmode_t                 f_mode;
        struct mutex            f_pos_lock;
        loff_t                  f_pos;
        struct fown_struct      f_owner;
        const struct cred       *f_cred;
        struct file_ra_state    f_ra;


        u64                     f_version;
#ifdef CONFIG_SECURITY
        void                    *f_security;
#endif
        /* needed for tty driver, and maybe others */
        void                    *private_data;


#ifdef CONFIG_EPOLL
        /* Used by fs/eventpoll.c to link all the hooks to this file */
        struct list_head        f_ep_links;
        struct list_head        f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
        struct address_space    *f_mapping;
#ifdef CONFIG_DEBUG_WRITECOUNT
        unsigned long f_mnt_write_state;
#endif
} __attribute__((aligned(4)));  /* lest something weird decides that 2 is OK */
```




所以，即使有superuser权限，也不可能任意修改/proc/sys/fs/file-max文件，会受限于内存资源。这个文件是在编译kernel时，根据一个kernel constant NR_OPEN（可以理解为Number of OPEN files）决定的。

在绝大多数情况下，NR_OPEN都是够用的。如果程序确实出现ENFILE的错误（注意，是“确实”，因为在一些开源程序中，ENFILE被滥用，直接errno = ENFILE），那么可以echo XXXXX > /proc/sys/fs/file-max解决。如果file-max设置为了最大值（NR_OPEN）还不能解决，根据不同kernel版本，有两种进一步的解决办法。

2.6.25之前，只能重新编译，手动指定NR_OPEN的值。2.6.25之后，kernel提供了/proc/sys/fs/nr_open，可以由程序修改，比如调用sysctl。

最后，对于线上“Too many open files”问题，比较简单并且比较彻底的解决方法，应该是根据需要修改/proc/sys/fs/file-max和/etc/security/limits.conf，而且，只修改limits.conf，一般就够了。



## 非阻塞socket accept出现EMFILE错误

测试连接的时候，连接到达1010的时候，accept返回-1，errno=EMFILE。

把系统的fd软限制和硬限制都抬高。打开的文件句柄数过多。ulimit -n 看看文件描述符限制 如果是1024的话，需要改大