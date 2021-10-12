# （1）基础设施之日志打印实战代码一

日志：供日后运行维护人员去查看，定位和解决问题
新文件：ngx_printf.cxx以及ngx_log.cxx
ngx_printf.cxx : 放和打印格式相关的函数
ngx_log.cxx ： 放和日志相关的函数

通过printf函数来认识ngx_log_stderr函数
printf("aaa=%s, inta = %d", "dssf", 233);
可以看见printf有的能力：
1）根据可变参数，组合出字符串
2）往屏幕上输出组合出来的字符串

那既然 ngx_log_stderr()的功能和printf类似，为何不直接使用printf呢？
*1、学习printf函数如何实现*
*2、ngx_log_stderr()可以支持任意格式化字符，对于扩展原有功能非常有帮助*

ngx_log_stderr函数

```cxx
void ngx_log_stderr(int err, const char *fmt, ...)
{
    va_list args;
    // 创建一个va_list数据类型变量
    u_char errstr[NGX_MAX_ERROR_STR+1];
    // 2048 -- *************** +1 感觉官方写法有点瑕疵
    u_char *p, *last;

    memset(errstr, 0, sizeof(errstr));
    // 这块有必要加，至少在va_end之前有必要，否者字符串没有结束标记是不行的

    last = errstr + NGX_MAX_ERROR_STR;
    // last指向的是整个buffer最后，【指向最后一个有效位置的后面也就是非有效位】，作为一个标记，
    // 防止输出内容超出这么长
    // 这里认为是有问题的，所以才在上面 u_char errstr[NGX_MAX_ERROR_STR+1]; 给了加1
    // 比如你定义了char tmp[2]，你如果last = tmp+2，那么last实际上指向了tmp[2]，而tmp[2]在使用中是无效的

    p = ngx_cpymem(errstr,"nginx: ", 7);    // p指向“nginx: ”之后

    va_start(args,fmt); // 使用args指向起始的参数
    p = ngx_vslprintf(p, last,fmt, args);   // 组合出这个字符串保存在errstr中
    va_end(args);   // 释放args

    if(err) // 如果错误代码不是0,表示有错误发生
    {
        // 错误代码和错误信息也要显示出来
        p = ngx_log_errno(p, last, err);

    }

    // 若位置不够，那换行也要硬插到末尾，哪怕覆盖到其他内容
    if(p>= (last -1))
    {
        p = (last - 1)-1;
        // 把尾部空格留出来，这里感觉nginx的处理有点问题
        // last-1，才是最后一个有效的内存，而这个位置要保存\0，所以这里需要再-1，这个位置才适合保存\n
    }

    *p++ = '\n';    // 增加换行符

    // 往标准错误【一般是屏幕】输出信息
    write(STDERR_FILEND, errstr, p-errstr);
    

    // 测试代码
    //printf("ngx_log_stderr()处理结果=%s\n",errstr);
    //printf("ngx_log_stderr()处理结果=%s",errstr);

    return;

}

```

然后看到ngx_log_stderr中实现了一个 ngx_vslprintf()函数

