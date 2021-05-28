# 0 序言
本系列将介绍如何使用googleTest测试框架。

google显然高估了用户的技术水平，为了使用google的测试框架，cmake、编译、链接等技术知识需要学习和温习。

本质上，无论qtest，还是googleTest，测试框架都是通过调用函数，将函数产生的结果，与期望结果进行对比。因此，一旦掌握了其中一个框架， 其他的框架学习难度就会降低很多。

本reprository中，qt文件夹下的1.x——qtest测试框架的内容，为google测试框架提供了一些铺垫。如果熟悉qt编程框架，可以先学习一下qtest。

目前google将test和mock两个项目都合并到googletest中。两者具有相似功能。我计划下个系列介绍googlemock。此篇文章所说的googletest，都是有别于googlemock的那个项目。

```
// googletest项目结构目录
|---googletest
  |---googletest(本篇介绍的内容)
  |---googlemock
```

# 1 简单测试一下
googletest虽然在开源代码的同时，都给出了相应的工程管理文件（包括cmake, 微软的VS，xcode），但是需要马上在Linux上运行一个demo来说，需要熟悉工程管理工具，大大提高了学习门槛。

因此，在googletest/googletest/README.md中，给出了简单的使用方式。本篇文章的核心，将介绍如何使用g++编译器，直接构建(build)一个测试程序，并运行这个测试程序。

### 1 googletest项目目录结构
当前，我们只涉及googletest中下面5个目录的内容。

|目录|内容|
|--|--|
|docs|文档信息|
|include|googletest项目的头文件|
|samples|googletest项目的示例程序|
|src|googletest项目源代码|
|test|一些测试程序的源代码|


### 2 将googletest编译成静态库
为了方便使用，我们首先将googletest工程编译成静态库，测试程序链接生成的静态库，此时，只需要源代码(src)和头文件(include)。

注意：googltest需要g++支持c++11标准，pthread运行库，因此需要g++版本在4.8以上。

为了防止demo污染到项目文件，我们单独建一个文件夹，并将googletest/include和googletest/src复制到新的目录下。目录结构如下。

```
|---demo
  |---include // 为googletest/include
  |---src     // 为googletest/src
  |---sampleBuild // 若无特别说明，以下文件来自googletest/samples目录下
    |---其他文件
```

下面，我们就可以在sampleBuild目录下构建静态库。切换到sampleBuild目录，输入如下指令，指令的含义是：
1. -std=c++11：采用C++11标准进行编译.
2. -isystem：该选项后面指定头文件目录，gcc会在搜索默认头文件目录之前，搜索该选项后面的目录。
3. -I：该选项指定头文件目录，gcc会优先搜索该目录下的头文件。
4. -pthread：使用pthread库。
5. -c：生成没有链接的二进制文件，扩展名为.o。

ar命令为打包程序，将所有的二进制文件打包，生成静态库。其参数意义如下：
1. -r：replace，替换库中同名的函数。
2. -v: verbose，详细显示打包过程。

```
g++ -std=c++11 -isystem ../include -I../ -pthread -c ../src/gtest-all.cc
ar -rv libgtest.a gtest-all.o
```

这样，我们就获得了一个文件名为libgtest.a的静态库文件。



### 3 使用测试框架

##### 1 目录结构
此时，demo具有如下结构。sample1为我们添加的，用于使用测试框架的代码文件。

```
|---demo
  |---include // 为googletest/include
  |---src     // 为googletest/src
  |---sampleBuild
    |---libgtest.a // 生成的静态库
    |---sample1.h
    |---sample1.cpp
    |---sample1_unittest.cpp
    |---gtest_main.cc
```

##### 2 定义被测试的函数
下面声明了sample1.h，其中声明了求阶乘的函数，和判断素数的函数。

```
// sample1.h
int Factorial(int n);
bool IsPrime(int n);
```

下面定义了两个函数。

```
// sample1.cpp
#include "sample1.cpp"

int Factorial(int n)
{
  int result = 1;
  for (int i = 1; i <= n; i++) {
    result *= i;
  }
  return result;
}

int IsPrime(int n)
{
  if (n <= 1) return false;
  if (n % 2 == 0) return n == 2;
  for (int i = 3; ; i += 2) {
    if (i > n/i) break;
    if (n % i == 0) return false;
  }
  return true;
}
```

