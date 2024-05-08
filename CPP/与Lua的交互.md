## Lua虚拟栈

C/C++与Lua交互的基础源于**虚拟栈**。在Lua中，Lua堆栈就是一个struct，堆栈索引的方式可是是正数也可以是负数，区别是：正数索引1永远表示栈底，负数索引-1永远表示栈顶

![image-20230728163912427](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20230728163912427.png)

入栈的数据类型包括数值, 字符串, 指针, talbe, 闭包等, 下面是一个栈的例子:

![image-20230728164749472](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20230728164749472.png)

```c
#include <lua.h>
#include <stdio.h>
#include "lauxlib.h"

int main()
{
    lua_State *L = luaL_newstate();

    // 创建并压入一个闭包
    lua_pushcclosure(L, func, 0);

    // 新建并压入一个表
    lua_createtable(L, 0, 0);

    // 压入一个数字
    lua_pushnumber(L, 100);

    // 压入一个字符串
    lua_pushstring(L, "hello，lua");
    
    return 0;
}
```



操作数据时，首先将数据拷贝到栈上，然后获取数据，栈中的每个数据通过索引值进行定位，索引值为正时表示相对于栈底的偏移索引，索引值为负时表示相对于栈顶的偏移索引。索引值以1或 -1起始值，因此栈顶索引值永远为-1, 栈底索引值永远为1 。 "栈"相当于数据在Lua和C之间的中转地，每种数据都有相应的存取接口 。

另外，还可以使用栈来保存临时变量。栈的使用解决了C和LUA之间两个不协调的问题：

1.  Lua使用自动垃圾收集机制，而C要求显式的分配和释放内存
2. Lua使用动态数据类型，而C使用静态类型；

详细看一下Lua中数据类型的底层实现

![image-20230731120701805](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20230731120701805.png)



TValue结构对应于Lua中的所有数据类型，是一个 {值，类型} 结构，这就是Lua的动态类型的实现。它把值和类型绑在一起，使用 tt 来记录value的类型。 value是一个联合结构，由 Value 定义，可以看到这个联合有四个域

- p ：可以存放一个指针，实际上是lua中的 light userdata结构
- n ：所有的数值存在这里，lua_pushinteger存放到这里
- b ：Boolean值存在这里
- gc ：其他诸如table，thread，string，clouser等需要内存管理垃圾回收的类型都存放到这里，gc是一个指针，指向的类型由联合体 GCObject决定
  - 明显看出，GCObject有 string，userdata，table，thread等




### 堆栈操作

http://cloudwu.github.io/lua53doc/contents.html

#### 初始化lua状态机

- lua_State* lua_open()
- lua_State* lua_newState(lua_Alloc f, void *ud);

lua_newState 创建一个新的，独立的lua状态机，如果因为内存不足导致创建失败，返回NULL，参数f 指定内存分配函数, 参数ud是传给f 函数的指针

lua_open 没有指定内存分配函数的功能，不建议使用

注意：lua_State表示的一个Lua程序的执行状态，它代表一个新的线程（注意是指Lua中的thread类型，不是指操作系统中的线程），每个thread拥有独立的数据栈以及函数调用链，还有独立的调试钩子和错误处理方法。

#### 加载lua库

- void luaL_openlibs(lua_State *L);          
- void luaopen_base(lua_State *L);
- void luaopen_package(lua_State *L);
- void luaopen_string(lua_State *L);
- void luaopen_io(lua_State *L);
- void luaopen_table(lua_State *L);
- void luaopen_math(lua_State *L);

#### stack操作

- void lua_pushboolean (lua_State *L, int b);
- void lua_pushcclosure (lua_State *L, lua_CFunction fn, int n);
- void lua_pushcfunction (lua_State *L, lua_CFunction f);
- const char *lua_pushfstring (lua_State *L, const char *fmt, ...);    将一个格式化字符串压入堆栈，并返回指向这个字符串的指针；
- void lua_pushinteger (lua_State *L, lua_Integer n);
- void lua_pushlightuserdata (lua_State *L, void *p);
- void lua_pushliteral (lua_State *L, const char *s);
- void lua_pushlstring (lua_State *L, const char *s, size_t len);    将一个指定大小的字符串压入堆栈；
- void lua_pushnil (lua_State *L);
- void lua_pushstring (lua_State *L, const char *s);
- void lua_pushvalue (lua_State *L, int idx);     将栈中指定索引的元素复制一份到栈顶；

值得注意的是，向栈中压入一个元素时，应该确保栈中具有足够的空间，可以调用lua_checkstack来检测是否有足够的空间。

实质上这些API是把C语言里面的值封装成Lua类型的值压入栈中的，对于那些需要垃圾回收的元素（如string、full userdata），在压入栈时，都会在Lua（也就是Lua虚拟机中）生成一个副本，从此不会再依赖于原来的C值。例如lua_pushstring 向栈中压入一个以'\0'结尾的字符串，在C中调用这个函数后，可以任意修改或释放这个字符串，也不会出现问题。



