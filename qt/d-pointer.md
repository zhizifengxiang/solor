# 综述
本篇将介绍什么是d-pointer, 以及相关的q-pointer，宏Q_D和Q_Q等三个变量。
# 1 程序兼容性
### 1 提出问题
对于大型软件，我们会使用动态链接的方式，即程序运行时，会将需要的程序文件，分批次导入到内存中，而不是一次性全部将程序导入到内存。这种做法提高内存的利用率。

在编译时，我们也分模块编译。下面例子中，Label是Widget的子类，其依赖于Widget类.
```
class Widget {
  private:
    int m_id;
}

class Label : public Widget {
  public:
    std::string text() const { return m_text;}
  private:
    std::string m_text;
}
```
我们称上面代码为版本0.0，称下面代码为版本1.0。1.0代码对Widget做如下扩展，Label类定义不变。
```
class Widget {
  private:
    int         m_id;
    std::string m_name;
}
```

如果父类Widget和子类Label分别编译。版本0.0程序编译通过，运行正确。我们在版本1.0中，只修改Widget定义，因此只编译Widget对应的模块。此时，编译通过，但是运行出问题（crash）。

### 2 分析问题
1.0版本运行出错，是因为Widget被重新定义。Label通过偏移量来访问变量，现在变量在Widget中的位置发生变化，因此Widget重新编译后，程序运行错误。
