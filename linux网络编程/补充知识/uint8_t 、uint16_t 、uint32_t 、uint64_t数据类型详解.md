uint8_t / uint16_t / uint32_t /uint64_t 是什么数据类型?
在nesc的代码中，你会看到很多你不认识的数据类型，比如uint8_t等。咋一看，好像是个新的数据类型，不过C语言（nesc是C的扩展）里面好像没有这种数据类型啊！怎么又是u又是_t的？很多人有这样的疑问。论坛上就有人问：以*_t结尾的类型是不是都是long型的？在baidu上查一下，才找到答案，这时才发觉原来自己对C掌握的太少。

那么_t的意思到底表示什么？具体的官方答案没有找到，不过我觉得有个答案比较接近。它就是一个结构的标注，可以理解为type/typedef的缩写，表示它是通过typedef定义的，而不是其它数据类型。

uint8_t，uint16_t，uint32_t等都不是什么新的数据类型，它们只是使用typedef给类型起的别名，新瓶装老酒的把戏。不过，不要小看了typedef，它对于你代码的维护会有很好的作用。比如C中没有bool，于是在一个软件中，一些程序员使用int，一些程序员使用short，会比较混乱，最好就是用一个typedef来定义，如：
typedef char bool;

一般来说，一个C的工程中一定要做一些这方面的工作，因为你会涉及到跨平台，不同的平台会有不同的字长，所以利用预编译和typedef可以让你最有效的维护你的代码。为了用户的方便，C99标准的C语言硬件为我们定义了这些类型，我们放心使用就可以了。 按照posix标准，一般整形对应的*_t类型为：
1字节     uint8_t
2字节     uint16_t
4字节     uint32_t
8字节     uint64_t

这些数据类型是 C99 中定义的，具体定义在：/usr/include/stdint.h    ISO C99: 7.18 Integer types <stdint.h>

```cpp
/* There is some amount of overlap with <sys/types.h> as known by inet code */  
#ifndef __int8_t_defined  
# define __int8_t_defined  
typedef signed char             int8_t;   
typedef short int               int16_t;  
typedef int                     int32_t;  
# if __WORDSIZE == 64  
typedef long int                int64_t;  
# else  
__extension__  
typedef long long int           int64_t;  
# endif  
#endif  
  
/* Unsigned.  */  
typedef unsigned char           uint8_t;  
typedef unsigned short int      uint16_t;  
#ifndef __uint32_t_defined  
typedef unsigned int            uint32_t;  
# define __uint32_t_defined  
#endif  
#if __WORDSIZE == 64  
typedef unsigned long int       uint64_t;  
#else  
__extension__  
typedef unsigned long long int  uint64_t;  
#endif  
```

格式化输出：

unit64_t     %llu   

unit32_t     %u

unit16_t    %hu


注意：

必须小心 uint8_t 类型变量的输出，例如如下代码，会输出什么呢？

```cpp
uint8_t fieldID = 67;
cerr<< "field=" << fieldID <<endl;
```

结果发现是：field=C 而 不是我们所想的 field=67

这是由于 typedef unsigned char uint8_t; 
uint8_t 实际是一个 char, cerr << 会输出 ASCII 码是 67 的字符，而不是 67 这个数字.

因此，输出 uint8_t 类型的变量实际输出的是其对应的字符, 而不是真实数字.

若要输出 67,则可以这样：

```cpp
cerr<< "field=" << (uint16_t) fieldID <<endl;

结果是：field=67

同样： uint8_t 类型变量转化为字符串以及字符串转化为 uint8_t 类型变量都要注意, uint8_t类型变量转化为字符串时得到的会是ASCII码对应的字符, 字符串转化为 uint8_t 变量时, 会将字符串的第一个字符赋值给变量.

例如如下代码：

```cpp
#include <iostream>  
#include <stdint.h>  
#include <sstream>  
using namespace std;  
  
  
int main()  
{  
    uint8_t fieldID = 67;  
  
    // uint8_t --> string  
    string s;  
    ostringstream strOStream;  
    strOStream << fieldID;  
    s = strOStream.str();  
    cerr << s << endl;  
      
    // string --> uint8_t  
    s = "65";   
    stringstream strStream;  
    strStream << s;  
    strStream >> fieldID;  
    strStream.clear();  
    cerr << fieldID << endl;  
} 

```

上述代码输出的是：

C

6