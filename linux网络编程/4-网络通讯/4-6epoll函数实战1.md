# （1）配置文件的修改

```makefile
#epoll连接的最大数【是每个worker进程允许连接的客户端数】，实际其中有一些连接要被监听socket使用，实际允许的客户端连接数会比这个数小一些
worker_connections = 1024
```



# （2）epoll函数实战

## （2.1）ngx_epoll_init函数内容

## （2.2）ngx_epoll_init函数的调用

