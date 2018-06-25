---
title: WinDBG观世界（三）
date: 2016-04-21 00:48:07
categories: windbg
tags: [windbg,disassembly,c++]
---

## 导言
已经介绍了函数和结构体的内容了，我想这次应该可以大致研究一下类相关的知识了。不过这次照例还是没有release版本的分析内容，而且这次讲类也只是讲解基础知识，对于类的高级用法，比如继承、成员函数指针这些都留到以后去研究。那么进入正题吧（貌似导言越来越短了，我在想以后要不要去掉）。
<!--more-->
## 给结构体加上一点控制！再加上一点函数！
国际惯例，先贴分析的代码：
```cpp
class SimpleClass
{
public:
    SimpleClass(int m)
        : a(m)
    {
        b = 3;
    }

    SimpleClass(const SimpleClass& rhs)
        : a(rhs.a)
        , b(rhs.b)
    {

    }

    ~SimpleClass()
    {

    }

public:
    void TestMemberFunc()
    {
        a += b;
    }

    void TestConstMemberFunc() const
    {
        int m = a;
        m += b;
    }

public:
    virtual void TestVirtual(int m)
    {
        a += m;
    }

public:
    static int TestStatic(int m, int n)
    {
        return m + n;
    }

private:
    int a;
    int b;

private:
    static const char* str;
};

const char* SimpleClass::str = "joker";

SimpleClass k(12);

int main(int argc, char** argv)
{
    SimpleClass j = 233;
    SimpleClass k = ::k;
    SimpleClass a(k);

    k.TestMemberFunc();
    k.TestConstMemberFunc();

    a.TestVirtual(14);

    int d = SimpleClass::TestStatic(12, 13);

    return 0;
}
```

这次需要分析的东西还是挺多的，我把几个需要带到分析过程中的问题罗列出来吧：
1. 全局变量什么时候初始化
2. 类的初始化器和普通的赋值有什么不同
3. 成员变量的初始化顺序是什么
4. 没有给初始化值的成员变量编译器会做什么操作
5. 静态函数存放在什么地方，是否存放在类空间内部
6. 静态成员变量又存放在什么地方，是否存放在类空间的内部
7. 虚函数和普通成员函数有什么区别
8. 普通成员函数和const成员函数有什么区别
9. 静态函数和普通成员函数有什么区别
10. 全局变量和局部变量编译器是怎么区分开的

看了下真的是任重而道远，其实我已经做了很多压缩了，比如“静态成员变量和全局变量谁先被初始化”这个问题由于难度问题，没有把它列进来。

那么我们先挑个最恶心人的问题来吧，全局变量什么时候初始化。要想了解这个问题，最好的方法就是在类的构造函数处下一个断点，来，跟着我输入指令`bp HelloWorld2005!SimpleClass::SimpleClass`，然后回车。好吧，坑爹了，WinDBG认为这个断点有歧义，因为`SimpleClass`类有两个构造函数，所以它不知道给哪个构造函数下断点，哎，真是笨：
{% asset_img SetConstructorBreakPoint.png 设置构造函数断点 %}

没办法了，只能根据WinDBG给我们的提示，用构造函数的具体地址来设置断点，来输入`bp 00401140`，然后输入`g`继续执行代码，直到运行到断点处：
{% asset_img HitConstructorBreakPoint.png 执行到断点处 %}

这时我们只是想直到这个构造函数是谁调用过来的，那么我们就需要直到函数调用的堆栈，输入`kbn`回车即可：
{% asset_img GlobalVarCallStack.png 全局变量构造时的调用堆栈 %}

可以看到它是由一个叫做`_initterm`的函数调用过来的，上面的哪个`dynamic initializer for 'k'`可以先忽略，那个就是初始化全局变量k的地方，简单来说这个函数就对应了SimpleClass k(12)这句代码。那么我们现在要做的事情就是在_initterm这个函数调用的地方下一个断点。好了，按下`Ctrl+Shift+F5`重启程序，然后输入`bp MSVCR80D!_initterm`，输入`g`回车即可：
{% asset_img InInitTermFunction.png 调用_initterm的地方 %}