```cxx
// ---------------------------------------------------------------------------------------------------------------------------------------
// 对于nginx自定义的数据结构进行标准化格式输出
// 例如：给进来一个 “abc = %d",13  最终buf里应该得到的是 abc=13 这种结果
// 
// buf: 往这里放数据
// last：放的数据不要超过这里
// fmt: 以这个为首的一系列可变参数
// 支持的格式：%d[%xd/%Xd]： 数字，     %s:字符串       %f:浮点数       %p: pid_t
// 对于：ngx_log_stderr(0, "invalid option:\"%s\", %d", "testinfo", 123);
//      fmt = "invalid option: \"%s\", %d"
//      args = "testinfo", 123

u_char *ngx_vslprintf(u_char *buf, u_char *last,const char *fmt, va_list args)
{
    // 比如说你需要调用 ngx_log_stderr(0, "invalid option: \"%s\"", argv[i]);
    // 那么这里的fmt就应该是：invalid option: "%s"
    // printf("fmt = %s", fmt);

    u_char      zero;

    #ifdef _WIN64
        typedef unsigned __int64 uintptr_t;
    #else
        typedef unsigned int uintptr_t;
    #endif

    uintptr_t width,sign,hex,frac_width,scale,n;    // 临时用到的一些变量

    int64_t     i64;    // 保存 %d  对应的可变参数
    uint64_t    ui64;   // 保存 %ud 对应的可变参数，临时作为 %f 可编程的整数部分也是可以的
    u_char      *p;     // 保存 %s  对应的可变参数
    double      f;      // 保存 %f  对应的可变参数
    uint64_t    frac;   // %f可变参数。根据 %.2f等，取得小数部分的后两位内容


    while (*fmt && buf < last)  // 每次处理完一个字符，处理的是 “invalid option: \"%s\", %d"中的字符
    {
        if(*fmt == '%') // %开头的一帮都是需要被可变参数取代的
        {
            // ----------------------------变量初始化工作开始------------------------------
            // ++fmt是先加后用，也就是fmt先往后走一个字节位置，然后再判断该位置的内容
            zero = (u_char)((*++fmt == '0') ? '0':' ');
            // 判断%后面接的是否是个'0',如果是zero='0',否者zero=' '，一般比如想显示10位，而实际数字是7位，前面填充3个字符，这就是这里的zero用于填充
            // ngx_log_stderr(0, "数字是%010d", 12);

            width = 0;                  // 格式字符 % 后面如果是个数字，这个数字最终会弄到width里面来，这个东西目前只对数字格式有效， 比如%d,%f这种
            sign  = 1;                  // 显示 是否是有符号数，这里给1， 表示是有符号数，除非你用 %u ，这个 u 表示无符号数
            hex   = 0;                  // 是否以16进制形式显示（比如显示一些地址）0：否， 1：是，并以小写字母显示a-f，     2：是，并以大写字母显示A-F
            frac_width = 0;             // 小数点后为数字，一帮需要和 %.10f 配合使用功能，这里10就是frac_width
            i64   = 0;                  // 一帮用 %d 对应的可变参数中的实际数字，会保留在这里
            ui64  = 0;                  // 一般用 %ud 对应的可变参数中的实际数字，会保留在这里

            // ------------------------------变量初始化结束------------------------------

            // 这个while就是判断 % 后面是否是一个数字，如果是一个数字，那就把这个数字取出来，比如 %16 ，最终这个循环就能够把 16 取出来弄到 width 里面去
            // %16d 这里最终 width =16;
            while (*fmt >= '0' && *fmt <= '9')  // 如果 % 后面接的字符是 '0' - '9' 之间的内容， 比如 %16 这种
            {
                // 第一次：width = 1; 第二次：width = 16, 所以整个 width = 16
                width = width * 10 + (*fmt++ - '0');
            }
            
            for (;;)    // 一些特殊格式，我们做一些特殊的标记【给一些变量特殊值等等】
            {
                switch (*fmt)   // 处理一些特殊字符
                {
                case 'u':       // %u 这个u表示无符号
                    sign = 0;   // 标记这是一个无符号数
                    fmt++;      // 往后走一个字符
                    continue;   // 回到for继续判断    

                case 'X':       // %X 这个X表示十六进制，并且十六进制A-F为大写，不要单独使用，一般都是 %Xd
                    hes = 2;    // 标记以大写字母显示十六进制中的A-F
                    fmt++;      // 往后走一个字符
                    continue;   // 回到for继续判断 

                case 'x':       // %x 这个x表示十六进制，并且十六进制a-f为小写，不要单独使用，一般都是 %xd
                    hes = 2;    // 标记小写字母显示十六进制中的a-f
                    fmt++;      // 往后走一个字符
                    continue;   // 回到for继续判断 

                case '.':       // %. 其后面必须跟一个数字，必须与 %f 配合使用，形如 %.f10f : 表示转换浮点数的小数部分位数， 比如 %.10f 表示转换浮点数时，小数点后面必须保证10为数字，不足10位则用0来补充
                    fmt++;      // 往后走一个字符，后面这个字符肯定是0-9之间，因为%.要求接一个数字先
                    while (*fmt >= '0' && *fmt <= '9')  // 如果是数字，一直循环，这个循环就能最终把诸如 %.10f 中的10提取出来
                    {
                        frac_width = frac_width * 10 + (*fmt++ - '0');
                    }
                    break;

                default:
                    break;
                }
            }

            switch (*fmt)
            {
            case '%':       // 只有 %% 时才会遇到这个情形，本意是打印一个 % 
                *buf++ = '%';
                fmt++;
                continue;
            case 'd':       // 显示整形数据，如果配置u一起使用，也就是 %ud ，则是显示无符号整形数据
                if(sign)    // 如果是有符号数
                {
                    i64 = (int64_t)va_arg(args, int);
                    // va_arg():遍历可变参数， var_arg 的第二个参数表示遍历的这个可变参数的类型
                }
                else
                {
                    // 如果是和 %ud 配合使用 ，则本条件成立
                    ui64 = (uint64_t) va_arg(args, u_int);
                }
                break;
                // 这个break掉直接跳转switch后面的代码去执行，这种凡是break的，都不做fmt++;
                // switch后面仍需进一步处理
            case 's':   // 一般用于显示字符串
                p = va_arg(args, u_char *);
                // va_arg()：遍历可变参数， var_arg的第二个参数表示遍历的这个可变参数的类型

                while (*p && buf < last)    // 没遇到字符串结束标记，并且buf值能都装下这个参数
                {
                    /* code */
                    *buf++ = *p++;  //那就装，比如  "%s"    ，   "abcdefg"，那abcdefg都被装进来
                }
            fmt++;
                continue; //重新从while开始执行 

            case 'P':  //转换一个pid_t类型
                i64 = (int64_t) va_arg(args, pid_t);
                sign = 1;
                break;

            case 'f': //一般 用于显示double类型数据，如果要显示小数部分，则要形如 %.5f  
                f = va_arg(args, double);  //va_arg():遍历可变参数，var_arg的第二个参数表示遍历的这个可变的参数的类型
                if (f < 0)  //负数的处理
                {
                    *buf++ = '-'; //单独搞个负号出来
                    f = -f; //那这里f应该是正数了!
                }
                //走到这里保证f肯定 >= 0【不为负数】
                ui64 = (int64_t) f; //正整数部分给到ui64里
                frac = 0;

                //如果要求小数点后显示多少位小数
                if (frac_width) //如果是%d.2f，那么frac_width就会是这里的2
                {
                    scale = 1;  //缩放从1开始
                    for (n = frac_width; n; n--) 
                    {
                        scale *= 10; //这可能溢出哦
                    }

                    //把小数部分取出来 ，比如如果是格式    %.2f   ，对应的参数是12.537
                    // (uint64_t) ((12.537 - (double) 12) * 100 + 0.5);  
                                //= (uint64_t) (0.537 * 100 + 0.5)  = (uint64_t) (53.7 + 0.5) = (uint64_t) (54.2) = 54
                    frac = (uint64_t) ((f - (double) ui64) * scale + 0.5);   //取得保留的那些小数位数，【比如  %.2f   ，对应的参数是12.537，取得的就是小数点后的2位四舍五入，也就是54】
                                                                             //如果是"%.6f", 21.378，那么这里frac = 378000

                    if (frac == scale)   //进位，比如    %.2f ，对应的参数是12.999，那么  = (uint64_t) (0.999 * 100 + 0.5)  = (uint64_t) (99.9 + 0.5) = (uint64_t) (100.4) = 100
                                          //而此时scale == 100，两者正好相等
                    {
                        ui64++;    //正整数部分进位
                        frac = 0;  //小数部分归0
                    }
                } //end if (frac_width)

                //正整数部分，先显示出来
                buf = ngx_sprintf_num(buf, last, ui64, zero, 0, width); //把一个数字 比如“1234567”弄到buffer中显示

                if (frac_width) //指定了显示多少位小数
                {
                    if (buf < last) 
                    {
                        *buf++ = '.'; //因为指定显示多少位小数，先把小数点增加进来
                    }
                    buf = ngx_sprintf_num(buf, last, frac, '0', 0, frac_width); //frac这里是小数部分，显示出来，不够的，前边填充'0'字符
                }
                fmt++;
                continue;  //重新从while开始执行

            //..................................
            //................其他格式符，逐步完善
            //..................................

            default:
                *buf++ = *fmt++; //往下移动一个字符
                continue; //注意这里不break，而是continue;而这个continue其实是continue到外层的while去了，也就是流程重新从while开头开始执行;
            } //end switch (*fmt) 
            
            //显示%d的，会走下来，其他走下来的格式日后逐步完善......

            //统一把显示的数字都保存到 ui64 里去；
            if (sign) //显示的是有符号数
            {
                if (i64 < 0)  //这可能是和%d格式对应的要显示的数字
                {
                    *buf++ = '-';  //小于0，自然要把负号先显示出来
                    ui64 = (uint64_t) -i64; //变成无符号数（正数）
                }
                else //显示正数
                {
                    ui64 = (uint64_t) i64;
                }
            } //end if (sign) 

            //把一个数字 比如“1234567”弄到buffer中显示，如果是要求10位，则前边会填充3个空格比如“   1234567”
            //注意第5个参数hex，是否以16进制显示，比如如果你是想以16进制显示一个数字则可以%Xd或者%xd，此时hex = 2或者1
            buf = ngx_sprintf_num(buf, last, ui64, zero, hex, width); 
            fmt++;
        }
        else  //当成正常字符，源【fmt】拷贝到目标【buf】里
        {
            //用fmt当前指向的字符赋给buf当前指向的位置，然后buf往前走一个字符位置，fmt当前走一个字符位置
            *buf++ = *fmt++;   //*和++优先级相同，结合性从右到左，所以先求的是buf++以及fmt++，但++是先用后加；
        } //end if (*fmt == '%') 
    }  //end while (*fmt && buf < last) 
    
    return buf;
}

```

