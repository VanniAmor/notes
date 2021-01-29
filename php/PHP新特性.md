[toc]



# PHP7新特性

【博客】https://blog.csdn.net/wuxing26jiayou/article/details/80036963

1.PHP7带来的好处 
2.PHP7带来的新东西 
3.PHP7带来的废弃 
4.PHP7带来的变更 
5.如何充分发挥PHP7的性能 
6.如何更好的写代码来迎接PHP7?
7.如何升级当前项目代码来兼容PHP7

## PHP7带来的好处

![image-20210123222045337](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210123222045337.png)

- 处理速度比之php5将近快了10倍

- qps更是增加了将近三倍

## PHP7新特性

https://www.jb51.net/article/73788.htm

#### 类型声明

```c
declare(strict_types=1);
function add(int $a, int $b): int {
    return $a+$b;
}
 
echo add(1, 2);
echo add(1.5, 2.6);
```

使用declare(strict_types = 1)开启严格模式，在严格模式之下，只能接受完全匹配的类型（包括参数类型，函数返回类型），否则会抛出 TypeError。唯一的例外是int值也可以传入声明为float的类型。

如果不开启，默认是弱类型校验，此时会强制转化不合适的类型为想要的标量类型。比如，参数想要String，传入的是int，则会获取string类型的变量，但是此时不会抛出TypeError



> **注意**:
> 文件开启严格类型后的*内部*调用函数将应用严格类型， 而不是在声明函数的文件内开启。 如果文件没有声明开启严格类型，而被调用的函数所在文件有严格类型声明， 那将遵循调用者的设置（开启类型强制转化）， 值也会强制转化。
>
> 只有为标量类型的声明开启严格类型。

#### set_exception_handler() 修改

set_exception_handler()，设置用户自定义的异常处理类。**用于没有用try/catch块捕获的异常。**

set_exception_handler()可以接受除Exception外的对象，例如大多数Error对象。

在php7中，很多致命错误以及可恢复的致命错误，都被转化为异常来处理了。这些异常继承自Error类。此类实现了Throwable接口。

PHP7进一步方便开发者处理, 让开发者对程序的掌控能力更强. 因为在默认情况下, Error会直接导致程序中断, 而PHP7则提供捕获并且处理的能力, 让程序继续执行下去, 为程序员提供更灵活的选择。

#### **AST: Abstract Syntax Tree, 抽象语法树**

AST在PHP编译过程作为一个中间件的角色, 替换原来直接从解释器吐出opcode的方式, **让解释器(parser)和编译器(compliler)解耦**, 可以减少一些Hack代码, 同时, 让实现更容易理解和可维护.

PHP5 : PHP代码 -> Parser语法解析 -> OPCODE -> 执行 
PHP7 : PHP代码 -> Parser语法解析 -> AST -> OPCODE -> 执行

#### **ZVAL结构优化**

![image-20210124144147791](C:%5CUsers%5C99380%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210124144147791.png)

左边是php5的zval（24字节），右边是php7的zva（16）字节

可以看出php7的zval要比上一代的还要复杂，但却从24字节下降到16字节，这是因为C语言中struct的每一个成员变量要各自占据一块独立的内存空间，而union里的成员变量是共用一块内存空间（php7中大量使用了union来代替了struct）。



#### **新的数组实现**

在php5中，数组是使用HashTable来实现，而php7中用了新的Zend Array类型，性能和访问速度都有大幅提升。

**一些非常常用，开销不大的的函数直接变成了引擎支持的opcode**

```php
call_user_function(_array) => ZEND_INIT_USER_CALL
is_int/string/array/* => ZEND_TYPE_CHECK
strlen => ZEND_STRLEN
defined => ZEND+DEFINED
```

#### **64位的INT支持**

​	支持存储大于2GB的字符串
​	支持上传大小大于2GB的文件
​	保证字符串在所有平台上【64位】都是64bit



#### 其他语法改动

##### 1. 新增操作符“<=>”

```php
$c = $a <=> $b

如果$a > $b, $c 的值为1

如果$a == $b, $c 的值为0

如果$a < $b, $c 的值为-1
```

##### 2. 新增操作符“??”

```php
//原写法
$username = isset($_GET['user]) ? $_GET['user] : 'nobody';
 
//现在
$username = $_GET['user'] ?? 'nobody';
```

##### 3. define定义常量数组

```php
define('ARR',['a','b']);
echo ARR[1];// a
```

##### 4. a的n次方

```php
// ** - 【a的b次方】
echo 2 ** 3;//8
```



## PHP7废弃