前面保存堆栈就不用看了，主要看下面这块
```asm
mov eax, dword ptr [ebp+8]
cmp eax, dword ptr [ebp+0Ch]
jae MSVCR80D!_initterm+0x25
```

注意`ebp+8`指向了上一个堆栈的栈顶的第一个DWORD，`ebp+0Ch`指向了上一个堆栈栈顶的第二个DWORD，其实应该也是传递给`_initterm`函数的两个参数，这里只是比较这两个数值是否相等。这时肯定有人会问了，`cmp`比较之后的结果存放在哪里呢？当然是寄存器了，只是这个寄存器比较特殊，它叫标志寄存器，你平时在WinDBG中看不到这个寄存器，但是你可以通过按下`Alt+4`弹出一个罗列了所有寄存器的对话框：
{% asset_img AllRegisters.png 所有的寄存器 %}

确实相当多，不过这里我只介绍`cmp`会用到的，一个是进位标记`cf`（carry flag）和零位标记`zf`（zero flag）。`cmp`比较下来不过就是小中大三种结果，那么对应到标志寄存器上的表现如下：
{% asset_img CmpResult.png cmp比较的结果 %}

那么我们这里执行完`cmp`之后看到这两个寄存器的内容如下：
{% asset_img AfterCmpOperation.png cmp执行之后寄存器的内容 %}

我们再人肉比较一下这两个值吧：
{% asset_img ActualValueForCmp.png 两个值的具体内容 %}

可以发现`ebp+8`的值小于`ebp+0Ch`的值，所以说`zf`应该是0，`cf`应该是1，对照寄存器的内容，怎么样，我没有忽悠你吧。

下面是一条新的指令`jae MSVCR80D!_initterm+0x25`，当然其实我现在也记不住那么多指令，只能依靠谷歌和翻书。`jae`指令的意思是：基于无符号数的比较结果，如果大于等于就执行后面的跳转。而前面比较的结果是小于，所以不会执行后面的跳转，指令选择继续往下执行。同样的指令还有很多，我这里不想介绍太多，大家可以自行翻书。

下面三条语句也执行了同样的操作
```asm
mov ecx, dword ptr [ebp+8]
cmp dword ptr [ecx], 0
je MSVCR80D!_initterm+0x1a
```

这里的意思也差不多，它先取`ebp+8`指向地址的值，把它放到ecx中，然后在将这个值指向的值与0进行比较，如果等于0的话就跳转到`MSVCR80D!_initterm+0x1a`处执行。这里具体比较内容和结果自己去探索，我这里就看到它执行了后面的跳转。然后他又执行了下面的这段代码：
```asm
mov ecx, dword ptr [ebp+8]
add ecx, 4
jmp MSVCR80D!_initterm+0x3
```

它又一次取了`ebp+8`的值，并把它加4,所以更加肯定了`ebp+8`存放的是一个指针，这里只是对指针做了一个偏移操作。然后将`ecx`的值又放回了`ebp+8`指向的地址，接着跳转到了`_initterm+0x3`的位置，也就是下面这段代码：
{% asset_img InitTermJmpBackToFuncHead.png 跳转回去开头 %}

好了，整个函数就只剩下一段汇编没有分析了：
```asm
mov edx, dword ptr [ebp+8]
mov eax, dword ptr [edx]
call eax
```

看到`call eax`第一反应应该就是这肯定是调用了一个函数，只是这个函数可能是通过参数传递进来的。这里继续引用`ebp+8`指向的值，放到edx中，然后把这个值指向的内存内容放到eax中，而这个值则正好是一个函数的地址。

