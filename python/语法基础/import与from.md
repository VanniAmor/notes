Python通过import与from导入其他模块，分别有以下方式

- import xxx
- from xxx import yyy
- from xxx.zzz import yyy
- from xxx import *



**强烈禁止使用最后一个 from xxx import * **， 这样导入会容易出现变量或函数命名的覆盖

**所谓模块，其实就是一个py文件**

## 标准导入模块



在py中，所有加载到内存中的模块都存在于sys.modules

当 import 一个模块时首先会在这个列表中查找是否已经加载了这个模块

如果加载了则只是将模块的名字加入到正在调用 import 的模块的 Local 名字空间中。

如果没有加载则从 sys.path 目录中按照模块名称查找模块文件，模块可以是py、pyc、pyd，找到后将模块载入内存，并加到 sys.modules 中，并将名称导入到当前的 Local名字空间

**一个模块不会重复载入。多个不同的模块都可以用 import 引入同一个模块到自己的 Local 名字空间，其实背后的 PyModuleObject 对象只有一个。**

注意

- import导入为绝对导入
- import 只能导入模块，不能导入模块中的对象（类、函数、变量等)



## 嵌套导入

### 顺序嵌套

例如：本模块导入 A 模块（import A），A 中又 import B，B 模块又可以 import 其他模块……
注意：各个模块的 Local 名字空间是独立的；

对于上面的例子，本模块 import A 之后本模块只能访问模块 A，不能访问模块 B 及其他模块。**虽然模块 B 已经加载到内存了，如果访问还要再明确的在本模块中 import B。**



### 循环嵌套

例如：文件[ A.py ]
          from B import D
          class C:pass


​          文件[ B.py ]
​          from A import C
​          class D:pass

 为什么执行 A 的时候不能加载 D 呢？
 如果将 A.py 改为：import B 就可以了。



RobertChen：这跟Python内部 import 的机制是有关的，具体到 from B import D，Python 内部会分成几个步骤：
    （1）在 sys.modules 中查找符号 “B”
    （2）如果符号 B 存在，则获得符号 B 对应的 module 对象。
        从 <modult B> 的 __dict__ 中获得符号 “D” 对应的对象，如果 “D” 不存在，则抛出异常。
    （3）如果符号 B 不存在，则创建一个新的 module 对象 <module B>，注意，此时，module 对象的 __dict__ 为空。
        执行 B.py 中的表达式，填充 <module B> 的 __dict__。
        从 <module B> 的 __dict__ 中获得 “D” 对应的对象，如果 “D” 不存在，则抛出异常。



 所以这个例子的执行顺序如下：

- 执行 A.py 中的 from B import D 由于是执行的 python A.py，所以在 sys.modules 中并没有 <module B> 存在， 首先为 B.py 创建一个 module 对象 (<module B>) ， 注意，这时创建的这个 module 对象是空的，里边啥也没有， 在 Python 内部创建了这个 module 对象之后，就会解析执行 B.py，其目的是填充 <module B> 这个 __dict__。
- 执行 B.py中的from A import C， 在执行B.py的过程中，会碰到这一句，首先检查sys.modules这个module缓存中是否已经存在<module A>了， 由于这时缓存还没有缓存<module A>，所以类似的，Python内部会为A.py创建一个module对象(<module A>)， 然后，同样地，执行A.py中的语句
- 再次执行A.py中的from B import D 这时，由于在第1步时，创建的<module B>对象已经缓存在了sys.modules中，所以直接就得到了<module B>，但是，注意，从整个过程来看，我们知道，这时<module B>还是一个空的对象，里面啥也没有， 所以从这个module中获得符号"D"的操作就会抛出异常。如果这里只是import B，由于"B"这个符号在sys.modules中已经存在，所以是不会抛出异常的。

![image-20210331102909537](C:%5CUsers%5Cwb.luoweile%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210331102909537.png)



## 包import

`只要一个文件夹下面有个 __init__.py 文件，那么这个文件夹就可以看做是一个包`

```txt
包导入的过程和模块的基本一致，只是导入包的时候会执行此包目录下的 __init__. py 而不是模块里面的语句了, 另外，如果只是单纯的导入包，而包的__init__.py 中又没有明确的其他初始化操作，那么此包下面的模块是不会自动导入的。
```

假设

     有下面的包结构：
            PA
            |---- __init__.py
            |---- wave.py
            |---- PB1
               |---- __init__.py
               |---- pb1_m.py
            |---- PB2
               |---- __init__.py
               |---- pb2_m.py 
            有如下程序：
            import sys
            import PA.wave                 #1
            import PA.PB1                  #2
            import PA.PB1.pb1_m as m1      #3
            import PA.PB2.pb2_m            #4
            
    PA.wave.getName()                      #5
    m1.getName()                           #6
    PA.PB.pb2_m.getName() 				   #7
1. 当执行 #1 后，sys.modules 会同时存在 PA、PA.wave 两个模块，此时可以调用 PA.wave 的任何类或函数了。但不能调用 PA.PB1(2) 下的任何模块。当前 Local 中有了 PA 名字。
2. 当执行 #2 后，只是将 PA.PB1 载入内存，sys.modules 中会有 PA、 PA.wave、PA.PB1 三个模块，但是 PA.PB1 下的任何模块都没有自动载入内存，此时如果直接执行 PA.PB1.pb1_m.getName() 则会出错，因为 PA.PB1 中并没有 pb1_m 。当前 Local 中还是只有 PA 名字，并没有 PA.PB1 名 字。
3. 当执行 #3 后，会将 PA.PB1 下的 pb1_m 载入内存，sys.modules 中会有 PA、PA.wave、PA.PB1、PA.PB1.pb1_m 四个模块，此时可以执行 PA.PB1.pb1_m.getName() 了。由于使用了 as，当前 Local中除了 PA 名字，另外添加了 m1 作为 PA.PB1.pb1_m 的别名。
4. 当执行 #4 后，会将 PA.PB2、PA.PB2.pb2_m 载入内存，sys.modules 中会有 PA、PA.wave、PA.PB1、PA.PB1.pb1_m、PA.PB2、PA.PB2.pb2_m 六个模块。当前 Local 中还是只有 PA、m1。
   下面的 #5，#6，#7 都是可以正确运行的。
5. 

**注意的是：如果 PA.PB2.pb2_m 想导入 PA.PB1.pb1_m、PA.wave 是可以直接成功的。最好是采用明确的导入路径，对于 ./.. 相对导入路径还是不推荐用。**



## 参考文献

https://blog.csdn.net/cainiao_python/article/details/106964674

