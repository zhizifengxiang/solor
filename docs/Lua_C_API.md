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
// 若元素值为一个数字，则将数字按照ASCII码转换成字符（此转换会让lua_next产生困惑）。
// 返回字符串尾部自动补充"\0"，但是有可能在串内也有\0出现，可以使用函数lua_strlen获得字符串实际长度。
// 由于Lua回收机制，需要将获得的字符串复制出来，或者放到注册表（registry)，否则将来可能会被移出栈。
size_t      lua_strlen          (lua_State *L, int index); 
lua_CFunction lua_tocfunction   (lua_State *L, int index);
// 该函数返回一个C函数指针，失败返回NULL
void          *lua_touserdata   (lua_State *L, int index);
lua_State     *lua_tothread     (lua_State *L, int index);
// 返回值必须是个thread，转换失败则返回NULL
void          *lua_topointer    (lua_State *L, int index);
// 该函数返回一个void *的C指针，其指向对象可能是userdata, table, thread, function.若都不是，则返回NULL。
// 通常该函数用于debug。
```

## 6，Push values onto stack

下面函数将C语言中的值压入栈中：

```
void lua_pushboolean    (lua_State *L, int b);
void lua_pushnumber     (lua_State *L, lua_Number n);
void lua_pushlstring    (lua_State *L, const char *s, size_t len);
void lua_pushstring     (lua_State *L, const char *s);
void lua_pushnil        (lua_State *L);
void lua_pushcfunction  (lua_State *L, lua_CFunction f);
void lua_pushuserdata   (lua_State *L, void *p);
```

（1）上面函数将C语言类型的值转换成Lua值，并压入栈。
（2）其中，lua_pushlstring和lua_pushstring复制原有字符串。对于lua_pushstring, 其只接受末尾有，且其他地方没有“\0”的字符串。否则就需要调用lua_pushlstring()来指定字符串的长度。
（3）也可以使用格式化的字符串函数，这些函数将格式化字符串压入栈，并返回指向这些字符串的指针。

```
const char *lua_pushfstring(lua_State *L, const char *fmt, ...);
const char *lua_pushvfstring(lua_State *L, const char *fmt, va_list argp);
```

上面函数与sprintf和vsprintf基本相同，有以下几点不同：
（1）不需要对空间分配进行额外管理。Lua自动分配回收空间。
（2）转义字符仅限以下几个，不能使用flag, 宽度，精度限定。%%（插入符号%）, %s（插入以\0结尾的字符串）, %f（插入lua_Number）, %d(插入证书), %c（插入以字符形式呈现的整数）。

>void lua_concat(lua_State *L, int n)
>该函数将栈顶上n个值弹出，连接他们，形成新的字符串，并将新的字符串压入栈中。若n=1,则还是栈顶字符串，没变化；若n=0，则旧的栈顶元素变成一个空串。

## 7，control garbage collection

Lua使用两个数来控制其垃圾回收的时机：count 和 threshold。count计数lua使用的内存量，当count达到threshold时，lua开始进行垃圾回收。当垃圾回收结束，count被更新，threshold变为原来值的两倍。可以通过如下函数获得这两个值：
>int lua_getgccount(lua_State *L);
>int lua_getcthreshold(lua_State *L);

两个函数均以KB为单位，可以通过下面函数修改threshold的值：
>void lua_setgcthreshold(lua_State *L, int newthreshold);

如果threshold的值比当前的count小，则立刻进行垃圾回收。若新的threshold=0，则强制进行垃圾回收操作。进行回收后，新threshold值更新。

## 8，userdata
Userdata指Lua中C语言的值。Lua支持两种类型的Userdata：full userdata 和 light userdata。lua中没有对方法可以判断是full还是light，但是在C语言中，可以使用lua_type来判断：LUA_TUSERDATA表示full userdata, LUA_TLIGHTUSERDATA表示light data。
一个Full userdata代表一块内存数据，其为一个对象（例如table）：需要手动创建该对象，并在其回收时能够探测到，其可以拥有metatable。full data只能和自己相等（在raw equality)。
一个light user代表一个指针，其为一个值（比如数字），该值无需被创建，也没有销毁操作，其不拥有metatable。Userdata只能和具有相同C 指针地址相同的其他userdata相等。

下面函数可以创建一个新的full userdata,该函数分配一块大小为size的内存块，将执行该内存块的指针压入栈，并返回。若将light userdata压入栈，参见第6节，lua_pushlightuserdata()。
>void *lua_newuserdata(lua_State *L, size_t size);

lua_touserdata（第5节）函数返回userdata的值，对于full userdata，该函数返回内存块的地址；对于light userdata，其返回一个指针；对于其他类型，则返回NULL。当Lua垃圾回收full userdata，其调用userdata的metamethod，并释放相应内存。

## 9，metatable
下面函数为操作metatable的函数：
```
int lua_getmetatable(lua_State *L, int index);
// 该函数将指定Index的元素对应的metatable压入栈，若index无效，或者访问的对象没有metatable
// 则函数返回0，且任何元素都不会压入栈。