因此我有理由相信，这个函数接受两个参数，第一个是`ebp+8`,第二个是`ebp+0Ch`，并且从`ebp+8`到`ebp+0Ch`这个范围中存放了一系列的函数的地址，而这个函数做的事情就是不断地循环，如果遇到不是空指针的就调用这个函数。我把这段函数具体化为C++代码，应该大致如下：
```cpp
void _initterm(void* from, void* to)
{
    while (from < to)
    {
        if (*from != NULL)
        {
            functype func = (functype)*from;
            func();
        }

        from++;
    }
}
```

其中functype我也不知道是什么类型，所以就遗留着吧。搞定了这个函数的含义，下面就好办了，就是不断地尝试进入函数体的内部，直到调用到了`SimpleClass`的构造函数位置。

那我们调用到`eax`中函数的时候，就用F11调试进入函数内部（记得用`kbn`看下调用堆栈，以防调试错了地方，因为一共会调用两次`_initterm`函数，我们需要分析的是`__tmainCRTStartup`调用过来的`_initterm`函数）

第一次调进来好像是调到了一个叫做`pre_cpp_init`的函数中了，这货居然还是HelloWorld2005中的函数，看看也无妨，可以更加了解一下运行的机制：
{% asset_img HelloWorld2005PreCppInit.png pre_cpp_init函数 %}

国际惯例，前面两句保存堆栈，但是注意这里其实少了我们以前看到的那些`sub esp 120h`这样的语句，额，可能是没有声明局部变量？或者是被优化掉了？以后讨论。然后它将一个叫做`_RTC_Terminate`的函数进行压栈，这个函数我查了好久都没有找到是干嘛的，有熟悉的可以告诉我一下(cloudy064@gmail.com)，或者等我加了评论模块再说，万分感谢。

然后就是熟悉的两个个操作：
```asm
call HelloWorld2005!atexit
add esp, 4
```

现在我们应该会知道之前压栈的`_RTC_Terminate`是一个`atexit`需要用到的参数，然后`pre_cpp_init`调用`atexit`函数，返回之后，利用`add esp, 4`清除临时变量，恢复堆栈。这里我就不进去`atexit`里面看了，我觉得会吐血，以后安排补丁文章来搞定这些一直没有分析的问题吧。

继续看后面几句：
```asm
mov eax, dword ptr [HelloWorld2005!_newmode]
mov dword ptr [HelloWorld2005!startinfo], eax
push offset HelloWorld2005!startinfo
```

依据以前的经验，这里的`HelloWorld2005!_newmode`和`HelloWorld2005!startinfo`应该是`HelloWorld2005`里面的全局变量，虽然我不知道它是在哪里初始化的，但是不妨碍我们继续分析。我们可以通过`dv`（dump value）指令来查看这两个变量的内容：
{% asset_img PreCppInitDvResult.png 用dv查看结果 %}

发现我们只能看到startinfo的类型信息，却得不到它的内容，这个时候应该介绍新指令:`dt`（dump type），这个指令可以在变量类型为符合类型（比如结构体）的时候发挥作用。利用`dt`指令查看到的结果如下：
{% asset_img PreCppInitDtValueResult.png dt查看得到的结果%}

恩，这下能够看到它的成员变量的值这些信息了。额，你还不放心，还想看它的结构体内部变量是怎么分布的？没问题，用`dt _startupinfo`来查看就行了，这里我就不贴出来结果了，大家自己试验吧。

所以这里就只是初始化了全局变量`startinfo`，最后一句`push offset HelloWorld2005!startinfo`你就可以把它理解为将`startinfo`的地址入栈了。将一个变量或者地址进行入栈，在没有优化的情况下，十有八九就是准备调用函数了。

接着看下面的语句中，有连续好几个压栈的操作，然后接一个函数调用，最后恢复堆栈：
```asm
mov ecx, dword ptr [HelloWorld2005!_dowildcard]
push ecx
push offset HelloWorld2005!envp
push offset HelloWorld2005!argv
push offset HelloWorld2005!argc
call dword ptr [HelloWorld2005!_imp____getmainargs]
add esp, 14h
```

