`GCC` 是 Linux 下的编译工具集，是 `GNU Compiler Collection` 的缩写，`包含 gcc、g++` 等编译器。这个工具集不仅包含编译器，还包含其他工具集，例如 ar、nm 等。

GCC 工具集不仅能编译 C/C++ 语言，其他例如 Objective-C、Pascal、Fortran、Java、Ada 等语言均能进行编译。GCC 在可以根据不同的硬件平台进行编译，即能进行交叉编译，在 A 平台上编译 B 平台的程序，支持常见的 X86、ARM、PowerPC、mips 等，以及 Linux、Windows 等软件平台。

## 1.安装GCC

> 有些纯净版的 Linux 默认没有 gcc 编译器，需要自己安装，在线安装步骤如下:

```shell
# ubuntu
$ sudo apt update   		# 更新本地的软件下载列表, 得到最新的下载地址
$ sudo apt install gcc g++	# 通过下载列表中提供的地址下载安装包, 并安装

# centos
$ sudo yum update   		# 更新本地的软件下载列表, 得到最新的下载地址
$ sudo yum install gcc g++	# 通过下载列表中提供的地址下载安装包, 并安装

```

gcc安装完成后，可以查看版本

```shell
# 查看 gcc 版本
$ gcc -v
$ gcc --version

# 查看 g++ 版本
$ g++ -v
$ g++ --version
```

## 2.GCC工作流程

> GCC 编译器对程序的编译下图所示，分为 4 个阶段：`预处理（预编译）`、`编译和优化`、`汇编`和`链接`。GCC 的编译器可以将这 4 个步骤合并成一个。 先介绍一个每个步骤都分别做了写什么事儿:

1. 预处理：在此阶段完成三件事，`展开头文件`，`宏替换`，`去掉注释行`
   - 这个阶段需要 GCC 调用处理器来完成，最终得到的还是源文件，文本格式
2. 编译：这个阶段需要 GCC 调用编译器对文件进行编译，最终得到一个汇编文件
3. 汇编：这个阶段需要 GCC 调用汇编器对文件进行汇编，最终得到一个二进制文件
4. 链接：这个阶段需要 GCC 调用链接器对程序需要调用的库进行链接， 最终得到一个可执行的二进制文件

| 文件名后缀 |             说明             | gcc参数 |
| :--------: | :--------------------------: | :-----: |
|     .c     |            源文件            |   无    |
|     .i     |        预处理后的文件        |   -E    |
|     .s     | 编译后得到的汇编语言的源文件 |   -S    |
|     .o     |    汇编后得到的二进制文件    |   -o    |

## 3.GCC常用参数

> 下面的表格中列出了常用的一些 gcc 参数，这些`参数在 gcc命令中没有位置要求`，只需要编译程序的时候将需要的参数指定出来即可。

|                gcc编译选项                |                           选项说明                           |
| :---------------------------------------: | :----------------------------------------------------------: |
|                    -E                     |                预处理指定的源文件，不进行编译                |
|                    -S                     |                 编译指定源文件，但不进行汇编                 |
|                   `-c`                    |             编译、汇编指定的源文件，但不进行链接             |
| `-o [file1] [file2] / [file2] -o [file1]` |                将文件 file2 编译成文件 file1                 |
|                   `-I`                    |    指定 include 包含的搜索目录（一般用于指定头文件位置）     |
|                   `-g`                    |       在编译的时候，生成调试信息，该程序可被调试器调试       |
|                    -D                     |                      编译时，指定一个宏                      |
|                    -w                     |                      不生成任何警告信息                      |
|                   -Wall                   |                       生成所有警告信息                       |
|                    -On                    | n 的取值范围：0~3。编译器的优化选项的 4 个级别，-O0 表示没有优化，-O1 为缺省值，-O3 优化级别最高 |
|                   `-l`                    |                    在编译时，使用指定的库                    |
|                   `-L`                    |                   指定编译时，搜索的库路径                   |
|                -fPIC/fpic                 |                     生成与位置无关的代码                     |
|                 `-shared`                 |            生成共享目标文件。通常用于建立共享库时            |
|                  `-std`                   |       指定 C 方言，如:-std=c99，gcc 默认的方言是 GNU C       |

### 	3.1 指定生成文件名（-o）

> 该参数用于指定源文件通过 gcc 处理后生成的新文件名字，有两种写法

```shell
# 参数 -o的用法 , 原材料 test.c 最终生成的文件名为 app
# test.c 写在 -o 之前
$ gcc test.c -o app

# test.c 写在 -o 之后
$ gcc -o app test.c
```

### 	3.2 搜索头文件（-l）