- void lua_pop(lua_State *L, int n);
- int lua_gettop (lua_State *L);    返回栈顶元素的索引（也即元素个数）
- void lua_concat (lua_State *L, int n);    将栈顶开始的n个元素连接起来，并将它们出栈，然后将结果入栈；
- void lua_getfield (lua_State *L, int index, const char *k);    将t[k]压入堆栈，t由参数index指定在栈中的位置；
- void lua_setfield (lua_State *L, int index, const char *k);    相当于t[k]=v，t由参数index指定在栈中的位置，v是栈顶元素，改函数会将栈顶的value出栈；
- void lua_getglobal(lua_State *L, char *name);    等价于 lua_getfield(L, LUA_GLOBALSINDEX, s)，注意：栈中LUA_GLOBALSINDEX索引位置处是当前Lua状态机的全局变量环境。
- void lua_setglobal (lua_State *L, const char *name);    等价于 lua_setfield(L, LUA_GLOBALSINDEX, s)；
- void lua_insert (lua_State *L, int index);    移动栈顶元素到index指定的位置；
- void lua_remove (lua_State *L, int index);    移除index处的元素，index之上的元素均下移一个位置；
- void lua_replace (lua_State *L, int index);    栈顶元素移到指定位置，并取代原来的元素，原先的栈顶元素弹出；
- int lua_next (lua_State *L, int index);    弹出一个key，然后将t[key]入栈，t是参数index处的table；在利用lua_next遍历栈中的table时，对key使用lua_tolstring尤其需要注意，除非知道key都是string类型。
- size_t lua_objlen (lua_State *L, int index);    返回index处元素的长度，对string，返回字符串长度；对table，返回"#"运算符的结果；对userdata，返回内存大小；其它类型返回0；
- void luaL_checkstack (lua_State *L, int sz, const char *msg);     增加栈大小（新增sz个元素的空间），如果grow失败，引发一个错误，msg参数传递错误消息



### 类型转换

int lua_toboolean (lua_State *L, int index);
lua_CFunction lua_tocfunction (lua_State *L, int index);
lua_Integer lua_tointeger (lua_State *L, int index);
const char *lua_tolstring (lua_State *L, int index, size_t *len);
lua_Number lua_tonumber (lua_State *L, int index);
const void *lua_topointer (lua_State *L, int index);
const char *lua_tostring (lua_State *L, int index);
lua_State *lua_tothread (lua_State *L, int index);
void *lua_touserdata (lua_State *L, int index);

lua_toboolean 将给定index索引处的元素转换为bool类型（0或1）；
lua_tocfunction 将给定index索引处的元素转换为C函数；
lua_tointeger 将给定index索引处的元素转换为int类型；
lua_tolstring 将给定index索引处的元素转换为char*类型，如果len不为空，同时还设置len为字符串长度；该函数返回的指针，指向的是Lua虚拟机内部的字符串，这个字符串是以'\0'结尾的，但字符串中间也可能包含值为0的字符。
lua_tostring 等价于参数len=NULL时的lua_tolstring；
lua_tonumber 将给定index索引处的元素转换为double类型；
lua_topointer 将给定index索引处的元素转换为void*类型；
lua_tothread 将给定index索引处的元素转换为lua_State*类型（即一个thread）；
lua_touserdata 返回给定index索引处的userdata对应的内存地址；



## C调用Lua

https://www.cnblogs.com/pied/archive/2012/10/26/2741601.html





## Lua调用C

#### 方式1：

https://www.cnblogs.com/djzny/p/11050789.html

做法是把c文件打包成一个动态库，在lua中进行require，并调用



#### 方式二：

https://blog.csdn.net/Dr_chaser/article/details/117628726

将c的函数，注册到lua虚拟机中，在lua中直接使用即可

有多种方式将c函数注册到lua中

- lua_register
- luaL_requiref + luaL_newlib + luaL_Reg[] 的方式，这个比较主流和方便

注意，需要在加载lua文件前，注册c的函数



## LuaBridge





## 参考文献

- http://cloudwu.github.io/lua53doc/contents.html
- https://www.cnblogs.com/chenny7/p/3993456.html
- https://blog.csdn.net/zhoutianzi12/article/details/108253246?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-5-108253246-blog-83344996.pc_relevant_multi_platform_whitelistv3&spm=1001.2101.3001.4242.4&utm_relevant_index=8

- https://blog.csdn.net/feihe0755/article/details/124672798?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-4-124672798-blog-83344996.pc_relevant_multi_platform_whitelistv3&spm=1001.2101.3001.4242.3&utm_relevant_index=7

- https://blog.csdn.net/weixin_46935110/article/details/128588463

- 

  