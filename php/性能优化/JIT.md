JIT可以说是php8最为亮眼的新特性了

https://www.laruence.com/2020/06/27/5963.html

https://blog.p2hp.com/archives/7577

**对于CPU密集型计算，会有明显的性能提升**

## JIT原理

以java为例，通常javac将程序源代码编译，转换成java字节码，JVM通过解释字节码将其翻译成对应的机器指令，逐条读入，逐条解释翻译。很显然，经过解释执行，其执行速度必然会比可执行的二进制字节码程序慢。为了提高执行速度，引入了JIT技术。

​	在运行时JIT会把翻译过的机器码保存起来，已备下次使用，因此从理论上来说，采用该JIT技术可以接近以前纯编译技术。下面我看看，JIT的工作过程。

#### **JIT编译过程**

​	当JIT编译启用时（默认是启用的），JVM读入.class文件解释后，将其发给JIT编译器。**JIT编译器将字节码编译成本机机器代码**

​    我们了解了JIT的工作原理及过程，同样也发现了个问题，由于JIT对每条字节码都进行编译，造成了编译过程负担过重。为了避免这种情况，**当前的JIT只对经常执行的字节码进行编译**，如循环等。

   需要说明的是，JIT并不总是奏效，不能期望JIT一定能够加速你代码执行的速度，更糟糕的是她有可能降低代码的执行速度。这取决于你的代码结构，当然很多情况下我们还是能够如愿以偿的。

## PHP-JIT

简而言之： **当 JIT 按预期工作时，您的代码不会通过 Zend VM 执行，而是作为一组 CPU 级指令直接执行。**

解释器原来的执行流程如下：

1. 读取 PHP 代码并将其解释为一组称为 Tokens 的关键字。这个过程让解释器知道各个程序都写了哪些代码。 这一步称为 Lexing 或 Tokenizing 。
2. 拿到 Tokens 集合以后，PHP 解释器将尝试解析他们。通过称之为 Parsing 的过程生成抽象语法树（AST）。这里 AST 是一个节点集表示要执行哪些操作。比如，「 echo 1 + 1 」实际含义是 「打印 1 + 1 的结果」 或者更详细的说 「打印一个操作，这个操作是 1 + 1」。
3. 有了 AST ，可以更轻松地理解操作和优先级。将抽象语法树转换成可以被 CPU 执行的操作需要一个用于过渡的表达式 (IR)，在 PHP 中我们称之为 Opcodes 。将 AST 转换为 Opcodes 的过程称为 compilation 。
4. 有了 Opcodes ，有趣的部分就来了： executing 代码！ PHP 有一个称为 Zend VM 的引擎，该引擎能够接收一系列 Opcodes 并执行它们。执行所有 Opcodes 后， Zend VM 就会将该程序终止。

![image-20210128152811397](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210128152811397.png)

**当PHP开启opCache后，可以直接从AST中提取opCache，跳过Lexing/Tokenizing 和 Parsing 步骤。**

**而JIT则是可以直接跳过Zend VM，代码直接被CPU执行**

php的JIT使用了名为 DynASM的库，该库将一种特定格式的CPU指令映射为许多不同的CPU类型汇编代码。因此，编译器只需要使用DynASM就可以将opCodes转换为特定结构的机器码



## 编译顺序的先后问题

有一个哲学的问题

**既然预加载能够在执行之前将PHP代码解析为opCodes，并且DynASM可以将opCode编译为机器码（Just In Time编译），那么为什么不干脆运行前编译（Ahead of Time 编译）PHP代码呢**



这是因为PHP是弱类型语言，这意味着Zend VM在尝试执行某个操作码之前，PHP通常是不知道变量类型的。

通过查看 zend_value联合类型得知，很多指针指向不同类型的变量。每当Zend VM尝试从zend_value获取值时，它会使用像ZSTR_VAL这样的宏，获取联合类型中字符串的指针。

因此， **使用机器码执行类型推断逻辑是不可行的，并且可能变得更慢。**

但是，先求值在编译也不是一个好的选择，因为编译为机器码是CPU密集型任务，即在运行时编译所有内容，会严重占据CPU资源。

**换句话而言，就是JIT的运算成本很高**



