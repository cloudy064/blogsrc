---
title: WinDBG观世界——番外[一]（在C语言中模拟类的虚函数）
date: 2016-05-02 23:08:05
categories: windbg
tags: [windbg,disassembly,c++,c]
---

## 导言
很多同学在面试的时候都会遇到这个问题吧，请问在C里面我要怎么实现虚函数呢？我记得当时大多数都是死记硬背的答案：通过函数指针就可以了啊。但是本后的机制当时有弄懂么？如果你懂了，那么这一节你就不需要继续往下看了；如果你是个C语言高手，那么你也可以给我这篇文章提一点意见，因为我用C语言的经验至今就三千行代码，而且还是四年前写的。其他同学就跟着我一起来模拟一下在C语言里面究竟如何才能实现C++中的虚函数。
<!--more-->
## 我记得是这样运作的
回顾一下上面两章的内容，首先我们要弄清楚三个问题：
1. 类的继承是怎么实现的，内存结构如何
2. 虚函数表存放在类的什么地方
3. 虚函数表里面存放了什么

如果上面三个问题还没有弄清楚，我建议先把第三四章的内容搞定再来研究这个吧。

我们这里先来模拟继承，看C++下面这段代码：
```cpp
class TestBase
{
private:
    int a;
};

class TestDerive : public TestBase
{
private:
    int b;
};
```

首先，在C语言里面并没有`class`关键字，只有通过`struct`来定义结构体。并且也没有`public`等访问控制符，但是正如我以前所说，其实这些访问控制符在实际生成的代码中并没有任何作用，他们的作用只是将代码语义上的错误在编译期抛出来，我们这里也可以忽略。而父类在子类中的布局其实前面一章也说明了，父类对象的内容位于子类对象最开始的地方（这里我们先不考虑多重继承）。我们所需要做的就是将一个父类对象放到子类对象最开始的地方就行了。所以它所对应的C代码如下：
```C
typedef struct tagTestBase
{
    int a;
} TestBase;

typedef struct tagTestDerive
{
    TestBase base;
    int b;
} TestDerive;
```

接着我们需要加的就是构造函数了，考虑下面这段C++代码：
```cpp
class TestBase
{
public:
    TestBase()
        : a(10)
    {

    }

private:
    int a;
};

class TestDerive : public TestBase
{
public:
    TestDerive()
        : b(12)
    {

    }

private:
    int b;
};
```

但是在C语言里面，其实我们并不能在结构体里面加入函数（你不要和我说你的能加，你先把cpp后缀改成c再说），所以C语言里面的函数都是全局函数，并且提前说明一下，C语言不支持函数的重载。前面我们研究了`this`指针的由来，还记得么？
1. 将局部变量的堆栈地址存放到`ecx`寄存器
2. `call`构造函数
3. 将`ecx`的值存放到`ebp-8`的位置（也出现了`ebp-14h`的情况）
4. 将`ebp-8`处的值存放到`eax`寄存器中

也就是说，`this`指针也是每次函数调用时，外部传入的一个指针变量。
接着构造函数会执行下列操作：
1. 调用父类构造函数
2. 调用成员变量的初始化器
3. 执行函数体

那么这里就好办了，我们可以直接写出下面这段代码：
```c
typedef struct tagTestBase
{
    int a;
} TestBase;

typedef struct tagTestDerive
{
    TestBase base;
    int b;
} TestDerive;

void TestBaseConstructor(TestBase* pThis)
{
    pThis->a = 10;
}

void TestDeriveConstructor(TestDerive* pThis)
{
    TestBaseConstructor(pThis->base);
    pThis->b = 12;
}
```

那个说我没有判断指针为空的，你给我出去。好了，我们继续。所以这里构造函数已经模拟完成。

下面该清楚本章的主角了：虚函数。考虑下面这个C++代码：
```cpp
class TestBase
{
public:
    TestBase()
        : a(10)
    {

    }

public:
    virtual int add(int k)
    {
        return a + k;
    }

    virtual int sub(int k)
    {
        return a - k;
    }

private:
    int a;
};

class TestDerive : public TestBase
{
public:
    TestDerive()
        : b(12)
    {

    }

public:
    virtual int add(int k)
    {
        return b + k;
    }

    virtual int sub(int k)
    {
        return b - k;
    }

private:
    int b;
};
```

因为刚讲虚函数，所以可能大家还没有消化完全，所以这里我大致画一下这两个类的结构图吧，画完之后大家应该就知道如何用C语言实现虚函数了：
{% asset_img vftables.png 类的内存结构 %}

我来解释下其中的`TestDerive`吧，`TestBase`同样的道理。虚函数表的地址存放在`this`指针指向的头四个字节，我们把这个地址用变量的方式标记为`vftable`吧，那么`vftable`指向的内存，第一个`DWORD`存放了`TestDerive`类中虚函数`add`的地址，第二个`DWORD`存放了`TestDerive`类中虚函数`sub`的地址。

