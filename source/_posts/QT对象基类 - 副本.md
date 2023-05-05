---
title: QT信号和槽解析
abbrlink: 51714
date: 2023-05-05 19:50:59
updated:
tags:
 - QT
categories:
 - QT
keywords:
description:
top_img:
comments:
cover:
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

## 机制

- Qt使用信号与槽机制进行对象间的通信，当某个对象的状态发生变化时，该对象会触发一个信号，该信号和另外一些对象的槽函数绑定，信号的触发将导致执行这些槽函数，这些槽函数将处理第一个对象状态的变化，完成对象之间的通信。
- 信号和槽本质上还是回调函数。

## 概念

信号槽是 Qt 框架引以为豪的机制之一。所谓信号槽，实际就是观察者模式。当某个事件发生之后，比如，按钮检测到自己被点击了一下，它就会发出一个信号（signal）。这种发出是没有目的的，类似广播。如果有对象对这个信号感兴趣，它就会使用连接（connect）函数，意思是，将想要处理的信号和自己的一个函数（称为槽（slot））绑定来处理这个信号。也就是说，当信号发出时，被连接的槽函数会自动被回调。

槽的本质是类的成员函数，其参数可以是任意类型的。和普通C++成员函数几乎没有区别，它可以是虚函数；也可以被重载；可以是公有的、保护的、私有的、也可以被其他C++成员函数调用。唯一区别的是：槽可以与信号连接在一起，每当和槽连接的信号被发射的时候，就会调用这个槽。

------

自定义信号槽注意事项：

> 1. 发送者和接收者都需要是QObject的子类（当然，槽函数是全局函数、Lambda 表达式等无需接收者的时候除外）；
> 2. 使用 signals 标记信号函数，信号是一个函数声明，返回 void，不需要实现函数代码；
> 3. 槽函数是普通的成员函数，作为成员函数，会受到 public、private、protected 的影响；
> 4. 使用 emit 在恰当的位置发送信号(emit 是一个空宏定义,只作为一个信号发送的标志,没有实际含义,不适用emit也可以发送信号)；
> 5. 使用QObject::connect()函数连接信号和槽；
> 6. 任何成员函数、static 函数、全局函数和 Lambda 表达式都可以作为槽函数。

## 信号

### 信号的发出

由于某种条件到达可能引起了对象改变，其内部状态将发生改变，这时候对象就会发出信号。信号是公共访问函数，可以从任何地方发出，但是建议：**【只从定义该信号的类及其子类发出信号】**。

在Qt框架下，信号发出分为两种：

1. 【每个类预定义的信号】：这些信号何时发出可以通过查看官方文档获知。
2. 【自定义的信号】：这些信号的发出由开发人员自行定义。

### **信号的处理**

当一个信号发出时，连接到它的槽函数通常会立即执行，就像一个普通函数调用一样。在这种情况下，信号和槽函数机制是完全独立于GUI事件循环的，也并不会干扰GUI的事件循环。`emit`语句之后的代码将在所有槽函数都返回之后才执行。如果使用**排队连接（queued connections）**，情况略有不同，在这种情况下，emit关键字后面的代码将立即继续，槽函数将在后续执行。

如果几个槽函数连接到同一个信号上，当信号发出时，这些槽函数将按照它们连接时的顺序依次执行【这一点很重要】。

信号是由`moc`工具自动生成，不能在.cpp文件中实现，所以信号永远不能有返回类型(必须使用`void`关键字定义信号)。

关于信号和槽参数的注意事项：经验表明，如果信号和槽函数不使用特殊类型，那么代码具有极强的可重用性。

下表是使用`connect()`创建信号和槽函数连接时，可以指定5种不同的连接类型：

