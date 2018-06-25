---
title: WinDBG观世界（四）
date: 2016-04-26 12:22:37
categories: windbg
tags: [windbg,disassembly,c++]
---

## 内容概要
前面一章在探究类的时候，遗留了很多问题，比如参数的构造顺序，虚函数的调用实现等等，那么这一章的就是来填这些坑的。当然这一章不会把这些坑都填完，因为类里面需要讲的东西实在是太多了。那么我先把这一章里面需要既觉得几个问题都罗列出来吧：
1. 如果我在构造函数中调用虚函数，会执行到正确的（或者说我们所希望的）虚函数么
2. 构造函数成员初始化的顺序
3. 在子类中，父类的数据被放在什么地方
4. 普通成员覆盖和虚函数重写有什么区别
5. 虚函数重写之后，子类的虚函数表会发生什么变化
6. 发生继承时，子类是会完全复制父类的虚函数表，还是只会将自己类中的虚函数写入表中
7. 通过函数指针进行调用和直接调用成员函数有什么区别
8. 父类能够通过成员函数指针来调用子类的普通成员函数和虚函数么
9. 子类能够通过成员函数指针来调用父类的普通成员函数和虚函数么
<!--more-->
## 执行哪个函数？运行的时候才知道啊！
本章进行分析的代码内容如下：
```cpp
class TestBase
{
public:
    TestBase()
        : e(12)
        , c(3)
        , a(12)
        , b(1)
        , f('l')
    {
        d = 19;

        VirtualFunc1();
    }

    ~TestBase()
    {
        a = 12;
    }

public:
    virtual void VirtualFunc1()
    {
        a += 12;
    }

    virtual void VirtualFunc2()
    {
        b += 22;
    }

public:
    void NormalFunc1()
    {
        c += 14;
    }

protected:
    int a;
    int b;
    char c;
    char d;
    short e;

private:
    char f;
};

class TestDerive 
    : public TestBase
{
public:
    TestDerive(int a)
        : TestBase()
        , f(a)
    {
        VirtualFunc1();
    }

    ~TestDerive()
    {
        b = 12;
    }

public:
    virtual void VirtualFunc1()
    {
        a += 99;
    }

public:
    void NormalFunc1()
    {
        c += 54;
    }

private:
    int f;
};

int main(int argc, char** argv)
{
    typedef void (TestBase::*BaseNormalFuncType)();
    BaseNormalFuncType baseNormalFuncPointer = &TestBase::NormalFunc1;

    typedef void (TestDerive::*DeriveNormalFuncType)();
    DeriveNormalFuncType deriveNormalFuncPointer = &TestDerive::NormalFunc1;

    typedef void (TestBase::*BaseVirtualFunc1Type)();
    BaseVirtualFunc1Type baseVirtualFunc1Pointer = &TestBase::VirtualFunc1;

    typedef void (TestBase::*BaseVirtualFunc2Type)();
    BaseVirtualFunc2Type baseVirtualFunc2Pointer = &TestBase::VirtualFunc2;

    typedef void (TestDerive::*DeriveVirtualFunc1Type)();
    DeriveVirtualFunc1Type deriveVirtualFunc1Pointer = &TestDerive::VirtualFunc1;

    typedef void (TestDerive::*DeriveVirtualFunc2Type)();
    DeriveVirtualFunc2Type deriveVirtualFunc2Pointer = &TestDerive::VirtualFunc2;

    TestBase base1;
    base1.NormalFunc1();
    base1.VirtualFunc1();
    base1.VirtualFunc2();

    TestDerive derive1(77);
    derive1.NormalFunc1();
    derive1.VirtualFunc1();
    derive1.VirtualFunc2();


    (base1.*baseNormalFuncPointer)();
    (derive1.*deriveNormalFuncPointer)();
    (derive1.*baseNormalFuncPointer)();
    
    (base1.*baseVirtualFunc1Pointer)();
    (base1.*baseVirtualFunc2Pointer)();
    
    (derive1.*baseVirtualFunc1Pointer)();
    (derive1.*baseVirtualFunc2Pointer)();

    (derive1.*deriveVirtualFunc1Pointer)();
    (derive1.*deriveVirtualFunc2Pointer)();
}
```

