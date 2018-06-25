---
title: WinDBG观世界（二）
date: 2016-04-20 13:19:11
categories: windbg
tags: [windbg,disassembly,c++]
---

## 导言
前面一章主要讲述了C++函数的调用过程在汇编中的具体执行过程，如果还不熟悉的同学也没关系，后面仍然会不断重复函数的讲解。之所以没有对release版本的exe进行分析，是因为编译器优化后，很多未使用的变量和函数都会被丢弃掉，所以其实优化后的版本没有多少参考意义。对于release的分析将安排到程序稍微复杂一点时进行。

本章的内容仍然很简单，主要引入了struct关键字，所以现在大家可以开始定义自己的结构体了。并且本章还会稍微分析一下指针、引用的区别，以及const在最终代码中的作用。好了，闲话少叙，看看今天的代码吧。
<!--more-->
## 慢慢地趋向结构化
今天的代码内容如下：
```cpp
typedef struct Simple
{
    int a;
    int b;
    char c;
    short d;
    unsigned int e;
} Simple;

void TestValue(Simple s)
{

}

void TestReference(Simple& s)
{

}

void TestPointer(Simple* s)
{

}

void TestConstReference(const Simple& s)
{

}

Simple TestReturn()
{
    Simple s;
    s.a = 10;
    s.b = 20;
    s.c = 30;
    s.d = 40;
    s.e = 50;

    return s;
}

int main(int argc, char** argv)
{
    Simple s = TestReturn();
    TestValue(s);
    TestReference(s);
    TestPointer(&s);
    TestConstReference(s);

    Simple e = {0};

    return 0;
}
```

这次的主人公是结构体，所以函数的实现基本上都是空。大家在继续阅读的同时，最好能够带着下面几个问题：
1. Simple结构体的内存分布
2. 指针和引用的区别
3. const引用和非const引用有什么区别
4. 返回值存放在局部堆栈的什么位置，能不能用ebp将这个位置表示出来
5. 返回值是怎么传递给调用的地方的
6. Simple e = {0}这句代码做了什么事情

如果你上面的问题的答案都能够如数家珍，那么这篇博客不适合你，你可以不用往下面读了。没有了解的同学跟着我一起来看看windows最终是怎么执行我们写的这段代码的吧。

依旧是打开WinDBG，然后加载HelloWorld2005.exe，然后输入`bp HelloWorld2005!main`，让它能够在main函数的时候断下来，输入g运行。按下`alt+7`打开汇编代码，并将`Source Mode`勾选框去掉。做好这一切之后，就可以开始一步一步调试代码了。

F10单步调试，当`main`函数保存完所有寄存器的值之后，然后可以看到第一句话`TestReturn`是从下图开始执行的：
{% asset_img begin.png 主代码开始执行 %}

还记得`ebp`是什么吗？当前函数堆栈的基地址，`ebp~esp`的范围就组成了当前函数的局部变量的存放地址范围（包括过程中可能出现的`push`操作所占用的栈内存）。下面要执行的`lea eax, [ebp-104h]`就是把`ebp-104h`的值赋值给`eax`，（这里注意取内存中的值和取表达式的值是不一样的，`mov eax, qword ptr [ebp-104h]`就是取`ebp-104h`指向地址的内存内容，而`lea eax, [ebp-104h]`则是把`ebp-104`的结果赋值给`eax`）现在并不知道这句话有什么卵用。然后它又把`eax`压栈了，好吧，这个具体什么作用先遗留着，也许以后会用到。

然后终于开始调用TestReturn函数了，我们用F11进入到函数的内部查看一下，这里我就把堆栈的初始化代码给跳过了，如下：
{% asset_img TestReturnAssembly.png TestReturn函数体的内容 %}

好了现在开始就是要接触结构体的相关内容了。在开始这个之前，我想说明的是，WinDBG给我们提供了一个查看结构体内存分布的命令dt（应该是dump type的缩写吧），输入`dt HelloWorld2005!Simple`，可以看到Simple这个结构体在内存中的相对分布情况，如下图所示：
{% asset_img dtHelloWorld2005Simple.png Simple结构体的分布 %}

不如再用一个图来表示一下Simple结构体成员变量的分布情况吧：
{% asset_img SimpleStructMemory.png Simple结构体的分布图 %}

