# 1 定义变量
上一篇简要介绍了gflags的使用。gflags通过宏定义，让用户自定义一些字段。gflags提供了定义多种基础类型的宏。如下表所示。

|类型|说明|
|----|-----|
|DEFINE_bool|布尔值|
|DEFINE_int32|32位带符号整形|
|DEFINE_int64|64位带符号整形|
|DEFINE_uint64|64位不带符号整形|
|DEFINE_double|double类型|
|DEFINE_string|C++字符串类型|

1. 变量定义接受3个参数：（变量名，默认值，help信息）。当程序运行起来时，输入flag：--help，就会显示所有的提示信息。

2. 通常在一个source文件中定义变量，在header文件中声明。其他source文件引用header文件。

3. gflags提供validator来验证参数是否合法。比如文件目录是否存在等。

4. 多数宏（函数）都定义在google命名空间，因此，需要在全局命名控件使用这些函数。

# 2 访问变量
通过在变量前加上“FLAGS_”前缀，来访问变量。如上篇文章所示：

```
DEFINE_string(name, "nizhang", "input ur name");

int main(int ac, char** av)
{
  std::cout << FLAGS_name << std::endl;
  return 0;
}
```

# 3 声明：多文件共享变量
如下代码，若在foo.cc中定义变量name，希望在bar.cc中访问变量name，则可以使用"DECLARE_type"，type为变量的类型。

```
// foo.cc
DEFINE_string (name, "nick", "user's name");

// bar.cc
DECLARE_string (name);

// 上面的DECLARE等效于：
extern FLAGS_name;
```

使用extern会让大型工程难以维护，因此，最好直接在foo.cc对应的头文件中声明：

```
// foo.h
DECLARE_string (name);
```

# 4 RegisterFlagValidator: 检测flag的值
定义变量后，就可以为该变量注册检测函数。每次变量从命令行传入程序，或调用函SetCommandLineOption()来修改变量值，检测函数都会测试变量的新值是否合法。

值合法，检测函数返回true,不合法返回false，此时变量值不变。若变量的默认值为非法，则ParseCommandLindFlags()函数终止。

下面举一个例子：

```
bool validatePort(const char* flagName, int32 port)
{
  if (port > 0 && port < 32768)
    return true;
  return false;
}
DEFINE_int32 (port, 0, "The port listen to");
DEFINE_validator (port, &validatePort); // 注册检测函数
```

在定义变量后，马上注册检测函数，从而保证在进入main()函数，变量被转换时得到检测。

DEFINE_validator函数调用函数RegisterFlagValidator()，注册成功返回true，失败返回false：
1. 注册的变量不存在。
2. 注册的变量被多个检测函数检验。

这个返回值为global static bool <flagName\>\_validator\_registered。


# 5 解析变量
下面介绍如何通过"FLAGS_*"来解析，命令行中传入的变量。如下代码所示。

```
int main(int ac, char** av)
{
  // 下面的函数可能会修改ac，av的值
  // 最后的布尔值为：remove_flags。
  gflags::ParseCommandLineFlags(&ac, &av, true);
}
```

若remove_flags=true，则av中的所有flag都会被删除，并相应修改ac，av只会含有argument。

若remove_flags=false，则ac不会修改，av会做调整，里面的flags都会在命令行开头。并且，函数ParseCommandLineFlags会返回av中第一个出现的arguement的位置。此处为“arg1”的位置，为2——av[2]="arg1"

比如：

```
av = {"/bin/foo", "arg1", "-q", "arg2"}
```

经过调整后：

```
av = {"/bin/foo", "-q", "arg1", "arg2"}
```