| 序号 | 类型                         | 含义                                                         |
| ---- | ---------------------------- | ------------------------------------------------------------ |
| 1    | Qt::AutoConnection           | 如果接收者生活在发出信号的线程中，Qt::DirectConnection被使用。否则，使用Qt::QueuedConnection。连接类型是在信号发出时确定。【这是Qt创建信号和槽函数时的默认连接方式】 |
| 2    | Qt::DirectConnection         | 当信号发出时，槽函数立即被调用。槽函数在发送信号的线程中执行。 |
| 3    | Qt::QueuedConnection         | 当控制返回到接收方线程的事件循环时，将调用槽函数。槽函数在接收方的线程中执行。 |
| 4    | Qt::BlockingQueuedConnection | 与Qt::QueuedConnection相同，只是在槽函数返回之前线程会阻塞。如果接收方存在于发送信号的线程中，则不能使用此连接，否则应用程序将会死锁。 |
| 5    | Qt::UniqueConnection         | 这是一个标志，可以使用按位OR与上述的连接类型进行组合。当Qt::UniqueConnection被设置时，如果连接已经存在，QObject::connect()将失败(例如，如果相同的信号已经连接到同一对对象的相同槽位)。注：这个标志在Qt 4.6中引入。 |
|      |                              |                                                              |

## 槽函数

当一个连接到槽函数的信号被发射时，槽函数将被调用。槽函数是普通的C++函数，在实际开发中也可以正常调用；它们唯一的特点是：【信号可以与它们相连接】。

由于槽是普通的成员函数，所以它们在直接调用时遵循普通的C++规则。但是，作为槽函数时，任何组件都可以通过信号连接从而调用它们。

还可以将槽函数定义为虚拟的，这在开发中非常有用。

与回调机制相比，信号和槽函数机制的速度稍微慢一些，这一点对于实际应用程序来说，这种差别并不显著。一般来说，发送一个连接到某些槽函数的信号，比直接调用非虚函数要慢大约10倍。这是定位连接对象、安全地遍历所有连接(即检查后续接收方在发射过程中没有被销毁)以及以函数调用增加的开销。虽然10个非虚函数调用听起来很多，但是它比new操作或delete操作的开销要小得多。一旦在后台执行一个需要new或delete的字符串、向量或列表操作，信号和槽函数的开销只占整个函数调用开销的很小一部分。在槽函数中执行系统调用时也是如此（或间接调用超过十个函数）。因此信号和槽函数机制的简单性和灵活性是值得的，这些开销在实际应用场景下甚至不会注意到。

注意，当与基于Qt的应用程序一起编译时，定义为信号或槽的变量的第三方库可能会导致编译器出现警告和错误。要解决这个问题，使用`#undef`来定义出错的预处理器符号即可。

## 用法

1. 一个信号可以和多个槽相连

   > 如果是这种情况，这些槽会一个接一个的被调用，但是它们的调用顺序是不确定的。

2. 多个信号可以连接到一个槽

   > 只要任意一个信号发出，这个槽就会被调用。

3. 一个信号可以连接到另外的一个信号

   > 当第一个信号发出时，第二个信号被发出。除此之外，这种信号-信号的形式和信号-槽的形式没有什么区别。

4. 槽可以被取消链接

   > 这种情况并不经常出现，因为当一个对象delete之后，Qt自动取消所有连接到这个对象上面的槽。

5. 使用Lambda 表达式

   > 在使用 Qt 5 的时候，能够支持 Qt 5 的编译器都是支持 Lambda 表达式的。

**总结:**