一目了然，在确定了Simple结构体实例的基地址之后（假设为`p`吧），`a`变量就存放在`p+0x000`的位置，`b`变量就存放在`p+0x004`的位置，以此类推。然后 你就会发现`c`和`d`这两个变量共享了同一个DWORD大小的空间，这就是我们以前曾经学过的内存对齐。了解了这些是不是更加清楚了呢，那么我提一个小问题，当成员变量只有一个`char`的时候会发生内存对齐么？当一个`short`和两个`char`作为成员变量的时候，会发生怎么样的对齐？修改一下变量声明的顺序又会出现什么情况？留着自己去发现吧。

言归正传，我们下面看到的几句话是：
```asm
mov dword ptr [ebp-14h], 0Ah ; s.a = 10;
mov dword ptr [ebp-10h], 14h ; s.b = 20;
mov byte ptr [ebp-0Ch], 1Eh  ; s.c = 30;
mov word ptr [ebp-0Ah], 28h  ; s.d = 40;
mov dword ptr [ebp-8], 32h   ; s.e = 50;
```

这里的分号表示注释，我这里在每一句汇编的后面都加上了它所对应的C++代码。诶，第一句的`Simple s;`去哪里了？难道是没有执行？当然不是了。再回到标识符的概念上来，永远记住标识符只是代表了一块内存，它并不占用内存，你可以把标识符就理解为一块内存的名字。而这里s代表的起始就是从`ebp-14h`到`ebp-4`范围的这块内存，根据上面得出的`Simple`的内存结构，其实不难得出`ebp-14h`就是`Simple`实例`s`的基地址，也就是a存放的位置，我们可以用`dd ebp-14h L4`（L4用来声明查看多少个DWORD长度）来查看一下这一块的内存，如下图：
{% asset_img ddebp-14hL4.png s结构体的内容 %}

可以看出从左向右依次是a到e的值，额，貌似有点不对，对于c和d需要重点关注一下。第三个DWORD的内容是`0028cc1e`，首先30的16进制是1eh，40的16进制是28h，我们对应到内存中的内容时，可以发现`0028cc1e`的前两个字节（高字节）保存的是d的值；第三个字节的内容是`cc`，是windows的无效数据；第四个字节保存的是c的值。恩，好像有点明白了，windows采用的是大端模式来保存内存数据的。如果还不明白的话，我们不如用`db ebp-14h L10`来以字节为单位查看内存的内容（db的意思是dump byte，dw表示dump word，dd表示dump dword）:
{% asset_img dbebp-14hL10.png s结构体中字节内容 %}

好，现在是不是发现所有的字节都颠倒了？就拿a变量来说，它标识的是前四个字节的内容，但是我们在上图中看到的这四个字节依次是`0a 00 00 00`，也就是说大端模式下，最低位的字节会放在最高的地址上，所以阅读这个内存的真正顺序是从右往左读，读取四个字节拼接起来。你之所以用dd看到的是正确的顺序，是因为winDBG给你优化了。那么我们继续观察s实例中第三个DWORD的内存内容，现在应该知道怎么读取了吧，成员变量c相对于结构体的偏移地址是0x008，从开头向后数8个字节（当然你如果熟悉命令的话，也可以输入`dd ebp-14h+8 L4`），然后c是一个char，所以只需要读取一个字节，那么它的内容就是`1e`；同样的方式你可以读取一下d的内存内容。这里再拓展一下，不仅读取内存是按照大端的方式来读取，写入内存也是同样的，这个之后再看汇编的时候大家注意一下。

解决了结构体的内存分布问题，我们再看一下成员变量`e`，它和`a`的区别就是多了一个`unsigned`修饰符，但是汇编代码中没有体现出这个特性。编译器出错了？呵呵，这个问题等以后将数值的表示的时候再来慢慢研究（推荐可以去看CSAPP这本书）。

恩，我重新提一下局部变量s表示的内存范围是`ebp-14h`到`ebp-4`，在之后会用到。继续人肉执行代码：
```asm
mov eax, dword ptr [ebp+8]

mov ecx, dword ptr [ebp-14h]
mov dword ptr [eax], ecx

mov edx, dword ptr [ebp-10h]
mov dword ptr [eax+4], edx

mov ecx, dword ptr [ebp-0Ch]
mov dword ptr [eax+8], ecx

mov edx, dword ptr [ebp-8]
mov dword ptr [eax+0Ch], edx
```

第一句代码我现在也没有看懂是干嘛的，但是我知道ebp+8这个地址其实已经到了main函数的局部内存地址空间了，可能是和返回值有关我觉得。往下走，`move ecx, dowrd ptr [ebp-14h]`这里是把变量`a`放到`ecx`中我知道，貌似开始挪数据了。接着的`mov dword ptr [eax], ecx`，就把`ecx`的内容，也就是变量`a`的值拷贝到了eax指向的内存中了。下面的三组语句貌似和这两句相同，都是在拷贝变量的值。但是有5个变量，为什么只拷贝了四次内存呢？我觉得应该不用我解释吧。

