# 0 序言
gflags是谷歌公司的一个开源库，这个软件主要用于处理，用户通过终端命令行输入的参数。

通常，我们在执行命令行软件时，会向软件中传入若干参数。gflags将参数分成两类：

>1 flag : 这种参数会以若干个“-”（破折号）开始。   
2 command argument: 这种参数直接给出一些参数。

比如下面的命令行指令。-l 和 -f都是flag, -f后面还带着一个参数/var/tmp/foo。Nick, Rhan, 和 bonjure都是command-argument。

```
./software -l -f /var/tmp/foo Nick Rhan bonjure
```

所以说，flag可以看成是传给软件的(key, value)，因为(key, value)可以以多种形式传入，比如--key=value, 或者--key value。因此，gflags也支持多种形式的(key, value)传入操作。

gflags的document列举了多种好处，此处不再赘述，通过使用，大家可以逐步体验。

# 1 构建gflags库
gflags支持多种构建工具的编译，由于之前学了一些cmake的内容，因此本文章采用cmake构建gflags库。

执行下面命令。其中-DCMAKE_INSTALL_PREFIX用于定义gflags的安装目录。

```
cd gflags-2.x
mkdir build
cd build
cmake ../ -DCMAKE_INSTALL_PREFIX=/home/nickRhan/usr/local

make
make install
```

安装完后，我们就会在/home/nickRhan/usr/local目录下看到三个目录：include(头文件目录),lib(库文件目录，如果在构建过程中未特别指定，gflags将编译成静态库), bin(测试程序)

```
|---local
  |---lib
    |---cmake #
    |---libgflags.a
    |---libgflags_nothreads.a
    |---pkgconfig
  |---include # 头文件目录
  |---bin
```
其中，libgflags_nothreads.a就是我们需要链接的库（libgflags.a支持多线程，目前我们不需要）。
到此，我们就编译好了静态库。

下面将链接静态库，使用gflags。

# 2 一个小demo
下面将给出一个简单的例子，演示如何使用gflags。

使用gflags中预定义的宏，分别在程序中定义如下三个变量。其中：
1. DEFINE_x：x为需要定义的变量类型。
2. 括号中的参数分别为：变量名称，默认值，提示信息。
3. 若需要访问变量，则直接在变量名称前加前缀：FLAGS_即可。

```
// main.cpp
#include <gflags/gflags.h>
#include <iostream>

DEFINE_string(name, "nizhang", "input ur name");
DEFINE_int32(age, 27, "input ur age");
DEFINE_bool(sex, true, "r u a man?");

int main(int ac, char** av)
{
      std::cout << FLAGS_name << std::endl
                << FLAGS_age << std::endl
                << FLAGS_sex << std::endl;
        return 0;
}
```

执行下面命令，编译、链接程序。

```
g++ --std=c++11 -I/home/nickRhan/usr/local/include -L/home/nickRhan/usr/local/lib -l/home/nickRhan/usr/local/lib/gflags -o main main.cpp
```

最后生成可执行文件main。输出相应的值。
