所谓AOP即 **Aspect Oriented Programing**，即面向切面编程

AOP是一种编程思想，是对OOP的补充



![image-20201205135823098](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201205135823098.png)





AOP的实现有两种方式：

- AspectJ
- Spring AOP



## AspectJ



AspectJ是语言级的实现，是JAVA语言的拓展，定义了AOP语法

AspectJ在编译时织入代码，它有一个专门的编译器，用来生成遵守Java字节码规范的class文件



## Spring AOP

Spring Aop使用纯Java实现，它不需要专门的编译过程，也不需要特殊的类加载器

Spring AOP在运行时通过代理的方式织入代码，只支持方法类型的连接点

Spring支持对AspectJ的集成



### JDK动态代理

是Java提供的动态代理技术，可以在运行时创建接口的代理实例

Spring AOP默认采用此种方式，在接口的代理实例中织入代码

### CGLib动态代理

采用底层的字节码技术，在运行时创建子类代理实例

当目标对象不存在接口时，SpringAOP会采用此种方式，在子类织入代码



Spring AOP中有几个织入代码的位置，分别为：

- before
- after
- around
- afterReturning
- afterThrowing



其优先级为



aroundBefore

before

aroundAfter

after

afterReturning



