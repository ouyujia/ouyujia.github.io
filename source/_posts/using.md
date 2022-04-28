---
title: using
tags:
  - C++11新特性
categories:
  - - C++
    - C++11新特性
description: C++11新特性，using
cover: 'https://cdn.jsdelivr.net/gh/ouyujia/blogImg/img/202204271704943.jpg'
abbrlink: 55596
date: 2022-04-27 16:59:44
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

在 C++ 中 using 用于声明命名空间，使用命名空间也可以防止命名冲突。在程序中声明了命名空间之后，就可以直接使用命名空间中的定义的类了。在 C++11 中赋予了 <font color='orange'>using </font>新的功能，让 C++ 变得更年轻，更灵活。

#### 定义别名

在 C++ 中可以通过 <font color='orange'>typedef </font>重定义一个类型，语法格式如下：

```C++
typedef 旧的类型名 新的类型名;
// 使用举例
typedef unsigned int uint_t;
```

<font color='red'>`被重定义的类型并不是一个新的类型，仅仅只是原有的类型取了一个新的名字`</font>。和以前的声明语句一样，这里的声明符也可以包含类型修饰，从而也能由基本数据类型构造出复合类型来。C++11 中规定了一种新的方法，使用别名声明 (alias declaration) 来定义类型的别名，即使用 <font color='orange'>`using`</font>。

在使用的时候，关键字 using 作为别名声明的开始，其后紧跟别名和等号，其作用是把等号左侧的名字规定成等号右侧类型的别名。<font color='orange'>`类型别名和类型的名字等价，只要是类型的名字能出现的地方，就能使用类型别名。使用typedef定义的别名和使用using定义的别名在语义上是等效的。`</font>

使用 using 定义别名的语法格式是这样的：

```C++
using 新的类型 = 旧的类型;
// 使用举例
using uint_t = int;
```

通过 using 和 typedef 的语法格式可以看到二者的使用没有太大的区别，假设我们定义一个函数指针，using 的优势就能凸显出来了，看一下下面的例子：

```C++
// 使用typedef定义函数指针
typedef int(*func_ptr)(int, double);

// 使用using定义函数指针
using func_ptr1 = int(*)(int, double);
```

如果不是特别熟悉函数指针与 typedef，第一眼很难看出 func_ptr 其实是一个别名，其本质是一个函数指针，指向的函数返回类型是 int，函数参数有两个分别是 int，double 类型。

使用 using 定义函数指针别名的写法看起来就非常直观了，<font color='orange'>`把别名的名字强制分离到了左边，而把别名对应的实际类型放在了右边`</font>，比较清晰，可读性比较好。

#### 模板的别名

使用 typedef 重定义类似很方便，但是它有一点限制，比如无法重定义一个模板，比如我们需要一个固定以 int 类型为 key 的 map，它可以和很多类型的 value 值进行映射，如果使用 typedef 这样直接定义就非常麻烦:

```C++
typedef map<int, string> m1;
typedef map<int, int> m2;
typedef map<int, double> m3;
```

在这种情况下我们就不自觉的想到了模板：

```C++
template <typename T>
typedef map<int, T> type;	// error, 语法错误
```

使用 typename 不支持给模板定义别名，这个简单的需求仅通过 typedef 很难办到，需要添加一个外敷类：

```C++
#include <iostream>
#include <functional>
#include <map>
using namespace std;

template <typename T>
// 定义外敷类
struct MyMap
{
    typedef map<int, T> type;
};

int main(void)
{
    MyMap<string>::type m;
    m.insert(make_pair(1, "luffy"));
    m.insert(make_pair(2, "ace"));

    MyMap<int>::type m1;
    m1.insert(1, 100);
    m1.insert(2, 200);

    return 0;
}
```

通过上边的例子可以直观的感觉到，需求简单但是实现起来并不容易。**在 C++11 中，新增了一个特性就是可以通过使用 using 来为一个模板定义别名**，对于上面的需求可以写成这样：

```C++
template <typename T>
using mymap = map<int, T>;
```

完整的示例代码如下:

```C++
#include <iostream>
#include <functional>
#include <map>
using namespace std;

template <typename T>
using mymap = map<int, T>;

int main(void)
{
    // map的value指定为string类型
    mymap<string> m;
    m.insert(make_pair(1, "luffy"));
    m.insert(make_pair(2, "ace"));

    // map的value指定为int类型
    mymap<int> m1;
    m1.insert(1, 100);
    m1.insert(2, 200);

    return 0;
}
```

上面的例子中通过使用 using 给模板指定别名，就可以基于别名非常方便的给 value 指定相应的类型，这样使编写的程序变得更加灵活，看起来也更加简洁一些。

<font color='red'>`最后在强调一点：using 语法和 typedef 一样，并不会创建出新的类型，它们只是给某些类型定义了新的别名。using 相较于 typedef 的优势在于定义函数指针别名时看起来更加直观，并且可以给模板定义别名。`</font>


