# Lua C API编程初探

## 序

由于业务需要学习C中调用Lua函数接口。查看最近的Lua 5.3文档，以及相关博客教程，其中原理机制相对不甚清晰，复杂内容较多。偶然下载到Lua 5.0的手册，里面说明十分详尽，尤其在原理部分，说明十分清晰。虽然相隔十年，5.0-5.3三个子版本原理基本不变，相应添加的函数相对比较简单，因此，目前对于入门来说，5.0版本的手册不失为一把很好的钥匙。

## 综述

在当前的版本（5.0）中，以下函数的声明可在lua.h中找到。其中函数可能以macro形式进行定义。但是所有函数都没有任何副作用，即函数不会影响到调用其的C代码中的变量，除了Lua state变量。Lua state变量记录之前执行的状态，可以认为是一个状态机。

## 1， States

Lua库完全“可重入”，即其不改变C语言中的变量，所有的运算结果和运算状态，都被保存在一个动态分配的结构体中：lua_State。在调用所有Lua库函数时，都需传入指向该结构体的指针。该结构体使用函数lua_open()函数来创建。因此通常调用函数的结构是：
```
lua_State *luaState = lua_open();
// do something
```
函数原型为：
```
lua_State *lua_open(void); // 创建state
void lua_close(lua_State *L) // 释放state
```
当程序结束时，创建的state会被自动释放掉。若程序需要长时间执行，最好手动释放掉这些state。

## 2，Stack 和 indices
Lua使用一个虚拟栈（virtual stack）来和C进行通信，即通过虚拟栈来互相传递值。栈中的每个元素代表Lua中的value(nil, number, string等)。当Lua调用C函数时，则Lua会创建一个新的栈，该栈与之前创建的任何栈没有任何关系。在栈刚创建时，栈中存放调用C函数后返回的值。

为使用方便，该栈并不严格执行栈的操作，而是可以通过索引（index）来对栈中元素进行访问：
1，正数表示从栈底开始的位置，比如index = 1 ，表示从栈底开始的第一个元素。
2，负数表示从栈顶开始的位置，比如index = -1，表示从栈顶开始的第一个元素。

>有效索引（valid index)：指介于栈底和栈顶之间索引。假设栈中共有n个元素，则1 <= abs(valid index) <=n。

可以通过调用函数lua_gettop来获取栈顶元素的索引（也可得到栈中元素数量，若为0，则表示栈空）：
```
int lua_gettop(lua_State *L);
```
栈的空间无法自动增长，因此需要用户确定当前栈的空间足够大，从而不会出现栈溢出（stack overflow）：
```
int lua_checkstack(lua_State *L, int extra)
```
上面函数将栈大小调整到：top + extra，即栈中可以容纳元素个数为：top+extra。该函数不会收缩栈大小；若扩容失败，则返回false；若栈已经比指定的要大，则不发生任何变化。栈最少可以容纳LUA_MINSTACK个元素，该宏定义于lua.h中，其值为20.

>可接受索引（acceptable index）：上面提到，栈本身空间容量大于栈中的元素数量（即栈顶元素索引），因此凡是在栈最大容量以内的索引都是可接受索引。其范围为： （index < 0 && abs(index) <= top）(此为有效索引) || (index > 0 && index <= stackspace)（此为可接受索引）

注意：0永远为不可接受索引

若为特殊说明，对于任何可以接受valid index的函数，也可以使用伪索引（pseudo-indices）进行访问，其代表那些可以访问C代码的Lua value，且不在栈中。伪索引用于访问全局环境（global environment——即不在Lua调用函数体内的变量），注册值（registry）和C函数的上值（upvalue)。

