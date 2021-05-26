- shell中的五大运算
- if语法



------



当我们在写程序的时候，时常对上一步执行是否成功如何判断苦恼，当我们今天学习了if就可以解决你的苦恼。if语句在我们程序中就是用来做判断的，以后大家不管学习什么语言，以后只要涉及到判断的部分，大家就可以直接拿if来使用，不同的语言之间的if只是语法不同，原理是相同的。

## 一、shell中的运算

### 1.1）数学比较运算

```
 运算符解释
 -eq         等于
 -gt         大于
 -lt         小于
 -ge         大于或等于
 -le         小于或等于
 -ne         不等于
```

### 1.2）字符串比较运算

```
运算符解释，注意字符串一定别忘了使用引号引起来
==          等于   
!=          不等于
-n          检查字符串的长度是否大于0  
-z          检查字符串的长度是否为0
```

### 1.3）文件比较与检查

```
-d  检查文件是否存在且为目录
-e  检查文件是否存在
-f  检查文件是否存在且为文件
-r  检查文件是否存在且可读
-s  检查文件是否存在且不为空
-w  检查文件是否存在且可写
-x  检查文件是否存在且可执行
-O  检查文件是否存在并且被当前用户拥有
-G  检查文件是否存在并且默认组为当前用户组
file1 -nt file2  检查file1是否比file2新	（nt即 newer than）
file1 -ot file2  检查file1是否比file2旧	（ot即 older than）
```

### 1.4）逻辑运算

```
逻辑与运算       &&   
逻辑或运算       ||  
逻辑非运算      ！
逻辑运算注意事项：
    逻辑与 或 运算都需要两个或以上条件，逻辑非运算只能一个条件。
    口诀:    逻辑与运算               真真为真 真假为假   假假为假
             逻辑或运算               真真为真 真假为真   假假为假
             逻辑非运算               非假为真   非真为假
```

### 1.5）赋值运算

```
         =      赋值运算符         a=10   name='baism'
```

## 二、if 语法

### 2.1）语法一： 单if语句

适用范围:只需要一步判断，条件返回真干什么或者条件返回假干什么。

语句格式

```
if [ condition ]           #condition 值为true or false   then      commandsfi
```

该语句翻译成汉语大意如下：

```
假如  条件为真 那么    执行commands代码块结束
```

通过一段代码来演示一下吧，判断 当前用户是不是root，如果不是那么返回”ERROR: need to be root so that!“

实验代码

![img](https://book.apeland.cn/media/images/2019/05/09/if-1.png)

执行以下看看吧

![img](https://book.apeland.cn/media/images/2019/05/09/if-1-result.png)

### 2.2）语法二： if-then-else语句

适用范围:两步判断，条件为真干什么，条件为假干什么。

```
if [ condition ]     then          commands1else          commands2fi
```

该语句翻译成汉语大意如下：

```
假如条件为真  那么        执行commands1代码块否则        执行commands2代码块结束
```

通过一段代码演示一下吧，判断当前登录用户是管理员还是普通用户,如果是管理员输出”hey admin“ 如果是普通用户输出”hey guest“

实验代码

![img](https://book.apeland.cn/media/images/2019/05/09/if-2.png)

执行结果

![img](https://book.apeland.cn/media/images/2019/05/09/if-2-result.png)

### 2.3）语法三: if-then-elif语句

适用范围:多于两个以上的判断结果，也就是多于一个以上的判断条件。

```
if [ condition ]
     then
          commands1
else
          commands2
fi
```

通过一段代码演示一下吧，通过一个脚本，判断两个整数的关系。

实验代码

![img](https://book.apeland.cn/media/images/2019/05/09/if-3.png)

执行结果

![img](https://book.apeland.cn/media/images/2019/05/09/if-3-result.png)

## 三、if 高级应用

1、条件符号使用双圆括号，可以在条件中植入数学表达式

通过代码来看下吧

![img](https://book.apeland.cn/media/images/2019/05/09/for-4.png)

注意 双小圆括号中的比较运算符 使用的是我们传统的比较运算符 >>= == <<= !=

2、使用双方括号,可以在条件中使用通配符

通过代码看下 ，为字符串提供高级功能，模式匹配 r* 匹配r开头的字符串

![img](https://book.apeland.cn/media/images/2019/05/09/if-5.png)

执行结果

![img](https://book.apeland.cn/media/images/2019/05/09/if-5-result.png)