为了寻求平衡， **PHP 的 JIT 尝试只编译有价值的 Opcodes** 。为此， JIT 会分析 Zend VM 要执行的 Opcodes 并检查可能编译的地方。（根据配置文件）

当某个 Opcode 编译后，它将把执行交给该编译后的代码，而不是交给 Zend VM 。看起来如下：

![image-20210128171535637](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210128171535637.png)

在 Opcache 扩展中，有两条检测指令判断要不要编译 Opcode

如果要，编译器将使用DynASM将此opcode编译成机器码，并执行机器码

## phpJIT配置

https://blog.csdn.net/u010905752/article/details/109499013

https://www.cnblogs.com/wenhainan/p/14050955.html

如上介绍，JIT的开启前置条件为opCache的开启， 确保在php.ini中 `opcache.enable = 1`



JIT有两个重要配置

```php
opcache.jit=1205
opcache.jit_buffer_size=64M
```

**请记住，这`opcache.jit`是可选的。如果忽略该属性，则JIT将使用默认值tracing(1205)。**

opcache.jit这个配置有点复杂。它接受`disable`，`on`，`off`，`trace`，`function`，和按顺序排列的 4 个不同标志的 4 位值。

- **`disable`**：在启动时完全禁用JIT功能，并且在运行时无法启用。
- **`off`**：禁用，但是可以在运行时启用JIT。
- **`on`**：启用`tracing`模式。
- **`tracing`**：细化配置 的别名`1254`。
- **`function`**：细化配置 的别名`1205`。

### 粒度配置

- 是否在生成机器码点时候使用AVX指令，需要CPU支持：

  ```txt
  0: 不使用
  1: 使用
  ```

- 寄存器分配策略

  ```txt
  0: 不使用寄存器分配
  1: 局部(block)域分配
  2: 全局(function)域分配
  ```

- JIT触发策略

  ```txt
  0: PHP脚本载入的时候就JIT
  1: 当函数第一次被执行时JIT
  2: 在一次运行后，JIT调用次数最多的百分之(opcache.prof_threshold * 100)的函数
  3: 当函数/方法执行超过N(N和opcache.jit_hot_func相关)次以后JIT
  4: 当函数方法的注释中含有@jit的时候对它进行JIT
  5: 当一个Trace执行超过N次（和opcache.jit_hot_loop, jit_hot_return等有关)以后JIT
  ```

- JIT优化策略，数值越大优化力度越大

  ```txt
  0: 不JIT
  1: 做opline之间的跳转部分的JIT
  2: 内敛opcode handler调用
  3: 基于类型推断做函数级别的JIT
  4: 基于类型推断，过程调用图做函数级别JIT
  5: 基于类型推断，过程调用图做脚本级别的JIT
  ```

## 调试JIT（`opcache.jit_debug`）

在继续之前，让我们确保JIT确实有效，创建一个可通过浏览器或CLI访问的PHP脚本（取决于您测试JIT的位置），并查看以下输出`opcache_get_status()`：

```
var_dump(opcache_get_status()['jit']);
```

输出应该是这样的：

```
array:7 [
  "enabled" => true
  "on" => true
  "kind" => 5
  "opt_level" => 4
  "opt_flags" => 6
  "buffer_size" => 4080
  "buffer_free" => 0
]
```

如果`enabled`和`on`是正确的，那您就对了！

---

PHP JIT提供了一种通过设置INI配置来发出JIT调试信息的方法。设置后，它将输出汇编代码以供进一步检查。

```ini
opcache.jit_debug=1
```

该`opcache.jit_debug`指令接受位掩码值以切换某些功能。当前，它接受的值范围是1（`0b1`）至20位二进制。值`1048576`表示调试输出的最大级别。

```bash
php -d opcache.jit_debug=1 test.php
TRACE-1$/test.php$7: ; (unknown)
    sub $0x10, %rsp
    mov $EG(jit_trace_num), %rax
    mov $0x1, (%rax)
.L1:
    cmp $0x4, 0x58(%r14)
    jnz jit$$trace_exit_0
    cmp $0x4e20, 0x50(%r14)
    ...
```

## 向后兼容性影响

没有。JIT是PHP 8.0中新增的一项功能，不会引起任何问题。当遇到未知的INI指令时，PHP不会引发任何警告或错误，这意味着设置JIT INI指令不会引起任何问题。