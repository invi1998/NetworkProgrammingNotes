作用
exit()函数关闭所有打开的文件并终止程序。

### 原理

exit()函数的参数会被传递给一些操作系统，通常的约定是正常终止的程序传递值0,非正常终止的程序传递非0值。不同的退出值可能用来标识导致程序的失败的不同原因，ANSIC标准要求使用值0或宏EXIT_SUCCESS来指示程序成功终止，使用宏EXIT_FAILURE指示程序非成功中止。(宏和exit() 原型 在stdlib.h头文件中都可以找到 EXIT_SUCCESS EXIT_FAILURE)

### 和return的区别

按照ANSIC,在最初调用的main函数中使用和调用exit() 的效果相同，在main()中我们使用语句

return 0;

exit(0);

两个语句的作用相同

注意 如果main() 在一个递归程序中，exit函数仍然会终止程序，但是return 将控制权移交给递归的上一级，直到最初的那一级，此时return 才会终止程序。 return和exit的另一个区别在于 ， 及使在除main之外的函数中调用exit（）它也会终止程序

### 相关宏

```c
#define EXIT_FAILURE 1
#define EXIT_SUCCESS 0
```

### 举例

```c
#include <io.h>
#include <conio.h>
#include <stdlib.h>

int main(void){
    if((_unlink("D:\\sample.txt"))==1){
  cprintf("删除成功\n");
  exit(EXIT_SUCCESS);
 }else{
   cprintf("删除失败\n");
  exit(EXIT_FAILURE);
 }
    return 0;
}
```
