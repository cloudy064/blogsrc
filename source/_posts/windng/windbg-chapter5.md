---
title: WinDBG观世界（五）
date: 2016-06-15 23:34:06
categories: windbg
tags: [windbg,disassembly,c++]
---

## 导言
すみません，好久没有更新博客了，因为我觉得越写要写的东西越多，这真的是个大坑，而且我还只挑了`C++99`的相关特性，`C++11`那个真心不敢讲。而且用`hexo`写东西真的有点蛋疼，尤其是截图，简直是煎熬啊。好了闲话不多说了，我们这次要看的东西不多，我们只看虚继承的内容，怕同学们消化不良啊，多重继承的知识放到下一部分讲解。
<!--more-->
## 诶？我父亲呢？
国际惯例，看看今天要解读的代码吧：
```cpp
class Super
{
public:
    ~Super()
    {

    }

public:
    int NormalFunc1()
    {
        return a;
    }

    virtual int VirtualFunc1()
    {
        return a + c;
    }

protected:
    int a;
    int c;
};

class Derive
    : virtual public Super
{
public:
    Derive()
        : a(10)
        , b(12)
    {
        Super::a = 19;
    }

    ~Derive()
    {

    }

public:
    virtual int VirtualFunc1() override
    {
        return Super::a + a + c + b;
    }

private:
    int a;
    int b;
};

int main(int argc, char** argv)
{
    Derive d;
    d.VirtualFunc1();
    d.NormalFunc1();
}

```

这里大家带着下面几个问题来听我说吧：
1. `Derive`的内存分布是怎么样的？`Super`放在什么位置？
2. `Derive`对象是如何找到父类的虚函数和成员函数的
3. `Derive`内部是如何传递`this`指针给父类的

如何？没骗你们吧，内容真的不多哦！（不许说我越来越短了！）好了，打开你的`windbg`，把这个`exe`跑起来吧！对了，在此之前为了不让`Visual Studio`生成多余的代码，所以我修改了一些编译选项，包括增量式编译、c++异常检测、堆栈检测等等，设置如下：
{% asset_img vsoptions.png VS设置1 %}
{% asset_img linkoptions.png VS设置2 %}

那么下面我们就单步调试到第一行代码对应的汇编码吧：
{% asset_img firststep.png 第一行代码 %}

第一句话是`push 1`，这个其实以前也遇到过，但是我没有说明是啥，因为我也没有仔细研究过，这里我只是怀疑可能是堆栈上对象的个数。这个坑留着以后填吧。

之后的两句话
```asm
lea ecx, [ebp-1Ch]
call HelloWorld2005!Derive::Derive
```

很明显第一句就是在当前堆栈上计算出`d`对象的地址，然后将这个地址当做`this`指针的值传递给构造函数。我们先把当前的堆栈用一个图画出来吧：
{% asset_img beforedconstructorstack.png 当前堆栈的样子 %}

其中黄色区域是运行时栈上的内容，粉红色区域就是局部变量区域分配给`d`对象的栈上内存（当然那也可能有浪费的，我们前面也了解到了`VS`的尿性）。不过至少能够知道`d`对象的`this`指针是`ebp-1Ch`。那我们`F11`进去函数里面看下吧，我们可以一直走到`this`指针保存到`dword ptr [ebp-4]`的地方：
{% asset_img inderiveconstructor.png 开始运行Derive构造函数 %}

现在堆栈的图如下：
{% asset_img inderiveconstructorstack.png 当前的堆栈情况 %}

下面几句就是从来没有见过的了，因为我们知道，构造函数都是先构造父类，然后再构造自己的。所以这里用脚趾头想想应该也知道肯定是准备数据初始化父类的，当然只是猜想哈。来看下面几句：
```asm
mov dword ptr [ebp-48h], 0
cmp dword ptr [ebp+8], 0
je HelloWorld2005!Derive::Derive+0x31
mov eax, dword ptr [ebp-4]
mov dword ptr [eax], offset HelloWorld2005!Derive::`vbtable'
mov ecx, dword ptr [ebp-4]
add ecx, 10h
call HelloWorld2005!Super::Super
```

不要惊讶为什么一下子要看这么多代码，因为我抄到这里才看到熟悉的父类构造函数。前面第一句我真TM不知道是干嘛用的，我真的想一句一句讲，我们忽略它好不好。第二句，要知道`ebp+8`指向的内容就是我刚刚猜测是堆栈上对象个数的值，看上面的图就知道，就是`1`，这里和0进行比，如果等于`0`，就跳转到`Derive::Derive+0x31`的位置。意义何在啊？当然你可以用`u HelloWorld2005+Derive::Derive+0x31`指令去看下这个地方的汇编代码。可以看出来，这句话的作用其实是跳过了父类构造函数的调用，如下所示：
{% asset_img jumpoversupercall.png 跳过了父类构造函数 %}

那么应该存在某种情况，我们调用构造函数不会调用父类构造函数，但是我也不知道什么时候会发生这种情况。那么我们继续往下看：
```asm
mov eax, dword ptr [ebp-4]
mov dword ptr [eax], offset HelloWorld2005!Derive::`vbtable'
```