ngx_sprintf_num()函数

```cxx
//----------------------------------------------------------------------------------------------------------------------
//以一个指定的宽度把一个数字显示在buf对应的内存中, 如果实际显示的数字位数 比指定的宽度要小 ,比如指定显示10位，而你实际要显示的只有“1234567”，那结果可能是会显示“   1234567”
     //当然如果你不指定宽度【参数width=0】，则按实际宽度显示
     //你给进来一个%Xd之类的，还能以十六进制数字格式显示出来
//buf：往这里放数据
//last：放的数据不要超过这里
//ui64：显示的数字         
//zero:显示内容时，格式字符%后边接的是否是个'0',如果是zero = '0'，否则zero = ' ' 【一般显示的数字位数不足要求的，则用这个字符填充】，比如要显示10位，而实际只有7位，则后边填充3个这个字符；
//hexadecimal：是否显示成十六进制数字 0：不
//width:显示内容时，格式化字符%后接的如果是个数字比如%16，那么width=16，所以这个是希望显示的宽度值【如果实际显示的内容不够，则后头用0填充】
static u_char * ngx_sprintf_num(u_char *buf, u_char *last, uint64_t ui64, u_char zero, uintptr_t hexadecimal, uintptr_t width)
{
    //temp[21]
    u_char      *p, temp[NGX_INT64_LEN + 1];   //#define NGX_INT64_LEN   (sizeof("-9223372036854775808") - 1)     = 20   ，注意这里是sizeof是包括末尾的\0，不是strlen；             
    size_t      len;
    uint32_t    ui32;

    static u_char   hex[] = "0123456789abcdef";  //跟把一个10进制数显示成16进制有关，换句话说和  %xd格式符有关，显示的16进制数中a-f小写
    static u_char   HEX[] = "0123456789ABCDEF";  //跟把一个10进制数显示成16进制有关，换句话说和  %Xd格式符有关，显示的16进制数中A-F大写

    p = temp + NGX_INT64_LEN; //NGX_INT64_LEN = 20,所以 p指向的是temp[20]那个位置，也就是数组最后一个元素位置

    if (hexadecimal == 0)  
    {
        if (ui64 <= (uint64_t) NGX_MAX_UINT32_VALUE)   //NGX_MAX_UINT32_VALUE :最大的32位无符号数：十进制是‭4294967295‬
        {
            ui32 = (uint32_t) ui64; //能保存下
            do  //这个循环能够把诸如 7654321这个数字保存成：temp[13]=7,temp[14]=6,temp[15]=5,temp[16]=4,temp[17]=3,temp[18]=2,temp[19]=1
                  //而且的包括temp[0..12]以及temp[20]都是不确定的值
            {
                *--p = (u_char) (ui32 % 10 + '0');  //把屁股后边这个数字拿出来往数组里装，并且是倒着装：屁股后的也往数组下标大的位置装；
            }
            while (ui32 /= 10); //每次缩小10倍等于去掉屁股后边这个数字
        }
        else
        {
            do 
            {
                *--p = (u_char) (ui64 % 10 + '0');
            } while (ui64 /= 10); //每次缩小10倍等于去掉屁股后边这个数字
        }
    }
    else if (hexadecimal == 1)  //如果显示一个十六进制数字，格式符为：%xd，则这个条件成立，要以16进制数字形式显示出来这个十进制数,a-f小写
    {
        //比如我显示一个1,234,567【十进制数】，他对应的二进制数实际是 12 D687 ，那怎么显示出这个12D687来呢？
        do 
        {            
            //0xf就是二进制的1111,大家都学习过位运算，ui64 & 0xf，就等于把 一个数的最末尾的4个二进制位拿出来；
            //ui64 & 0xf  其实就能分别得到 这个16进制数也就是 7,8,6,D,2,1这个数字，转成 (uint32_t) ，然后以这个为hex的下标，找到这几个数字的对应的能够显示的字符；
            *--p = hex[(uint32_t) (ui64 & 0xf)];    
        } while (ui64 >>= 4);    //ui64 >>= 4     --->   ui64 = ui64 >> 4 ,而ui64 >> 4是啥，实际上就是右移4位，就是除以16,因为右移4位就等于移动了1111；
                                 //相当于把该16进制数的最末尾一位干掉，原来是 12 D687, >> 4后是 12 D68，如此反复，最终肯定有=0时导致while不成立退出循环
                                  //比如 1234567 / 16 = 77160(0x12D68) 
                                  // 77160 / 16 = 4822(0x12D6)
    } 
    else // hexadecimal == 2    //如果显示一个十六进制数字，格式符为：%Xd，则这个条件成立，要以16进制数字形式显示出来这个十进制数,A-F大写
    { 
        //参考else if (hexadecimal == 1)，非常类似
        do 
        { 
            *--p = HEX[(uint32_t) (ui64 & 0xf)];
        } while (ui64 >>= 4);
    }

    len = (temp + NGX_INT64_LEN) - p;  //得到这个数字的宽度，比如 “7654321”这个数字 ,len = 7

    while (len++ < width && buf < last)  //如果你希望显示的宽度是10个宽度【%12f】，而实际想显示的是7654321，只有7个宽度，那么这里要填充5个0进去到末尾，凑够要求的宽度
    {
        *buf++ = zero;  //填充0进去到buffer中（往末尾增加），比如你用格式  
                                          //ngx_log_stderr(0, "invalid option: %10d\n", 21); 
                                          //显示的结果是：nginx: invalid option:         21  ---21前面有8个空格，这8个弄个，就是在这里添加进去的；
    }
    
    len = (temp + NGX_INT64_LEN) - p; //还原这个len，也就是要显示的数字的实际宽度【因为上边这个while循环改变了len的值】
    //现在还没把实际的数字比如“7654321”往buf里拷贝呢，要准备拷贝

    //如下这个等号是我加的【我认为应该加等号】，nginx源码里并没有加;***********************************************
    if((buf + len) >= last)   //发现如果往buf里拷贝“7654321”后，会导致buf不够长【剩余的空间不够拷贝整个数字】
    {
        len = last - buf; //剩余的buf有多少我就拷贝多少
    }

    return ngx_cpymem(buf, p, len); //把最新buf返回去；
}

```