![信号和槽](https://cdn.jsdelivr.net/gh/ouyujia/blog-img/img/20190109151732226.png)

### **带有默认参数的信号和槽函数**

信号和槽可以包含参数，参数可以有默认值。例如：`QObject::destroyed()`：

```c++
void destroyed(QObject* = nullptr);
```

当`QObject`被删除时，它会发出这个`QObject::destroyed()`信号。无论我们在哪里有一个悬空引用指向已删除的QObject，都希望捕捉到这个信号，这样就可以清除它。合适的槽参数可以是：

```c++
void objectDestroyed(QObject* obj = nullptr);
```

### **使用QObject::connect()将信号连接到槽函数的三种方法**

- 第一种方法：使用函数指针(qt5)

  ```c++
  connect(sender, &QObject::destroyed, this, &MyObject::objectDestroyed);
  ```

  将`QObject::connect()`与函数指针一起使用有几个优点。它允许编译器检查信号的参数是否与槽的参数兼容。当然，编译器还可以隐式地转换参数。

- 第二种方法：连接到C++ 11的lambdas

  ```c++
  connect(sender, &QObject::destroyed, this, [=](){ this->m_objects.remove(sender); });
  ```

  在这种情况下，我们在connect()调用中提供这个上下文。上下文对象提供关于应该在哪个线程中执行接收器的信息。

  当发送方或上下文被销毁时，`lambda`将断开连接。注意：当信号发出时，函数内部使用的所有对象依然是激活的。

- 第三种方法：使用`QObject::connect()`以及信号和槽声明宏。(qt4)

  ```c++
  在SIGNAL()和SLOT()宏中包含参数(如果参数有默认值)的规则是：传递给SIGNAL()宏的参数不能少于传递给SLOT()宏的参数。
  ```

### **使用disconnect断开信号/槽连接**

`disconnect()`用于断开对象发送器中的信号与对象接收器中的方法的连接。如果连接成功断开，则返回true；否则返回false。

当对信号/槽关联的两方中的任何一个对象进行销毁时，信号/槽连接将被移除。

`disconnect()`有三种使用方法，如下示例所示：

1、断开所有与对象相连的信号/槽：

```c++
disconnect(myObject, nullptr, nullptr, nullptr);
```

相当于非静态重载函数：

```c++
myObject->disconnect();
```

2、断开所有与特定信号相连的对象：

```c++
disconnect(myObject, SIGNAL(mySignal()), nullptr, nullptr);
```

相当于非静态重载函数：

```c++
myObject->disconnect(SIGNAL(mySignal()));
```

3、断开特定接收对象的连接：

```c++
disconnect(myObject, nullptr, myReceiver, nullptr);
```

相当于非静态重载函数：

```c++
myObject->disconnect(myReceiver);
```

`nullptr`可以用作通配符，分别表示“任何信号”、“任何接收对象”或“接收对象中的任何槽”。

如下格式的使用示例：

```c++
disconnect(发送对象，信号，接收对象，方法）
```

- **发送对象**不会是nullptr。
- 如果**信号**为nullptr，将断开**接收对象**和**槽函数**与所有**信号**的连接。否则只断开指定的**信号**。
- 如果**接收对象**是nullptr，它断开所有关联该信号的连接。否则，只断开与**接收对象**的槽函数连接。
- 如果方法是nullptr，它会断开任何连接到**接收对象**的连接。如果不是，只有命名为**方法**的槽函数连接将被断开。如果没有**接收对象**，**方法**必须为nullptr。即：

```c++
disconnect(发送对象，信号，nullptr，nullptr）
```

### **使用Qt与第三方信号和槽函数**

当第三方库中也有信号/槽函数机制时，这时候又需要使用Qt的信号和槽函数机制。对于这种开发场景，Qt可以在同一个项目中使用这两种机制。需将下面一行添加到qmake项目(.pro)工程配置文件中：

```c++
CONFIG += no_keywords
```

该配置将告诉Qt不要定义moc关键字信号、槽函数和`emit`，因为这些名称将被第三方库使用（例如`Boost`）。如果要在使用no_keywords标志下继续使用Qt信号和槽机制，需将源文件中所有的Qt moc关键字替换为对应的Qt宏：Q_SIGNALS(或Q_SIGNAL)、Q_SLOT(或Q_SLOT)和Q_EMIT。

## 信号和槽优缺点

### 问题：

为什么Qt使用信号与槽机制而不是传统的回调函数机制进行对象间的通信呢？
回调函数的本质是“你想让别人的代码执行你的代码，而别人的代码你又不能动”这种需求下产生的。
回调函数是函数指针的一种用法，如果多个类都关注某个类的状态变化，此时需要维护一个列表，以存放多个回调函数的地址。对于每一个被关注的类，都需要做类似的工作，因此这种做法效率低，不灵活。

### 解决办法：

Qt使用信号与槽机制来解决这个问题，程序员只需要指定一个类含有哪些信号函数、哪些槽函数，Qt会处理信号函数和槽函数之间的绑定。当信号函数被调用时，Qt会找到并执行与其绑定的槽函数。允许一个信号函数和多个槽函数绑定，Qt会依次找到并执行与一个信号函数绑定的所有槽函数，这种处理方式更灵活。

#### 优点:

- 类型安全：需要关联的信号槽的签名必须是等同的。即信号的参数类型和参数个数同接受该信号的槽的参数类型和参数个数相同。若信号和槽签名不一致，则编译器会报错。
- 松散耦合：信号和槽机制减弱了Qt对象的耦合度。激发信号的Qt对象无需知道是哪个对象的那个槽接收它发出的信号，它只需要在适当的时间发送适当的信号即可，而不需要关心是否被接收和哪个对象接收了。Qt保证适当的槽得到了调用，即使关联的对象在运行时删除，程序也不会崩溃。
- 灵活性：一个信号可以关联多个槽，多个信号也可以关联同一个槽。

### 缺点

- 速度较慢：与回调函数相比，信号和槽机制运行速度比直接调用非虚函数慢10倍左右。

#### 原因

- 需要定位接收信号的对象。
- 安全地遍历所有槽。
- 编组，解组传递参数。
- 多线程的时候，信号需要排队等候。（然而，与创建对象的new操作及删除对象的delete操作相比，信号和槽的运行代价只是他们很少的一部分。信号和槽机制导致的这点性能损失，对于实时应用程序是可以忽略的。）

## 信号和槽无法连接问题分析

1. 使用qt4信号和槽方法转发信号到qt5信号和槽方法中

   ```c++
   void signal();
   void signal2();
   void slot();
   connect(this, SIGNAL(void signal()), this, SIGNAL(void signal2()));
   connect(this, &test::siganl2, this, &test::slot);
   ```

   原因:qt4使用宏连接信号和槽,本质是将信号和槽函数转为字符串,而qt5是使用函数指针,因此无法混用转发信号.

2. 使用qt4信号和槽的方式连接时带了形参

   ```c++
   void singal(QString text);
   void slot(QString text);
   connect(this, SIGNAL(void signal(QString text)), this, SLOT(void slot(QString text))); // 可能会在编译时报错,也可能不会,但在运行时slot一定不会生效
   ```

3. 使用自定义类型参数而没有进行注册

   ```c++
   class Myclass
   {
   };
   void signal(Myclass ha);
   void slot(Myclass he);
   connect(this, &test::siganl2, this, &test::slot);// 槽函数没有生效
   
   // 自定义类型注册
   	1、注册位置：在第一次使用此类链接跨线程的signal/slot之前，一般在当前类的构造函数中进	行注册；
     	2、注册方法：在当前类的顶部包含：#include <QMetaType>，构造函数中加入代码：		qRegisterMetaType<MyClass>("Myclass")；
     	3、Myclass的引用类型需单独注册：qRegisterMetaType<MyClass>("Myclass&")；
   
   ```

4. 连接信号和槽时,对象是空指针(未定义)

   ```c++
   class Myclass
   {
   };
   void signal();
   void slot();
   connect(ha, &test::siganl2, he, &test::slot);// 槽函数没有生效
   Myclass ha = new Myclass();
   Myclass he = new Myclass();
   ```

   信号和槽一定要在对象定义之后进行连接才能生效

5. 信号声明为私有方法其他类对象连接失败(其他博客里提及,本人验证了放在私有里也是可以的,索性也列在这里供大家参考)

   test.h

   ```c++
   #ifndef TEST_H
   #define TEST_H
   #include <QObject>
   
   class Test : public QObject
   {
       Q_OBJECT
   public:
       explicit Test(QObject* parent = nullptr);
   
   private slots:
       void print(); //! 私有槽函数可以被其他类对象连接，并不是只能由本类内部连接
   
   };
   
   class Test2 : public QObject
   {
       Q_OBJECT
   public:
       explicit Test2(QObject* parent = nullptr);
   
   signals:
       void signalPrint();
   
   };
   
   #endif // TEST_H
   ```

   test.cpp

   ```c++
   #include "test.h"
   #include <stdio.h>
   
   Test::Test(QObject *parent)
       : QObject(parent)
   {
   
   }
   
   void Test::print()
   {
       printf("private is connected\n");
   }
   
   
   Test2::Test2(QObject *parent)
       : QObject(parent)
   {
       Test test;
       connect(this, SIGNAL(signalPrint()), &test, SLOT(print()));
       emit signalPrint();
   }
   
   ```

   执行结果

   ![执行结果](https://cdn.jsdelivr.net/gh/ouyujia/blog-img/img/1683285517791.png)

### 虚拟槽函数

## 参考资料

[Qt之信号signals和槽slots详解_private slots](https://blog.csdn.net/qq_30725967/article/details/104515447)

[Qt之信号槽机制](https://kevin-org.blog.csdn.net/article/details/121232524?spm=1001.2014.3001.5502)

[Qt一篇全面的信号和槽函数机制总结](https://zhuanlan.zhihu.com/p/603617075)