下面的一句`mov eax, dowrd ptr [ebp+8]`，又一次把`ebp+8`指向的内存的内容拷贝到了eax中，哎，没有优化的代码就是一堆傻逼操作。但是`ebp+8`保存的究竟是什么呢？额，我忘了保存这些东西了，所以再跟着我`Ctrl+Shift+F5`重新执行一次程序吧，回到main函数调用的地方：
{% asset_img BeforeCallTestReturn.png 调用TestReturn之前 %}

我们画一个图吧，以免忘记了当前堆栈的样子：
{% asset_img BeforeCallTestReturnStack.png 调用TestReturn之前的堆栈 %}

我特意标红了eax中保存的地址对应的地址空间，后面会用到这个地址，现在先忽略它。

好了，然后我们执行`TestReturn`这个函数，堆栈内容如下：
{% asset_img AfterCallTestReturn.png 调用TestReturn之后 %}

在执行完前面三句代码之后，所形成的内存堆栈的图如下：
{% asset_img AfterCallTestReturnStack.png 调用TestReturn之后的堆栈 %}

这样你应该就能清楚地看到`ebp+8`保存的内容就是上一个堆栈中的eax的值，也就是0019fe24，也就是我标红的堆栈空间。但是至于为什么没有从上一个堆栈的头或者尾开始使用，我也暂时不知道，我暂时认为它是乱选的一个空闲内存。

这样的话，经过刚刚的赋值拷贝，结果就是将变量s的内存原封不动拷贝到了`0019fe24~0019fe24+14h`地址空间中，所以你也可以这样理解，`main`函数传递了一个隐形的指针给`TestReturn`函数，而这个指针指向的内存就用于保存返回值。这个问题解决了我们就可以继续往下面执行了。

下面的几句代码如下：
```asm
push edx
mov ecx, ebp
push eax
lea edx, [HelloWorld2005!TestReturn+0x74]
call HelloWorld2005!_RTC_CheckStackVars
```

从函数名能够看出这个其实是检查堆栈的，至于内部具体实现，本次不做任何分析，以后单独立一篇文章来搞定它。只是注意这里之所以要`push eax`，应该是_RTC_CheckStackVars这个函数会用到`eax`这个寄存器来保存返回值，所以说将它放到堆栈上保存起来，以供以后恢复。

继续看下面的代码：
```asm
pop eax
pop edx
pop edi
pop esi
pop ebx
mov esp, ebp
pop ebp
ret
```

这里第一句就是恢复刚刚保存的`eax`的值，至于你想说“它傻么，为什么不先调用_RTC_CheckStackVars，再执行`mov eax, dowrd ptr [ebp+8]`”，额~~~~，好吧它确实很傻。之后的`pop edx`也是同样的作用。然后后面的三句是用来恢复上一个堆栈中的寄存器信息，与函数开始处的`push`对应。然后的两句则是用来恢复上一个堆栈的地址空间的，最后一句`ret`标识返回。

直到这里我们知道了eax中保存着一个指针，它指向了`0019fe24`，之后的14h个字节的地址内容都是返回值的内容，那么我们就继续来看看返回到`main`函数之后又做了哪些事情吧：
{% asset_img AfterTestReturnReturn.png TestReturn返回之后 %}

首先我们再回顾一下它的堆栈空间的内容：
{% asset_img BeforeCallTestReturnStack.png 此时main函数中堆栈分布 %}

然后执行`add esp, 4`，这句话等价于将栈顶元素出栈，只不过栈顶元素没有变量来保存，而只是单纯的丢弃掉。此时栈顶元素就是eax的值，也就是返回值的地址。

看接下来执行的几句代码：
```asm
mov ecx, dword ptr [eax]
mov dword ptr [ebp-11Ch], ecx

mov edx, dword ptr [eax+4]
mov dword ptr [ebp-118h], edx

mov ecx, dword ptr [eax+8]
mov dword ptr [ebp-114h], ecx

mov edx, dword ptr [eax+0Ch]
mov dword ptr [ebp-110h], edx
```

