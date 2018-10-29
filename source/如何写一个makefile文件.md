如何写一个Makefile文件
===
目录
[toc]

## 目的
编写一个实用的makefile， 能自动编译当前目录下所有.c/.cpp源文件， 支持二者的混合编译。 并且当某个.c/.cpp、.h或者依赖的源文件被修改后， 仅重编涉及到的源文件， 未涉及的文件不编译。

## 用到的技术
1. 使用wildcard函数来获得当前目录下所有.c/.cpp文件的列表
2. make多目标的规则
3. make的模式规则
4. 用`gcc -MM`命令得到一个.c/.cpp文件include了哪些文件([具体使用细节，请点击博文链接](https://blog.csdn.net/qq1452008/article/details/50855810))
5. 用set命令对`gcc -MM`命令的结果作修改
6. 用include命令包含依赖描述文件`.d`

## 知识点准备
### 多目标
对makefile里下面2行，可以看出多目标特征, 执行make bigoutput或make littleoutput可以看到结果:
```makefile
bigoutput littleoutput:defs.h push.h
    @echo $@ $(subst output, PUTPUT,#@) $^
注释: $@指该规则目标集合中能引起该规则执行的目标, $^指这个规则里所有依赖的集合. 
该行是把目标(bigoutput或者littleoutput)里的子串output替换成大写的OUTPUT
```

### 隐含规则
对makefile里下面4行, 可以看出make的隐含规则, 执行foo可看到结果:
第3行和第四行由.c得到.o文件, 第1行和第二行表示由.o得到可执行文件.
如果把第3, 4行注释的话, 效果一样.

即不写.o来自.c的规则, 它会自动执行`gcc -c -o foo.o foo.c`这条命令, 由.c编译出.o(其中-c表示只编译不链接), 然后自动执行`gcc -o foo foo.o`链接为可执行文件.

```makefile
foo:foo.o
    gcc -o foo foo.o; ./foo # 链接目标文件成可执行文件, 并运行该文件
foo.o:foo.c #注释该行看效果
    gcc -c foo.c -o foo.o # 注释该行看效果
```

## 定义规则模式
下面定义了一个规则模式, 即如何由.c文件生成.d文件的工资总额.

```makefile
foobar: foo.d bar.d
    @echo complete generate foo.d and bar.d

#make会对当前目录下的每个.c文件, 依次执行该依赖规则里面的命令,使每个.c文件生成对应的.d文件
%.d:%.c
    @echo from $< to $@
    g++ -MM $ < > $@
```
假定当前目录下有2个.c文件: foo.c和bar.c(文件的内容随意).
验证的方法有两种, 都可:
1. 运行make foo.d(或者make bar.d), 表示想生成foo.d这个目标.
根据规则%.d: %.c这时%匹配foo, 这样%.c等于foo.c, 即foo.d这个目标依赖于foo.c
此时会自动执行该规则里的命令gcc-MM foo.c>foo.c, 来生成foo.d这个目标.
2. 运行make foo bar, 因为foo bar 依赖于foo.d和bar.d这两个文件, 即会一次性生成这两个文件

## 下面详述如何生成依赖性, 从而实现本例的makefile
### 本例使用了makefile的模式工资总额, 目的是对当前目录下每个.c文件, 生成其对应的.d文件, 例如由main.c生成的.d文件内容为:
```makefile
main.o : main.c command.h
```
这里指示了main.o目标依赖于那几个源文件, 我们只要把这一行的内容, 通过make的include指令包含到makefile文件里, 即可在其任意一个依赖文件被修改后, 重新编译目标main.o

下面详解如何生成这个.d文件
### gcc/g++编译器有一个-MM选项, 可以对某个.c/.cpp文件, 分析其依赖的源文件, 例如假定main.c的内容为:
```c
#include <stdio.h> // 标准头文件(以<>方式包含的), 被-MM选项忽略, 被-M选项收集
#include "stdlib.h" // 标准头文件(以""方式包含的), 被-MM选项忽略, 被-M选项收集
#include "command.h"
int main()
{
    printf("##### Hello Makefile #####\n");
    return 0;
}
```
则执行
```
gcc -MM main.c后, 屏幕输出:
    main.o: mian.c command.h

cc -M main.c 后, 屏幕输出:
    main.o: main.c /usr/include/stdio.h /usr/include/features.h \  
    /usr/include/bits/predefs.h /usr/include/sys/cdefs.h \  
    /usr/include/bits/wordsize.h /usr/include/gnu/stubs.h \  
    /usr/include/gnu/stubs-64.h \  
    /usr/lib/gcc/x86_64-linux-gnu/4.4.3/include/stddef.h \  
    /usr/include/bits/types.h /usr/include/bits/typesizes.h \  
    /usr/include/libio.h /usr/include/_G_config.h /usr/include/wchar.h \  
    /usr/lib/gcc/x86_64-linux-gnu/4.4.3/include/stdarg.h \  
    /usr/include/bits/stdio_lim.h /usr/include/bits/sys_errlist.h \  
    /usr/include/stdlib.h /usr/include/sys/types.h /usr/include/time.h \  
    /usr/include/endian.h /usr/include/bits/endian.h \  
    /usr/include/bits/byteswap.h /usr/include/sys/select.h \  
    /usr/include/bits/select.h /usr/include/bits/sigset.h \  
    /usr/include/bits/time.h /usr/include/sys/sysmacros.h \  
    /usr/include/bits/pthreadtypes.h /usr/include/alloca.h command.h
```

### 可见, 只要把这些行挪到makefile, 就能自动定义main.c的依赖是哪些文件了, 做法是把命令的输出重定向到.d文件里面:`gcc -MM main.c > main.d`, 再把这个.d文件include到makefile里.('>'是重定向符号, 把生成的数据重定向写入制定的文件中)

如何include当前目录每个.c文件生成的.d文件:
```makefile
#使用$(wildcard *.c)来获取工作目录下的所有.c文件的列表.
sources:=$(wildcard *.c)
#这里的dependence是所有.d文件的列表, 即把source串里的.c换成.d
dependence=$(sources:.c=.d)
#include后面可以跟若干个文件名, 用空格分开, 支持通配符,
include $(dependence)

-------
#例如:
#这里是把所有的.d文件一次性全部include进来 注意该句要放在终极目标all的规则之后, 否则.d文件里的规则会被误当作终极规则了.
include foo.make *.mk
```

### 重编译
现在main.c command.h这几个文件, 任何一个改了都会重编main.o. 但这里还有一个问题, 如果修改了command.h, 即在command.h中假如#include "pub.h", 这时:
1. 再make, 由于command.h改了, 这时会重编main.o, 并且会使用新加的pub.h, 看起来是正常的.
2. 这时打开main.d查看, 发现main.d中未假如pub.h, 因为根据模式规则%.d:%.c中的定义, 只有依赖的.c文件变了, 才会重新生成.d, 而刚刚改得是command.h, 不会重新生成main.d, 即不会在main.d中假如对pub.h的依赖关系, 这样以来会导致问题
3. 修改新的pub.h的内容, 再make, 果然出问题了, make报告up to date, 没有像期望那样重编译main.o
**现在的问题在于, main.d里的某个.h文件改了, 没有重新生成main.d, 进一步说, main.d里给出的每个依赖文件, 任何一个改了, 都要生成这个main.d. 所以main.d也要作为一个目标来生成, 它的依赖应该是main.d里的每个依赖文件, 也就是说make里要有这样的定义:**
main.d:main.c command.h
这时我们发现, main.d与main.o的依赖是完全相同的, 可以利用make的多目标规则, 把main.d与main.o这两个目标的定义合并为一句:
```makefile
main.o main.d: main.c command.h
```
现在, main.o:main.c command.h这一句我们已经有了, 如何进一步得到main.o main.d:main.c command.h呢?

###
解决方法是行内字符串替换, 对main.o, 取出其中的字串main, 加上.d后缀得到的main.d, 再插入到main.o后面, 能实现替换这种功能的命令是sed.
实现的时候, 先把gcc -MM命令生成临时文件main.d.temp,再用sed命令从该临时文件中读出内容(用<重定向输入). 做替换后, 再用>输出到最终文件main.d
命令可以这么写:
```makefile
g++ -MM main.c > main.d.temp
sed 's,main\.o[ :]*,\1.o main.d :,g' < main.d temp > main.d
```


### 
```c
1. 这条sed命令的结构是s/match/replace/g有时为了清晰, 可以把每个/写成逗号,即这里的格式s,match,replace,g
2. 该命令表示被源串的match都替换成replace, s指示match可以是正则表达式. g表示把每行内所有match都替换, 如果去掉g, 则只有每行的第一处match被替换(实际上都不需要g, 因为一个.d文件中, 只会在开头有一个main.o:).
3. 这里match是正则式\(main\)\.o[ :]*,它分成3段:
第1段是main, 在sed命令里被main用括号括起来, 使接下来的replace中可以用\1引用main.
第2段是\.o
```




[参考地址](https://blog.csdn.net/qq1452008/article/details/50865535)

# makefile详解

## cmake的大致介绍
写程序的大致步骤为:
1. 用编译器编写源代码, 如.c文件.
2. 用编译器编译代码生成目标文件, 如.o
3. 用链接器连接目标代码生成可执行文件, 如.exe

但是如果源文件太多, 一个一个编译时就会特别麻烦, 于是人们想到, 为什么不设计一种类似批处理的程序, 来批处理编译源文件呢, 于是就有了make工具, 它是一个自动化编译工具, 你可以使用一条命令实现完全编译. 但是你需要编写一个规则文件呢, 于是就有了make工具, 它是一个自动化编译工具, 你可以使用一条命令实现完全编译. 但是你需要编写一个规则文件, make依据它来批处理编译, 这个文件就是makefile, 所以编写makefile文件也是一个程序员所必备的技能.

对于一个大工程, 编写makefile实在是件复杂的事, 于是人们又想, 为什么不设计一个工具, 读入所有源文件之后, 自动生成makefile呢, 于是出现了cmake工具, 它能够输出各种各样的makefile获得project文件, 从而帮助程序员减轻负担. 但是随之而来的就是编写cmakelist文件, 它是cmake所依据的规则. 所以在编程的世界里没有捷径可走, 还是要脚踏实地的.

[参考地址](https://blog.csdn.net/freeking101/article/details/51610782)