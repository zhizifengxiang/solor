# Basics of plugin

## plugin overview
plugin为程序的一种扩展方式，比如，我有一个计算器程序，可以通过加载插件，使其支持列表处理。即插件允许第三方开发者对计算器程序进行扩展，而不需要访问计算器程序本身。——计算器只是调用接口，具体实现交由第三方开发者来进行

## create plugins
C语言按照以下步骤创建插件：

>首先为插件定义接口，这些接口由插件程序给出具体实现。
>插件实现接口，并编译成shared object
>application发现插件，动态加载插件，解析插件中的symbols/functions，并调用插件的具体定义。

首先定义插件的接口：
```
// interface.h
double operation(double arg1, double arg2);
typedef double(*operation_pointer)(double, double);
```

给出接口的具体实现：
```
// addtion.c
#include "interface.h"
double operation(double arg1, double arg2) {
    return arg1 + arg2;
}
```
将插件编译成shared object: 

>gcc -shared -fPIC addition.c -o addition.so

下面代码使用定义的插件。在程序运行时，其会自动搜索之前定义的插件：
```
// application.c
#include <dlfcn.h> // linux support header-file
#include "interface.h"

int main() {
    const char *plugin_path = "/home/nick_rhan/pluginTest/addition.so"; // any arbitrary plugin supporting interface.h
    void *plugin = dlopen(plugin_path, RTLD_LAZY);
    operation_pointer ptr = (operation_pointer)dlsym(plugin, "operation"); //get pointer to "operation" function
                                                                           // "operation"为需要获取的函数名
    double ret = ptr(20, 10);
    printf("%f\n", ret);
    return 0;
}
```

## export symbols in C
shared object中有些函数不需要暴露给外部程序，在gcc中，可以使用下面的前缀来控制function的可见性：
> gcc中： __attribute__((visibility("default")))

因此，插件可以写成如下形式：
```
// addition.c
#include "interface.h"
__attribute__((visibility("default"))) double operation(double arg1, double arg2) {
    return arg1 + arg2;
}
```

## export symbols in C++
在C中，函数签名完全可以保证符号正确解析，但是对于c++，由于重载，重写等多态功能，导致不同编译器对C++的符号修饰不一样，即对根据namespace, class的等对symbol进行进一步修饰，形成最后的函数签名，我们称作name mangle。所以：
> 上面调用dlsym解析symbol "operation"，如果使用C++编译器则会失败。程序需要解析mangled name。

由于mangled name依赖于不同的C++编译器，因此不同编译器解析的插件实现，无法互相通用。解决的办法是，在C++中定义接口，然后使用一个C函数返回一个指向接口对象的指针。该C函数使用C-linkage进行编译（specified using extern "C"）。下面的代码展现了使用C++实现插件的技术：

```
// interface.h
class Interface {
    public:
        virtual double operation(double arg1, double arg2)'
};

extern "C" __attribute__((visibility("default"))) Interface *getInterface();
typedef Interface *(*GetInterFacePointer)(); // define a function returning object Interface
```

```
// addition.cpp
#include "interface.h"

class AdditionOperator : public Interface {
    public :
        double operation(double arg1, double arg2) {
            return arg1 + arg2;
        }
;
extern "C" Interface *getInterface() {return new AdditionOperator;}
```

```
// application.cpp
#include "interface.h"
#include <dlfcn.h>

int main() {
    void *plugin = dlopen(plugin_path, RTLD_LAZY);
    GetInterfacePointer getInterFacePointer = (GetInterfacePointer)dlsym(plugin, "getInterface"); // resolve the "C" function
    Interface *interface = getInterfacePointer(); // use C function to get C++ interface
    interface->operation(10, 20);
```