是不是似曾相识呢？和刚刚返回值的拷贝一样对不对，只是这一次是把`0019fe24`处的内存地址拷贝到`ebp-11Ch`。好了往下看还有一样的代码：
```asm
mov eax, dword ptr [ebp-11Ch]
mov dword ptr [ebp-14h], eax

mov ecx, dword ptr [ebp-118h]
mov dword ptr [ebp-10h], ecx

mov edx, dword ptr [ebp-114h]
mov dword ptr [ebp-0Ch], edx

mov eax, dword ptr [ebp-110h]
mov dword ptr [ebp-8], eax
```

这里又发生了一次内存的拷贝，将`ebp-11Ch`的内容拷贝到了`ebp-14h`的地方。所以想想吧，在完全没有开优化的情况下，返回一个结构体居然要拷贝三次内存，确实很浪费内存和时间啊。当然我现在也不知道优化之后会怎么样，可能会变好吧，以后再研究release。

那经过这样一番折腾，返回值终于到了我们熟悉的位置`ebp-14h~ebp-8`这个区间。好吧，它还是给你空出来4个字节，我还没弄清楚这空出来的4个字节究竟能用来干嘛。到现在为止，`TestReturn`函数的调用已经讲完了，大家现在可以慢慢消化一下再继续。

---

跨过华丽的分割线，我们继续看下面的`TestValue`函数（现在s的值保存在`ebp-14h~ebp-8`的区间中）。从准备参数到调用`TestValue`，汇编代码如下：
{% asset_img CallTestValue.png 调用TestValue %}

首先它执行`sub esp, 10h`，应该是给`TestValue`传入的参数预留空间，我猜的。然后又是一坨类似的拷贝代码：
```asm
mov eax, esp

mov ecx, dword ptr [ebp-14h]
mov dowrd ptr [eax], ecx

mov edx, dword ptr [ebp-10h]
mov dword ptr [eax+4], edx

mov ecx, dowrd ptr [ebp-0Ch]
mov dword ptr [eax+8], ecx

mov edx, dword ptr [ebp-8]
mov dword ptr [eax+0Ch], edx
```

我不想详细说了，第一句`eax`保存了`esp`的值，`esp`指向了当前的栈顶。而这段代码就是把`s`的内容拷贝到刚刚预留的堆栈空间中，也就是`esp~esp+10h`这块内存中。所以其实你可以理解为现在堆栈的顶部16个字节保存着s的内容。然后我们就开始调用`TestValue`函数了，记得这里有一个把`TestValue`返回继续执行的地址进行压栈的隐式操作。然而`TestValue`函数的实现我并没有写任何东西，所以说可以直接过了。

所以这里我想让大家知道的是，函数参数的地址是通过堆栈进行传递的，而这个堆栈的内存其实是由上一个函数的局部堆栈提供的，保存在上一个函数局部堆栈的栈顶位置。所以说在函数中使用这些内存的时候，是通过`esp+8`这样的偏移的形式来进行引用的。具体细节大家可以自己写代码进行研究啦，我就不赘述了。

继续往下面执行`TestReference`函数，它执行的代码逻辑如下：
{% asset_img CallTestReference.png 调用TestReference %}

哦哦对，还有一个没有讲，在`TestValue`返回之后，又执行了这样一句代码：`add esp, 10h`，这个其实就是清空按值传递时，堆栈上产生的临时对象的（虽然没有真正清空）。

接着执行`lea eax, [ebp-14h]`，应该还记得`ebp-14h`的位置上保存着什么么？s变量在堆栈上的基地址。把它赋值给`eax`寄存器后，利用`push eax`进行压栈，然后接着就调用了`TestReference`函数。而在函数中你要使用这个栈上变量的方式，和上面相同，也是同样引用`esp+8`指向的值（这里说明下`esp`指向的是上一个函数的堆栈基地址，`esp+4`指向的是该函数返回后继续执行的地址，`esp+8`开始才是上一个函数堆栈的顶部），比如如果这里我需要在函数中执行一个`s.a=10`的语句的话，那么汇编代码可能是如下所示的：
```asm
lea eax, [ebp+8]            ;得到引用的变量的指针
mov ecx, 0ah;               ;将10这个值放到ecx寄存器中
mov dword ptr [eax], ecx    ;将10这个值真正放到目标地址
```

当然这些都是我自己写的，只是为大家做一个演示，如何读取和写入引用变量的内容。如果你要验证的话最好以实际生成的代码为准。

接着往下看`TestPointer`函数的调用，诶，居然和`TestReference`生成的代码一模一样：
{% asset_img CallTestPointer.png 调用TestPointer %}

当然一开始恢复堆栈的那句代码不算啦。所以这个函数我们就不分析了，直接过。

往下再瞄几行，发现调用`TestConstReference`的代码好像也是一模一样：
{% asset_img CallTestConstReference.png 调用TestConstReference %}