int lua_setmetatable(lua_State *L, int index);
// 该函数从栈顶弹出一个table，并作为执行Index对应元素的metatable，若无法设置metatable(index对应的对象不是userdata或者table)
// 则返回0，即便顶层的table被弹出。
```

## 10， load Lua chunks
使用lua_load函数加载Lua chunk:

```
typedef const char *(*lua_Chunkreader)
                      (lua_State *L, void *data, size_t *size);
int lua_load(lua_State *L, lua_Chunkreader reader,
                void *data, const char *chunkname);
```

该函数的返回值为：
```
0 ： 没错误
LUA_ERRSYNTAC : 预编译时产生syntax（语法）错误
LUA_ERRMEM:   内存分配错误
```
若没错误，则该函数将编译好的函数压入栈顶，否则错误消息压入栈顶。该函数自动检测代码块是文本还是二进制代码，并相应进行加载（调用luac）。
该函数使用用户提供的自定义reader函数来读取chunk，每次其需要一个chunk，lua_load就调用一次reader，并传递数据参数。reader必须返回一个新创建的指向内存块的指针，size参数指定内存块大小。使用NULL来指出chunk结束，reader函数可能返回任意大小的chunk。

在当前实现中，reader函数无法调用所有的lua函数，需要将lua_State*指定为NULL来调用。

chunkname参数用于错误输出和debug。具体参见lauxlib.c中的例子，来使用lua_load，以及一些现有函数来加载file和string chunk。

## 11，manipulate tables
调用下面函数创建一个新的空table,并将该table压入栈中。
>void lua_newtable(lua_State *L)

下面函数读取栈中指定index的table，其将table中的某个key弹出，并将key对应value返回并压入栈。注意table还会在栈中。该函数可能调用metamethod函数index。
>void lua_gettable(lua_State *L, int index);
若只是打算获得原始值，而不调用metamethod，则使用下面函数
> void lua_rawget(lua_State *L, int index);

下面函数将提前压入栈中的Key和value存储在index对应的table中。table还留在原位，而key和value将会被弹出。lua可能会触发metamethod函数settable或者newindex。
>void lua_settable(lua_State *L, int index);

若不打算调用metamethod函数，则调用下面函数：
>void lua_rawset(lua_State *L, int index);

可以使用下面函数来遍历table,index指出table在栈中的位置，该函数从栈顶弹出一个key,并将下一个key-value pair压入栈中（注意是从栈中弹出key）。若table中没有元素，则函数返回0，并什么也不压入。使用key=nil来开始遍历操作。
>int lua_next(lua_State *L, int index);

下面是个典型遍历操作的代码：
```
/*table 位于栈中Index位置上*/
lua_pushnil(L); // first key
while (lua_next(L,index) != 0) {
  // key is at index=-1, value is at index=-1
  printf("%s - %s\n", lua_typename(L, lua_type(L, -2)), lua_typename(L, lua_type(L, -1)));
  lua_pop(L, 1); // remove value, keep key for next iteration.
}
```

若可以确定key是个string，在调用lua_tostring函数，否则不可调用。调用lua_tostring将会改变给定index的value值，这回让下一轮的lua_next调用感到困惑。

## 12 manipulate environment
所有全局变量都保存在ordinary lua table中，这些确具变量叫做环境变量。初始环境变量被称作global environment。该table可通过伪索引LUA_GLOBALSINDEX访问。为了访问和修改全局变量，可以之用regular table operation对environment talbe进行操作。比如访问全局变量的值，如下面代码所示：
```
lua_pushstring(L, varname);
lua_gettable(L, LUA_GLOBALSINDEX);
```

可以使用lua_replace函数来改变某个Lua线程的全局环境变量：
```
void lua_getfenv(lua_State *L, int index);
// 该函数将指定索引Index处函数的environment table压入栈顶。
// 若函数为C函数，lua_getfenv则压入global environment。
void lua_setfenv(lua_State *L, int index);
// 该函数弹出栈顶上的table，并将其设置为指定index处函数的新environment。
// 若index处的对象不为Lua 函数，则该函数返回0.
```

## 13，use table as array
下面可以让lua的table以数组形式进行访问：
```
void lua_rawgeti(lua_State *L, int index, int n);
// 该函数将index处的table中的第n个元素的值压入栈顶
void lua_rawseti(lua_State *L, int index, int n);
// 该函数将index处的table中的第n个元素设置为栈顶元素，并将其移出栈顶。
```

## 14，call function
lua中定义的函数，以及在lua中注册的C函数，可通过以下函数由宿主程序进行调用：
>void lua_call(lua_State *L, int nargs, int nresults);

（1）首先，被调用的函数压入栈。
（2）然后，传入参数按先后顺序压入栈（从左到右，第一个参数先压入）。
（3）最后，调用lua_call函数。

nargs为压入栈的传入参数数量。调用后，传入参数和函数都会弹出栈，函数返回结果再压入栈。返回值将根据nresults来返回，当nresults=LUA_NULTRET时，则所有返回值都会被压栈。结果的压入顺序也是第一个先压入，因此最后一个结果在栈顶。

对于lua程序：
>a = f("how", t.x, 14)
对应的C程序如下：
```
lua_pushstring(L, "t");
lua_gettable(L, LUA_GLOBALSINDEX); // 将t设置为全局变量
lua_pushstring(L, "a") // 变量名
lua_pushstring(L, "f") // 函数名
lua_gettable(L, LUA_GLOBALSINDEX); // 将要被调用的函数
lua_pushstring(L, "how") // 第一个参数
lua_pushstring(L, "x"); 
lua_gettable(L, -5); // 压入t.x的结果（第二个参数）
lua_pushnumber(L, 14) // 第三个参数
lua_call(L, 3, 1); //调用3个参数，1个返回值的函数
lua_settable(L, LUA_GLOBALSINDEX); // 设置全局变量为'a'
lua_pop(L, 1);
```

## 15, protected calls
调用lua_call产生的任何错误都会向上推（使用longjmp）。若需要处理error，则用下面函数：
>int lua_pcall(lua_State *L, int nargs, int nresults, int errfunc);

若没有错误，则和lua_call没差别。若有错误，该函数将错误信息压入栈中，并返回一个错误代码。注意：lua_pcall在调用后，也将移除被调用的函数和传入参数。
若errfunc=0，返回的错误信息为原始错误信息，否则，errfunc将会是在stack中错误处理函数的索引。（当前实现没有将其作为伪索引）。对于运行时错误，错误处理函数将接收错误消息，其返回值将传递给lua_pcall，由lua_pcall将处理函数的返回值返回。通常，错误处理函数为了调试所用，这些debug information无法在lua_pcall返回后在册收集，因为栈已经发生改变。

lua_pcall函数在以下情况下返回0：成功，或者产生如下错误（定义域lua.h）：
```
LUA_ERRRUN : 运行时错误
LUA_ERRMEM : 内存分配错误。此类错误不会调用错误处理函数。
LUA_ERRERR : 运行错误处理函数的时候产生错误。
```

## 16， define C function
C 函数需要以如下形式定义，C 函数接收Lua state并返回一个整数，该整数将返回给Lua。

> typedef int (*lua_CFunction) (lua_State *L);

C函数应遵循下面的原型，来进行参数传递和返回结果：函数通过栈来接收Lua传入的参数，第一个参数位于栈的index=1位置。函数通过压入结果到栈来返回结果，并返回压入栈中元素的数量。就像Lua函数，C函数也可以返回多个值。

下面的例子接收一个数值型参数，并返回其均值与和。

```
static int foo ( lua_State *L) {
  int n = lua_gettop(L); //获得传入参数
  lua_Number sum = 0;
  int i;
  for (i = 1; i < n; ++i) {
    if (!lua_isnumber(L, i)) {
      lua_pushstring(L, "incorrect argument to function 'average'");
      lua_error(L);
    }
    sum += lua_tonumber(L, i);
  }
  lua_pushnumber(L, sum/n); //第一个结果
  lua_pushnumber(L, sum); // 第二个结果
  return 2; // 结果数量
}
```

对于注册C函数，可使用下面的宏。该宏接收函数在Lua中的名字，以及指向该函数的指针。
```
#define lua_register(L, n, f) \
        (lua_pushstring(L, n), \
        lua_pushcfunction(L, f), \
        lua_settable(L, LUA_GLOBALSINDEX))
