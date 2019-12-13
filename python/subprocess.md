# 0， 综述
subprocess创建新进程，给出输入input, output, error管道，并返回结果码。其为了替换如下模块：
os.spawn\*, os.system, os.popen\*, popen2.\*和command.\*。

# 1 函数

命令1：执行命令，返回值设置returncode
```
subprocess.call(args, \*, stdin=None, stdout=None, stderr=None, shell=False)
举例：
call(['ls', '-l'])
call('exit 1', shell=True)
```

命令2：执行命令，返回值为0，返回程序，否则抛出异常CalledProcessError,该异常带有returncode属性。
```
subprocess.check_all(args, \*, stdin=None, stdout=None, stderr=None, shell=False)
举例：
check_all('exit 1', shell=True) #抛出异常
```

命令3：运行指定命令，并以byte形式返回输出。非0返回值抛异常，带returncode.
```
subprocess.check_output(args, \*, stdin=None, stderr=None, shell=False, universal_newlines=False)
```
# 2 常量

subprocess.PIPE : 指定stdin/out/err可使用管道通信。
subprocess.STDOUT: 可以指定给stderr参数，表示错误从标准输出中打印。

subprocess.CalledProcessError: 异常类，用于check_all()和check_output()返回非0值时。属性有：
>returncode:子进程退出码
>cmd：启动子进程的命令
>output:如果check_output()引发异常，则存储对应的返回值，否则为None。


# 3 函数输入参数
### 1 args
当args为一个字符串，shell=true，或者字符串只是一个单一命令。
当args为一个字符串数组，则第一个字符串为程序名称，后面跟参数。

### 2 stdin/stdout/stderr
指定文件描述符，值可以是PIPE，文件描述符，None。为PIPE，系统将会自动创建一个PIPE。为None，表示使用默认输入输出。子进程可以继承父进程描述符。stderr可以被赋值为stdout，重定向错误到标准输出。

当universal_newlines=True，stdout/err为PIPE，所有换行符均为"\\n"。和open()中universal newline中的“U”模式一致。

### 3 shell
为True将会调用shell来启动命令。通常不建议这么用，如下命令会造成风险。

```
>>>filename = input("input filename you want to review")
input filename you want to review
non_existFilename; rm -rf /
>>>subprocess.call("cat"+filename, shell=True) #删盘
```

# 4 Popen类
subprocess.call()实际上直接构建Popen类对象，然后调用Popen类对象的函数，创建并执行子进程，只是call隐藏了一些参数。

```
class subprocess.Popen(args, bufsize=0, executable=None, stdin=None, stdout=None, stderr=None, preexc_fn=None, close_fds=False, shell=False, cwd=None, env=None, universal_newlines=False, startupinfo=None, creationflags=0)
同等函数：
os.execvp()
```
### 1 args
当shell=True, 默认的shell解析器时/bin/sh，此时如果args为一个序列，args[1]及后面的数组元素将会被认为是命令参数，args[0]被认为是命令。

### 2 bufsize
0（默认值）表示输出不缓存，1表示缓存行，其他正数表示缓存字节数，负数表示系统默认（通常为全部缓存）。

### 3 executable
shell=False, executable指定了实际的可执行程序，但是args给出的程序将作为名称，用于ps这类程序来显示执行名称。（名义上是args，实际上是executable）
shell=True，executable替换/bin/sh

### 4 preexc_fn
值为一个可调用对象，会在实际执行程序之前调用。

### 5 close_fns
True，除了stdin/out/err，所有的文件描述符都会在程序执行前关闭。

### 6 cwd
程序执行前，会切换大cwd指定的目录，不能使用相对于将要执行程序的相对路径。

### 7 env
指定环境变量映射，其用于替换（注意是替换，不是与系统环境变量叠加）系统环境变量。

### 8 universal_newlines
行结束符以任何\\r， \\n， \\r\\n等结束。

### 9 startinfo
对应一个STARTUPINFO类对象。其会传递给CreateProcess函数。creationflags只用于windows。

# 5 Popen的异常
常被引起OSError异常，其包含child_traceback描述异常原因。
使用无效参数，会引发Popen抛出ValueError异常。
check_call()和check_output()引出CalledProcessError异常。

# 6 Popen中的公开成员
### 1 Popen.poll()
检测子进程完成，设置并返回returncode的值。
### 2 Popen.wait()
等待并设置、返回returncode的值。
### 3 Popen.communicate(input=None)
从stdin中读取数据，从stdout/stderr中输出数据。input参数为发送给子进程的字符串。
函数返回一个tuple：(stdoutdata, stderrdata)

如果希望发送给子进程数据，需要在创建Popen的时候指定stdin=PIPE,如果希望获得输出，则指定stdout=PIPE和stderr=PIPE。由于数据会缓存，所以不要用在大量数据中使用。

### 4 Popen.send_signal(signal)
向子进程发送信号

### 5 Popen.terminate()
其会向子进程发送SIGNTERM信号。

### 6 Popen.kill()
向子进程发送SIGNKILL信号。

### 7 成员变量
Popen.stdin.write, Popen.stdout.read, Popen.stderr.read不要用三个变量对应的read/write来交换数据。防止引起死机——利用系统缓存来进行pipe传输。
Popen.pid 子进程号，如果shell=true，则为/bin/sh的进程号。
Popen.returncode被wait(), poll()设置，间接被communicate()设置。如果为None，表示进程尚未结束。负数-N表示子进程被SIGNAL N终止。

# 7 重要举例
### 1 管道传输
```
output = `dmsg | grep hda`
可替换为：
p1 = Popen(['dmesg'], stdout=PIPE)
p2 = Popen(['grep', 'hda'], stdin = p1.stdout, stdout= PIPE)
p1.stdout.close() # 如果p2提前退出，p1可以接收到SIGPIPE信号
output = p2.communicate()[0]
```
