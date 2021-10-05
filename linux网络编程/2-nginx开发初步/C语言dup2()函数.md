# C语言dup2()函数：复制文件描述词

相关函数：open, close, fcntl, dup

头文件：#include <unistd.h>

定义函数：int dup2(int odlfd, int newfd);

函数说明：dup2()用来复制参数oldfd 所指的文件描述词, 并将它拷贝至参数newfd 后一块返回. 若参数newfd为一已打开的文件描述词, 则newfd 所指的文件会先被关闭. dup2()所复制的文件描述词, 与原来的文件描述词共享各种文件状态, 详情可参考dup().

返回值：当复制成功时, 则返回最小及尚未使用的文件描述词. 若有错误则返回-1, errno 会存放错误代码.

附加说明：dup2()相当于调用fcntl(oldfd, F_DUPFD, newfd); 请参考fcntl().

错误代码：EBADF 参数fd 非有效的文件描述词, 或该文件已关闭

## 额外补充dup

相关函数：open, close, fcntl, dup2

头文件：#include <unistd.h>

定义函数：int dup (int oldfd);

函数说明：dup()用来复制参数oldfd 所指的文件描述词, 并将它返回. 此新的文件描述词和参数oldfd 指的是同一个文件, 共享所有的锁定、读写位置和各项权限或旗标. 例如, 当利用lseek()对某个文件描述词作用时, 另一个文件描述词的读写位置也会随着改变. 不过, 文件描述词之间并不共享close-on-exec 旗标.

返回值：当复制成功时, 则返回最小及尚未使用的文件描述词. 若有错误则返回-1, errno 会存放错误代码.

错误代码：EBADF 参数fd 非有效的文件描述词, 或该文件已关闭.