当然这些也全部都是全局变量了，不过第一个是将`_dowildcard`这个变量压栈，下面三个是将`envp`,`argv`,`argc`的内存地址进行压栈，所以你可以把`_imp____getmainargs`函数看做是下面这个签名：
```cpp
void _imp____getmainargs(
    void* argc, 
    void* argv, 
    void* envp, 
    int _dowildcard,
    _startupinfo* startinfo)
{
//.......
}
```

关于函数调用的参数顺序和入栈顺序以后会安排专题详细分析，这里就当参数是这个顺序吧。而这个函数从名字上来看就是用来初始化参数信息的，和我们的全局变量没有多大的关系，所以就不进去继续看了，直接跳过。

往下再看几行：
```asm
mov dword ptr [HelloWorld2005!argret], eax
cmp dword ptr [HelloWorld2005!argret], 0
jge HelloWorld2005!pre_cpp_init+0x56
```

这里eax应该存放了刚刚调用`_imp____getmainargs`的返回值，之所以把它存放到另外一个全局变量里面，恩，可能是因为没有优化吧= =。这三句概括起来就是将返回值和0进行比较，如果返回值大于等于0（`jge`的意思就是jump if greater than or equal），那么就跳转到`pre_cpp_init+0x56`的位置；否则就继续执行。

还剩下最后一段代码了：
```asm
push 8
call HelloWorld2005!amsg_exit
add esp, 4
```

这段应该是强迫程序退出的一个函数吧，8是错误码。意思就是说，如果上面的返回值小于0，那么肯定就是程序出现了什么错误了，然后执行到这里就让程序已错误码8的结果退出。

好了，我们整理一下这个`pre_cpp_init`函数，其实可以用C++代码表示成下面这个样子：
```cpp
void pre_cpp_init()
{
    atexit(&_RTC_Terminate);
    startinfo.newmode = _newmode;
    argret = _imp____getmainargs(&argc, &argv, &envp, _dowildcard, &startinfo);
    if (argret >= 0)
        return;

    amsg_exit(8);
}
```

虽然可能和真实实现会有不一样，但是大致的流程应该不会错的。

好吧，分析了半天我们还是没有找到调用`SimpleClass`构造函数的地方。不要急嘛，函数返回了继续看下去。

第二次进入`call eax`，我们这个时候就看到了全局变量k的初始化了，代码如下所示：
{% asset_img DynamicInitializerForK.png 初始化全局变量K %}

又是人肉分析代码的艰辛历程，前面一大坨就是我们以前熟悉的，先保存上一个堆栈以及寄存器的内容，然后分配一个`30h`大小局部变量堆栈空间给当前函数（我也不知道函数名叫啥），然后将这块内存初始化为`0xCCCCCCCC`，下面的这段代码才是真正执行构造函数的地方：
```cpp
push 0Ch
mov ecx, offset HelloWorld2005!k
call HelloWorld2005!SimpleClass::SimpleClass
```

这里其实可以看到在调用构造函数之前，编译器已经确定好全局变量`k`应该存放在什么位置了，而我们调用构造函数只是初始化`k`这块内存区域而已。这里讲12这个数进行压栈，然后将`k`的地址保存到`ecx`寄存器中，我猜ecx保存就一定是this指针！不然构造函数不知道初始化哪片内存。最后我们调用了构造函数。因为这里二进制代码已经写死了调用函数的地址，而不是通过函数名进行调用，所以不存在函数名有二义性的问题。

我们进入到了构造函数里面，看到如下的代码：
{% asset_img SimpleClassConstructorCode.png SimpleClass的构造函数 %}

前面里面有一个不同，那就是多了一句`push ecx`，要知道后面再执行初始化堆栈的时候，需要使用到`ecx`这个计数器的，而ecx之前保存了this指针的地址，所以说需要先将`ecx`的值压栈进行保存，这样才能保证this指针地址不会丢失。然后继续往下看，看到`pop ecx`处，这里就是执行一次出栈操作，恢复`ecx`中原有的this指针。

