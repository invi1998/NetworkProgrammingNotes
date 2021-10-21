# （1）ngx_epoll_process_events函数调用位置

之前的代码实现了 epoll_create()，epoll_ctl()：现在我们已经做好尊卑，等待迎接客户端主动发起三次握手连接

下面将研究，如何获取用户的连接事件，并怎样把用户接入进来

──master process ./nginx 

──worker process

───ngx_master_process_cycle()     //创建子进程等一系列动作

────ngx_setproctitle()       //设置进程标题   

────ngx_start_worker_processes()  //创建worker子进程  

─────for (i = 0; i < threadnums; i++)  //master进程在走这个循环，来创建若干个子进程

──────ngx_spawn_process(i,"worker process");

───────pid = fork(); //分叉，从原来的一个master进程（一个叉），分成两个叉（原有的master进程，以及一个新fork()出来的worker进程

─────── //只有子进程这个分叉才会执行ngx_worker_process_cycle()

───────ngx_worker_process_cycle(inum,pprocname);  //子进程分叉

────────ngx_worker_process_init();

─────────sigemptyset(&set);  

─────────sigprocmask(SIG_SETMASK, &set, NULL); //允许接收所有信号

─────────g_socket.ngx_epoll_init();  //初始化epoll相关内容，同时 往监听socket上增加监听事件，从而开始让监听端口履行其职责

──────────m_epollhandle = epoll_create(m_worker_connections); 

──────────ngx_epoll_add_event((*pos)->fd....);

───────────epoll_ctl(m_epollhandle,eventtype,fd,&ev);

────────ngx_setproctitle(pprocname);      //重新为子进程设置标题为worker process

────────for ( ;; ) {

─────────//子进程开始在这里不断的死循环

─────────ngx_process_events_and_timers();	// 处理网络事件和定时器事件

──────────**ngx_socket_epoll_process_events(-1);**	// -1表示卡着等待

────────}



────sigemptyset(&set); 

────for ( ;; ) {}.         //父进程[master进程]会一直在这里循环



从这个函数的调用位置上，可以预见：

1）这个函数任然是在子进程中进行调用，

2）这个函数，是放在子进程的for(;;)中，意味着这个函数会被不断的，多次调用

# （2）ngx_epoll_process_events函数内容

用户三次握手成功，然后连入进来，这个 “连入进来” 事件对于服务器来讲，就是一个监听套接字上的可读事件