## 3，Stack manipulation
下面是对栈的基本操作函数：
```
void lua_settop(lua_State *L, int index); // 将栈顶指针值设置为index，可以设为包括0在内的任何整数。
//若新栈顶比旧栈顶低，则栈收缩，新栈顶之上数据被删除。
//若新栈顶比旧栈顶高，则多出来的元素用nil填补；若index=0，则所有元素都被删除。下面宏定义于lua.h中
#define lua_pop(L, n) lua_settop(L, -(n)-1) // 弹出n个元素
void lua_pushvalue(lua_State *L, int index); // 将指定index的元素复制一份，并压入栈中
void lua_remove(lua_State *L, int index); //移除指定index的元素
void lua_insert(lua_State *L, int index); // 将栈顶元素移动到指定index位置上
void lua_replace(lua_State *L, int index); // 将指定index的元素删除，并将栈顶元素插入到index处
```
注意：不能对“伪索引”进行lua_remove或者lua_insert操作，因为伪索引并不指向栈内元素。

## 4，Query the stack
下面函数用于确定栈内元素的类型：
```
int lua_type(lua_State *L, int index);// 返回值为LUA_TNONE表示non-valid index。
//返回值是lua.h中定义的某个值：LUA_TNIL,LUA_TNUMBER, LUA_TBOOLEAN, LUA_TSTRING, 
// LUA_TTABLE, LUA_TFUNCTION, LUA_TUSERDATA, LUA_TTHREAD, LUA_TLIGHTUASERDATA

const char *lua_typename(lua_State *L, int type); //返回元素对应类型的字符串
int lua_isnil(lua_State *L, int index);
int lua_isboolean(lua_State *L, int index);
int lua_isnumber(lua_State *L, int index);
int lua_isstring(lua_State *L, int index);
int lua_istable(lua_State *L, int index);
int lua_isfunction(lua_State *L, int index);
int lua_iscfunction(lua_State *L, int index);
int lua_isuserdata(lua_State *L, int index);
int lua_islightuserdata(lua_State *L, int index);
```
（1）上面所有类型，若与给定类型兼容(compatible)，则返回1，否则返回0。lua_isboolean比较特殊的一点是，其仅在boolean值是有意义，其他值无意义。因为其他值都可以看作是boolean值。
（2）若索引无效，则返回0.
（3）lua_isnumber接受数字或者数字字符串。
（4) lua_isstring接受字符串和数字。为了区分数字和字符串，可使用lua_type函数。
（5）lua_isfunction可同时接受Lua函数和C函数。可使用lua_iscfuntion来区分两种函数。
（6）lua_isuserdata接受full 和 light userdata。可使用lua_islightuserdata来区分两种数据类型。

下面为比较大小函数（索引无效则返回0）：
```
int lua_equal(lua_State *L, int index1, int index2);
int lua_rawequal(lua_State *L, int index1, int index2); //该函数只是进行原始比较，而不调用metamethods。
int lua_lessthan(lua_State *L, int index1, int index2);
```

## 5，从Stack中获得值

下面函数将栈中元素转换为特定C类型：
```
int         lua_toboolean       (lua_State *L, int index); 
// 将元素转换为0或1，若元素不是false或nil，则返回1，否则返回0.
//若索引无效，返回0.若确定是否真的是boolean值，则需要使用lua_isboolean进行检测。
lua_Number  lua_tonumber        (lua_State *L, int index); 
// 将元素转换为数值，默认lua_Number为double类型。
//Lua value必须是一个数字或者可以转换成数字的字符串，否则返回0.
const char  *lua_tostring       (lua_State *L, int index); 
// 将元素转换成字符串，被转换元素必须是字符串或者数字，否则返回NULL。
// 
//
//
size_t      lua_strlen          (lua_State *L, int index); 
size_t      lua_strlen          (lua_State *L, int index);
size_t      lua_strlen          (lua_State *L, int index);
lua_CFunction lua_tocfunction   (lua_State *L, int index);
void          *lua_touserdata   (lua_State *L, int index);
lua_State     *lua_tothread     (lua_State *L, int index);
void          *lua_topointer    (lua_State *L, int index);
```

















