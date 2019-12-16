该模块用于解析命令行参数，支持sys.argv传过来的参数。sys.argv返回一个list，argv[0]为执行程序名称，argv[1:]为参数。

解析参数主要分三个步骤：创建解析器对象，添加需要解析的选项，传入命令行字符串、命令list后进行解析。

# 1 创建对象
```
class argparse.ArgumentParser(prog=None, usage=None, description=None, epilog=None, parents=[], formatter_class=argparse.HelpFormatter, prefix_chars='-', fromfile_prefix_chars=None, argument_default=None, conflict_handler='error', add_help=True)
```
下面介绍各个参数的意义
### 1 prog
默认使用sys.argv[0]作为帮助信息中，程序的名称。指定该参数的值：prog='somename'，就会以somename来表示名称。使用对象时，可使用%(prog)s作为变量来指是该值。

### 2 usage
打印出的帮助信息中，usage域指出如何使用当前命令。usage，改变使用说明的内容。

### 3 description
修改对命令功能的简短描述。

### 4 epilog
命令帮助信息结尾显式的字符串内容。默认格式是line-wrapped，可以通过修改formatter_class来定义新的形式。

### 5 parents
一个列表，将父类的选项继承。使用该选项，add_help=False，否则父子都会打印help信息，引发错误。
parent后面再改，子对象也不会随着改变。

### 6 formatter_class
目前有三种格式类型可以选择：
```
class argparse.RawDescriptionHelpFormatter
class argparse.RawTextHelpFormatter
class argparse.ArgumentDefaultsHelpFormatter
```
前两个允许对显式文本有更多的控制，最后一个会添加默认值。默认，ArgumentParser会将description和epilog为line-wrap模式。RawDescriptionHelpFormatter 认为description和epilog已经格式化好，不用换行等操作。 ArgumentDefaultsHelpFormatter会添加变量的默认信息。

### 7 prefix_chars
flag参数中，默认使用"-"和“--”作为前缀。如果使用“+”等，则需要指定该值。

### 8 fromfile_prefix_chars
有时参数比较长，放入文件中。该参数指定一个前缀，凡是在该前缀后面的字符串，都被认为是文件。文件中的参数关键词每个占用一行，且和命令行出现顺序一致。如下面例子：
```
with open('ha.txt', 'w') as fp:
  fp.write('-f\nbar')
parser = argparse.ArgumentParser(fromfile_prefix_chars="@")
parser.add_argument('-f')
parser.parse_arg(['@ha.txt'])
```

### 9 argument_default (没明白)
全局设置默认参数。可以设置Suppress属性来抑制解析。

### 10 conflict_handler
模块禁止同一个flag来对应两个不同的动作，此时会引出error。比如下面情况会抛出异常：
```
>>> parser = argparse.ArgumentParser(prog='PROG')
>>> parser.add_argument('-f', '--foo', help='old foo help')
>>> parser.add_argument('--foo', help='new foo help')
Traceback (most recent call last):
 ..
ArgumentError: argument --foo: conflicting option string(s): --foo
```
该参数指定某一些字符串，来让parser采取行动来处理冲突。如下面的例子，采用了折中的方案。
```
>>> parser = argparse.ArgumentParser(prog='PROG', conflict_handler='resolve')
>>> parser.add_argument('-f', '--foo', help='old foo help')
>>> parser.add_argument('--foo', help='new foo help')
>>> parser.print_help()
usage: PROG [-h] [-f FOO] [--foo FOO]

optional arguments:
 -h, --help  show this help message and exit
 -f FOO      old foo help
 --foo FOO   new foo help
```

### 11 add_help
通常模块会自动加上-h/--help来支持显式帮助信息。add_help=False，将禁止这个选项。可以直接调用函数print_help()打印帮助信息。如果prefix_chars=发生变动，也将改变选项。

# 2 添加解析选项
其描述了解析选项的行为。
```
ArgumentParser.add_argument(name or flags...(用逗号隔开的多个字符串)， action, nargs, const, default, type, choices, required, help, metavar, dest)
```

### 1 name or flags
分option（如-f, --help）和position （如一系列文件名）。
```
option: parser.add_argument('-f', '--foo')
position: parser.add_argument('bar')
```

进行解析时，option参数根据"-"来解析，剩下的就都是position参数。

### 2 action
该选项指出针对命令行参数如何进行处理。
1. 'store': 仅存储参数值，其为默认行为
2. 'store_const'：存储“const”参数指定的值。当使用某些开关参数flag的时候比较有用。比如
```
>>> parser = argparse.ArgumentParser()
>>> parser.add_argument('--foo', action='store_const', const=42)
>>> parser.parse_args(['--foo'])
Namespace(foo=42)
```

3. 'store_true', 'store_false':和'store_const'相同，指是更强调bool值。
4. 'append':指出如果多次出现某个参数，则将其值追加到一个list中
5. 'append_const':将多个const的值追加到列表中。如下面例子。
```
>>> parser = argparse.ArgumentParser()
>>> parser.add_argument('--str', dest='types', action='append_const', const=str)
>>> parser.add_argument('--int', dest='types', action='append_const', const=int)
>>> parser.parse_args('--str --int'.split())
Namespace(types=[<type 'str'>, <type 'int'>])
```
6. 'count':计数某个关键字参数出现次数。比如统计递增的详细信息层次。
```
>>> parser = argparse.ArgumentParser()
>>> parser.add_argument('--verbose', '-v', action='count')
>>> parser.parse_args(['-vvv'])
Namespace(verbose=3)
```

7.



### 3 nargs


### 4 const
### 5 default
### 6 type
### 7 choices
### 8 required
### 9 help
### 10 metavar
### 11 dest
### 12 Action类


# 3 解析命令行
