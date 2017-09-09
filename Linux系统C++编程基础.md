最近在复习Linux系统的一些知识，Linux系统下C/C++编程最基本的就是这三部分：GCC，GDB和Makefile。这个笔记做的很简略，只是为了帮助自己回忆最基本的知识点，想要深入了解还需要看更多的文档。

<h2><font color=#0099FF>GCC</font></h2> 

GCC是Linux最常用的编译器，基本语法格式为：`gcc [options][filenames]`

options参数有很多种，现列一些常用的参数如下：

> -c，只编译，不链接成为可执行文件，编译器只是由输入的.c等源代码文件生成.o为后缀的目标文件，通常用于编译不包含主程序的子程序文件。
> -o output_filename，确定输出文件的名称为output_filename，同时这个名称不能和源文件同名。如果不给出这个选项，gcc就给出预设的可执行文件a.out。
> -g，产生符号调试工具(gdb)所必要的符号资讯，要想对源代码进行调试，我们就必须加入这个选项。
> -O，对程序进行优化编译、链接，采用这个选项，整个源代码会在编译、链接过程中进行优化处理，这样产生的可执行文件的执行效率可以提高，但是编译、链接的速度就相应地要慢一些。
> -O2，比-O更好的优化编译、链接，当然整个编译、链接过程会更慢。
> -Wall，开启编译器几乎所有常用的警告，有助于检测程序存在的问题

下面用一些例子来说明如何简单使用gcc命令

home目录下有一个`hello.c` 源文件，其内容如下：

```c
#include <cstdio.h>
int main()
{
    printf("hello,world\n");
  	return 0;
}
```

<h3><font color=#0099FF>case1</font></h3> 

```shell
gcc -c hello.c # 对hello.c只编译不链接
```

生成一个`hello.o` 文件，其扩展名为`.obj`，是目标文件，无法执行。想要执行，必须再将其转换为可执行文件

```shell
gcc hello.o -o abc # 如果不加abc，则默认生成a.out可执行文件
```

执行命令后，生成一个`abc` 文件，无扩展名，linux下的可执行文件，这时，再用`./abc` 即可执行，打印输出。当然，也可以将多个`.o` 文件一起链接，生成一个可执行文件。

<h3><font color=#0099FF>case2</font></h3> 

```shell
gcc hello.c -o abc
```

直接生成了`abc` 可执行文件，说明同时完成了编译链接的工作。

```shell
gcc hello.c -O0
gcc hello.c -O1
gcc hello.c -O2 # 使用优化编译和链接，速度慢
```

这个貌似无法添加输出的name参数，只能这样，默认输出`a.out` 文件，`./a.out` 一样能执行

<h3><font color=#0099FF>case3</font></h3> 

链接多个程序，假设`/home` 目录下有三个文件：hello.c， hello_fn.c和头文件hello.h，其内容分别如下：

```c
// hello.h
void hello (const char * name);

// hello_fun.c
#include <stdio.h> 
#include "hello.h" 
void hello (const char * name) 
{ 
	printf ("Hello, %s!\n", name);
}

// hello.c
#include "hello.h" 
int main() 
{ 
	hello("world"); // hello函数在hello.h声明，在hello_fun.c中定义
	return 0;
}
```

下面编译源文件，使用如下命令：

```shell
gcc -Wall hello.c hello_fn.c -o newhello
```

注意到头文件`hello.h`并未在命令行中指定，源文件中的的 `#include "hello.h"` 指示符使得编译器自动将其包含到合适的位置。执行后生成`newhello` 可执行文件。

<h3><font color=#0099FF>gcc和g++</font></h3> 

gcc既可以编译C程序，也可以编译C++程序（通过扩展名自动确认），但是gcc无法自动链接C++库函数，比如STL。因此，编译C++程序的任务还是交给g++比较好。