要知道`ebp-4`的地方保存的是`this`指针，而它拿到`this`指针之后就把一个`vbtable`赋值给了`this`指针指向的起始地址。同学们（装腔作势样），你们还记得前面介绍虚函数的时候有一个`vftable`么，应该不难猜出这个地方的`vbtable`应该就是`virtual base table`的意思吧（我自己翻译的，官方翻译我没有查过）。那么这里就是和普通的继承的一点局别，引入了一个虚基类表指针。那么类比于`vftable`的内容，我猜`vbtable`里面保存的应该也是一个个的指针，而这些指针指向了某片基类内存。

继续往下看：
```asm
mov ecx, dword ptr [ebp-4]
add ecx, 10h
call HelloWorld2005!Super::Super
```

要知道在调用构造函数的时候，`ecx`寄存器保存的都是`this`指针（未开启优化的情况下），所以这里好像是把`this+10h`的位置作为了父类的`this`指针，我们看下`ecx`的内容：
{% asset_img contentofecx.png this指针的值 %}

然后我们进去`Super`构造函数里面看下吧：
{% asset_img superconstructor.png Super构造函数 %}

好像和平时没有什么两样，完全就是一个普通的构造函数，除了`this`指针是借的`Base`的之外。我们是时候画一下执行完该构造函数之后，`d`对象的内存分布了：
{% asset_img contentofobjectd.png d的内存分布 %}

好像和我们猜想的不一样，我以前以为是虚表中一个指针指向一块内存，然后这块内存里面有父类的成员啊什么的。

从构造函数里出来之后，第一句话又把我干懵了:
```asm
or dword ptr [ebp-48h], 1
```

因为我真的不知道`ebp-48h`这个地方存放的是个啥东西。。。

接下来的五句话其实很亲切，子类要开始改写虚函数表了：
```asm
mov eax, dword ptr [ebp-4]
mov ecx, dword ptr [eax]
mov edx, dword ptr [ecx+4]
mov eax, dowrd ptr [ebp-4]
mov dword ptr [eax+edx], offset HelloWorld2005!Derive::`vftable'
```

恩。。。。同学们要不然我们下课吧，好复杂T_T。

首先我们通过`eax`拿到`this`指针，然后把`this`指针指向的第一项内容取出来，放到`ecx`，可以看下上面的内存分布，这个地方存放的就是虚基类表指针。然后将虚基类表指针偏移四个字节，取出一个值存放在edx中，然后将`vftable`放到了`eax+edx`指向的地方。我也没想到会这么复杂。

我们先看看虚基类表指向的那块内存里面到底放了些什么东西吧：
{% asset_img vbtablecontent.png 虚基类表的内容 %}

还是没有头绪，它偏移四个字节之后，得到的值是`10h`，我们现在又把`this`指针加上`10h`。。。。

诶，好像得到的是父类的基类指针，然后我们把这个基类指针的第一个值改写为`Derive::vftable`，也就是子类的虚函数表，好像一切又明朗起来了。（那个问多重继承怎么办的同学请你以后上课站着）

那么我们是不是可以得出一个结论，虚基类表每个表项的内容其实是它所虚拟继承的所有父类的`this`指针偏移呢？这个问题留待下一章进行讲解。

有了这个解释之后下面的成员变量初始化也就不难解释了，我只在后面加注释了，就不一一讲解了：
```asm
mov eax, dword ptr [ebp-4]        ;this
mov ecx, dword ptr [eax]          ;vbtable
mov edx, dword ptr [ecx+4]        ;10h，Super指针偏移
sub edx, 10h                      ;吃饱了撑的？变成0了
mov eax dword ptr [ebp-4]         ;this
mov ecx, dword ptr [eax]          ;vbtable
mov eax, dword ptr [ecx+4]        ;10h
mov ecx, dword ptr [ebp-4]        ;this
mov dword ptr [ecx+eax-4], edx    ;vftable上面的那个值（暂时不知道啥用）

mov eax, dword ptr [ebp-4]        ;this指针
mov dword ptr [eax+4], 0Ah        ;a:10

mov eax, dword ptr [ebp-4]        ;this
mov dword ptr [eax+8], 0Ch        ;b:12