> 如果在程序中包含了一些头文件，但是包含的一些头文件在程序预处理的时候因为找不到无法被展开，导致程序编译失败，这时候我们可以在 gcc 命令中添加 `-I`参数重新制定要引用的头文件路径，以保证编译顺利完成

```shell
# -I, 指定头文件目录
$ tree
.
├── add.c
├── div.c
├── include
│   └── head.h
├── main.c
├── mult.c
└── sub.c

# 编译当前目录中的所有源文件，得到可执行程序
$ gcc *.c -o calc
main.c:2:18: fatal error: head.h: No such file or directory
compilation terminated.
sub.c:2:18: fatal error: head.h: No such file or directory
compilation terminated.
```

通过编译得到的错误信息可以知道，`源文件中包含的头文件无法被找到`。通过提供的目录结构可以得知`头文件 head.h 在 include 目录中`，因此可以在编译的时候重新指定头文件位置，具体操作如下

```shell
# 可以在编译的时候重新指定头文件位置 -I 头文件目录
$ gcc *.c -o calc -I ./include
```

### 	3.3 指定一个宏（-D）

> 在程序中我们可以使用宏定义一个常量，也可以通过宏控制某段代码是否能够被执行。在下面这段程序中第 8 行判断是否定义了一个叫做 DEBUG 的宏，如果没有定义第 9 行代码就不会被执行了，通过阅读代码能够知道这个宏是没有在程序中被定义的
>

```// test.c
#include <stdio.h>
#define NUMBER  3

int main()
{
    int a = 10;
#ifdef DEBUG
    printf("我是一个程序猿, 我不会爬树...\n");
#endif
    for(int i=0; i<NUMBER; ++i)
    {
        printf("hello, GCC!!!\n");
    }
    return 0;
}
```

如果不想在程序中定义这个宏，但又想让他存在，通过 gcc 的 `-D`就可以实现了，编译器会人为参数后指定的宏在程序中是存在的。

```shell
# 在编译命令中定义这个 DEBUG 宏, 
$ gcc test.c -o app -D DEBUG

# 执行生成的程序， 可以看到程序第9行的输出
$ ./app 
我是一个程序猿, 我不会爬树...
hello, GCC!!!
hello, GCC!!!
hello, GCC!!!
```

`-D 参数的应用场景`

在发布程序的时候，一般都会要求将程序中所有的 log 输出去掉，如果不去掉会影响程序的执行效率，很显然删除这些打印 log 的源代码是一件很麻烦的事情，解决方案是这样的：

将所有的打印 log 的代码都写到一个宏判定中，可以模仿上边的例子

- 在编译程序的时候指定 -D 就会有 log 输出
- 在编译程序的时候不指定 -D, log 就不会输出

## 4.多文件编译

GCC 可以自动编译链接多个文件，不管是目标文件还是源文件，都可以使用同一个命令编译到一个可执行文件中

### 	4.1 准备工作

- 头文件

  ```c
  #ifndef _STRING_H_
  #define _STRING_H_
  int strLength(char *string);
  #endif // _STRING_H_
  ```

- 源文件 string.c

  ```c
  #include "string.h"
  
  int strLength(char *string)
  {
  	int len = 0;
  	while(*string++ != '\0') 	// 当*string 的值为'\0'时, 停止计算
      {
          len++;
      }
  	return len; 	// 返回字符串长度
  }
  ```

- 源文件 main.c

  ```c
  #include <stdio.h>
  #include "string.h"
  
  int main(void)
  {
  	char *src = "Hello, I'am Monkey·D·Luffy!!!"; 
  	printf("string length is: %d\n", strLength(src)); 
  	return 0;
  }
  ```

### 	4.2 编译

> 因为头文件是包含在源文件中的，因此在使用 gcc 编译程序的时候不需要指定头文件的名字（`在头文件无法被找到的时候需要使用参数 -I 指定其具体路径而不是名字`）。我们可以通过一个 gcc 命令将多个源文件编译并生成可执行程序，也可以分多步完成这个操作。
>

- 直接链接生成可执行程序

  ```shell
  # 直接生成可执行程序 test
  $ gcc -o test string.c main.c
  
  # 运行可执行程序
  $ ./test
  ```

- 先将源文件变成目标文件，然后进行链接

  ```shell
  # 汇编生成二进制目标文件, 指定了 -c 参数之后, 源文件会自动生成 string.o 和 main.o
  $ gcc –c string.c main.c
  
  # 链接目标文件, 生成可执行程序 test
  $ gcc –o test string.o main.o
  
  # 运行可执行程序
  $ ./test
  
  ```

  

