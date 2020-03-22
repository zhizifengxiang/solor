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

# 3 q-pointer
上面设计中，调用是单方向的——只有Widget调用WidgetPrivate。如果WidgetPrivate也需要访问Widget中接口、成员变量怎么办？

所以我们引入q-pointer。q-pointer是私有类WidgetPrivate，指向公共接口类Widget对象的指针。如下代码说明了使用q-pointer的设计。

现在，我们可以正式称，将公共接口暴露给用户的类为public class, 而给public class 提供具体实现的类为private class.

```
// WidgetPrivate.h
class Widget;
class WidgetPrivate {
    public:
        WidgetPrivate(Widget* q);
    public:
        Widget* m_qptr;

        String m_styleSheet;
        Rect m_geometry;
}
```

```
// Widget.h
class WidgetPrivate;
class Widget {
    public:
        Widget();
        Rect geometry() const;
    private:
        WidgetPrivate* m_dptr;
}
```

下面为public/private class在源程序文件中的实现。
```
// Widget.cpp
#include "WidgetPrivate.h"
#include "Widget.h"

Widget::Widget() : m_dptr(new WidgetPrivate(this)) {}

Rect Widget::geometry()
{
    if (m_dptr) {
        return m_dptr->m_geometry;
    }
    return Rect();
}
```
下面为WidgetPrivate的实现。需要注意，由于我们只是保存了一个指向Widget对象指针，所以没必要在WidgetPrivate.cpp中#include "Widget.h"。但是，如果需要调用Widget类的接口，则需要添加#include "Widget"。
```
// WidgetPrivate.cpp
#include "WidgetPrivate.h"

WidgetPrivate::WidgetPrivate(Widget* qptr) : m_qptr(qptr) {}
```
类似地，我们定义并实现了子类Label，并实现对应的private class。具体代码如下。

```
// LabelPrivate.h
class Label;
class LabelPrivate {
    public:
        LabelPrivate(Label* qptr);
    public:
        Label* m_qptr;
        String text;
};
```


```
// Label.h
class LabelPrivate;
class Label {
    public:
        Label();
        String text();
    private:
        LabelPrivate *m_dptr;
}
```
下面是Label的public/private class实现。同样地，由于LabelPrivate并没有调用Label中声明的接口，因此，我们不需要再LabelPrivate.cpp中#include “Label.h”

```
// LablePrivate.cpp
#include "LabelPrvate.h"

LabelPrivate::LabelPrivate(Label* qptr) : m_qptr(qptr) {}
```


```
// Label.cpp
#include "LabelPrivate.h"
#include "Label.h"

Label::Label() : m_dptr(new LabelPrivate(this)) {}

String Label::text()
{
    if (m_dptr) {
        return m_dptr->text;
    }
    return String();
}
```

# 4 多层类对象衍生问题及优化
上面我们加了q-pointer和d-pointer，使public class和private class互指，并使用private class隐藏了public class的实现细节。但是，有一个问题是，我们每创建一个public class对象，就会产生对应的private class对象。如果继承层次比较多，那么创建顶层派生类对象，将导致无数个private class 对象被创建。

具体点说，如果创建Label对象，因为Label继承自Widget，Label和Widget共享一个对象。但是，我们需要创建WidgetPrivate对象、LabelPrivate对象。这样，创建一个public class对象，就需要创建三个对象。进一步说，如果继承层次为n，如果创建一个顶层派生类对象，不仅要创建派生类对象本身，还要创建n个private class对象。这种创建/销毁对象的开销会严重效率。

所以，我们进一步考虑，是不是可以让private class，像public class一样，也有一套继承体系？下面，我们就实现这种想法。我们将private class和public class分化为两个继承体系，在继承体系中，每个类对应着自己的public class和private class。

```
// WidgetPrivate.h
class Widget;
class WidgetPrivate {
    public:
        WidgetPrivate(Widget *qptr);
    private:
        Widget* m_qptr;
}
```

```
// Widget.h
class WidgetPrivate;
class Widget {
    public:
        Widget();
    private：
        WidgetPrivate* m_dptr;
}
```

```
// WidgetPrivate.cpp
#include "WidgetPrivate.h"
WidgetPrivate::WidgetPrivate(Widget *qptr) : m_qptr(qptr) {}
```


