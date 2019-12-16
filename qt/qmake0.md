# 1 初阶
qmake工程文件扩展名为".pro"，qmake程序根据描述语句，构建工程，在Unix系统中，生成makefile文件，再进一步进行编译。

使用关键字“SOURCES”添加源文件：
```
SOURCES += a.cpp b.cpp c.pp
或者
SOURCES += a.cpp \
          b.cpp
或者
SOURCES = a.cpp b.cpp
```

使用关键字"HEADERS"添加头文件：
```
HEADERS += a.h b.h
```

使用关键字“TARGET”定义生成的目标文件。因为每个工程只有一个程序入口，所以TARGET只能有一个值。
```
TARGET = helloworld
```

使用关键字"CONFIG"添加库。qt表示使用qt库，debug表示程序中需要使用qDebug()输出调试信息。
```
CONFIG += qt debug
```
如果希望项目能在unix和windows平台上同时兼容，则需要判断平台：
```
win32 {
  SOURCES += winHello.cpp
}
unix {
  SOURCE += unixHello.cpp
}
```
如果发现文件不存在，停止生成makefile：
```
!exists(main.cpp) {
  error("no main.cpp file found")
}
```

希望当"CONFIG"中添加debug，则链接console库。可以：
```
win32{
  debug {
    CONFIG += console
  }
}
或者
win32 : debug {
  CONFIG += console
}
```

# 2 qmake common project
qmake可以构建三种类型的项目，这些项目与平台无关（macos, window, unix）。分别是application（应用程序）、library（库）和plugin（插件）。

#### 1 app template
应用程序分"window(窗口)"和"console(控制台)"两种类型。下面值赋值给"CONFIG"变量来控制生成的app类型，来添加对应的库。
```
window: GUI application
console: 仅app template拥有， 为Windows的控制台程序。
```

当生成app template的使用，下面变量可以使用.有些变量只能为一个值，比如TARGET，所以用"="。有些变量可以有多值，所以用"+="。"="会抹除前面的变量赋值。

|变量名|说明|
|--|--|
|HEADERS|头文件列表|
|SOURCES|源文件列表|
|FORMS|UI文件列表|
|LEXSOURCES|lex源文件列表|
|YACCSOURCES|yacc源文件列表|
|TARGET|目标文件，默认为项目名称，文件扩展名自动添加|
|DESTDIR|目标可执行文件的目录|
|DEFINES|预处理器需要查看的宏|
|INCLUDEPATH|额外需要搜索的头文件目录|
|DEPENDPATH|额外搜索的依赖路径|
|VPATH|寻找支持我文件的路径|
|DEF_FILE|窗口应用：连接到.def文件|
|RC_FILE|窗口应用：资源文件|
|RES_FILE|窗口应用：连接到应用的资源文件|

#### 2 library
库类型的目标需要将以下值赋给CONFIG。另外，除了以上定义的变量，库类型新加了"VERSION"变量，比如“VERSION = 1.0.0”
```
dll: 生成shared library
staticlib: 生成static library
plugin:生成插件
```

生成的目标文件名依平台而变化。比如x11或者macos系统，库名前加lib。window不会有前后缀。

### 3 plugin
CONFIG=plugin就可以生成插件目标了。下面演示创建designer plugin的简便方法。
```
CONFIG += designer plugin
```

# 3 其他说明
#### 1 debug & release
我们可以设置CONFIG的值为：debug，或者release。当两者同时设置时，debug会覆盖release的目标。

如果希望同时构建两种版本，则使用debug_and_release值给CONFIG.需要注意，两种模式名字不能相同。然后我们就可以使用"make all"命令，同时生成两个目表。
```
CONFIG += debug_and_release
CONFIG(debug, debug | release) {
  TARGET += debug_binary
} else {
  TARGET += release_binary
}
```
更简便的方式是：
```
CONFIG += build_all
$ make # shell命令
```
"build_all"选项保证两种文件也会被同时安装上：
```
make install
```

#### 2 平台相关
构建lib或者plugin，我们可以根据平台来命名。relase按照默认命名即可。
```
CONFIG (debug, debug | release) {
  mac : TARGET = $$join(TARGET, , , _debug)
  win32: TARGET = $$join(TARGET, , d)
}
```