好吧，现在应该能够清楚下面几个问题了：
1. 编译器如何实现引用的？ 答案是指针，至少在VS2005上是的。
2. 指针和引用有什么区别？ 好像也没什么区别
3. const引用和引用有什么区别？ 好像也是一样的

那么问题来了，既然指针、引用、const引用都是同一个实现，那么为什么C++还要分那么多东西呢？有想过没有？自己想1分钟，再往下看。

---

好吧，我来说下我的理解，其实指针和引用还是有区别的啊，指针有空指针，引用却没有空引用，这是指针和引用的本质区别。使用引用的一个好处就是，能够从语法的层面杜绝你出现空指针的情况，减少因为空指针导致的崩溃。而使用const引用呢，则是从语义的层面，在编译期就能够判断出你非法的赋值操作，比如说你传了一个const引用给函数，那么你的本意肯定是不希望这个变量被改变，但是你在函数里面对这个const引用赋值了，那么编译器在进行语义分析的时候就能够帮你纠正出这个错误，至于减少bug的话，不同场景不同表现啦。因为有一堆人在遇到这种语义错误的时候，选择的是去掉const，而不是继续调整代码的结构。所以，对于const的使用，见仁见智吧。

最后一个`Simple e = {0}`算是一个额外的扩展研究吧，我们可以看下他的代码：
{% asset_img InitSimple.png 对结构体进行初始化 %}

我们可以看到下面这四句代码是初始化成员变量的内容的：
```asm
mov dword ptr [ebp-2Ch], 0      ;a
mov dword ptr [ebp-28h], eax    ;b
mov dword ptr [ebp-24h], eax    ;c和d
mov dword ptr [ebp-20h], eax    ;e
```

但是为什么第一个变量的初始化方式不同，我也不知道，我也想掀桌啊妈蛋，不过反正结果都一样，VS这样做可能是傻逼了吧。可以知道他其实是以DWORD（4个字节）为单位进行初始化的，而不是一个变量一个变量进行初始化的，其实相当于做了一个`memset`的过程。

你还不知道`eax`是啥？看这句代码`xor eax, eax`，`xor`就是亦或，任何值和自己亦或得到的结果都是0，所以这句代码就已经将eax的值覆盖成0了。至于它和`mov eax, 0`有什么区别，嘿嘿，我们不妨看下面两个语句的二进制代码，都是我从真实的代码里面摘抄的，分号之前的是汇编代码对应的二进制代码，分号后面的是汇编代码：
```asm
bacccccccc      ;mov     edx,0CCCCCCCCh
33c0            ;xor     eax,eax
```

能看出区别来了么，对，保存`mov`语句比保存`xor`语句需要更多的存储空间，所以用xor生成代码更加简短。这里也算是一种优化吧。

那么此时我们不妨看一下e在内存中的内容，利用`dd ebp-2Ch L4`即可：
{% asset_img ddebp-2ChL4.png e在内存中的内容 %}

好吧全都是0，没啥好看的。

那么到这里讲解就全都结束了，至于main函数后面还有的那部分，可以完全参考我上一章将的内容，只做了两件事情，一个就是检查堆栈，另外一个就是恢复上一个函数的堆栈和寄存器。由于该代码过于简单，release版本基本上会被优化得不剩什么代码，所以照旧不讲解release版本的汇编代码。

那么下次再见吧，拜拜！

## 回顾：

### 关于WinDBG
无


### 关于汇编
1. ebp指向了上一个函数堆栈的基地址
2. ebp+4指向了函数返回时，继续执行的地址
3. ebp+8是上一个函数局部堆栈的栈顶，一般函数参数都保存在这个地方
4. 调用函数时，会发生一个隐式的压栈操作，即把`call`指令的下一个指令的地址进行压栈，（其实这个地址的值就是`eip`，在函数ret时，同样也会执行一个`pop eip`的操作）
5. `mov eax, 0`需要5个字节记录，`xor eax, eax`只需要两个字节
6. 区分取寄存器的值（`lea eax, [ebp+8]`）和取寄存器指向地址的内容（`mov eax, dword ptr [ebp+8]`）

### 关于C++
1. 结构体初始化器所做的操作其实和`memset`类似，不会按照变量为单位进行初始化，而是按照DWORD为单位进行初始化
2. 指针、引用在VS2005上的实现并没有什么区别，唯一的区别就是指针可空，引用不可空
3. 引用和const引用在实现上也没有任何区别，只是方便了编译器在语义分析时及时报错