接着函数执行了`mov dword ptr [ebp-8], ecx`这句话，要知道在没有开优化的情况下，`ebp-8`这个位置是第一个局部变量声明的位置，也就是说其实构造函数中隐藏了一个定义变量的操作`SimpleClass* this = ecx;`，所以现在大家知道this指针是怎么来的了吗？它保存在哪里也应该知道了吧。

继续看，下面一句话是`mov eax, dword ptr [ebp-8]`，我要掀桌了（(╯‵□′)╯︵┻━┻），好好一个this指针到处传，就不能一步到位么。把桌子摆好，努力思考一下，我觉得将this指针放在局部变量的位置，可能是为了将this指针保存起来，能够随时获取，因为寄存器是经常变化的，一堆操作需要用到寄存器。而把this指针放到寄存器中，则是因为对于`dword ptr`这种类似于析址操作只支持寄存器的。哎，Intel的指令就是这么麻烦（我感到背后有一股杀气）。

接下来就开始进行调用初始化器以及调用函数体了：
```asm
mov dword ptr [eax], offset HelloWorld2005!SimpleClass::`vftable'
```

= =天哪，忘记了还有这货，这个看名字就知道是SimpleClass的虚函数表，因为我这里声明了一个虚函数，所以说这个类就会有一个虚函数表来承载虚函数的地址。而这个虚函数表就存放在this指针的基地址的位置。我们现在可以简单瞄一眼虚函数表，利用`dd eax`可以看到this指针指向内存的内容：
{% asset_img SimpleClassThisPointToMemory.png this指针指向的内容 %}

第一个值就是虚表的地址了，至于虚拟地址里面存放了什么，我现在还不想研究，等真正讲到虚函数的时候再说吧。

然后继续往下看：
```asm
mov eax, dword ptr [ebp-8] ;取this指针
mov ecx, dword ptr [ebp+8] ;获取最后一个压栈的参数，这里只有一个就是12
mov dword ptr [eax+4], ecx ;给this指针偏移四个字节的地址进行赋值
```

前面两个都比较好理解吧，就是单纯的取值，对于最后一个的话，恩，看一看SimpleClass的内存分布大家应该就了解了，输入`dt SimpleClass`即可：
{% asset_img DtSimpleClass.png SimpleClass的内存分布 %}

可以很容易看到，对于一个具体的`SimpleClass`实例，它的基地址存放了虚函数表指针，偏移4字节的地方存放了`a`变量的值，偏移8字节的地方存放了`b`变量的值。额最后一个就太奇怪了，它给我们的信息是它存放在一个固定的地址，并且它的类型是(null)，明明是`char*`好吧。所以它的内存结构用图来表示的话如下：
{% asset_img SimpleClassMemoryMap.png SimpleClass的内存结构图 %}

最后那个静态变量不管它，讲到它的时候再说。

所以上面的那句`mov dword ptr [eax+4], ecx`其实就是给成员变量`a`进行赋值，所以这三句其实就对应了代码中的初始化器`:a(m)`了。

然后在往下看两行：
```asm
mov eax, dowrd ptr [ebp-8]
mov dword ptr [eax+8], 3
```

不用我说大家也明白了，这个就是给成员变量`b`进行赋值。

那么这么一看，其实初始化器和在构造函数体中对变量进行初始化并没有什么区别啊。当然有！最明显的区别就是初始化的顺序，这个顺序后面会用多个例子来进行试验，这里先不说；另外一个区别就是初始化器调用的是构造函数，而函数体中的初始化调用的是赋值操作符，因为这里都是`int`，所以无法看出区别。

好了分析完成构造函数了。等等，最后那个`ret`后面怎么多了个4，以前都是没有的啊。嗯嗯，这里的4只是恢复堆栈用的，这个具体会在函数调用约定里面详细讲解。这里只要知道`ret 4`其实就相当于下面两条指令就行了:
```asm
ret
add esp, 4
```

---

画一个分隔线，只是因为前面讲的太多了，其实这里还有一个问题没有讲清楚，就是那个`dynamic initializer for 'k'`的地址是谁传递过来的，这又是一个坑，还是得填啊。但是我现在不想填，好麻烦= =，留着以后想了解main函数之前做了什么的时候再来看这个坑吧，恩。（我中途发现了这样一个东西[http://wiki.osdev.org/Visual_C%2B%2B_Runtime](http://wiki.osdev.org/Visual_C%2B%2B_Runtime)，打开有兴趣可以参考参考）

然后不知不觉我们就来到了`main`函数调用的地方：
{% asset_img ThePlaceWhereCallMain.png main函数调用的地放 %}

我们仍然可以看一下`main`函数它完整的函数签名是什么：
```asm
mov eax, dword ptr [HelloWorld2005!envp]
push eax

