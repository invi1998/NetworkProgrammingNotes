# [Linux errno详解](https://www.cnblogs.com/x_wukong/p/10848069.html)

转自：https://www.cnblogs.com/Jimmy1988/p/7485133.html

 

## 1. 错误码 / errno

Linux中系统调用的错误都存储于 `errno`中，`errno`由操作系统维护，存储就近发生的错误，即下一次的错误码会覆盖掉上一次的错误。

> *PS: 只有当系统调用或者调用lib函数时出错，才会置位`errno`！*

查看系统中所有的`errno`所代表的含义，可以采用如下的代码：

```c++
/* Function: obtain the errno string
*   char *strerror(int errno)
*/

#include <stdio.h>
#include <string.h>     //for strerror()
//#include <errno.h>
int main()
{
    int tmp = 0;
    for(tmp = 0; tmp <=256; tmp++)
    {
        printf("errno: %2d\t%s\n",tmp,strerror(tmp));
    }
    return 0;
}
```

输出信息如下：

```c++
errno:  0       Success
errno:  1       Operation not permitted
errno:  2       No such file or directory
errno:  3       No such process
errno:  4       Interrupted system call
errno:  5       Input/output error
errno:  6       No such device or address
errno:  7       Argument list too long
errno:  8       Exec format error
errno:  9       Bad file descriptor
errno: 10       No child processes
errno: 11       Resource temporarily unavailable
errno: 12       Cannot allocate memory
errno: 13       Permission denied
errno: 14       Bad address
errno: 15       Block device required
errno: 16       Device or resource busy
errno: 17       File exists
errno: 18       Invalid cross-device link
errno: 19       No such device
errno: 20       Not a directory
errno: 21       Is a directory
errno: 22       Invalid argument
errno: 23       Too many open files in system
errno: 24       Too many open files
errno: 25       Inappropriate ioctl for device
errno: 26       Text file busy
errno: 27       File too large
errno: 28       No space left on device
errno: 29       Illegal seek
errno: 30       Read-only file system
errno: 31       Too many links
errno: 32       Broken pipe
errno: 33       Numerical argument out of domain
errno: 34       Numerical result out of range
errno: 35       Resource deadlock avoided
errno: 36       File name too long
errno: 37       No locks available
errno: 38       Function not implemented
errno: 39       Directory not empty
errno: 40       Too many levels of symbolic links
errno: 41       Unknown error 41
errno: 42       No message of desired type
errno: 43       Identifier removed
errno: 44       Channel number out of range
errno: 45       Level 2 not synchronized
errno: 46       Level 3 halted
errno: 47       Level 3 reset
errno: 48       Link number out of range
errno: 49       Protocol driver not attached
errno: 50       No CSI structure available
errno: 51       Level 2 halted
errno: 52       Invalid exchange
errno: 53       Invalid request descriptor
errno: 54       Exchange full
errno: 55       No anode
errno: 56       Invalid request code
errno: 57       Invalid slot
errno: 58       Unknown error 58
errno: 59       Bad font file format
errno: 60       Device not a stream
errno: 61       No data available
errno: 62       Timer expired
errno: 63       Out of streams resources
errno: 64       Machine is not on the network
errno: 65       Package not installed
errno: 66       Object is remote
errno: 67       Link has been severed
errno: 68       Advertise error
errno: 69       Srmount error
errno: 70       Communication error on send
errno: 71       Protocol error
errno: 72       Multihop attempted
errno: 73       RFS specific error
errno: 74       Bad message
errno: 75       Value too large for defined data type
errno: 76       Name not unique on network
errno: 77       File descriptor in bad state
errno: 78       Remote address changed
errno: 79       Can not access a needed shared library
errno: 80       Accessing a corrupted shared library
errno: 81       .lib section in a.out corrupted
errno: 82       Attempting to link in too many shared libraries
errno: 83       Cannot exec a shared library directly
errno: 84       Invalid or incomplete multibyte or wide character
errno: 85       Interrupted system call should be restarted
errno: 86       Streams pipe error
errno: 87       Too many users
errno: 88       Socket operation on non-socket
errno: 89       Destination address required
errno: 90       Message too long
errno: 91       Protocol wrong type for socket
errno: 92       Protocol not available
errno: 93       Protocol not supported
errno: 94       Socket type not supported
errno: 95       Operation not supported
errno: 96       Protocol family not supported
errno: 97       Address family not supported by protocol
errno: 98       Address already in use
errno: 99       Cannot assign requested address
errno: 100      Network is down
errno: 101      Network is unreachable
errno: 102      Network dropped connection on reset
errno: 103      Software caused connection abort
errno: 104      Connection reset by peer
errno: 105      No buffer space available
errno: 106      Transport endpoint is already connected
errno: 107      Transport endpoint is not connected
errno: 108      Cannot send after transport endpoint shutdown
errno: 109      Too many references: cannot splice
errno: 110      Connection timed out
errno: 111      Connection refused
errno: 112      Host is down
errno: 113      No route to host
errno: 114      Operation already in progress
errno: 115      Operation now in progress
errno: 116      Stale file handle
errno: 117      Structure needs cleaning
errno: 118      Not a XENIX named type file
errno: 119      No XENIX semaphores available
errno: 120      Is a named type file
errno: 121      Remote I/O error
errno: 122      Disk quota exceeded
errno: 123      No medium found
errno: 124      Wrong medium type
errno: 125      Operation canceled
errno: 126      Required key not available
errno: 127      Key has expired
errno: 128      Key has been revoked
errno: 129      Key was rejected by service
errno: 130      Owner died
errno: 131      State not recoverable
errno: 132      Operation not possible due to RF-kill
errno: 133      Memory page has hardware error
errno: 134~255  unknown error!
```

