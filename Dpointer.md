# D-Pointer

## 1， What is the d-pointer

Qt源代码中可以看到Q_D和Q_Q宏，两个宏是被称作d-pointer的设计模式的一部分，该设计模式使库的具体实现向用户隐藏，从而保证了库内部实现的改变，不会影响到binary的兼容性。

## 2，Binary compatibility — what is that?

在设计类似Qt库的时候，最好能够让动态链接Qt的软件，即使在库升级或替换的情况下，依然能够正常运行，而无需重新编译。例如，软件CuteApp给予Qt4.5，但是在将Qt4.5升级到Qt4.6后，使用Qt4.5编译的CuteApp依然能够正常运行。

### 2.1 What breaks binary compatibility?

下面来看一个简单例子，说明在什么情况下，更换库后，需要编译。

```
class Widget {
    // ....
private:
    Rect m_geometry;
};

class Label : public Widget {
public:
    //...
    String text() const {
        return m_text;
    }
private:
    String m_text;

};
```
上面，我们定义了包含了成员变量geometry的基类Widget，将该类编译到WidgetLib 1.0。有人想到可以在Widget中添加支持stylesheet的功能，这样就生成了WidgetLib 1.1。下面为加入新功能后的代码：

```
class Widget {
    // ....
private:
    Rect m_geometry;
    String m_stylesheet; // new in WidgetLib 1.1
};

class Label : public Widget {
public:
    // ...
    String text() const {
        return m_text;
    }
private:
    String m_text;
};
```
经过上面的修改后，我们的应用CuteApp无法运行。

### 2.2 Why did it crash?

原因在于，加入新的数据导致Widget和Label对象的大小发生改变，C++编译器在生成代码时，其使用"offsets"来访问对象（object）中的数据。下面大致展示了上面POD对象在内存中的样子：

        Label object layout in WidgetLib 1.0   |  Label object layout in WidgetLib 1.1
                m_geometry <offset 0>          |     m_geometry <offset 0>
                - - -                          |     m_stylesheet <offset 1>
                m_text <offset 1>              |     - - -
                - - -                          |     m_text <offset 2>
                
在WidgetLib 1.0中，Label对象中m_text成员变量逻辑上偏移量为1，与Label::text()对应的代码访问偏移量1。但是在WidgetLib 1.1中，m_text的偏移量变成了2，由于CuteApp程序没有重新编译，其仍然认为m_text的偏移量为1，实际上偏移量为1的位置为m_stylesheet。

产生这种问题的原因在于Label::text()定义于头文件，因此编译器将该函数内联起来。即便将Label::text()的定义移动到源文件，也无济于事，因为无论运行时，还是编译阶段，C++都依赖于对象的大小。编译时，编译器生成在栈中分配空间的代码，此时就已经为Label分配了空间。由于两个版本分配的空间不一致，因此Label的构造函数覆盖了已有数据，并产生corruption。

### 2.3 Never change the size of an exported C++ class

总之，在发布库的时候，不要改变C++类的大小，或者数据的排布（即数据声明的先后顺序），以及数据的可见性。C++编译器认为对象的大小以及对象内数据的排布顺序在客户程序编译后是不会发生变化的。

那么如何在不改变对象大小的情况下添加新功能？

## 3，The d-pointer

为了保证库中公有类的大小一致，可以只存储一个类对象的指针，该指针指向一个private/internal的数据结构，被指向的数据结构包含了所有的数据。被指向的数据结构尺寸无论变大变小，都不会对客户程序有任何影响，因为内部的指针仅由库代码访问，而暴露给客户程序的接口类的大小始终没有变化——其总是指针的大小。这个指针就被称作d-pointer。

下面的代码揭示了该模式的核心（本篇中的所有代码都没有析构函数，应用中需自己添加）。
```
/**
由于d_ptr为一个指针，且用于不会在头文件中访问实体（referended）（会造成编译错误），因此WidgetPrivate不必包含在头文件中，只需前置声明即可。WidgetPrivate类定义于widget.cpp中，或者一个单独的文件，比如：widget_p.h中。
*/
class WidgetPrivate;
class Widget {
public:
    Rect geometry() const;
private:
    WidgetPrivate *d_ptr;

};

```
下面是widget_p.h的定义，为Widget类的私有头文件
```
struct WidgetPrivate {
    Rect geometry;
    String stylesheet;
};

// widget.cpp
#include "widget_p.h"

Widget::Widget() : d_ptr(new WidgetPrivate) {
    // creation of private data
}

Rect Widget::geometry() const {
    // the d_ptr is only accessed in the library code
    return d_ptr->geometry;
}
```

