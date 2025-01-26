## Lua内存泄漏

一般来说，lua自带GC，理论上是不会有内存泄漏的。当它进行GC时，会从根部开始扫描所有的对象，如果某个地方对这个对象还有引用，就不会把这个对象内存 collect，这个对象就没有被GC。所以lua中的内存泄漏是指：已经没有被使用，但外部仍还有引用存在的对象

```lua
--函数中应该被申明为local的对象忘记加local  
local function test()   
    testTable = {} --这个testTabel会被存放在 全局表_G 中，GC时由于此对象还有引用存在，所以这里总是会有一个table泄露。   
    local mt = {} --mt加了local修饰，函数调用完后，引用也不复存在了，GC时会被回收。   
    setmetatable(testTable, mt)   
end 
```

以下是一些常见的错误引用：

- 本应该local 的变量进入global空间或者module空间了(忘记写local)
- c/c++部分调用的lua_ref是否有正常lua_unref释放

- 其他各种各样的实际bug造成的泄漏。



可以建立一个weak table, 把你所有创建过的能够称之为资源的，包含但不限于“战斗对象，玩家，npc，物品，场景，邮件”等等对象全部扔到这个table里面。当你知道玩家
已经下线、战斗已经销毁了，但通过连续的强制full gc以后weak table里面还有这个变量，这就证明了这个变量的引用没有被完全释放，