```
// Widget.cpp
#include "WidgetPrivate.h"
#include "Widget.h"
Widget::Widget() : m_dptr(new WidgetPrivate(this)) {}
```

```
// LabelPrivate.h
#include "WidgetPrivate.h"
class LabelPrivate : public WidgetPrivate {
    public:
        LabelPrivate(Widget* qptr);
    public:
        // 因为基类WidgetPrivate包含了q-pointer，所以LabelPrivate无需再保存q-pointer信息
        String text;
}
```

```
// Label.h
#include "Widget.h"
class Label : public Widget {
    public:
        Label();
        // 因为Widget中包含了d-pointer，所以Label中无需再保存d-pointer信息
}
```


```
// LabelPrivate.cpp
#include "LabelPrivate.h"

LabelPrivate::LabelPrivate(Widget* qptr) : WidgetPrivate(qptr) {}
```

```
// Label.cpp
#include "Label.h"
#include "LabelPrivate.h"

Label::Label() : Widget(new LabelPrivate(this)) {}
```
在上面代码中，Label对象将自己的this指针传递给Widget，再经过Widget将指针传递给WidgetPrivate，即q-pointer由WidgetPrivate来维护——上层派生的public类指针传递到下层。同时，Widget在构造期间，保存LablePrivate对象指针到m_dptr中。此时，q-pointer和d-pointer都由最下层对应的基类来保存。而程序中只存在两个对象，分别是顶层派生的public class对象和private class对象。

现在，我们进一步考虑d-pointer在Qt中的应用。在Qt中，继承体系完全按照我们上面描述的方式设计实现。出了一些已知的，未来不会再改变的类，如QPoint, QRect等，一般来说，public class会继承类QObject，而对应private class会继承类QObjectPrivate.


# 5 基类指针访问派生类接口问题
在上面的设计中，我们将q-pointer和d-pointer都交由public class和private class继承体系中的最底层类来保管，而这两个指针的类型也是基类。
但是，当我们需要用q-pointer，或者d-pointer来访问派生类的公共接口是，就必须进行类型转换。如下面的代码所示。
```
void Label::setText(const String& text)
{
    m_dptr->m_text = text;
}
```
在上面代码中，m_dptr的类型是WidgetPrivate，WidgetPrivate中没有数据成员m_text，而其派生类LabelPrivate中包含数据成员m_text。因此，我们需要对指针进行类型转换。如下代码所示。
```
void Lable::setText(const String& text)
{
    LabelPrivate* dptr = reinterpreter_cast<LabelPrivate*>(m_dptr);
    if (dptr) {
        dptr->m_text = text;
    }
}
```
上面的类型转换不仅不太优雅，而且每次类型转换十分麻烦。所以，Qt在文件src/corelib/global/qglobal.h中定义两个宏来对q-pointer和d-pointer进行类型转换。具体定义如下。

```
/ global.h
#define Q_D(Class) Class##Private * const d = d_func()
#define Q_Q(Class) Class *const q = q_func()
```
下面代码展示如何使用两个宏函数。
```
// label.cpp
void Label::setText(const String &text) 
{
    Q_D(Label);
    d->text = text;
}

void LabelPrivate::someHelperFunction()
{
    Q_Q(Label);
    q->selectAll();
}
```
我们可以注意到，上面定义的宏Q_D和Q_Q分别定义了两个函数指向的指针。这两个函数d_func()和q_func()由定义在qglobal.h的宏Q_DECLARE_PRIVATE和Q_DECLARE_PUBLIC来实现。其中Q_DECLARE_PRIVATE宏定义了d_func()的实现，具体如下所示。
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
这样，我们就可以像下面代码一样，直接用宏来定义在public class中的private class指针了。

需要注意的是，上面不仅声明了d_func()函数，同时还声明了Class##Private为友元类。因为d_func()通常作为public class的私有成员函数，为了在使类型转换可以访问，所以声明对应的private class为友元类。

```
// qlabel.h
class QLable {
private:
    Q_DECLARE_PRIVATE(QLabel);
};
```

# 附录
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
        return a.m_bAccess; // 类B中可以直接访问A的私有成员
    }
}
```