# （2）设置时区

上面只是实现了日志格式的转换，但是并没有真正的提供日志的输出接口，因为一条日志，肯定是需要时间信息的

```shell
# 查看本机（虚拟机）的时区信息
date
# 如果打印不是 CST 时区，那么就需要调整为CST（北京时区）
```

我们需要设置成CST时区，以保证日期时间显示正确

常见的时区归纳

<strong style="color: #E91E63">
GMT 格林威治标准时间 GMT    等同于英国伦敦本地时间

UTC 全球标准时间 GMT    通用协调时

ECT 欧洲中部时间 GMT+1:00                     ** 也有用CET的

CST 中部标准时间 GMT-6:00       北京时间

PST 美国太平洋标准时间 GMT-8:00                   **

</strong>
EET 东欧时间 GMT+2:00

ART （阿拉伯）埃及标准时间 GMT+2:00

EAT 东非时间 GMT+3:00

MET 中东时间 GMT+3:30
、NET 近东时间 GMT+4:00

PLT 巴基斯坦拉合尔时间 GMT+5:00

IST 印度标准时间 GMT+5:30

BST 孟加拉国标准时间 GMT+6:00

VST 越南标准时间 GMT+7:00

CTT 中国台湾时间 GMT+8:00

JST 日本标准时间 GMT+9:00

