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

subprocess.PIPE : 指定stdin/out/err可使用管道通信。例子如下：