**Linux中，在头文件 `/usr/include/asm-generic/errno-base.h` 对基础常用errno进行了宏定义**：

```c++
#ifndef _ASM_GENERIC_ERRNO_BASE_H
#define _ASM_GENERIC_ERRNO_BASE_H

#define EPERM        1  /* Operation not permitted */
#define ENOENT       2  /* No such file or directory */
#define ESRCH        3  /* No such process */
#define EINTR        4  /* Interrupted system call */
#define EIO      5  /* I/O error */
#define ENXIO        6  /* No such device or address */
#define E2BIG        7  /* Argument list too long */
#define ENOEXEC      8  /* Exec format error */
#define EBADF        9  /* Bad file number */
#define ECHILD      10  /* No child processes */
#define EAGAIN      11  /* Try again */
#define ENOMEM      12  /* Out of memory */
#define EACCES      13  /* Permission denied */
#define EFAULT      14  /* Bad address */
#define ENOTBLK     15  /* Block device required */
#define EBUSY       16  /* Device or resource busy */
#define EEXIST      17  /* File exists */
#define EXDEV       18  /* Cross-device link */
#define ENODEV      19  /* No such device */
#define ENOTDIR     20  /* Not a directory */
#define EISDIR      21  /* Is a directory */
#define EINVAL      22  /* Invalid argument */
#define ENFILE      23  /* File table overflow */
#define EMFILE      24  /* Too many open files */
#define ENOTTY      25  /* Not a typewriter */
#define ETXTBSY     26  /* Text file busy */
#define EFBIG       27  /* File too large */
#define ENOSPC      28  /* No space left on device */
#define ESPIPE      29  /* Illegal seek */
#define EROFS       30  /* Read-only file system */
#define EMLINK      31  /* Too many links */
#define EPIPE       32  /* Broken pipe */
#define EDOM        33  /* Math argument out of domain of func */
#define ERANGE      34  /* Math result not representable */

#endif
```

**在 `/usr/include/asm-asm-generic/errno.h` 中，对剩余的errno做了宏定义**：