所以对应到C语言中时，因为这些虚函数表的内容和地址在编T译期就已经固定了，我们这里需要模拟的话可以用一个全局的变量来唯一标识这个虚函数表。那么接下来的工作就容易很多了，虽然跨度有一点大，但是看下代码应该也能够很容易理解了：
```c
#include <stdio.h>
#include <stdlib.h>

typedef struct tagTestBase
{
    void** pVftable;
    int a;
} TestBase;

typedef struct tagTestDerive
{
    TestBase base;
    int b;
} TestDerive;

void** g_pBaseVftable = 0; // base的虚函数表
void** g_pDeriveVftable = 0; // derive的虚函数表

void TestBaseConstructor(TestBase* pThis)
{
    pThis->pVftable = g_pBaseVftable; //初始化虚函数表
    pThis->a = 10;
}

void TestDeriveConstructor(TestDerive* pThis)
{
    TestBaseConstructor(&pThis->base);
    pThis->base.pVftable = g_pDeriveVftable; //改写虚函数表地址
    pThis->b = 12;
}

// 这个就是前面一章讲到的虚函数的跳转函数
int CallVirtualAdd_int(TestBase* pThis, int m)
{
    typedef int (*FuncType)(TestBase*, int);
    FuncType func = (FuncType)pThis->pVftable[0];
    return func(pThis, m);
}

// 另外一个跳转函数
int CallVirtualSub_int(TestBase* pThis, int m)
{
    typedef int (*FuncType)(TestBase*, int);
    FuncType func = (FuncType)pThis->pVftable[1];
    return func(pThis, m);
}

int TestBaseVirtualAdd_int(TestBase* pThis, int m)
{
    return pThis->a + m;
}

int TestBaseVirtualSub_int(TestBase* pThis, int m)
{
    return pThis->a - m;
}

int TestDeriveVirtualAdd_int(TestDerive* pThis, int m)
{
    return pThis->b + m;
}

int TestDeriveVirtualSub_int(TestDerive* pThis, int m)
{
    return pThis->b - m;
}

void InitVftables()
{
    // 构造Base的虚函数表
    g_pBaseVftable = (void**)malloc(sizeof(void*) * 2);
    g_pBaseVftable[0] = (void*)&TestBaseVirtualAdd_int;
    g_pBaseVftable[1] = (void*)&TestBaseVirtualSub_int;

    // 构造Derive的虚函数表
    g_pDeriveVftable = (void**)malloc(sizeof(void*) * 2);
    g_pDeriveVftable[0] = (void*)TestDeriveVirtualAdd_int;
    g_pDeriveVftable[1] = (void*)TestDeriveVirtualSub_int;
}

void UninitVftables()
{
    free(g_pBaseVftable);
    free(g_pDeriveVftable);
}

int main_core(int argc, char** argv)
{
    TestDerive d;
    TestDeriveConstructor(&d);
    CallVirtualAdd_int((TestBase*)&d, 19);

    return 0;
}

int main(int argc, char** argv)
{
    InitVftables();

    // 这里之所以要弄一个单独的函数，是因为C语言中变量声明必须放在函数的开头
    // 这里你可以理解为只是为了对称吧
    main_core(argc, argv);

    UninitVftables();

    return 0;
}
```

那么这样做的话肯定不能满足大多数人的需求，因为不可能每次要用到虚函数都要让我写这么一大坨东西。所以这个时候就需要用到C语言中的利器——宏了，但是我这里不打算介绍宏的编写，因为确实好复杂，以后研究出来了放出来给大家好了。

而其实C语言中使用虚函数更好的方式并不是利用虚函数表，因为这样的话，如果有n个虚函数那么我们就需要写相当多个虚函数的call函数，那么针对上面的虚函数调用，C语言中更常用的代码版本如下：
```c
struct tagTestBase;

typedef int (*AddFuncType)(struct tagTestBase* pThis, int);
typedef int (*SubFuncType)(struct tagTestBase* pThis, int);

typedef struct tagTestBase
{
    AddFuncType addFunc;
    SubFuncType subFunc;
    int a;
} TestBase;

typedef struct tagTestDerive
{
    TestBase base;
    int b;
} TestDerive;

int TestBaseVirtualAdd_int(TestBase* pThis, int m)
{
    return pThis->a + m;
}

int TestBaseVirtualSub_int(TestBase* pThis, int m)
{
    return pThis->a - m;
}

int TestDeriveVirtualAdd_int(TestDerive* pThis, int m)
{
    return pThis->b + m;
}

int TestDeriveVirtualSub_int(TestDerive* pThis, int m)
{
    return pThis->b - m;
}

void TestBaseConstructor(TestBase* pThis)
{
    pThis->addFunc = &TestBaseVirtualAdd_int;
    pThis->subFunc = &TestBaseVirtualSub_int;
    pThis->a = 10;
}

void TestDeriveConstructor(TestDerive* pThis)
{
    TestBaseConstructor(&pThis->base);
    pThis->base.addFunc = (AddFuncType)&TestDeriveVirtualAdd_int;
    pThis->base.subFunc = (SubFuncType)&TestDeriveVirtualSub_int;
    pThis->b = 12;
}


int main_core(int argc, char** argv)
{
    TestDerive d;
    TestDeriveConstructor(&d);
    d.base.addFunc((TestBase*)&d, 19);

    return 0;
}

int main(int argc, char** argv)
{
    main_core(argc, argv);

    return 0;
}
```

那么这次番外篇就这么结束好了，只是一个简单的介绍，不理解也没有大碍。晚安