代码确实越来越长了，但是其实分析的内容不是特别多，那么就照旧，打开windbg一步一步来分析代码吧。

跳过初始化的部分，我们来到第一句代码，诶，为什么`typedef void (TestBase::*BaseNormalFuncType)()`没有执行啊喂，第一句代码直接运行赋值操作了？
{% asset_img AssignMemberFuncPointer.png 第一句代码 %}

恩，其实我也不知道，而且我google也没有找到答案，咱忽略这个问题好不好？

那么看下面六个函数指针的赋值吧，前面两个没话说，和普通的函数地址一样，因为经过上一章的研究，其实`this`指针并不是类内部或者函数内部的，而是函数调用时外部通过`ecx`寄存器传递给函数的一个隐式参数。（当然如果你想知道函数的地址是怎么得到的，可以等我开一个讲编译器的新坑）那么这里我们主要需要看的是最后四个虚函数的指针的赋值。可以看到它的代码如下：
```asm
mov dword ptr [ebp-2Ch], offset HelloWorld2005!TestBase::`vcall'{0}'
mov dword ptr [ebp-38h], offset HelloWorld2005!TestBase::`vcall'{4}'
mov dword ptr [ebp-44h], offset HelloWorld2005!TestBase::`vcall'{0}'
mov dword ptr [ebp-50h], offset HelloWorld2005!TestBase::`vcall'{4}'
```

不对啊喂，我后面两个明明是取的`TestDerive`中的虚函数，为什么在汇编代码中还是给我`TestBase`中虚函数的地址啊？好吧，我暂时也无法给你解答，把所有的疑问全部都放一放，然后继续研究吧。

然后我们开始定义第一个类变量：
{% asset_img FirstClassVariable.png 定义TestBase变量 %}

