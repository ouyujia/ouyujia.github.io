---
title: final和override
tags:
  - C++11新特性
  - 类继承
categories:
  - - C++
    - C++11新特性
description: C++11新特性，final和override
cover: 'https://cdn.jsdelivr.net/gh/ouyujia/blogImg/img/202204271635488.jpg'
abbrlink: 7146
date: 2022-04-27 16:32:22
updated:
keywords:
top_img:
comments:
toc:
toc_number:
toc_style_simple:
copyright:
copyright_author:
copyright_author_href:
copyright_url:
copyright_info:
mathjax:
katex:
aplayer:
highlight_shrink:
aside:
---

### final

C++ 中增加了 <font color='orange'>`final` </font>关键字来<font color='red'>`限制某个类不能被继承，或者某个虚函数不能被重写`</font>，和 Jave 的 <font color='orange'>`final`</font> 关键字的功能是类似的。如果使用 <font color='orange'>`final` </font>修饰函数，<font color='red'>`只能修饰虚函数`</font>，并且<font color='red'>`要把final关键字放到类或者函数的后面。`</font>

#### 修饰函数

如果使用 <font color='orange'>`final` </font>修饰函数，<font color='red'>只能修饰虚函数，这样就能阻止子类重写父类的这个函数了</font>：

```C++
class Base
{
public:
    virtual void test()
    {
        cout << "Base class...";
    }
};

class Child : public Base
{
public:
    void test() final
    {
        cout << "Child class...";
    }
};

class GrandChild : public Child
{
public:
    // 语法错误, 不允许重写
    void test()
    {
        cout << "GrandChild class...";
    }
};   
```

在上面的代码中一共有三个类：

- 基类：Base
- 子类：Child
- 孙子类：GrandChild

<font color='orange'>`test()` </font>是基类中的一个虚函数，在子类中重写了这个方法，但是不希望孙子类中继续重写这个方法了，因此在子类中将 <font color='orange'>`test()` </font>方法标记为 <font color='orange'>`final`</font>，孙子类中对这个方法就只有使用的份了。

#### 修饰类

使用 <font color='orange'>`final` </font>关键字修饰过的类是不允许被继承的，也就是说这个类不能有派生类。

```C++
class Base
{
public:
    virtual void test()
    {
        cout << "Base class...";
    }
};

class Child final: public Base
{
public:
    void test()
    {
        cout << "Child class...";
    }
};

// error, 语法错误
class GrandChild : public Child
{
public:
};
```

<font color='orange'>`Child` </font>类是被 <font color='orange'>final </font>修饰过的，因此 <font color='orange'>Child </font>类不允许有派生类 <font color='orange'>GrandChild </font>类的继承是非法的，<font color='orange'>Child </font>是个断子绝孙的类。

### override

<font color='orange'>override </font>关键字确保在派生类中声明的重写函数与基类的虚函数有相同的签名，同时也明确表明将会重写基类的虚函数，这样就可以保证重写的虚函数的正确性，也提高了代码的可读性，和 final 一样<font color='orange'>这个关键字要写到方法的后面。</font>使用方法如下：

```C++
class Base
{
public:
    virtual void test()
    {
        cout << "Base class...";
    }
};

class Child : public Base
{
public:
    void test() override
    {
        cout << "Child class...";
    }
};

class GrandChild : public Child
{
public:
    void test() override
    {
        cout << "Child class...";
    }
};
```

上述代码中第 13 行和第 22 行就是显示指定了要重写父类的<font color='orange'> test()</font> 方法，使用了<font color='orange'> override </font>关键字之后，假设在重写过程中因为误操作，写错了函数名或者函数参数或者返回值编译器都会提示语法错误，提高了程序的正确性，降低了出错的概率。

### 参考资料

[final 和 override | 爱编程的大丙 (subingwen.cn)](https://subingwen.cn/cpp/final/)