ACT 澳大利亚中部时间 GMT+9:30

AET 澳大利亚东部时间 GMT+10:00

SST 所罗门标准时间 GMT+11:00

NST 新西兰标准时间 GMT+12:00

MIT 中途岛时间 GMT-11:00

HST 夏威夷标准时间 GMT-10:00

AST 阿拉斯加标准时间 GMT-9:00

PNT 菲尼克斯标准时间 GMT-7:00

MST 西部山脉标准时间 GMT-7:00            



修改设置时区

```shell
# 方法一
tzselect

# 然后选择 Asia 亚洲

# 然后选择国家 中国

# 然后选择时间 北京时间

# 方法二 仅限于RedHat Linux 和 CentOS系统

# 然后上述操作执行完之后，生成了一个文件，需要把这个文件拷贝到一个目录下才能保证修改后下次启动也生效

sudo cp /usr/share/zoneinfo/Asia/ShangHai /etc/localtime

timeconfig
# 方法三 适用于Debian
dpkg-reconfigure tzdata
# 方法四 复制相应的时区文件，替换CentOS系统时区文件；或者创建链接文件
cp /usr/share/zoneinfo/EST5EDT /etc/localtime
# 或者
ln -s /usr/share/zoneinfo/EST5EDT /etc/localtime
# 时间同步
ntp
yum instlal ntp -y
# 加入crontab
0-59/10 * * * * /usr/sbin/ntpdate us.pool.ntp.org | logger -t NTP
```