我们进入到构造函数内部看下吧（这里`this`指针的值为`0019febc）：
{% asset_img TestBaseConstructor.png TestBase的构造函数 %}

还记得下面构造函数会做哪些事情吗？
1. 将`this`指针放到`ebp-8`的位置
2. 将`ebp-8`位置的值放到`eax`中
3. 如果有虚函数，下面就会把虚函数表的地址放到`this`指针指向的第一个地址

这一章里面主要研究虚函数表，之所以叫虚函数表，我想应该就是一张表里面有很多的`entry`，然后每个`entry`指向了一个对应的虚函数地址吧，好吧这都是我意淫出来的。所以，我们首先用`x`指令来查看`TestBase`中所有虚函数的地址是多少：
{% asset_img TestBaseVirtualFunctionsAddress.png TestBase中的虚函数 %}

然后我们用`dd eax`来查看`this`指针指向的内存：
{% asset_img TestBaseLookupThis.png this指针指向的内存 %}

第一个4字节的值`00404130`就是虚函数表的地址，我们再用`dd 00404130`查看这个地址下是不是存放了上面两个虚函数的地址：
{% asset_img TestBaseContentOfVirtualTable.png TestBase虚函数表的内容 %}

这不是演习，果然后面一个一个存放着虚函数的指针，暂且认为它们是按照声明的顺序来存放的吧，毕竟其实这个顺序如果你不是想`hack`的话，对于我们来说没什么意义。所以从汇编的层面来看的话，虚函数表其实没啥神奇的。

好吧，那么我们继续往下面看，需要注意的是，我这里把成员变量声明的顺序和类初始化器的顺序安排成不一致，那么现在我们需要一边看一边关注是在为哪个变量赋值。或者。。。。。我们不妨用`dt`看下`TestBase`的内存结构吧：
{% asset_img dtTestBase.png TestBase的内存结构 %}

额，内存结构里面的内存对齐我觉得可以自己分析，我这里就不多说了。然后就是对照着目标地址和内存结构进行对应了：
{% asset_img TestBaseInitSequence.png TestBase成员变量初始化顺序 %}

诶，好像和我在初始化器里面摆放的顺序不一样，完全就是和成员变量声明的顺序是一致的，当然除了最后一个在函数体中进行的一次赋值。这样的话概要中的第二个问题就已经解决了。

接着下面调用了一个虚函数`VirtualFunc1`，进去看了下好像和我们以前调用的普通函数没有什么区别，那我们就直接跳过吧：
{% asset_img TestBaseVirtualFunc2Function.png VirtualFunc1的汇编实现 %}

然后我们跳过函数的调用，等下一起看，直接来看下`TestDerive`对象的定义，进去它的构造函数看下汇编实现（此时`this`指针的值是`0019fe9c`）：
{% asset_img TestDeriveConstructor.png TestDerive的构造函数 %}

在看调用父类构造函数之前，我们继续用`dt`来看下`TestDerive`的内存结构：
{% asset_img dtTestDerive.png TestDerive的内存结构 %}

可以看出，`TestDerive`是把父类的内存结构拷贝过来（而不是在开始的位置放一个`TestBase`），然后在最后加上一个`TestDerive`中定义的变量`f`，而虚函数表指针也只有一个。根据之前的经验，类会先初始化虚函数表，在进行成员变量的初始化，然而在往下看的过程中，可以看到：
```asm
mov dword ptr [ebp-14h], ecx
mov ecx, dword ptr [ebp-14h]
call HelloWorld2005!TestBase::TestBase
```

尼玛，为什么这里`this`指针就存放在了`ebp-14h`这个位置上，之前都是放在`ebp-8`的位置的。哎，我觉得我又要画一个此时堆栈的内存图了：
{% asset_img TestDeriveCtorBeforeBaseCtorStackMemory.png 此时堆栈的内存图 %}

= =，为什么这个构造函数里面又多了这么多东西？说好的只需要初始化堆栈区域呢？至于为什么会多出这么多东西，我暂时也不知道，所以无法给大家解答。那这样的话，加上第一个局部变量都会空出4个字节出来的尿性，这里`this`指针确实应该放在`ebp-14h`的位置。

接着它就开始调用父类构造函数`TestBase`了。这么一来，虚函数表就肯定是在父类中进行赋值了，所以这样的话，父类中调用的`VirtualFunc1`肯定也就是`TestBase`中的`VirtualFunc1`了。这个我就不跟进去看了，那么现在的问题就是，我在子类中重写了虚函数`VirtualFunc1`，那么这个虚函数的地址是怎么写入到虚函数表的呢？继续往下看就知道了。

下面一句`mov dword ptr [ebp-4], 0`这个我还是没有明白是啥意思，忽略它。主要是下面这两句：
```asm
mov eax, dword ptr [ebp-14h]
mov dword ptr [eax], offset HelloWorld2005!TestDerive::`vftable'
```

看到这里的动作就是修改`this`指向的基地址的值，也就是虚表的地址，我们前面拿到的虚函数表的地址是`00404130`，那我们这里看下它的地址是多少：
{% asset_img TestDeriveAfterAssignVftable.png 虚函数表的地址 %}

可看到这个地址变成了`0040413c`。那么查看这个地址下的内容：
{% asset_img TestDeriveVtableContent.png 虚函数表中的内容 %}

我们可以来看下是不是正好就是我们虚函数的地址，利用`x`指令即可：
{% asset_img TestDeriveVirtualFunctionAddress.png 虚函数地址 %}

居然没有输出`VirtualFunc2`的地址，只能猜测`x`指令只能输出类中实际实现的函数的地址。但是这里我们仍然可以看到`VirtualFunc1`的地址明显变成了`00401460`，正好就是虚函数表中第一个地址。而虚函数表中第二个地址也正好就是父类中`VirtualFunc2`的地址。所以其实有几个点我们就弄清楚了：
1. 在构造时会先调用父类的构造函数，并且将父类的虚函数表地址传递给`this`指针，所以在父类构造函数中仍然可以调用虚函数，但是调用的只是父类中实现的版本
2. 在父类构造函数返回后，会将子类中的虚函数表地址传递给`this`指针，也就是会覆盖掉父类的虚函数表地址。所以在此之后，构造函数中调用的虚函数就是子类中的版本了。