// lua_State *L;
// const char *n;
// lua_CFunction f;
```

上面定义的函数使用下列语句注册：
>lua_register(L, "average", foo);

## 17, define C closure
当一个C函数创建，我们可以与其关联一些值，从而建立C闭包（C closure）。每当这个函数调用时，这些值都可以被函数访问。为了实现闭包，首先需要将这些被关联的值压入栈中（第一个值首先被压入），然后调用下面函数，将C函数压入栈中。
> void lua_pushcclosure(lua_State *L, lua_CFunction fn, int n);

（1）参数n指明有需要与函数关联的值的数量。（lua_pushcclosure将这些被关联的值弹出栈）。事实上，宏lua_pushcfunction就是调用lua_pushcclosure时，n=0。
（2）每当C函数被调用，这些值可以通过伪索引进行访问，这些伪索引由宏lua_upvalueindex产生，第一个关联的值可通过调用此函数访问：lua_upvalueindex(1)。当n的值大于当前函数上值（upvalue)数量时，调用lua_upvalueindex(n)会产生一个可接受（acceptable）但是无效（invalid）的索引（index）。

查看src/lib/*.c文件可找到对应的C函数以及闭包。

## 18， registry
lua提供了一个registry（注册表），该注册表为一个pre-defined table， 可被C代码用来存储任何Lua值，尤其是在需要存储超过当前C函数周期的Lua变量时。该table可以通过伪索引：LUA_REGISTRYINDEX来访问。认可C库都可在该表中存储数据，只要其选择的Key与其他库的不同。通常，可以使用包含库名的字符串，或者以C指针形式的light userdata来作为key。

在引用机制下，registry中使用整数key，由auxiliary library来实现，因此不能用于其他目的。

## 19 error handling in C
Lua 内部使用C语言的longjmp机制来处理error。当Lua需要处理error（如内存分配错误，类型错误，语法错误）时，其将raise error，即进行一个long jump。一个protected environment使用setjmp函数来设置恢复点，任何错误都会跳到最近的active recover pointer处。

若在protected environment之外发生错误，Lua调用panic function，并调用exit(EXIT_FAILURE)。通过下面函数来设置panic函数：
>lua_CFunction lua_atpanic(lua_State *L, lua_CFunction panicf);

在你新设置的panic函数中，通过禁止返回操作，来防止程序退出（比如使用long jump）。但是，这样做会让lua状态不一致，处理此种情况最安全的方式就是关闭这个应用。在API中，几乎所有函数都有raise error功能。下列函数运行在protected mode下（即这些函数创建protected environment来运行），则他们永远不会raise error: lua_open, lua_close， lua_load, lua_pcall。

下面也有在protected mode下运行一个给定C函数的函数：

> int lua_cpcall(lua_State *L, lua_CFunction func, void *ud);

该函数在protected mode下运行c函数func.func只使用栈中一个元素作为参数启动——包含ud的一个light userdata。为了处理错误，lua_cpcall返回与lua_pcall相同的错误代码，并将error object压入栈顶；否则，其返回0， 且不改变堆栈。该函数返回的任何值都会被丢弃。

C代码可以通过调用下面函数来生成lua error：
>void lua_error(lua_State *L);

error message（实际上可以为任何类型的对象）需要在栈顶，该函数产生一个long jump, 因此，永远不会返回。

## 20， threads
lua部分支持多线程编程。若C库支持多线程，则lua与该库协同使用，相当于是使用多线程在lua中。同时，lua在线程之上应用其协程系统（coroutine system）。下面的函数在Lua中创建一个新线程：

> lua_State *lua_newthread(lua_State *L);

该函数将新创建的线程压入栈中，并返回一个指向lua_State的指针来代表新创建的线程。返回的新创建的线程共享原有state的所有global object(比如table)，但是有一个独立的运行时栈。没有函数可以显式关闭或销毁线程，和其他lua对象相同，其服从垃圾回收。

为了操纵以coroutine形式的线程，lua提供如下函数：
```
int lua_resume(lua_State *L, int narg);
int lua_yield(lua_State *L, int nresults);
```

（1）为了执行coroutine，首先需要创建一个新线程，然后将需要执行的函数体及其参数压入新线程的栈中。
（2）然后调用lua_resume函数，narg指出传入的参数数量。当coroutine执行结束或者挂起时，该函数调用返回，此时，栈中存放了需要传递给lua_yield函数的值，或者存放了由主题执行函数(body function)返回的值。若成功运行coroutine，则lua_resume返回0，否则，返回error code。
（3）对于任何错误，栈中仅存放error message。若打算重启coroutine，只有在来自于yield函数的结果放入该coroutine对应的栈中后，才可以再次调用lua_resume。lua_yield函数只能以C函数中，返回表达式中的值进行调用，就像下面的代码：
>return lua_yield(L, nresults);

当c函数以上面形式调用lua_yield，正在运行的coroutine将会挂起，启动该coroutine的lua_resume函数调用将会返回。参数nresults为栈中将要传递给lua_resume函数的参数个数。

若要在两个进程中交换数据，则可以使用下面函数：
>void lua_xmove(lua_State *from, lua_State *to, int n);
该函数从栈from中弹出n个值，并将这些值压入到to中。





**本篇完**