```c++
// 开始获取发生的事件消息
// 参数 unsigned int timer ： 表示epoll_wait() 阻塞的时长，单位是 毫秒
// 返回值，1：正常返回，0：有问题返回，一帮不管是正常还是问题返回，都应该保持进程继续运行
// 本函数被ngx_process_events_and_timers()调用，而ngx_process_events_and_timers()是在子进程的死循环中被反复调用
int CSocket::ngx_epoll_process_events(int timer)
{
    // 等待事件，事件会返回到m_events里，最多返回NGX_MAX_EVENTS个事件【因为我们这里只提供了这些内存】
    // 阻塞timer这么长的时间，除非：1）：阻塞时间到达，2）阻塞期间收到了事件会立即返回，3）调用的时候有事件也会立即返回，4）如果来个信号，比如使用kill -1 pid测试
    // 如果timer为-1则一直阻塞，如果timer为0则立即返回，即便没有任何事件
    // 返回值：  有错误发生返回-1，错误在errno中，比如：你发送了一个信号过来，就返回-1，错误信息（4：Interrupted system call)（系统回调中断）
    //          如果你等待的是一段时间，并且超时了，则返回0
    //          如果返回 > 0 则表示成功捕获到这么多个事件【返回值里】
    int events = epoll_wait(m_epollhandle,m_events,NGX_MAX_EVENTS,timer);

    if(events == -1)
    {
        // 有错误发生，发送某个信号给本进程，就可以导致这个条件成立，而且错误码为 4
        // #define EINTR 4, EINTR错误的产生：当阻塞于某个慢系统调用的一个进程捕获某个信号且响应的信号处理函数返回时，该系统调用可能会返回一个EINTR错误
        // 例如：在socket服务器端，设置了信号捕获机制，有子进程，在父进程阻塞于慢系统调用时，由于父进程捕获到一个有效的信号，内核会致使accept返回一个EINTR错误（被中断的系统调用）
        if(errno == EINTR)
        {
            // 信号所导致，直接返回，一般认为这不是什么毛病，但是还是打印日志记录一下，因为一般也不会认为给worker进程发送消息
            ngx_log_error_core(NGX_LOG_INFO,errno,"CSocekt::ngx_epoll_process_events()中epoll_wait()失败!");
            return 1;   // 正常返回
        }
        else
        {
            // 这里被认为应该是有问题的，记录日志
            ngx_log_error_core(NGX_LOG_ALERT,errno,"CSocekt::ngx_epoll_process_events()中epoll_wait()失败!");
            return 0;   // 非正常返回
        }
    }

    if(events == 0)     // 超时，但是事件没来
    {
        if(timer != -1)
        {
            // 要求epoll_wait阻塞一定的时间而不是一直阻塞，这属于是阻塞时间到了，正常返回
            return 1;
        }
        // 无限等待【所以不存在超时】，但是却没有任何返回事件，这应该不正常有问题
        ngx_log_error_core(NGX_LOG_ALERT,0,"CSocekt::ngx_epoll_process_events()中epoll_wait()没超时却没返回任何事件!"); 
        return 0; //非正常返回 
    }

    // 会惊群，一个telnet上来，4个worker进程都会被惊动，都执行下面这个
    // ngx_log_stderr(errno,"惊群测试1:%d",events); 

    // 走到这里，就属于有事件收到了
    lpngx_connection_t      c;
    uintptr_t               instance;
    uint32_t                revents;
    for(int i = 0; i < events; ++i)     // 遍历本次epoll_wait返回的所有事件，注意，events才是返回的实际事件数量
    {
        c = (lpngx_connection_t)(m_events[i].data.ptr);         // ngx_epoll_add_event()给进去的，这里就能取出来
        instance = (uintptr_t)c & 1;                            // 将地址的最后一位取出来，用instance变量进行标识，见：ngx_epoll_add_event函数。该值当时是随着连接池中的连接一起给传进来的
        c = (lpngx_connection_t)((uintptr_t) c & (uintptr_t) ~1);   // 最后一位去掉，得到真正的c地址
		// 运算符m&n的结果是求出m/n的剩余数据个数（余数）
		// 运算符m&~n的结果是求出剩余数据的起始位置
        
        // 仔细分析一下官方nginx这个判断
        // 一个套接字，当关联一个连接池中的连接【对象】时，这个套接字值是要给到c->fd的
        // 那么什么时候这个c->fd会变成-1呢？关闭连接时，这个fd会被设置为-1,具体哪行代码设置的-1后续再看，但是这里应该不是ngx_free_connection()函数设置的-1
        // 处理过期事件
        if(c->fd == -1)
        {
            // 比如我们使用epoll_wait取得三个事件，处理第一个事件时，因为业务需要，我们把这个链接关闭，那么我们应该会把c->fd设置为-1；
            // 第二个事件照常处理
            // 第三个事件：假如这个第三个事件也跟第一个事件对应的是同一个连接，那么这个条件就会成立，那么这种事件就属于过期事件，不应该处理

            // 这里可以增加个日志，
            ngx_log_error_core(NGX_LOG_DEBUG,0,"CSocekt::ngx_epoll_process_events()中遇到了fd=-1的过期事件:%p.",c); 
            continue; //这种事件就不处理即可
        }

        if(c->instance != instance)
        {
            // 什么时候这个条件成立呢？【换种问法：instance标志为什么可以判断事件是否过期？】
            // 比如我们使用epoll_wait取的第单个事件，处理第一个事件时，因为业务需要，我们把这个链接关闭【麻烦就麻烦在这个连接被服务器关闭上了】，但是恰好第三个事件也和这个连接有关；
            // 因为第一个事件就把socket连接关闭了，显然第三个事件我们是我们不应该处理的【因为这是一个过期事件】，若处理，肯定会导致错误。
            // 那我们把上述c->fd设置为-1,可以解决这个问题吗？能解决问题的一部分，但是另外一部分不能解决，不能解决的部分描述如下：【这种情况应该很少见到】
            // 1）处理第一个事件时，一位业务需要，我们把这个连接【假设套接字为50】关闭，同时设置c->fd = -1；并且调用ngx_free_connection将这个连接归还给连接池
            // 2）处理第二个事件，恰好第二个事件时建立新连接事件，调用ngx_get_connection从连接池中取出的连接非常可能是刚刚释放的第一个事件对应的连接池中的连接
            // 3）又因为 （1）中套接字50被释放了，所以会被操作系统拿过来进行复用，复用给了（2）【一般这么快被复用也是很少见了】
            // 4）当处理第三个事件时，第三个事件其实是已经过期的，应该不处理，那么怎么判断第三个事件是过期的呢？
            //      【假设现在处理的是第三个事件，此时这个连接池中的该连接实际上已经被作用于第二个事件对应的socket了】
            //      依靠instance标志位能够解决这个问题，当调用ngx_get_connection从连接池中获取一个新连接时，我们把instance标志位置反，所以这个条件如果不成立，说明这个连接已经被挪作他用了

            // 总结：
            // 如果收到了若干个事件，其中连接关闭也搞了多次，导致这个instance标志位被取反两次，那么造成的结果就是：还是有可能遇到某些过期事件没有被发现【这里也就没有被continue】，
            // 然后照旧被单做新事件处理了，
            // 如果是这样，那就只能被照旧处理了，可能会照成偶尔某个连接被误关闭？但是整体服务器程序运行应该是平稳的，问题不大。这种漏网之鱼而被单做没过期来的过期事件理论上是极少发生的

            ngx_log_error_core(NGX_LOG_DEBUG,0,"CSocekt::ngx_epoll_process_events()中遇到了instance值改变的过期事件:%p.",c); 
            continue;   // 这种事件不处理即可
        }

        // 能走到这里，说明这些事件我们都认为没有过期，就开始正常处理
        revents = m_events[i].events;   // 取出事件类型
        if(revents & (EPOLLERR | EPOLLHUP))     // 例如对方close掉套接字，这里会感应到，【换句话说，就是如果发生了错误或者客户端断联】
        {
            // 这里加上读写标记，方便后续代码处理
            revents |= EPOLLIN | EPOLLOUT;  // EPOLLIN: 表示对应的连接上有数据可以读出，（tcp连接的远端主动关闭连接，也相当于可读事件，因为本服务器是要处理发送过来的FIN包）
                                            // EPOLLOUT: 表示对应的连接上可以写入数据发送【写准备好】

            //ngx_log_stderr(errno,"2222222222222222222222222.");
        }
        if(revents & EPOLLIN)	// // 如果是读事件
        {
            // 一个客户端新连入，这个就会成立
            // c->r_ready = 1;              // 标记可读。【从连接池中拿出一个连接时，这个连接的所有成员都是0】
            (this->*(c->rhandler))(c);      // 注意括号的运用，来正确的设置优先级，防止编译出错；如果是个新客户连入
                                            // 如果新连接进入，这里执行的应该是CSocket::ngx_event_accept(c)
                                            // 如果是已经连入，发送数据到这里，则这里执行的应该是CSocket::ngx_wait_request_handler
        }

        if(revents & EPOLLOUT) // 如果是写事件
        {
            // 。。。等待扩展
            ngx_log_stderr(errno,"111111111111111111111111111111.");
        }
    }
    return 1;

}
```