mov ecx, dword ptr [HelloWorld2005!argv]
push ecx

mov edx, dword ptr [HelloWorld2005!argc]
push edx

call HelloWorld2005!main 
```

可以看到它其实传给main函数三个参数，分别是`argc`,`argv`,`envp`，至于具体什么意思，自己百度一下啦。

现在我们进入到`main`函数的主体，并定位到第一句代码执行的位置，如下：
{% asset_img MainSimpleClassJ=233.png main函数执行的第一句代码 %}

准备数据没啥好说的，将`0E9h`(也就是233)进行压栈，然后将变量`j`的栈上地址，也就是this指针的值传递给`SimpleClass`的构造函数：
```asm
push 0E9h
lea ecx, [ebp-1Ch]
call HelloWorld2005!SimpleClass::SimpleClass
```

这里唯一的问题就是为什么`j`是保存在那个地址上，我们来算嘛。首先我们根据上面各种语句，可以得到当前函数局部堆栈的内存分布图如下：
{% asset_img MainSimpleClassJ=233Stack.png 此时的堆栈情况 %}

我把函数参数的区域标为黄色，分配给变量j的内存标为红色。好吧，这样看来没优化的时候，系统真的是有浪费4字节内存的习惯。

然后代码就开始调用`SimpleClass`构造函数了，如此看来单参数构造函数的调用，下面两种调用方法所生成的汇编代码是相同的（此处只针对了系统内置类型，结构体的之后再验证）：
```cpp
SimpleClass a(10);
SimpleClass b = 10;
```

构造函数已经详细分析过了，所以就不进去继续看了。

看下一句代码`SimpleClass k = ::k;`，这个从C++的层面来看的话，他应该调用我写的拷贝构造函数才对，那么我们来看对应的汇编代码吧：
```asm
push offset HelloWorld2005!k
lea ecx, [ebp-30h]
call HelloWorld2005!SimpleClass::SimpleClass
```

额，它好像没有对局部变量和全局变量进行直接的区分，因为，编译器TM就没有给局部变量取名字！！！局部变量是没有符号的。好吧，我的哪个局部变量和全局变量什么区别的问题就不是问题了。至于分配内存我也就不说了。这里主要就说明一下`SimpleClass`的拷贝构造函数好了：
{% asset_img SimpleClassCopyConstructorCode.png 拷贝构造函数 %}

我们直接跳到`004011b3`这一行
```asm
mov eax, dword ptr [ebp-8]
mov dword ptr [eax], offset HelloWorld2005!SimpleClass::`vftable'

mov eax, dword ptr [ebp-8] ; 拿到this指针
mov ecx, dword ptr [ebp+8] ; 拿到全局变量k的地址
mov edx, dword ptr [ecx+4] ; edx = k.a
mov dword ptr [eax+4], edx ; this.a = edx

mov eax, dword ptr [ebp-8] ; 拿到this指针
mov ecx, dword ptr [ebp+8] ; 拿到全局变量k的地址
mov edx, dword ptr [ecx+8] ; edx = k.b
mov dword ptr [eax+8], edx ; this.b = edx

mov eax, dword ptr [ebp-8]
```