mov eax, dowrd ptr [ebp-4]        ;this
mov ecx, dword ptr [eax]          ;vtable
mov edx, dword ptr [ecx+4]        ;10h
mov dword ptr [eax+edx+4], 13h    ;Super指针偏移4个字节
```

那么构造函数已经完全分析完了，我们跳出来，看虚函数的调用，如下：
```asm
lea ecx, [ebp-0Ch]
call HelloWorld2005!Derive::VirtualFunc1
```

我们从上面的对象内存分布图可以知道，父类指针存放在`+10h`的位置，这样这里的`ebp-0Ch`就很明白了，它取的是`Derive`中父类`Super`对应的`this`指针。为什么呢？留待以后进一步验证。但是它调用却是`Derive::VirtualFunc1`，恩，我现在和你有相同的疑问，但是不要问，我也不知道。

`F11`进去的代码如下：
{% asset_img VirtualFunc1Code.png VirtualFunc1的代码 %}

这里我直接跳到了`this`指针保存的后面一句代码。从这里开始计算`Super::a + a + c + b;`，我把代码摘录下来，并在后面加上注释，方便大家理解：
```asm
mov dword ptr [ebp-4], ecx          ; 保存this
mov eax, dword ptr [ebp-4]          ; this放到eax中
mov ecx, dword ptr [eax-10h]        ; eax-10后是Derive的this指针，然后取址得到虚类表
mov edx, dword ptr [ecx+4]          ; 得到Super父类的地址偏移，10h
mov eax, dword ptr [eax+edx-0Ch]    ; 将edx=10h带入到表达式，就是mov eax, dword ptr [eax+4]，也就是Super中的a，保存到了eax

mov ecx, dword ptr [ebp-4]          ; this放到ecx中
add eax, dword ptr [ecx-0Ch]        ; this-0Ch的地址保存的就是Derive中的a

mov edx, dword ptr [ebp-4]          ; this保存到edx中
mov ecx, dword ptr [edx-10h]        ; 虚表类地址
mov edx, dword ptr [ecx+4]          ; Super父类的地址偏移，10h
mov ecx, dword ptr [ebp-4]          ; this指针
add eax, dword ptr [ecx+edx-8]      ; 这里就是add eax, dword ptr [ecx+8]，也就是Super中的c

mov edx, dword ptr [ebp-4]          ; this指针
add eax, dword ptr [edx-8]          ; Derive中的b
```

可以看到，Derive中的成员变量获取还是很简单的，但是一旦涉及到父类的成员变量，就要先取到虚基类表，然后在通过各种地址偏移得到目标成员变量。我也不知道为什么要这么吃饱了撑得，可能有它自己的权衡吧，或者我猜在多重继承里面会有作用吧。

再往下就开始调用从父类继承过来的`NormalFunc1`函数了，看下给出的汇编指令：
```asm
mov eax, dword ptr [ebp-1Ch]
mov ecx, dword ptr [eax+4]
lea ecx, [ebp+ecx-1Ch]
call HelloWorld2005!Super::NormalFunc1
```

首先我们直到`d`在堆栈上的地址其实是`ebp-1Ch`（从调用构造函数时压入`ecx`中的值得到），那这里第一步就是先拿到`d`的指针（即`this`指针）指向的内容（这里千万要注意，如果是简单的取`this`指针的话,`lea eax, [ebp-1Ch]`就行了，这里还做了一层析址的操作）。但是我们这里调用的是父类继承过来的成员函数，所以要传递给该成员函数的`this`指针必须是指向`Super`部分的指针。

好，接着往下看，第二句`mov ecx, dword ptr [eax+4]`，`eax+4`保存的是`Super`在子类中分布的地址偏移。所以`ecx`理所当然这里就是`10h`。

第三句`lea ecx, [ebp+ecx-1Ch]`，恩，这里`ebp-1Ch`是对象`d`的`this`指针，然后加上偏移量`ecx`，得到的就是`d`中`Super`的内容的地址（好好想一下，有一点绕）。

最后一句就不用我解释了，很简单，简单的调用啦。

好了到目前为止第五章已经完全讲完了，大家慢慢消化吧。

## 写在后面的话
我写这个系列完全是在一点都不懂的情况下写的，大家会看到我会做很多猜想、假设，那只是因为我真的也不知道，并不是在卖关子。很多时候都是一边摸索一边写，把自己摸索的过程写下来，然后尽可能地把各种弯路给去掉，但是大家看起来肯定也会觉得有些绕，没办法，因为我也是在摸索阶段。

我在想等把这个出得差不多了，再出一个和编译器相关的系列文章。然后等把这些坑都填完了之后，就可以去整一整平台相关的东西了。恩恩，谢谢大家的支持。