## （2.1）事件驱动（官方nginx的架构也被称为事件驱动架构）

通常，我们写服务器处理模型的程序时，有以下几种模型：

```
（1）每收到一个请求，创建一个新的进程，来处理该请求； 
（2）每收到一个请求，创建一个新的线程，来处理该请求； 
（3）每收到一个请求，放入一个事件列表，让主进程通过非阻塞I/O方式来处理请求
```

第三种就是协程、事件驱动的方式，一般普遍认为第（3）种方式是大多数网络服务器采用的方式 

目前大部分的UI编程都是事件驱动模型，如很多UI平台都会提供onClick()事件，这个事件就代表鼠标按下事件。事件驱动模型大体思路如下：

1. 有一个事件（消息）队列；
2. 鼠标按下时，往这个队列中增加一个点击事件（消息）；
3. 有个循环，不断从队列取出事件，根据不同的事件，调用不同的函数，如onClick()、onKeyDown()等；
4. 事件（消息）一般都各自保存各自的处理函数指针，这样，每个消息都有独立的处理函数；

**事件驱动编程是一种编程范式，这里程序的执行流由外部事件来决定。它的特点是包含一个事件循环，当外部事件发生时使用回调机制来触发相应的处理**。另外两种常见的编程范式是（单线程）同步以及多线程编程。 

# （3）ngx_event_accept函数内容

​	

