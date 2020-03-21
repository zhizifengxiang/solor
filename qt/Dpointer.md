# 前言
D-pointer设计模式，使得代码中，调用接口和具体实现分离，从而保证，只要类的公共接口签名不发生变化，即公共接口的传入参数不发生变化，我们就不必重新编译上层调用库。

本篇文章将通过一个简单的例子，简要介绍Qt中的D-pointer，以及相关的宏Q_D和Q_Q.


# 1 兼容问题

现在，我们设计一个程序cuteApp，其依赖Qt4.5版本。 在后期维护中，我们希望将依赖的Qt库升级到4.7版本。假定cuteApp调用的Qt接口没有变化，所以我们只需要重新编译Qt库的代码。

显然，上面的做法效率更高。我们无需重新编译cuteApp程序，只需让cuteApp动态链接Qt库，每次更换Qt版本，只编译Qt库，cuteApp也可以正确运行。我们称此时Qt和cuteApp是兼容的。

如下代码，类Label是Widget的派生类。

```
// Widget.h
class Widget {
  public:
    // ...
  private:
    Rect m_geometry;
}
```
```
// Label.h
class Label : public Widget {
  public:
    String text() const { return m_text;}
  private:
    String m_text;
}
```
现在，我们扩展这个程序——在Widget中添加新的成员变量m_styleSheet。
```
// Widget.h
class Widget {
  public:
    // ...
  private:
    String m_styleSheet; // 新添加的变量
    Rect m_geometry;
}
```
如果只是重新编译Widget.h文件，而不重新编译Label.h文件，程序无法运行。

这是因为加入新的数据后，Widget的大小发生变化，m_geometry在Widget对象中的位置也发生了变化。如下图所示。

同时，Label::text()定义在类Label的声明中，即text()作为内联函数，因此，修改Widget后，访问text()函数也会使程序异常退出。

|变量名称（前）|变量位置（前）|变量名称（后）|变量位置（后）|
|---|---|---|---|
|m_geometry|0x0000|m_styleSheet|0x0000|
|xxxx|xxxx|m_geometry|0x0010|


综上，如果我们打算改变Widget的成员变量，就必须将所有依赖Widget的类重新编译。那么是否可以既改变底层类Widget的实现，又不必编译上层的派生类呢？即，将实现与接口解耦。

下面我们来介绍d-pointer设计模式。

# 2 d-pointer
上面问题的实质是，因为修改类定义，导致类大小变化，从而使上层依赖无法按照原来的地址，访问下层的数据结构。因此，我们只需要在扩展下层类的时候，不改变类的大小，和数据成员的内存布局即可。

利用指针可以实现这一点：指针实际上是一个存储内存地址的整数。如下代码，我们将类的具体实现封装到WidgetPrivate中，并在Widget中添加指向WidgetPrivate对象的指针，此时，我们称这个指针为“D-pointer”。

```
// WidgetPrivate.h
class WidgetPrivate {
  public:
    Rect geometry() const;
  private:
    Rect m_geometry;
}
```

```
// Widget.h
class WidgetPrivate;
class Widget {
  public:
    Widget();
    ~Widget();
    Rect geometry() const;
  private:
    WidgetPrivate* m_dptr;
}
```
下面是Widget和WidgetPrivate的实现。
```
// WidgetPrivate.cpp
#include "WidgetPrivate.h"

WidgetPrivate::geometry() { return m_geometry; }
```
```
// Widget.cpp
#include "WidgetPrivate.h"
#include "Widget.h"

Widget::Widget(): m_dptr(new WidgetPrivate) {}
Widget::~Widget()
{
    delete m_dptr;
    m_dpte = NULL;
}
Rect Widget::geometry()
{
    if (m_dptr) {
        return m_geometry;
    }
    return Rect();
}
```
通过上面的代码，只要Widget的public接口不发生变化，就无需重新编译子类，从而提高了开发效率。下面，与上面相同的套路，我们继续定义子类Label。

```
// LabelPrivate.h
class LabelPrivate {
    public:
        String m_text;
}
```
```
// Label.h
#include "Widget"
class LabelPrivate;
class Label : public Widget {
    public:
        Label();
        ~Label();
        String text();
    private:
        LabelPrivate* m_dptr;
}
```
下面是类Label对应的源文件。
```
// Label.cpp
#include "LablePrivate.h"
#include "Lable.h"

Label::Label() : m_dptr(new LablePrivate) {}
Label::~Label() 
{
    delete m_dptr;
    m_dptr = NULL;
}
String Lable::text()
{
    if (m_dptr) {
        return m_dptr->m_text;
    }
    return String();
}
```