下面定义Widget类的子类Label：
```
class Label: public Widget {
    String text();
private:
    // each class maintains its own d_pointer
    LabelPrivate *d_ptr;
};

// label.cpp
// unlike WidgetPrivate, the customer decides LabelPrivate
struct LabelPrivate {
    String text;
};

Label::Label() : d_ptr(new LabelPrivate) {

}
String Label::text() {
    return d_ptr->text;
}
```
上面设计使得CuteApp永远不会直接访问d_ptr(即d-pointer)，因为d-pointer仅有WidgetLib内部的类进行访问，而每次版本发布都会重新编译WidgetLib，因此Private类可以随意改动而不会影响到客户程序CuteApp。

### 3.1 other benefits of d-pointer
d-pointer不仅保证了binary的兼容性，还有以下好处：
1， 隐藏实现细节——只需要头文件和二进制执行包，源文件无需提供。
2， 头文件不包含实现细节，可作为API参考。
3， 定义从头文件移到源文件，加速编译。
实际上，主要原因还是在于binary compatible和Qt开始闭源代码。
## 4, the q-pointer
上面例子只是给出了对外接口Label和Widget包裹对应私有类的设计模式，通过调用私有类（如LabelPrivate和WidgetPrivate）的方法（helper function），向客户程序提供处理服务。实际实现时，Private中的方法有时需要从对外接口类处获取客户程序的信息，因此需要调用外层服务接口类的public function，所以WidgetPrivate需要通过存储一个被称为q-pointer的指针，来对外部的接口类进行引用。向上面例子中添加q-pointer的代码如下:
```
// widget.h
class WidgetPrivate;
class Widget {
    Rect geometry() const;
private :
    WidgetPrivate *d_ptr;

};

// widget_p.h
struct WidgetPrivate {
    // constructor initialaizing the q_ptr
    WidgetPrivate(Widget *q) : q_ptr(q) {}
    Widget *q_ptr; // q_ptr points to the API class
    Rect geometry;
    String stylesheet;
};

// widget.cpp
// create private data, pass the "this" pointer to initialize the q_ptr
Widget::Widget() : d_ptr(new WidgetPrivate(this)) {

}

Rect Widget::geometry() const {
    // the d_ptr is only accessed in library code
    return d_ptr->geometry;
}
```
继承自Widget的类Label定义如下：
```
// label.h
clss Lable: public Widget {
    String text() const;
};

//label.cpp
// unlike WidgetPrivate, the customer decides LabelPrivate
struct LabelPrivate {
    LablePrivate(Label *q) : q_ptr(q) {}
    Label *q_ptr;
    String text;
};

Lable::Label() : d_ptr(new LablePrivate(this)) {

}

String Label::text() {
    return d_ptr->text;
}
```
## 5, inheriting d-pointers for optimization

在上面代码中，创建Label类对象需要为LabelPrivate和WidgetPrivate类分配空间，若Qt采用此策略，情况会十分糟糕。比如QListWidget类，需要6层继承才能创建一个类对象，相应地，需要6次空间分配操作。通过向私有类采用层次式继承关系（inheritance hierarchy），以及向类传递d-pointer来实例化类，我们可以解决这个问题。

需要注意，当继承d-pointer时，私有类（Private class）需要声明在一个单独文件中，如声明在widget_p.h中，而不能声明在源文件widget.cpp中。

```
// widget.h
class Widget {
public:
    Widget();
protected:
    // only subclass can access below code
    // alow subclass to initialize with their own concrete Private
    Widget(WidgetPrivate &d);
    WidgetPrivate *d_ptr;
};

// widget_p.h
struct WidgetPrivate {
    WidgetPrivate(Widget *p) : q_ptr(q) {} // constructor initializing the q_ptr
    Widget *q_ptr; // q_ptr pointing to the API class
    Rect geometry;
    String stylesheet;
};

// widget.cpp
Widget::Widget() : d_ptr(new WidgetPrivate(this)) {

}

Widget::Widget(WidgetPrivate &d) : d_ptr(&d) {

}

// label.h
class Label : public Widget {
public:
    Label();
protected:
    Label(LabelPrivate &d); // allow Lable subclasses to pass on their Private
    // notice how Label doesen't have a d_ptr ! It just uses Widget's d_ptr
};

// label.cpp
class LabelPrivate : public WidgetPrivate {
public:
    String text;
};
Label::Label() : Widget(* new LabelPrivate) { // initialize the d_pointer with our-own-Private
}

Label::Label(LabelPrivate &d) : Widget(d) {
}

```
在上面代码中，当创建出Label类对象后，其会自动创建LabelPrivate对象（继承自WidgetPrivate）。Label会将d-pointer对象实体传递给Widget类的protected构造函数。所以，创建Label对象只会进行一次内存分配，而Label也提供了protected构造函数，该函数可被子类用来提供他们自己的Private class。
## 6， d-pointer in Qt

### 6.1 Q_D and Q_Q

### 6.2 Q_DECLARE_PRIVATE and Q_DECLARE_PUBLIC