1. 废弃拓展
   - Ereg 正则表达式 
   - mssql 
   - mysql 
   - sybase_ct

2. 废弃特性

   - 不能使用类同名的构造函数 
   - 实例方法不能用静态方法的方式调用

3. 废弃部分函数

   ```php
   call_user_method() 
   call_user_method_array()
       
   //采用call_user_func和call_user_func_array()来代替
   ```

4. 废弃用法

   $HTTP_RAW_POST_DATA 变量被移除, 使用php://input来代

   ini文件里面不再支持#开头的注释, 使用”;”

   移除了ASP格式的支持和脚本语法的支持: <% 和 < script language=php >

## PHP7重大变更

1. 字符串处理机制修改

   ```php
   var_dump("0x123" == "291"); // false
   var_dump(is_numeric("0x123")); // false
   var_dump("0xe" + "0x1"); // 0
   var_dump(substr("f00", "0x1")) // foo
   ```

2. 整型处理机制修改

   int64支持，统一不同平台下的整形长度。字符串和文件上传都支持最大2G，64位php7字符串长度可以超过2^31次方字节。

   ```php
   // 无效的八进制数字(包含大于7的数字)会报编译错误
   $i = 0681; // 老版本php会把无效数字忽略。
    
   // 位移负的位置会产生异常
   var_dump(1 >> -1);
    
   // 左位移超出位数则返回0
   var_dump(1 << 64);// 0 
    
   // 右位移超出会返回0或者-1
   var_dump(100 >> 32);// 0 
   var_dump(-100 >> 32);// -1 
   ```

3. foreach修改

   foreach循环对数组内部指针不再起作用

   ![image-20210124153947588](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210124153947588.png)

4. list修改

   ```php
   //$arr将会是[1,2,3]而不是之前的[3,2,1]
   list($arr[], $arr[], $arr[]) = [1,2,3];
   
   // $x = null 并且 $y = null
   $str = 'xy';
   list($x, $y) = $str;
   
   // list也适用于数组对象
   list($a, $b) = (object)new ArrayObject([0, 1]);
   ```

5. 处理变量机制修改

   ![image-20210124154231668](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210124154231668.png)

## **如何充分发挥PHP7的性能**

### **1.开启Opcache**

zend_extension=opcache.so 
opcache.enable=1 
opcache.enable_cli=1

### 2.使用GCC 4.8以上进行编译

只有GCC 4.8以上PHP才会开启Global Register for opline and execute_data支持, 这个会带来5%左右的性能提升(Wordpres的QPS角度衡量)

### 3. 使用HugePage

https://www.laruence.com/2015/10/02/3069.html

![image-20210124154503561](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210124154503561.png)

关于Hugepage是啥，简单的说下就是默认的内存是以4KB分页的，而虚拟地址和内存地址是需要转换的， 而这个转换是要查表的，CPU为了加速这个查表过程都会内建TLB（Translation Lookaside Buffer）， 显而易见如果虚拟页越小，表里的条目数也就越多，而TLB大小是有限的，条目数越多TLB的Cache Miss也就会越高， 所以如果我们能启用大内存页就能间接降低这个TLB Cache Miss



### 4. PGO

第一次编译成功后，用项目代码去训练PHP，会产生一些profile信息，最后根据这些信息第二次gcc编译PHP就可以得到量身定做的PHP7

需要选择在你要优化的场景中: 访问量最大的, 耗时最多的, 资源消耗最重的一个页面.

参考: http://www.laruence.com/2015/06/19/3063.html 
参考: http://www.laruence.com/2015/12/04/3086.html

# PHP8新特性

https://www.cnblogs.com/zyilong/p/php80_newfeature_learning.html



**PHP8最为吸引人的就是JIT即时编译**

在了解PHP8新特性前，先要了解下什么是JIT（Just In Time）即时编译

## 什么是JIT

​	以java为例，通常javac将程序源代码编译，转换成java字节码，JVM通过解释字节码将其翻译成对应的机器指令，逐条读入，逐条解释翻译。很显然，经过解释执行，其执行速度必然会比可执行的二进制字节码程序慢。为了提高执行速度，引入了JIT技术。

​	在运行时JIT会把翻译过的机器码保存起来，已备下次使用，因此从理论上来说，采用该JIT技术可以接近以前纯编译技术。下面我看看，JIT的工作过程。

#### **JIT编译过程**

​	当JIT编译启用时（默认是启用的），JVM读入.class文件解释后，将其发给JIT编译器。**JIT编译器将字节码编译成本机机器代码**