由于所有的d-pointer都被定义成private成员变量，因此我们永远不知道，也不必知道Private类到底做了什么，我们只需要只需要知道暴露出来的public接口。因此，d-pointer给我们带来如下好处：
1. 我们只需要提供头文件和库文件，告诉用户可以调用哪些接口，而无需提供源文件。
2. 修改底层实现代码，不会影响上层函数调用，解耦了上下层，提高编译速度，有更好的用户体验，每个库都是“即插即用”。
3. 分离实现与接口，代码更容易维护。

todo
# 3 q-pointer
现在让我们进一步思考。


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
在Qt 中，全部public class都是用d-pointer方法，唯一不适用该方法的情况是，已经预先知道该类，永远不会在未来版本中添加新的成员变量。比如，QPoint，QRect类，这两个类不会再添加新的成员变量，因此，他们的成员变量直接放入public class中，而不使用d-pointer。注意，根据上面的方法，所有的Private class都继承自QObjectPrivate。

### 6.1 Q_D and Q_Q
上面经过改进的方法有一个副作用是，我们将q-ptr和d-ptr分别作为Widget和WidgetPrivate类的指针，这就意味着下面的方法将会出问题：
```
void Label::setText(const String &text) {
// won't work, since d_ptr is of type WidgetPrivate
// even though it points to LabelPrivate object
// 即由于指针d_ptr指向基类WidgetPrivate ，因此无法访问变量LabelPrivate中的成员变量text
d_ptr->text = text; // 会产生错误
}
```
因此，当需要使用d-pointer来访问派生类时，我们需要static_cast来转换成适当的类型：
```
void Label::setText(const String &text) {
    LabelPrivate *d = static_cast<LabelPrivate*>(d_ptr);
    d->text = text;
}
```
使用static_cast 不够优雅，也比较麻烦，因此我们使用两个定义在src/corelib/global/qglobal.h中的宏，他们可以让操作更直接简便：
```
// global.h
#define Q_D(Class) Class##Private * const d = d_func()
#define Q_Q(Class) Class *const q = q_func()

// label.cpp
// with Q_D, you can use the members of LabelPrivate from Label
void Label::setText(const String &text) {
    Q_D(Label);
    d->text = text;
}

// with Q_Q, you can use the member of Label from LabelPrivate
void LabelPrivate::someHelperFunction() {
    Q_Q(Label);
    q->selectAll();
}
```
### 6.2 Q_DECLARE_PRIVATE and Q_DECLARE_PUBLIC

Q_DECLARE_PRIVATE宏声明于qglobal.h文件中：
```
#define Q_DECLARE_PRIVATE(Class) \
    inline Class##Private *d_func() { \
    return reinterpret_cast<Class##Private*>(qGetPtrHelper(d_ptr)); \
} \
inline const Class##Private d_func() const { \
    return reinterpret_cast<const Class##Private *>(qGetPtrHelper(d_ptr)); \
} \
friend class Class##Private;
```

上面的宏可以下面的方式使用：

```
// qlabel.h
class QLable {
private:
    Q_DECLARE_PRIVATE(QLabel);
};
```
该方法实际上为QLabel提供了一个函数d_func()，该函数允许访问其private internel class。由于宏被包含在private修饰符内，因此函数本身是私有的，但是可以被QLabel的友元类（friend class）来调用。该方法对于那些无法通过QLabel公有接口来访问的属性来说，特别有用。举一个不太恰当的例子，QLabel可能会跟踪用户单击了多少次链接，但是，并没有public API可以访问到这个信息。QStatictics就是这样一个需要此类信息的类，那么开发者就将QStatictics作为QLabel的友元类，这样QStatictics类型就可以这样做：
> label->d_func()->linkClickCount

d_func还有个好处是可以强制进行const-conrrectness：比如在类MyClass的一个const 成员函数中，你需要Q_D（const MyClass），这样你就只能在MyClassPrivate中调用const 成员函数了。对于没有const修饰符的d_ptr，你仍然可以调用非const函数。

本篇完。

## 附录
友元函数：存在两个类A和B，其中B需要访问一些A的私有变量，则：
```
class A {
private :
    int m_bAccess = 1;
    frind class B;
}
class B {
public:
    int returnA {
        A a;
        return a.m_bAccess;
    }
}
```