###### 3 定义测试逻辑
定义测试文件。如下展示了使用googletest的测试框架。
1. include声明被测试函数的头文件，注意包含头文件gtest.h。
2. 使用宏TEST定义测试。在googletest中，一个test-case包含若干个test，这些test在逻辑上具有相关性。宏TEST后面的圆括号分别传入两个参数：test-case的名称，和test的名称。test-case在逻辑上属于一个全集，如下所示，Factorial为一个test-case，其根据不同情况，分支处多个test。
3. 在每个宏TEST后，用花括号包含当前test的测试逻辑。测试逻辑使用宏来执行测试语句。
4. test-case和test的命名都遵循c++命名规范，但是名字中不能出现下划线。
5. googletest保证每个test都被执行，但是不能保证test之间从上至下顺序执行，因此，各个test之间不能存在依赖关系。

```
// sample1_unittest.cpp
#include <limits.h>
#include "sample1.h"
#include "gtest/gtest.h"
namespace {
  // test-case的名称为FactorialTest, test的名称为Negative。
  // EXPECT_EQ(期望值，实际值)与EXPECT_TRUE((期望值)==(实际值))作用相同。
  // 当检测错误时，宏EXPECT_*会打印出实际值和期望值。
  TEST(FactorialTest, Negative) {
    EXPECT_EQ(1, Factorial(-5)); // 期望两值相等
    EXPECT_EQ(1, Factorial(-1));
    EXPECT_GT(Factorial(-10), 0); // 期望左值大于右值
  }
  TEST(FactorialTest, Zero) {
    EXPECT_EQ(1, Factorial(0));
  }
  TEST(FactorialTest, Positive) {
    EXPECT_EQ(1, Factorial(1));
    EXPECT_EQ(2, Factorial(2));
    EXPECT_EQ(6, Factorial(3));
    EXPECT_EQ(40320, Factorial(8));
  }

  TEST(IsPrimeTest, Negative) {
    EXPECT_FALSE(IsPrime(-1)); // 期望值为false
    EXPECT_FALSE(IsPrime(-2));
    EXPECT_FALSE(IsPrime(INT_MIN));
  }
  TEST(IsPrimeTest, Trivial) {
    EXPECT_FALSE(IsPrime(0));
    EXPECT_FALSE(IsPrime(1));
    EXPECT_TRUE(IsPrime(2)); // 期望值为true
    EXPECT_TRUE(IsPrime(3));
  }
  TEST(IsPrimeTest, Positive) {
    EXPECT_FALSE(IsPrime(4));
    EXPECT_TRUE(IsPrime(5));
    EXPECT_FALSE(IsPrime(6));
    EXPECT_TRUE(IsPrime(23));
  }
}
```

最后，googletest给出了调用上面的测试主函数文件。
1. 我们不能直接调用宏TEST,而是需要在main函数中，使用宏RUN_ALL_TESTS()来运行所有的测试。
2. src/gtest_main.cc中给出了main函数，如果测试成功，则返回0，失败返回1。
3. 宏RUN_ALL_TESTS()知道所有的测试用例，我们无需手动注册测试用例。

```
// 代码在src/gtest_main.cc中
#include <cstdio>
#include "gtest/gtest.h"

#ifdef ARDUINO
void setup() {
  // Since Arduino doesn't have a command line, fake out the argc/argv arguments
  int argc = 1;
  const auto arg0 = "PlatformIO";
  char* argv0 = const_cast<char*>(arg0);
  char** argv = &argv0;

  testing::InitGoogleTest(&argc, argv);
}

void loop() { RUN_ALL_TESTS(); }

#else

GTEST_API_ int main(int argc, char **argv) {
  printf("Running main() from %s\n", __FILE__);
  testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
#endif
```

##### 4 编译测试程序
输入下面的指令，即可编译测试程序。注意：除了当前目录不需要加入到头文件外，因为使用静态库，所以需要指定头文件目录。

```
g++ -std=c++11 -isystem ../include -pthread sample1.cpp sample1_unittest.cpp gtest_main.cc libgtest.a  -o main
./main
```
