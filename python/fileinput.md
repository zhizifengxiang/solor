该模块用于快速迭代stdin和一系列输入文件，如果只是读写一个文件，参见open()。

```
import fileinput
for line in fileinput.input()
  process(line)
```

上面程序便利所有sys.argv[1:]中列出的文件。如果list为空，则迭代sys.stdin。文件名为“-”也由sys.stdin替换。
如果需要指定某个文件名，向input()传入文件名列表。如下：
```
import fileinput
filename = ['a', 'b', 'c']
for line in fileinput.input(filename):
  print(line)
```

所有文件默认以“文本模式”打开，用户可以修改相应读取模式。读取错误抛出IOError异常。

如果sys.stdin多次使用，第二次以后都不会返回数据，除非是在交互模式，或者被显式reset，比如sys.stdin.seek(0)。
除了作为最后一个文件，空文件打开后会被马上关闭。
可以指定input(), FileInput()打开文件的hook函数，其接受两个参数：filename和mode。并返回一个类似于文件的对象。

# 1 fileinput成员变量

### 1 fileinput.input(files:一个list或者string, inplace=False, backup, mode, openhook)
其创建一个FileInput类对象，参数会直接传递给FileInput构造函数。生成的对象作为全局变量状态（创建一次，整个模块都是用，而不是仅仅作为一个单一变量），供fileinput模块使用。
如果不是active state, 抛出RuntimeError异常。
```
import fileinput
filename = ['a', 'b', 'c']
for line in fileinput.input(filename):
  fileinput.filename() #输出当前遍历的文件名
```

### 2 fileinput.filename()
返回当前读取的文件名，读第一行文件之前，返回None。

### 3 fileinput.fileno()
返回当前文件描述符，如果没有打开的文件，返回-1

### 4 fileinput.lineno()
返回累计行号，在读取第一行之前，返回0，读取最后一行之后，返回所有行号。

### 5 fileinput.filelineno()
返回当前文件行号

### 6 fileinput.isfirstline()
如果刚刚读完第一行，返回True

### 7 fileinput.isstdin()
上一行从sys.stdin中读取，返回True

### 8 fileinput.nextfile()
关闭当前的文件，并开始都下一个文件。上一个文件剩余行不会累计行号。filename只有在读完下一个文件的第一行后，才会变化。该函数不能跳过第一个文件。

### 9 fileinput.close()
关闭文件

# 2 其创建一个FileInput类

```
class fileinput.FileInput(files, inplace, backup, mode, openhook)
```
FileInput除了提供上面的函数，还提供readline()，返回下一个读取的行。\_\_getitem\_\_()实现线性行为。
mode参数可以指定：r, rU, U, rb三种模式
openhook为接受两个参数的函数:filename, mode，返回一个类似文件的对象。不能同时使用openhook和inplace。

inplase:
=1:创建一个备份文件，stdout重定向到输入文件（如果有和备份文件同名的文件，该文件将会被删除）。backup参数指定了备份文件的扩展名，通常为.bak。当输入文件关闭后，备份文件会被删除。

hookup提供了两个已经实现的函数：
fileinput.hook_compressed(filename, mode): 打开gzip和bzip2（.gz, .bz2），如果不是指定扩展名，则按照普通文件打开。比如：f1 = fileinput.FileInput(openhook=fileinput.hook_compressed)

fileinput.hook_encoded(encoding):返回一个使用io.open()函数打开每个文件的hook函数。并以指定编码方式读取文件。
fi = fileinput.FileInput(openhook=fileinput.hook_encoded("iso-8859-1"))
其返回一个Unicode的字符串。