# （3）基础设施之日志打印实战代码二
## （3.1）日志等级划分

```cxx
// 日志相关-------------------------------------------------
// 这里把日志一共分为八个等级【级别从高到底，数字最小的级别最高，数字大的级别最低】
// 方便管理，显示，过滤等

#define NGX_LOG_STDERR            0    //控制台错误【stderr】：最高级别日志，日志的内容不再写入log参数指定的文件，而是会直接将日志输出到标准错误设备比如控制台屏幕
#define NGX_LOG_EMERG             1    //紧急 【emerg】
#define NGX_LOG_ALERT             2    //警戒 【alert】
#define NGX_LOG_CRIT              3    //严重 【crit】
#define NGX_LOG_ERR               4    //错误 【error】：属于常用级别
#define NGX_LOG_WARN              5    //警告 【warn】：属于常用级别
#define NGX_LOG_NOTICE            6    //注意 【notice】
#define NGX_LOG_INFO              7    //信息 【info】
#define NGX_LOG_DEBUG             8    //调试 【debug】：最低级别
```

## （3.2）配置文件中和日志有关的选项

```conf
# 日志相关
[Log]
# 日志文件输出目录和文件名
# Log=logs/error.log
Log=logs/error.log

# 只打印日志等级 <= 数字 的日志到日志文件中，日志等级 0 - 8 ，8 级别最低
LogLevel = 8

```

void ngx_log_init() 函数