知道有泄漏是比较容易的，能够完全揪出来就不是很容易了。是的，它究竟在哪儿呢? 一开始在此项目里面也是先发现比如某npc泄漏了，然后就去查代码，看看究竟哪个地方写得不对。这种方式效率极低，基本上查不到什么问题。在迟一点的时候才使用现在的方案：从_G深度遍历所有的table、metatable、funciton's upvalue、function's env、registentry(lua_ref)。 目前所知的所有引用必定存在于这几个空间， 遍历完成以后一定可以找到那个“迷失了的引用”。 这种方式在脚本层就可以完成所有事情，甚至你可以在运营环境中在线查证，其遍历的速度是非常快的，但内存开销非常大(:，可以考虑一边遍历一边gc，当然还要记得避免重复搜索。 在应用此方案以后，此项目解决了脚本中所有的泄漏问题。



## Lua弱引用

```lua
t = {};  
    
-- 使用一个table作为t的key值  
key1 = {name = "key1"};  
t[key1] = 1;  
key1 = nil;  

-- 又使用一个table作为t的key值  
key2 = {name = "key2"};  
t[key2] = 1;  
key2 = nil;  

-- 强制进行一次垃圾收集  
collectgarbage();  

for key, value in pairs(t) do  
    print(key.name .. ":" .. value);  
end  

--[[
输出如下
    [LUA-print] key1:1  
    [LUA-print] key2:1  
]]

这很符合常理，也在我们的预计当中，虽然我们在给t赋值之后，key1和key2都赋值为nil了。

但是，已经添加到table中的key值是不会因此而被当做垃圾的。
所以我们最后还是能输出key1和key2的name字段。
```



```lua
刚刚举例的只是正常情况，那么，如果我们把某个table作为另一个table的key值后，希望当table设为nil值时，另一个table的那一条字段也被删除。

t = {};  
   
-- 给t设置一个元表，增加__mode元方法，赋值为“k”  
setmetatable(t, {__mode = "k"})

-- 使用一个table作为t的key值  
key1 = {name = "key1"};  
t[key1] = 1;  
key1 = nil;  

-- 又使用一个table作为t的key值  
key2 = {name = "key2"};  
t[key2] = 1;  
key2 = nil;  

-- 强制进行一次垃圾收集  
collectgarbage();  

for key, value in pairs(t) do  
    print(key.name .. ":" .. value);  
end  


留意，在t被创建后，立刻给它设置了元表，元表里有一个__mode字段，赋值为”k”字符串。

如果这个时候大家运行代码，会发现什么都没有输出，因为，t的所有字段都不存在了。

这就是弱引用table的其中一种，给table添加__mode元方法，如果这个元方法的值包含了字符串”k”，就代表这个table的key都是弱引用的。

一旦其他地方对于key值的引用取消了（设置为nil），那么，这个table里的这个字段也会被删除。
 
通俗地说，因为t的key被设置为弱引用，所以，执行t[key1] = 1后，t中确实存在这个字段。

随后，又执行了key1 = nil，此时，除了t本身以外，就没有任何地方对key1保持引用，所以t的key1字段也会被删除。
```



lua有一个元方法 ==__mode== ，可以将Lua表设置为弱引用table，有三种形式

1）key值弱引用，也就是刚刚说到的情况，只要其他地方没有对key值引用，那么，table自身的这个字段也会被删除。设置方法：setmetatable(t, {__mode = “k”});
2）value值弱引用，情况类似，只要其他地方没有对value值引用，那么，table的这个value所在的字段也会被删除。设置方法：setmetatable(t, {__mode = “v”});
3）key和value弱引用，规则一样，但是key和value都同时生效，任意一个起作用时都会导致table的字段被删除。设置方法：setmetatable(t, {__mode = “kv”});

当然，这里所说的被删除，是指在Lua执行垃圾回收的时候，并不一定是立刻生效的。

demo如下

```lua
local findedObjMap = nil     
function _G.findObject(obj, findDest)    
    if findDest == nil then    
        return false    
    end    
    if findedObjMap[findDest] ~= nil then    
        return false    
    end    
    findedObjMap[findDest] = true    
    
    local destType = type(findDest)    
    if destType == "table" then    
        if findDest == _G.CMemoryDebug then    
            return false    
        end    
        for key, value in pairs(findDest) do    
            if key == obj or value == obj then    
                _info("Finded Object")    
                return true    
            end    
            if findObject(obj, key) == true then    
                _info("table key")    
                return true    
            end    
            if findObject(obj, value) == true then    
                _info("key:["..tostring(key).."]")    
                return true    
            end    
        end    
    elseif destType == "function" then    
        local uvIndex = 1    
        while true do    
            local name, value = debug.getupvalue(findDest, uvIndex)    
            if name == nil then    
                break    
            end    
            if findObject(obj, value) == true then    
                _info("upvalue name:["..tostring(name).."]")    
                return true    
            end    
            uvIndex = uvIndex + 1    
        end    
    end    
    return false    
end    
    
function _G.findObjectInGlobal(obj)    
    findedObjMap = {}    
    setmetatable(findedObjMap, {__mode = "k"})    
    _G.findObject(obj, _G)    
end    

```

思路:

1. 资源跟踪,定位哪些资源泄漏
2. 引用检索,查找泄漏的资源被哪个模块引用

##### 资源跟踪



定义:将应用中分配的lua对象添加到一个弱表中.执行完整的gc后,还能从弱表中索引到的对象表示它还在别的地方被引用着,可能是正常的引用,也可能是一处内存泄漏.我使用了一个弱键表,该表以要跟踪的lua对象为键,该对象的描述信息为值.其中的描述信息包含了对象描述和对象创建时间两项.对象描述用于区别不同的跟踪对象;创建时间则用来在打印弱表的时候判断对象的存活时间是否合理.我定义的接口是:function TraceMem(obj, description);

##### 引用检索

定义:从某个节点开始搜索所有该节点引用的对象以及递归搜索子节点,找到要搜索的对象,打印出引用路径.最常见的可以从_G开始搜索.搜索到的每个table,取其key和value递归搜索;搜索到的每个函数,取其upvalue递归搜索.至于是否需要搜索对象的环境表和metatable,以及全局registry表,则取决于具体需求.我因为用不上,就没有搜索这一部分.搜索的时候注意标记已经搜索过的节点,避免重复搜索.最好能缩小搜索范围,而不是从_G开始搜索,另外应该能每次只搜索指定的部分引用而非全部,可以极大的缩短等待时间.搜索所有的引用其实相当耗时.我定义的搜索接口是:function Search_r(obj, node, mark, result);



## Lua 垃圾回收算法

Lua的[GC算法](https://blog.csdn.net/qq_43801020/article/details/108073978)使用的所谓“Mark And Sweep”算法。简单的理解，这个算法将GC分为两个阶段，**一个是标记（mark）阶段，这一阶段将所有[系统](http://www.2cto.com/os/)中引用的对象都逐一标记**；**而在清理（sweep）阶段，将把在mark阶段中没有被标记的数据删除。**

在Lua中，使用几种颜色来区分不同的结点：

white：白色表示没有进行过标记的节点

gray：灰色表示已经进行过标记的节点，但是与它相关联的节点还没有进行过标记。

black：本节点和与之关联的节点都已经被扫描标记过了。通常会出现有关联数据的，包括有Table，upvalue等数据类型。



### 垃圾收集器函数

collectgarbage函数提供了多项功能：**停止垃圾回收**，**重启垃圾回收**，**强制执行一次回收循环**，**强制执行一步垃圾回收**，**获取Lua占用的内存**，以及**两个影响垃圾回收频率和步幅的参数**。collectgarbage(opt,[,arg])

| "stop"       | 停止垃圾收集器，如果它的运行。                               |
| ------------ | ------------------------------------------------------------ |
| "restart"    | 如果垃圾收集器已经停止，将重新启动它。                       |
| "collect"    | 执行一次全垃圾收集循环。默认执行此操作                       |
| "count"      | 返回当前Lua中使用的内存量(以KB为单位)                        |
| "step"       | 单步执行一个垃圾收集. 步长 "Size" 由参数arg指定　(大型的值需要多步才能完成)，如果要准确指定步长，需要多次实验以达最优效果。如果步长完成一次收集循环，将返回True |
| "setpause"   | 设置 arg/100 的值作为暂定收集的时长;并返回设置前的值。默认为200控制了收集器在开始一个新的收集周期之前要等待多久。 随着数字的增大就导致收集器工作工作的不那么主动。 小于 1 的值意味着收集器在新的周期开始时不再等待。 当值为 2 的时候意味着在总使用内存数量达到原来的两倍时再开启新的周期。 |
| "setstepmul" | 设置 arg/100 的值，作为步长的增幅(即新步长=旧步长*arg/100);并返回设置前的值。默认为200控制了收集器的工作速度，这个速度是一个相对于内存分配的速度。更大的数字将导致收集器工作的更主动的同时，也使每步收集的尺寸增加。 小于 1 的值会使收集器工作的非常慢，可能导致收集器永远都结束不了当前周期。 缺省值为200%，这意味着收集器将以内存分配器的两倍速运行。 |



```lua
function test1()  
    collectgarbage("collect")--为了有干净的环境，先把可以收集的垃圾收集了  
    collectgarbage()--为了保证内存的收集的相对干净，及内存的稳定，要执行多次收集  
    print("now,Lua内存为:",collectgarbage("count")) -->205.7158203125 KB  
    local colen = {} --现在是局部变量  
    for i=1,5000 do  
        table.insert(colen,{})  
    end  
    print("now,Lua内存为:",collectgarbage("count"))-->860.4111328125 KB  
    --创建5000个table，内存增加了655 KB  
end  
  
function collect1()  
    print("now,Lua内存为:",collectgarbage("count"))-->608.060546875 KB  
    collectgarbage()  
    collectgarbage()  
    print("now,Lua内存为:",collectgarbage("count"))-->204.8408203125 KB  
    --最后与一开始只差只有1KB  
end  
  
function test2()  
    collectgarbage("collect")--为了有干净的环境，先把可以收集的垃圾收集了  
    collectgarbage()--为了保证内存的收集的相对干净，及内存的稳定，要执行多次收集  
    print("now,Lua内存为:",collectgarbage("count")) -->205.7158203125 KB  
    colen = {} --现在是全局变量  
    for i=1,5000 do  
        table.insert(colen,{})  
    end  
    print("now,Lua内存为:",collectgarbage("count"))-->619.826171875 KB  
    --创建5000个table，内存增加了414 KB;这些增加的内存，由于已放到了全局函数中，是永远没有机会被回收到了!  
end  
  
function collect2()  
    print("now,Lua内存为:",collectgarbage("count"))-->596.7822265625 KB  
    collectgarbage()  
    collectgarbage()  
    collectgarbage()  
    print("now,Lua内存为:",collectgarbage("count"))-->489.189453125 KB  
    --最后内存增加了284KB(489-205)  
end 
```



垃圾回收器有两个参数用于控制它的节奏：

第一个参数，称为暂停时间，控制回收器在完成一次回收之后和开始下次回收之前要等待多久；

第二个参数，称为步进系数，控制回收器每个步进回收多少内容。粗略地来说，暂停时间越小、步进系数越大，垃圾回收越快。这些参数对于程序的总体性能的影响难以预测，更快的垃圾回收器显然会浪费更多的CPU周期，但是它会降低程序的内存消耗总量，并可能因此减少分页。只有谨慎地[测试](http://lib.csdn.net/base/softwaretest)才能给你最佳的参数值。