​    我们了解了JIT的工作原理及过程，同样也发现了个问题，由于JIT对每条字节码都进行编译，造成了编译过程负担过重。为了避免这种情况，**当前的JIT只对经常执行的字节码进行编译**，如循环等。

   需要说明的是，JIT并不总是奏效，不能期望JIT一定能够加速你代码执行的速度，更糟糕的是她有可能降低代码的执行速度。这取决于你的代码结构，当然很多情况下我们还是能够如愿以偿的。



## PHP-JIT

https://www.laruence.com/2020/06/27/5963.html

![image-20210124220907073](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210124220907073.png)

左图是PHP8之前的Opcache流程示意图，右图是PHP8中的Opcache示意图，可以裁剪几个关键点：

- Opcache会做opcode尺寸的优化，某些图中的俩条opcode合并为一条
- PHP8的JIT目前是在Opcache之中提供的
- JIT在Opcache优化之后的基础上，结合Runtime的信息再次优化，直接生成机器码
- JIT不是原来的Opcache优化的替代，是增强
- 目前PHP8只支持x86架构的CPU



在JVM中，JIT编译的对象是.class字节码，然后JIT编译为机器码

在Zend中，JIT编译的对象是Opcode，然后通过JIT编译为机器码，当开启Opcache后，更是把PHP脚本编译为Opcache的时间也省略了。



实际上JIT共享了很多原来的Opcache做优化的基础数据结构，例如数据流图，调用图，SSA等等

### php-JIT配置

下载安装后，除掉初始的opcache配置以外，对于JIT我们需要添加如下配置到

php.ini

```php
opcache.jit = 1205
opcache.jit_buffer_size = 64M
```

opcache.jit这个配置看起来稍微有点复杂，我来解释下，这个配置由4个独立的数字组成，从左到右分别是（**请注意，这个是基于现有alpha1的版本设置，一些配置可能会随身着后续版本做微调**）：

- 是否在生成机器码点时候使用AVX指令，需要CPU支持：

  ```php
  0：不使用
  1：使用
  ```

- 寄存器分配策略：

  ```php
  0：不使用寄存器分配
  1：局部（块）域分配
  2：全局（函数）域分配
  ```

- JIT触发策略：

  ```php
  0：PHP脚本加载的时候就JIT
  1：当函数第一次被执行时JIT
  2：在一次运行后，JIT调用次数最多的百分之（opcache.prof_threshold * 100）的函数
  3：当函数/方法执行超过N（N和opcache.jit_hot_func相关）次以后JIT
  4：当函数方法的注释中包含@jit的时候对它进行JIT
  5：当一个跟踪执行超过N次（和opcache.jit_hot_loop，jit_hot_return等有关）以后JIT
  ```

- JIT优化策略，数值最优优化速度指标：

  ```php
  0：不准
  1：做opline之间的撤退部分的JIT
  2：内敛操作码处理程序调用
  3：基于类型初步做函数等级的JIT
  4：基于类型变量，过程调用图做函数等级JIT
  5：基于类型变量，过程调用图做脚本等级的JIT
  ```

## PHP注解

PHP8之前PHP实现注解可以通过php-parser来实现，但现在可以直接通过Reflection 来获取。

```php
/**
* @param Foo $argument
* @see https:/xxxxxxxx/xxxx/xxx.html
*/    
function dummy($Foo) {}
# 现在获取这段注解则可以使用
$ref = new ReflectionFunction("dummy");
var_dump($ref->getAttributes("See")[0]->getName());
var_dump($ref->getAttributes("See")[0]->getArguments());

```

## 类成员变量初始赋值

```php

// PHP8前的写法
class User{
    public $username;
    public $phone;
    public $sex;
    
    public function __contruct(
        $username,$phone,$sex
    ){
        $this->username = $username;
        $this->phone = $phone;
        $this->sex = $sex;
    }
}

// PHP8新特性
class User{
    public function __contruct(
        public string $username = "zhangsan",
        public string $phone = "110110";
        public string $sex = "男"
    ){}
}
```

## 命名参数

```php

function roule($name,$controller,$model){
    // ... code
}
// PHP8前的写法
roule("user/login","UserController","login");

// PHP8
roule(name:"user/login",controller:"UserController",model:"login");


function roule($name,$controller="UserController",$model){
    // ... code
}
// 可以需要输入参数名来区分传入的字段，那么在一些函数中，类比中间某项这段需要默认值，那我们就可以跳过这个字段
roule(name:"user/login",model:"login");
// 新旧写法相结合
roule("user/login",model:"login");

```

## 联合类型

在强制限定参数类型时，可以使用多种预测类型

```php
function create() : bool|string
function create(bool|string $userId)
```