```cxx
// 描述：日志初始化，就是把日志文件打开，这里涉及到释放问题，如何解决
void ngx_log_init()
{
    u_char *plogname = NULL;
    size_t nlen;

    // 从配置文件中读取日志相关的配置信息
    CConfig *p_config = CConfig::GetInstance();
    plogname = (u_char *)p_config->GetString("Log");
    if(plogname == NUll)
    {
        // 没读到，就需要提供一个缺省的路径文件名
        plogname = (u_char *)NGX_ERROR_LOG_PATH;    // "logs/error.log", logs目录需要提前建立出来

    }

    ngx_log.log_level = p_config->GetIntDefault("LogLevel", NGX_LOG_NOTICE);
    // 缺省的日志等级为 6 【注意】,如果读失败，就给缺省的日志等级
    // nlen = strlen((const char *)plogname);

    // 只写打开|追加到末尾|文件不存在则创建文件 【这3个参数指定文件访问权限】
    // mode = 0644:文件访问权限， 6:110， 4:100， 【用户：读写，   用户所在组：读，   其他：读】
    ngx_log.fd = open((const char *)plogname, O_WRONLY|O_AOOEND|O_CREAT, 0644);
    if(ngx_log.fd==-1)  // 如果有错误，则直接定位到 标准错误上去
    {
        ngx_log_stderr(errno, "[alert] could not open error log file: open() \"%s\" failed", plogname);
        ngx_log.fd = STDERR_FILENO; // 直接定位到标准错误
    }
    
    return;
}
```

ngx_log_error_core()函数

```cxx
// --------------------------------------------------------------------------------------------------------------------------------------
// 往文件中写日志，代码中有自动加换行符，所以调用时字符串不用刻意加\n
// 日志定位标准错误，则直接往屏幕上写日志【比如日志文件打不开，这回直接定位到标准错误，此时日志就打印到屏幕上，参考 ngx_log_init()】
// 
// level：一个等级数字，如果我们把日志分为一些等级，已方便管理，显示，过滤等，如果这个等级数字比配置文件中的等级数字“LogLevel”大，那么这条信息就不会被写入到日志文件中
// err: 是个错误代码，如果不是0，就应该转换为显示对应的错误信息，一起写入到日志文件中
// ngx_log_error_core(5,7,"这个xxx工作空间有问题，显示的结果是= %s", "yyyyyyyyyy");

void ngx_log_error_core(int level, int err, const char *fmt, ...)
{
    u_char *last;
    u_char errstr[NGX_MAX_ERROR_STR+1];
    // 这个+1 可以参考ngx_log_stderr()函数的写法

    memset(errstr, 0, sizeof(errstr));
    last = errstr + NGX_MAX_ERROR_STR;

    struct timeval  tv;
    struct tm       tm;
    time_t          sec;    //  秒
    u_char          *p;     // 指向当前要拷贝数据到其中的内存位置
    va_list         args;

    memset(&tv, 0, sizeof(struct timeval));
    memset(&tm, 0, sizeof(struct tm));

    gettimofday(&tv, NULL);
    // 获取当前时间，返回的是自1970-01-01 00：00:00到现在经历的秒数【第二个参数是时区，一般不关心】

    sec = tv.tv_sec;                // 秒
    localtime_r(&sec, &tm);         // 把参数1的time_t转换为本地时间，保存到参数2中去，带_r的是线程安全版本
    tm.tm_mon++;                    // 月份要调整一下才正常
    tm.tm_year += 1900;             // 年份也要调整一下才正常

    u_char strcurrtime[40]={0};     // 先组合出一个当前时间字符串，格式形如： 2019/01/08 12:32:23

    ngx_slprintf(strcurrtime,
                (u_char *)-1,                       // 若是用一个u_char *接一个 (u_char *)-1,则得到的结果是 0xffffffff... 这个值足够大
                "%4d/%02d/%02d %02d:%02d:%02d",     // 格式是 年/月/日 时：分：秒
                tm.tm_year, tm.tm_mon,
                tm.tm_mday, tm.tm_hour,
                tm.tm_min, tm.tm_sec
    );
    p = ngx_cpymem(errstr, strcurrtime,strlen((const char *)strcurrtime));
    // 日期增加进来，得到形如   2019/01/08 20:26:07
    p = ngx_slprintf(p, last, " [%s] ", err_levels[level]);
    // 日志级别加进来，得到形如： 2019/01/08 20:26:07 [crit] 
    p = ngx_slprintf(p, last, "%p: ", ngx_pid);
    // 支持%p格式，进程ID增加进来，得到形如：2019/01/08 20:50:15 [crit] 2037:

    va_start(args, fmt);                // 使得args指向其实参数
    p = ngx_vslprintf(p, last, fmt, args); 
    // 把fmt和args参数弄进去，组合出来这个字符串
    va_end(args);                       // 释放args
    
    if(err)     // 如果错误代码不是0，表示有错误发生
    {
        // 错误代码和错误信息也要心事出来
        p = ngx_log_errno(p, last, err);
    }
    // 如果位置不够，那换行也要硬插入到末尾，哪怕覆盖其他内容
    if(p>=(last -1))
    {
        p = (last -1) - 1;
    }
    *p++ = '\n';
    // 增加换行符

    // 这么写是为了图方便：随时可以把流程弄到while后面去
    ssize_t n;
    while (1)
    {
        if(level > ngx_log.log_level)
        {
            // 要打印的这个日志等级态落后，（等级数字太大，比配置文件中的数字大）
            // 这种日志就不打印了
            break;
        }

        // 磁盘是否满了判断
        // todolist

        // 写日志文件
        n = write(ngx_log.fd, errstr, p-errstr);
        // 文件写入成功后，如果中途
        if(n==-1)
        {
            // 写入失败
            if(errno == ENOSPC) // 写失败了，且原因是磁盘没空间了
            {
                // todo
            }
            else
            {
                // 这里有其他错误，考虑把这些错误显示到标准错误设备
                if(ngx_log.fd != STDERR_FILENO) // 当前是定位到文件的，则条件成立
                {
                    n = write(STDERR_FILENO, errstr, p - errstr);
                }
            }
        }

        break;

    }
    
    return;
    
}

```