接着往下看到两句
```asm
mov eax, dword ptr [ebp-14h]    ;获取this指针
mov ecx, dword ptr [ebp+8]      ;拿到传递给构造函数的参数
mov dword ptr [eax+14h], ecx    ;执行f的初始化
```

加上注释之后也就不需要我解释了，大家都明白的。

再往下看，疑惑的地方就来了：
```asm
mov eax, dword ptr [ebp-14h]                    ;拿到this指针
call HelloWorld2005!TestDerive::VirtualFunc1    ;调用VirtualFunc1
```

说好的运行时多态呢？为什么好像在编译期直接就给我确定了调用的地址了？老师教的不是说先查找虚函数表，然后再找到对应的虚函数地址么？好像和我以前知道的有一点偏差。但是我现在也不知道为什么，我暂时理解为编译的优化吧。这里留一个疑问，以后看看能不能解决。

这时我们回到`main`函数中继续看那几个函数调用，貌似看起来也都是直接在编译期就决定了函数跳转到哪里了，和普通的函数完全没有区别。是不是被那些将虚函数的书坑了啊？我现在有一点怀疑了。当然不排除怀疑这是微软的特立独行。

算了，我们继续看下面通过成员函数指针进行的调用，首先是三个普通成员函数，它对应的汇编代码是：
```asm
lea ecx, [ebp-6Ch]
call dword ptr [ebp-14h]

lea ecx, [ebp-8Ch]
call dword ptr [ebp-20h]

lea ecx, [ebp-8Ch]
call dword ptr [ebp-14h]
```

可以看到其实对成员函数指针的调用和平时成员函数的调用没有什么区别，只是把`call`的函数名变成了一个地址，`this`指针仍然存放在`ecx`寄存器中。而且我们可以通过`x`指令来查看两个普通函数的地址：
{% asset_img NormalFunc1Addresses.png NormalFunc1的地址 %}

好像在这一块儿并没有任何歧义，因为知道地址对应的函数之后，都很好理解该跳转到哪里，所以我就不多说什么了。接下来

接着看虚函数，因为之前在前面虚函数地址进行赋值时，看到`TestBase`的虚函数地址和`TestDerive`的虚函数地址居然完全一样，这和我们刚刚用`x`命令看到的不同。好啊吧，闲话少说，首先看第一组，用`base1`调用`baseVirtualFunc1Pointer`和`baseVirtualFunc2Pointer`，对应的汇编代码是：
```asm
lea ecx, [ebp-6Ch]
call dword ptr [ebp-2Ch]

lea ecx, [ebp-6Ch]
call dword ptr [ebp-38h]
```

在执行`call`语句时用`F11`进入函数内部查看，发现这次调用到的函数不一样了，如下：
{% asset_img TestBaseVcall0.png 调用虚函数后的结果 %}

但是细看的话，其实很简(sha)单(bi)！！！！！因为它就只做了两个操作：
1. 获取到this指针指向的第一个值（四字节），而这个值正好就是虚函数表的地址
2. 然后直接对虚函数表指向的第一个函数地址进行一次跳转(`jmp`)

额。。。。好像这样做确实能够实现调用到正确的虚函数哦。恩，果然是源码之下，了无秘密。只怪我之前怎么没想到呢= =

那其实下面的那些虚函数的调用也就不需要我分析了吧，只是调用`TestVirtualFunc2`的时候，跳转函数里面在虚表的基础上，加了四个字节的偏移。把这里的虚函数调用也理解为做了一个函数的跳转，这个函数跳转中所做的事情就是获取到正确的虚函数地址，而这个虚函数地址的获取依赖于构造对象时，赋值给`this`指针的虚表地址。

那么今天的分析就到这里了，大家好好消化一下吧，如果对本系列有什么改进的建议，欢迎找我提出来(cloudy064@gmail.com)