```c++
#ifndef _ASM_GENERIC_ERRNO_H
#define _ASM_GENERIC_ERRNO_H

#include <asm-generic/errno-base.h>

#define EDEADLK     35  /* Resource deadlock would occur */
#define ENAMETOOLONG    36  /* File name too long */
#define ENOLCK      37  /* No record locks available */
#define ENOSYS      38  /* Function not implemented */
#define ENOTEMPTY   39  /* Directory not empty */
#define ELOOP       40  /* Too many symbolic links encountered */
#define EWOULDBLOCK EAGAIN  /* Operation would block */
#define ENOMSG      42  /* No message of desired type */
#define EIDRM       43  /* Identifier removed */
#define ECHRNG      44  /* Channel number out of range */
#define EL2NSYNC    45  /* Level 2 not synchronized */
#define EL3HLT      46  /* Level 3 halted */
#define EL3RST      47  /* Level 3 reset */
#define ELNRNG      48  /* Link number out of range */
#define EUNATCH     49  /* Protocol driver not attached */
#define ENOCSI      50  /* No CSI structure available */
#define EL2HLT      51  /* Level 2 halted */
#define EBADE       52  /* Invalid exchange */
#define EBADR       53  /* Invalid request descriptor */
#define EXFULL      54  /* Exchange full */
#define ENOANO      55  /* No anode */
#define EBADRQC     56  /* Invalid request code */
#define EBADSLT     57  /* Invalid slot */

#define EDEADLOCK   EDEADLK

#define EBFONT      59  /* Bad font file format */
#define ENOSTR      60  /* Device not a stream */
#define ENODATA     61  /* No data available */
#define ETIME       62  /* Timer expired */
#define ENOSR       63  /* Out of streams resources */
#define ENONET      64  /* Machine is not on the network */
#define ENOPKG      65  /* Package not installed */
#define EREMOTE     66  /* Object is remote */
#define ENOLINK     67  /* Link has been severed */
#define EADV        68  /* Advertise error */
#define ESRMNT      69  /* Srmount error */
#define ECOMM       70  /* Communication error on send */
#define EPROTO      71  /* Protocol error */
#define EMULTIHOP   72  /* Multihop attempted */
#define EDOTDOT     73  /* RFS specific error */
#define EBADMSG     74  /* Not a data message */
#define EOVERFLOW   75  /* Value too large for defined data type */
#define ENOTUNIQ    76  /* Name not unique on network */
#define EBADFD      77  /* File descriptor in bad state */
#define EREMCHG     78  /* Remote address changed */
#define ELIBACC     79  /* Can not access a needed shared library */
#define ELIBBAD     80  /* Accessing a corrupted shared library */
#define ELIBSCN     81  /* .lib section in a.out corrupted */
#define ELIBMAX     82  /* Attempting to link in too many shared libraries */

#define ELIBEXEC    83  /* Cannot exec a shared library directly */
#define EILSEQ      84  /* Illegal byte sequence */
#define ERESTART    85  /* Interrupted system call should be restarted */
#define ESTRPIPE    86  /* Streams pipe error */
#define EUSERS      87  /* Too many users */
#define ENOTSOCK    88  /* Socket operation on non-socket */
#define EDESTADDRREQ    89  /* Destination address required */
#define EMSGSIZE    90  /* Message too long */
#define EPROTOTYPE  91  /* Protocol wrong type for socket */
#define ENOPROTOOPT 92  /* Protocol not available */
#define EPROTONOSUPPORT 93  /* Protocol not supported */
#define ESOCKTNOSUPPORT 94  /* Socket type not supported */
#define EOPNOTSUPP  95  /* Operation not supported on transport endpoint */
#define EPFNOSUPPORT    96  /* Protocol family not supported */
#define EAFNOSUPPORT    97  /* Address family not supported by protocol */
#define EADDRINUSE  98  /* Address already in use */
#define EADDRNOTAVAIL   99  /* Cannot assign requested address */
#define ENETDOWN    100 /* Network is down */
#define ENETUNREACH 101 /* Network is unreachable */
#define ENETRESET   102 /* Network dropped connection because of reset */
#define ECONNABORTED    103 /* Software caused connection abort */
#define ECONNRESET  104 /* Connection reset by peer */
#define ENOBUFS     105 /* No buffer space available */
#define EISCONN     106 /* Transport endpoint is already connected */
#define ENOTCONN    107 /* Transport endpoint is not connected */
#define ESHUTDOWN   108 /* Cannot send after transport endpoint shutdown */
#define ETOOMANYREFS    109 /* Too many references: cannot splice */
#define ETIMEDOUT   110 /* Connection timed out */
#define ECONNREFUSED    111 /* Connection refused */
#define EHOSTDOWN   112 /* Host is down */
#define EHOSTUNREACH    113 /* No route to host */
#define EALREADY    114 /* Operation already in progress */
#define EINPROGRESS 115 /* Operation now in progress */
#define ESTALE      116 /* Stale file handle */
#define EUCLEAN     117 /* Structure needs cleaning */
#define ENOTNAM     118 /* Not a XENIX named type file */
#define ENAVAIL     119 /* No XENIX semaphores available */
#define EISNAM      120 /* Is a named type file */
#define EREMOTEIO   121 /* Remote I/O error */
#define EDQUOT      122 /* Quota exceeded */

#define ENOMEDIUM   123 /* No medium found */
#define EMEDIUMTYPE 124 /* Wrong medium type */
#define ECANCELED   125 /* Operation Canceled */
#define ENOKEY      126 /* Required key not available */
#define EKEYEXPIRED 127 /* Key has expired */
#define EKEYREVOKED 128 /* Key has been revoked */
#define EKEYREJECTED    129 /* Key was rejected by service */

/* for robust mutexes */
#define EOWNERDEAD  130 /* Owner died */
#define ENOTRECOVERABLE 131 /* State not recoverable */

#define ERFKILL     132 /* Operation not possible due to RF-kill */

#define EHWPOISON   133 /* Memory page has hardware error */

#endif
```

## 2. 打印错误信息

### 1). 打印错误信息 / perror

> - **作用**：
>   打印系统错误信息
>
> - **头文件**：
>
>   ```c++
>   #include <stdio.h>  
>   ```
>
> - **函数原型**：
>
>   ```c++
>   void perror(const char *s)
>   ```
>
> - **参数**：
>
>   > **s**: 字符串提示符
>
> - **输出形式**：
>   const char *s: strerror(errno) //提示符：发生系统错误的原因
>
> - **返回值**：
>   无返回值

### 2). 字符串显示错误信息 / strerror

> - **作用**：
>   将错误码以字符串的信息显示出来
>
> - **头文件**：
>
>   ```c++
>   #include <string.h>  
>   ```
>
> - **函数原型**：
>
>   ```c++
>   char *strerror(int errnum);
>   ```
>
> - **参数**：
>
>   > **errnum**: 即errno

> - **返回值**：
>   返回错误码字符串信息