```c++
// 建立新连接专用函数，当新连接进入时，本函数会被ngx_epoll_process_events()调用
void CSocket::ngx_event_accept(lpngx_connection_t oldc)
{
    // 因为listen套接字上用的不是ET【边缘触发】，而是LT【水平触发】,意味着客户端连入如果我不要处理，那这个函数会被多次调用，所以这里可以不必多次accept()，可以只执行一次accept()
    // 这也可以避免本函数被卡很久，注意：本函数应该尽快返回，以免阻塞程序运行
    struct sockaddr         mysockaddr;         // 远端服务器的socket地址
    socklen_t               socklen;
    int                     err;
    int                     level;
    int                     s;
    static  int             use_accept4 = 1;    // 我们先认为他能够使用accept4()函数
    lpngx_connection_t      newc;               // 代表连接池中一个连接【注意这里是指针】

    // ngx_log_stderr(0, "这是几个\n"); 这里会惊群，也就是说，epoll技术本身会有惊群的问题

    socklen = sizeof(mysockaddr);
    do      // 用do，跳到while后边去会更方便
    {
        if(use_accept4)
        {
            s = accept4(oldc->fd, &mysockaddr, &socklen,SOCK_NONBLOCK);     // 从内核中获取一个用户端连接，最后一个参数 SOCK_NONBLOCK表示返回一个非阻塞的socket，节省一次ioctl【设置非阻塞】调用
        }
        else
        {
            s = accept(old->fd,&mysockaddr, &socklen,SOCK_NONBLOCK);
        }

        //惊群，有时候不一定完全惊动所有4个worker进程，可能只惊动其中2个等等，其中一个成功其余的accept4()都会返回-1；错误 (11: Resource temporarily unavailable【资源暂时不可用】) 
        //所以参考资料：https://blog.csdn.net/russell_tao/article/details/7204260
        //其实，在linux2.6内核上，accept系统调用已经不存在惊群了（至少我在2.6.18内核版本上已经不存在）。大家可以写个简单的程序试下，在父进程中bind,listen，然后fork出子进程，
               //所有的子进程都accept这个监听句柄。这样，当新连接过来时，大家会发现，仅有一个子进程返回新建的连接，其他子进程继续休眠在accept调用上，没有被唤醒。
        //ngx_log_stderr(0,"测试惊群问题，看惊动几个worker进程%d\n",s); 【我的结论是：accept4可以认为基本解决惊群问题，但似乎并没有完全解决，有时候还会惊动其他的worker进程】

        if (s == 1)
        {
            err = errno;

            // 对于accept,send和recv而言，事件未发生时，errno通常会被设置为EAGAIN（意为“再来一次）或者EWOULDBLOCK（意为“期待阻塞”）
            if(err == EAGAIN)   // accept()没准备好，这个EAGAIN错误EWOULDBLOCK是一样的
            {
                // 除非你用一个循环不断的accept()取走所有的连接，不然一般不会有这个错误【我们在这里只取一个连接】
                return;
            }
            level = NGX_LOG_ALERT;
            if (err == ECONNABORTED)    // ECONNABORTED错误则发生在对方意外关闭套接字后【您的主机中的软件放弃了一个已经建立的连接--由于超时或者其他失败而终止连接（用户插拔网线就可能导致这个错误出现】
            {
                // 该错误被描述为“software caused connection abort” ， 即：软件引起的连接中断；原因在于当前服务和客户进程在完成用于TCP连接的“三次握手”后
                // 客户端 TCP 却发送了一个 RST （复位）分节，在服务器进程看来，就在该连接已经由TCP 排队，等待服务进程调用 accept 的时候 RST 却到达了。
                // POSIX 规定此时的 errno 值必须为 ECONNABORTED。源自 Berkeley 的实现完全在内核中处理终止的连接，服务器进程将永远不知道该终止的发生
                // 服务器进程一般可以忽略该错误，直接再次调用accept

                level = NGX_LOG_ERR;
            }

            // EMFILE: 进程的fd已经用尽【已达到系统所允许单一进程所能打开的文件/套接字总数】。
            // 使用命令 ulimit -n  ： 查看文件描述符显示，如果是1024的话，需要改大：打开的文件句柄数过多，把系统的fd软限制和硬限制都抬高
            // ENFILE：这个errno的存在，表明一定存在system-wide的resource limits，而不仅仅有process-specific的resouce limits。按照常识，
            // process-soecific的resource limits一定受限于system-wide的resource limits
            else if (err == EMFILE || err == ENFILE)
            {
                level = NGX_LOG_CRIT;
            }

            ngx_log_error_core(level, errno, "CSocket::ngx_event_accept()中accept4()失败!");

            if (use_accept4 && err == ENOSYS)   // accept4()函数没实现
            {
                use_accept4 = 0;                // 标记不使用accept4()函数，改用accept()函数
                continue;
            }

            if (err == ECONNABORTED)            // 对方关闭套接字
            {
                // 这个错误因为可以忽略，所以不用干啥

            }

            if (err == EMFILE || err == ENFILE)
            {
                // do nothing 官方nginx的做法是，先把读事件从listen socket中移除，然后再弄一个定时器，定时器到了则继续执行该函数，但是定时器到了有个标记，会把读事件增加到listen socket上去
                // 这里先暂时不处理，因为上面即写这个日志了
            }
            return;
            
        }

        // 走到这里，表示accept4()成功了
        // ngx_log_stderr(errno, "accept4成功s=%d", s);    // s这里就是一个句柄了
        newc = ngx_get_connection(s);
        if (newc == NULL)
        {
            // 连接池中的连接不够用，那么就得把这个socket直接关闭并返回了，因为ngx_get_connection()中已经写日志了，所以这里不再需要写日志了
            if (close(s) == -1)
            {
                ngx_log_error_core(NGX_LOG_ALERT, errno, "CSocket::ngx_event_accept()中close(%d)失败！", s);
            }
            return;
            
        }

        // ---------------将来这里会判断是否连接超过最大允许连接数，现在先暂时不处理

        // 成功拿到连接池中一个连接
        memset(&newc->s_sockaddr, &mysockaddr, socklen);    // 拷贝客户端地址到连接对象【要转成字符串IP地址，参考函数ngx_sock_ntop()】

        // {
        //     // 测试将收到的地址弄成字符串，格式形如“192.168.1.126"或者""192.168.1.126:40904"
        //     u_char ipaddr[100];
        //     memset(ipaddr,0, sizeof(ipaddr));
        //     ngx_sock_ntop(&newc->s_sockaddr, 1, ipaddr, sizeof(ipaddr)-10); // 宽度给小点
        //     ngx_log_stderr(0, "ip信息为%s\n", ipaddr);
        // }
        
        if(!use_accept4)
        {
            // 如果不是用accept4()取得的socket，那么就要设置为非阻塞【因为使用accept4()的已经被accept4()设置为非阻塞了】
            if (setnonblocking(s) == false)
            {
                // 设置非阻塞失败
                ngx_close_accepted_connection(newc);
                return;     // 直接返回
            }
            
        }

        newc->listening = oldc->listening;                      // 连接对象，和监听对象关联，方便通过连接对象找监听对象【关联到监听端口】
        newc->w_ready = 1;                                      // 标记可以写，新连接写事件肯定是ready的，【从连接池拿出一个连接时这个连接的所有成员都是0】
        newc->rhandler = &CSocket::ngx_wait_request_handler;    // 设置数据来的时候的读处理函数。其实官方nginx中是ngx_http_wait_request_handler()

        // 客户端应该主动发送第一次的数据，这里读事件加入epoll监控
        // s,                   socket句柄
        // 1, 0,                读， 写
        // EPOLLET,             其他补充标记【EPOLLET（高速模式，边缘触发ET）】
        // EPOLL_CTL_ADD,       事件类型【增加，还有删除/修改】
        // newc                 连接池中的连接
        if (ngx_epoll_add_event(s, 1, 0, EPOLLET, EPOLL_CTL_ADD, newc) == -1)
        {
            // 增加事件失败。失败日志在ngx_epoll_add_event中写过了
            ngx_close_accepted_connection(newc);
            return; // 直接返回
        }

        break;      // 一般就是循环一次就跳出去
    
    } while (1);
    
    return;

}

```



## （3.1）epoll的两种工作模式：LT和ET

LT：level trigged， 水平触发，这种工作模式 低速模式（效率差） -------------epoll缺省用次模式
ET：edge trigged， 边缘触发/边沿触发，这种工作模式 高速模式（效率好）
现状：所有的监听套接字用的都是 水平触发； 所有的接入进来的用户套接字都是 边缘触发

水平触发的意思：来 一个事件，如果你不处理它，那么这个事件就会一直被触发；
边缘触发的意思：只对非阻塞socket有用；来一个事件，内核只会通知你一次【不管你是否处理，内核都不再次通知你】；
边缘触发模式，提高系统运行效率，编码的难度加大；因为只通知一次，所以接到通知后，你必须要保证把该处理的事情处理利索；
程序高手能够洞察到很多普通程序员所洞察不到的问题，并且能够写出更好的，更稳定的，更少出错误的代码；

# （4）总结和测试

# （5）事件驱动总结

# （6）一道腾讯后台开发的面试题