= =好复杂的感觉，一句一句分析，此时`ebp-8`中保存的就是this指针的值。然后前面两句的意思就是初始化this指针的虚函数表，没有任何疑问。

接下来的四句，因为在调用拷贝构造函数之前调用了一句`push offset HelloWorld2005!k`，所以现在上一个堆栈的顶部保存的其实就是全局变量`k`的地址。所以这里`mov ecx, dword ptr [ebp+8]`就是讲全局变量`k`的地址保存到`ecx`寄存器中。所以有了上面的注释之后基本上就不需要我讲解了。拷贝构造函数搞定！

再往下看`Simple Class a(k)`的汇编代码：
```asm
lea     eax, [ebp-30h]  ; 将局部变量k的地址保存到eax
push    eax             ; 将eax的值压栈
lea     ecx, [ebp-44h]  ; 分配局部变量内存，从ebp-44h开始的12个字节
call    HelloWorld2005!SimpleClass:SimpleClass  ;调用拷贝构造函数
```

不解释，依然和上面的构造过程一模一样。所以其实拷贝构造函数的调用下面两种方式也是完全相同的：
```cpp
SimpleClass a = k;
SimpleClass b(k);
```

---

一个华丽的分割线之后，开始看函数的调用，我们这里主要是看各个函数之间的区别，对于函数主体，我们其实并不关注，看前面两个函数:
{% asset_img MainCompareTestAndTestConst.png 比较const和非const函数 %}

他们几乎完全一样，通过ecx传递指针，然后调用对应的函数，const函数最后的const修饰符可以说是没有起到任何作用，要知道这个const是用来修饰this指针的，也就是说const成员函数中你不能对this指针指向的成员进行修改。但是就如同我上一章里面说的，const只是从语义上让你能够在编译期就能发现问题，要知道编译期找bug的代价比运行期找bug代价小很多的。

这一次我们不去看它真正的实现，我们不如自己来写一个TestMemberFunc出来？
```asm
push ebp        ; 保存上一个堆栈的基地址
mov ebp, esp    ; 确定新的堆栈基地址
sub esp, 0C0h   ; 分配足够的堆栈，其实我觉得这里没有用到堆栈啊，可以不用这一步
push xxx        ; 忘记是啥
push xxx        ; 忘记是啥
push xxx        ; 忘记是啥
push ecx        ; 保存this指针
; 初始化这块堆栈区域，忘记什么命令了
pop ecx         ; 恢复this指针
mov dword ptr [ebp-8], ecx ; 将this指针保存到栈底
mov eax, dword ptr [ebp-8] ; 将this指针拷贝到eax中
mov ecx, dword ptr [eax+4] ; 拿到a的值
add ecx, dword ptr [eax+8] ; 加上b的值
mov dword ptr [eax+4], ecx ; 将得到的和保存到a的位置中
pop xxx
pop xxx
pop xxx 
mov esp, ebp
pop ebp
ret
```

我觉得差不多应该是这个样子，不管那么多了。这个只是一个我自己想做的一个练习而已，继续往下面执行。

接着就是虚函数，汇编代码如下图所示:
{% asset_img MainCallTestVirtual.png 调用虚函数 %}

好像我还是觉得没什么特别的地方，通过`push`传递参数，通过`ecx`传递this指针，进去函数实现的地方看也和普通函数的调用没有什么区别。好吧，可能要等到用到继承的时候才会出现区别吧。

所以在这种没有继承的情况下，成员函数，const成员函数和虚函数之间完全是没有任何区别的。

看完最后一个`int d = SimpleClass::TestStatic(12, 13)`就可以解放了，相信看了汇编代码的人会觉得，这个函数是最没有技术含量的了：
{% asset_img MainCallTestStatic.png 调用静态函数 %}

简单来说就和普通的全局函数是一模一样的，唯一的区别就是静态函数前面要加一个类名加一个域操作符。

