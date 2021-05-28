# 1 命令行设置变量
下面展示一个通过命令行传入变量的示例。

```
foo --nobigMenu -languages="chinese, english, spanish"
```

上面的命令行，通过函数ParseCommandLineFlags()解析，会得到如下结果：
> FLAGS_bigMenu = false   
FLAGS_languages = "chinese, english, spanish"

gflags允许不规范的flag形式：在变量前加no——"--nobig_menu".下面演示了针对languages变量，所有允许的flag形式。

```
1. --languages="chinese"
2. -languages="chinese"
3. --languages "chinese"
4. -languages "chinese"
```

对于布尔值，允许的形式有：

```
// 下面也允许单破折号开头：-bigMenu
1. --bigMenu
2. --nobigMenu
3. --bigMenu=true
4. --bigMenu=false
```

文档建议，对于"非布尔类型"，使用规范形式：--variable=value；对于"布尔类型"，使用形式：--variable, --novariable。

使用未定义flag将会导致严重错误。若多个程序的flag只是一个子集，我们可以指定"undefork"来抑制错误。

同getopt()函数，"--"会终止处理flag。比如下面代码，-f2不会作为flag被处理。

```
foo -f1 1 -- -f2 2
```

若flag被赋值多次，只有最后一个值会生效。

注意，gflags不支持单字母缩写（比如-l代表--library）和单字母组合（比如-la代表--list 和 --all）

# 2 修改flag的默认值
若flag在库中定义，我们希望修改flag的默认值，则可以在调用ParseCommandLineFlags之前，直接修改flag的值。

```
DECLARE_bool(libVerbose);

int main (int ac, char** av)
{
  FLAGS_libVerbose = true;
  ParseCommandLineFlags(...);
}
```

# 3 一些特殊Flags
commandlineflags模块中有一些内置flag，它们可分成三类：

#### 1 报告类
这类变量会输出关于程序的一些信息，然后直接退出程序。

|flag|说明|
|---|----|
|--help|显示所有source文件中定义的flags<br>按文件名和变量名顺序排序，对于每个flag，依次显示(flag名，默认值，提示信息)|
|--helpfull|功能同--help，显示所有的flag信息。<br>未来help功能会发生变化|
|--helpshort|只显示在与可执行文件名相同的文件中，定义的flag。<br>通常可执行文件中包含main()函数|
|--helpxml|功能同--help，但是采用xml格式|
|--helpon=FILE|只显示在FILE.*中定义的flag|
|--helpmatch=S|只显示在*S*.*中定义的flag|
|--helppackage|显示与main()所在文件相同目录定义的flag|
|--version|打印可执行文件的版本号|

#### 2 控制转换类
第2种flag用于控制其他的flag如何进行转换。

|flag|说明|
|---|----|
|--undefork=flagname0, flagname1|在--undefork后面列出的flag，如果程序中没有定义，但是命令行中出现，则会抑制错误输出，并不退出程序|

#### 3 递归类
第3中是“递归类型”，他们的变化会影响到其他flag的值发生变化。

###### (1) --fromenv
 --fromenv=foo, var，表示从环境变量中，为flag foo和bar读取对应的值。为了能正确读取变量，我们需要在环境变量中export变量。

```
export FLAGS_foo=xxx; export FLAGS_bar=yyy # sh
setenv FLAGS_foo=xxx; setenv FLAGS_bar=yyy #tcsh

上面的语句等同于在命令行输入：

--foo=xxx --bar=yyy
```

如果foo没有定义，使用--fromenv=foo会报出严重错误（fatal error),除非用--undefork=foo来抑制错误。

同样，如果没有在环境变量中定义FLAGS_foo，则使用--fromenv=foo也会报出严重错误。


###### (2) --tryfromenv
该flag与同--fromenv基本相同。区别在于若FLAGS_foo没有在环境变量中定义，FLAGS_foo使用在程序中的默认值。

若foo没有在程序中定义，--tryfromenv=foo依然会报错。

###### (3) --flagfile
某些程序运行起来需要指定大量复杂的flag，因此将这些flag写入文件，可以方便多次运行。

--flagfile=file即用于此种情景。commandlineflags模块将读取文件file中的所有flag赋值，然后转换给程序。如下所示为文件/tmp/myflags

```
// /tmp/myflags
--nobigMenu
--languages=chinese, english
```

一下两条命令作用相同

```
./myapp --foo --nobigMenu --languages=chinese,english --bar
./myapp --foo --flagfile=/tmp/myflags --bar
```

--flagfile会抑制很多错误信息，尤其无法识别的flag会被直接忽略，比如--languages这种flag，没有值的情况。

上面的例子比较简单，更通用的写法是，在file文件中，给出所有flag定义所在的文件名，以及参数。以文件名开始，每个文件名占一行，然后是flag，每个flag各占一行。

文件名可以使用通配符（\*和?），此时，只有在可执行文件名与通配符文件名匹配时，后面的flag才会被处理。file文件中也可以不指定文件名，直接给出flag，这些flag将会传递给可执行文件。

file文件中，以“#”开头的行为注释行，被直接忽略。以空格开头的行被认为是空行。

我们可以使用--flagfile来包含其他的flagfile。处理flagfile或者flag，先后顺序确定。

# 4 杂项
下面代码的第一行抑制help信息编译到源文件中。这样可以减少程序大小，并增加安全性。

```
#define STRIP_FLAG_HELP 1
#include <gflags/gflags.h>
```