- 编译阶段是相同的，链接阶段g++默认链接c++库，gcc没有。
- 一般情况下用gcc编译c文件，用g++编译cpp文件。
- 也可以用gcc编译cpp文件，但后面需要加一个选项`-l stdc++`，作用是链接C++库。
- `gcc train.cpp -o exe -l stdc++` 和`g++ train.cpp -o exe` 算是等效的。

<h2><font color=#0099FF>GDB</font></h2> 

GDB是Linux下常用的调试工具，其主要功能有如下4点：

- 启动你的程序，可以按照你的自定义的要求随心所欲的运行程序。
- 可让被调试的程序在你所指定的调置的[断点](https://baike.baidu.com/item/%E6%96%AD%E7%82%B9)处停住。（断点可以是[条件表达式](https://baike.baidu.com/item/%E6%9D%A1%E4%BB%B6%E8%A1%A8%E8%BE%BE%E5%BC%8F)）
- 当程序被停住时，可以检查此时你的程序中所发生的事。
- 你可以改变你的程序，将一个BUG产生的影响修正从而测试其他BUG。

<h3><font color=#0099FF>基本用法</font></h3> 

GDB主要调试的是C/C++程序。要调试C/C++的程序，首先在编译时，必须要把调试信息加到可执行文件中。使用编译器(cc/gcc/g++)的 -g 参数即可，如下所示：

```shell
gcc -g hello.c -o hello
g++ -g hello.cpp -o hello
```

编译成功后，可以使用`gdb hello` 启动GDB调试程序，然后使用一些命令调试程序:

- `l` 列出程序内容
- `break 16`在16行设置断点
- `break func`在func函数入口设置断点
- `info break`可以查看断点信息
- `r`执行程序，
- `n`是单步运行
- `c`继续运行程序
- `p sum`打印变量sum的值
- `bt`可以查看函数堆栈
- `finish`可以退出函数
- `until num` 跳到指定行num，结束循环
- `q`退出gdb

更多GDB调试技巧参考[100个gdb小技巧](https://gitlore.com/subject/15)

<h2><font color=#0099FF>Makefile</font></h2> 

Makefile文件中描述了整个工程所有文件的编译顺序、编译规则（哪些文件需要先编译，哪些文件需要后编译，哪些文件需要重新编译），可以使用make命令来解析这些规则。

C/C++源程序变为可执行程序需要两步，首先是**编译**（compile），将源文件转化为obj文件或o文件，第二步是**链接**（link） ，将大量的obj文件合成为可执行文件。

编译时，编译器需要的是语法的正确，函数与变量的声明的正确。对于后者，通常是你需要告诉编译器头文件的所在位置（头文件中应该只是声明，而定义应该放在C/C++文件中），只要所有的语法正确，编译器就可以编译出中间目标文件。

链接时，主要是链接函数和全局变量，所以，我们可以使用这些中间目标文件（o文件或是obj文件）来链接我们的应用程序。

<h3><font color=#0099FF>基本规则</font></h3> 

```shell
目标 : 需要的条件 （注意冒号两边有空格）
　　　　命令　  　（注意前面用tab键开头）
```

- 目标是想要生成的文件，可以是obj文件，也可以是可执行文件等等；
- 需要的条件是目标所需的源文件或者obj；
- 命令是生成目标执行的gcc/g++命令或其他命令；

总结就是：目标文件依赖于条件，生成则使用相应的命令。**如果需要的条件的文件比目标更新的话，就会执行生成命令来更新目标。**

<h3><font color=#0099FF>case1</font></h3> 

如果一个工程有3个头文件，和8个C文件，最朴素的Makefile应该是下面的这个样子的。

```makefile
output : main.o kbd.o command.o display.o /
           insert.o search.o files.o utils.o
            gcc -o edit main.o kbd.o command.o display.o /
                       insert.o search.o files.o utils.o

    main.o : main.c defs.h
            gcc -c main.c
    kbd.o : kbd.c defs.h command.h
            gcc -c kbd.c
    command.o : command.c defs.h command.h
            gcc -c command.c
    display.o : display.c defs.h buffer.h
            gcc -c display.c
    insert.o : insert.c defs.h buffer.h
            gcc -c insert.c
    search.o : search.c defs.h buffer.h
            gcc -c search.c
    files.o : files.c defs.h buffer.h command.h
            gcc -c files.c
    utils.o : utils.c defs.h
            gcc -c utils.c
    clean :
            rm output main.o kbd.o command.o display.o /
               insert.o search.o files.o utils.o
```

很明显，output是最终想得到的可执行文件（Linux系统下可执行文件一般不给扩展名），那具体的工作流程是：

1. make会在当前目录下找名字叫“Makefile”或“makefile”的文件。
2. 如果找到，它会找文件中的第一个目标文件（target），在上面的例子中，他会找到“edit”这个文件，并把这个文件作为最终的目标文件。
3. 如果edit文件不存在，或是edit所依赖的后面的o文件的文件修改时间要比edit这个文件新，那么就会执行后面所定义的命令来生成edit这个文件。
4. 如果edit所依赖的o文件也不存在，那么make会在当前文件中找目标为o文件的依赖性，如果找到则再根据那一个规则生成o文件。（这有点像一个堆栈的过程）
5. 当然c文件和h文件是存在的，于是make会生成o文件，然后再用o文件生成make的终极任务，也就是执行文件edit了。

<h3><font color=#0099FF>case2</font></h3> 

上述例程中，重复率比较高，可以试着使用变量，节省代码量。

```makefile
objects = main.o kbd.o command.o display.o /
              insert.o search.o files.o utils.o

    output : $(objects)
            gcc -o edit $(objects)
    main.o : main.c defs.h
            gcc -c main.c
    kbd.o : kbd.c defs.h command.h
            gcc -c kbd.c
    command.o : command.c defs.h command.h
            gcc -c command.c
    display.o : display.c defs.h buffer.h
            gcc -c display.c
    insert.o : insert.c defs.h buffer.h
            gcc -c insert.c
    search.o : search.c defs.h buffer.h
            gcc -c search.c
    files.o : files.c defs.h buffer.h command.h
            gcc -c files.c
    utils.o : utils.c defs.h
            gcc -c utils.c
    clean :
            rm output $(objects)
```

此例中，使用object变量代指一系列的o文件，此时修改其中的o文件，只用改定义的那一处，output行不用改。

<h3><font color=#0099FF>case3</font></h3> 

make命令可以自动推导，比如

```makefile
utils.o : utils.c defs.h
            gcc -c utils.c
```

完全可以写成`    utils.o : defs.h` ，因为make只要看到`utils.o`，就知道是`utils.c`生成的，而且还知道使用了` gcc -c utils.c` 命令，但是无法推导`utils.c`用了哪些头文件，所以要把用到的头文件写出来。于是，更为精简的Makefile如下：

```makefile
objects = main.o kbd.o command.o display.o /
              insert.o search.o files.o utils.o
gcc = cc # linux下cc和gcc是同一个东西
    output : $(objects)
            cc -o edit $(objects)
            
    main.o : defs.h
    kbd.o : defs.h command.h
    command.o : defs.h command.h
    display.o : defs.h buffer.h
    insert.o : defs.h buffer.h
    search.o : defs.h buffer.h
    files.o : defs.h buffer.h command.h
    utils.o : defs.h

　　clean :
            rm output $(objects)
```

上述功能只能适用于简单的程序，且所有文件都放在在同一目录下。更多Makefile知识可参考：[跟我一起写Makefile](http://wiki.ubuntu.org.cn/%E8%B7%9F%E6%88%91%E4%B8%80%E8%B5%B7%E5%86%99Makefile:MakeFile%E4%BB%8B%E7%BB%8D)

<h3><font color=#0099FF>高级功能</font></h3> 

看到再补充