# （4）捋顺main函数中代码执行顺序

```cxx
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

#include "ngx_c_conf.h"     // 和配置文件处理相关的类，名字带C_表示和类有关
#include "ngx_func.h"   // 头文件路径，已经使用gcc -I 参数指定了 各种函数声明
#include "ngx_signal.h"

// 和设置标题有关的全局量
char **g_os_argv;   // 原始命令行参数数组，在main中会被赋值
char *gp_envmem = NULL; // 指向自己分配的env环境变量的内存
int g_environlen = 0;   // 环境变量所占内存的大小

//和进程本身有关的全局量
pid_t ngx_pid;               //当前进程的pid


int main(int argc, char *const *argv)
{   
    int exitcode = 0;           //退出代码，先给0表示正常退出

    //(1)无伤大雅也不需要释放的放最上边    
    ngx_pid = getpid();         //取得进程pid
    g_os_argv = (char **) argv; //保存参数指针    

    //(2)初始化失败，就要直接退出的
    //配置文件必须最先要，后边初始化啥的都用，所以先把配置读出来，供后续使用 
    CConfig *p_config = CConfig::GetInstance(); //单例类
    if(p_config->Load("nginx.conf") == false) //把配置文件内容载入到内存        
    {        
        ngx_log_stderr(0,"配置文件[%s]载入失败，退出!","nginx.conf");
        //exit(1);终止进程，在main中出现和return效果一样 ,exit(0)表示程序正常, exit(1)/exit(-1)表示程序异常退出，exit(2)表示表示系统找不到指定的文件
        exitcode = 2; //标记找不到文件
        goto lblexit;
    }
    
    //(3)一些初始化函数，准备放这里
    ngx_log_init();             //日志初始化(创建/打开日志文件)


    //(4)一些不好归类的其他类别的代码，准备放这里
    ngx_init_setproctitle();    //把环境变量搬家

    
    //--------------------------------------------------------------    
    for(;;)
    //for(int i = 0; i < 10;++i)
    {
        sleep(1); //休息1秒        
        printf("休息1秒\n");        

    }
      
    //--------------------------------------
lblexit:
    //(5)该释放的资源要释放掉
    freeresource();  //一系列的main返回前的释放动作函数
    printf("程序退出，再见!\n");
    return exitcode;
}

//专门在程序执行末尾释放资源的函数【一系列的main返回前的释放动作函数】
void freeresource()
{
    //(1)对于因为设置可执行程序标题导致的环境变量分配的内存，我们应该释放
    if(gp_envmem)
    {
        delete []gp_envmem;
        gp_envmem = NULL;
    }

    //(2)关闭日志文件
    if(ngx_log.fd != STDERR_FILENO && ngx_log.fd != -1)  
    {        
        close(ngx_log.fd); //不用判断结果了
        ngx_log.fd = -1; //标记下，防止被再次close吧        
    }
}

```