啊！终于都分析完了。“诶，好像还有一个静态成员变量你还没说吧？”，我。。。。。。你这个同学记性能不能放差一点，我现在好累了= =

好的，之前我们用`dt`命令查看了`SimpleClass`的内存结构，我们再看一下：
{% asset_img DtSimpleClass.png SimpleClass的内存结构 %}

发现str的地址并不是一个偏移量，而是一个固定的地址`00406000`，记住这个是`str`变量保存的地址，而不是`"joker"`这个字符串的地址。如果要确定它指向字符串的地址，那么就输入`dd 0040600`：
{% asset_img dd00406000.png str指向的内存内容 %}

拿到第一个值，这个值就是`"joker"`存放的地址了，继续输入`da 0040412c`（`da`的意思应该是dump ansi）：
{% asset_img da0040412c.png 查看str指向的字符串 %}

好吧，既然讲到静态变量了，那么我问大家一个问题，静态变量存放在程序的什么区段：
1. text段
2. rdata段
3. data段
4. rsrc段

能搞明白么？好吧，为了寻找这个问题的答案，我们不妨打印出这个程序所有的头部信息吧。利用`dh! -s HelloWorld2005`（dh的意思是表示dump header,-s表示只展示区段信息）
{% asset_img DumpHelloWorldHeaders.png 程序的头部区段信息 %}

但是这里所有的数据都是相对于HelloWorld2005.exe最开始的地方，
{% asset_img lmvmHelloWorld2005.png HelloWorld2005.exe在程序中的真实地址范围 %}

那这样的话我们就能确认到各个区段的实际内存范围：
```
text段       00401000~0040313C
rdata段      00404000~0040515C
data段       00406000~00406418
rsrc段       00407000~004071B4
```

先来看`str`变量的地址，是`00406000`，它好像正好就是data段的起始地址，毫无疑问了那就，静态成员变量被存放在data段中。

我们再来看看`"joker"`这个字符串，它的地址是`0040412C`，而这个地址落在了rdata段中。我们不妨把rdata段中的所有的字符串都列举出来，眼见为实。我们可以用`s -sa HelloWorld2005+4000 L115C`命令来搜索这个rdata内存区间里面所有的ANSI字符串，我这里输出的结果如下：
{% asset_img s-saHelloWorld2005+4000L115C.png rdata段中的所有ANSI字符串 %}

毫无悬念你可以找到`"joker"`这个字符串。

好了这些就把所有的东西都讲完了，遗留下来的坑，下一个章节会填完。大家慢慢消化吧，拜拜！

## 回顾：

### 关于WinDBG
1. `dv`命令用于查看局部变量，如果想查看全局变量记得前面加上`模块名!`
2. `dt`命令用于查看结构体的分布情况，也可以以结构体来解析对应的内存
3. `!dh`命令可以用于查看模块的头部信息，具体参数请自行补充
4. `s`命令可以用于搜索内存，具体参数请自行补充


### 关于汇编
1. `ret 4`指令在返回的同时，也会将栈顶指针加4（清空堆栈）
2. 在调用类函数的时候，包括构造函数，通过`ecx`寄存器传递this指针，并在函数内部，将this指针的值保存在`ebp-8`的位置


### 关于C++
1. 全局变量在是main函数之前初始化的
2. 全局变量的内存是在编译期就已经被决定了的
3. 类的虚函数表是在构造函数中进行初始化的
4. 虚函数表放在this指针指向的基地址位置
5. const成员函数和非const成员函数在实现上并没有任何不同，只是便于编译器能够在编译期就把问题抛出来
6. 在没有继承的情况下，成员函数和虚函数并没有什么不同
7. 静态函数和普通函数也没有什么不同，只是静态函数调用的时候需要加上类名加上域作用符
8. 静态成员变量存放在`data`区段
9. 常量字符串存放在`rdata`区段
10. main函数的完整签名是int main(int argc, char** argv, char** envp);
11. 函数入口并不存放在类的结构中