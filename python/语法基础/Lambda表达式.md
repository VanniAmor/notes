Python使用lambda关键字创造匿名函数。所谓匿名，意即不再使用def语句这样标准的形式定义一个函数。这种语句的**目的是由于性能的原因，在调用时绕过函数的栈分配**。其语法是：

`lambda [arg1[, arg2, ... argN]]: expression`



```python
# 使用def定义函数的方法
def true():
  return True

#等价的lambda表达式
>>> lambda :True
<function <lambda> at 0x0000000001E42518>

# 保留lambda对象到变量中，以便随时调用
>>> true = lambda :True
>>> true()
True
```



要注意的是，**lambda表达式是可以嵌套的**

```python
>>> action = (lambda x : (lambda y : x + y))
>>> a = action(10)
>>> a(5)
15
```



## 进阶使用



### filter函数

```python
>>> list = [1, 2, 3]
>>> result = filter(lambda x: x%2==0, list)
>>> result
[2]
>>> result = [x for x in list if x%2==0]
>>> result
[2]
```

### map函数

```python
list = [1, 2, 3]
>>> result = map(lambda x: x*2, list)
>>> result
[2, 4, 6]
>>> result = [x*2 for x in list]
>>> result
[2, 4, 6]
```

### reduce函数

```python
list = [1, 2, 3]
>>> result = reduce(lambda x, y: x+y,list)
>>> result
6
>>> result = sum(list)
>>> result
6
```

### 跳表

```python
>>> key = "get"
>>> {"abc":(lambda : 2 + 2),"bcd" : (lambda : 3 + 3), "get" : (lambda : 4 + 4)}[key]()
8
```

这样在字典中，每个lambda都留下了一个后续可以调用的函数，通过索引可以取出来，并调用。这就使字段可以成为更加通用的多路分支工具。



## 参考文献

https://www.jb51.net/article/159980.htm