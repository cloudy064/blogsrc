---
title: DUMP下的世界（一）
date: 2016-06-17 23:45:47
categories: dump
tags: [windbg,disassembly,c++,dump]
---

## 前言
讲了那么多用`windbg`调试程序，至今还没有用过`windbg`的最大的作用，那就是分析程序dump，因为我也是个新手，所以只能讲一点皮毛。今天就和大家简单讲一下堆栈吧，如果有任何错误请各位大神指正。闲话不说了，来看看本系列的第一个程序吧，今天的程序仍然没有开优化，因为开了优化之后啥都没了！
<!--more-->
## 草！崩溃了！
当然现在还没有复杂到用真正的一个崩溃`dump`来进行分析，所以我们可以通过汇编指令`int 3`来进行触发一个异常中断，让程序崩溃，并在崩溃的时候取到程序的`dump`文件。本次的代码如下：
```cpp
#include <Windows.h>

void CallFunc3()
{
    _asm{
        int 3;
    }
}

void CallFunc2(const char* name)
{
    UNREFERENCED_PARAMETER(name);
    CallFunc3();
}

void CallFunc1(int a)
{
    UNREFERENCED_PARAMETER(a);
    CallFunc2("joker");
}

DWORD WINAPI ThreadProc(LPVOID lpParam)
{
    UNREFERENCED_PARAMETER(lpParam);

    CallFunc1(30);

    return 0;
}

INT main(INT argc, CHAR* argv[])
{
    HANDLE hThread = ::CreateThread(NULL, 0, &ThreadProc, NULL, 0, NULL);
    system("pause");
    if (hThread != NULL)
    {
        ::CloseHandle(hThread);
        hThread = NULL;
    }

    return EXIT_SUCCESS;
}
```

相当简单，最复杂的地方也就是创建了一个线程而已。好了，我们编译一个程序然后运行，它自己就会崩溃了。我们所需要做的就是打开任务管理器，然后右键目标进程，选择`Create dump File`即可。然后打开`windbg`，按下`Ctrl+D`，选择刚刚的`dump`文件。然后输入所有初学者都会的一个指令`!analyze -v`，然后`windbg`就会把一切和崩溃相关的信息提供给你了！怎么样，很简单吧。但是我们今天主要讨论堆栈，所以就把这个指令放到一边吧！

我们可以通过指令`~*kbn`打印出所有线程的运行时堆栈，我们可以得到下面的结果：
{% asset_img kbnresult.png kbn之后的结果 %}

0号线程永远都是你的`main`函数，这个你不要怀疑，然后1号线程和2号线程的话，不是我们创建的，我也不知道操作系统做了啥，所以忽略。今天的主人公就是3号线程。

这里可以很清楚得看到，该线程从运行了`CallFunc3`之后就抛出了异常：
{% asset_img number3threadstack.png 3号线程的堆栈 %}

这里先打住下，我说下为什么会有今天这个文章。这两天看我们公司的`dump`看到头疼，无一例外都是调用堆栈被破坏，这个时候用`kbn`命令看到的堆栈其实是不完整的，这个时候可能需要用到`kd`指令才行，甚至还需要自己手动恢复堆栈。所以今天就想写一下，如何自己手动恢复一个调用堆栈。

好了，回到程序中来，我们首先要将当前操作的对象选择为3号线程，这就需要用到`~3s`指令（指令我就不解释了，自己翻帮助文档去）。当左下角显示`0:003>`的时候就表示操作成功了。

然后第二步，我们需要知道线程的堆栈范围，这里又要用到`!teb`命令（Thread Environment Block），可以查询到线程的相关信息，其中就包括堆栈的范围，信息如下：
{% asset_img number3threadinfo.png 线程的信息 %}

这里注意的地方就是堆栈它是从大到小增长的，所以你会看到`StackBase`比`StackLimit`大。

最后我们要做的时候就是把这个区间的内存打印出来，用到`dds`指令，输入`dds 0089d000 008a0000`，后面两个数字表示范围，具体数值根据不同人的电脑有所不同，得到的结果因为太长了，自己看吧。

然后看线程堆栈，第一步需要做的事情就是找到`ntdll!_RtlUserThreadStart`，这是`Windows`所有线程开始的地方。在刚刚打印的内存中，可以在最底部看到这个函数，如下所示：
{% asset_img threadstartup.png 线程开始的地方 %}

还记得我们以前在『`windbg`观世界』系列中讲函数的时候，函数调用的过程么？函数调用做了下面几个操作：
1. 将函数的参数压入栈中，把`this`指针（如果有）放到`ecx`寄存器中
2. 将`EIP`寄存器的值（即返回地址）压入堆栈
3. 跳转到函数的起始地址
4. 将前一个堆栈的起始地址压入堆栈
5. 调整`EBP`和`ESP`寄存器，使得`EBP`指向的内容为上一个函数的起始地址

这里可以看到下一个函数的`EBP`寄存器保存的地址上的内容，永远都是上一个函数的`EBP`，那么根据这个`EBP`链，我们就可以把整个函数的调用链都恢复出来。

好了就在这里打住，我们可以想一下这里的`ntdll!_RtlUserThreadStart+0x1b`是个啥？它的代码已经执行到`ntdll!_RtlUserThreadStart`其实地址偏移`0x1b`的地方了，那么可以很容易知道啊，这个地方就是函数的返回地址啊。那么它上面的4个字节的内容肯定就是`_RtlUserThreadStart`堆栈的起始地址了，同时存储该地址的内存地址也是`_RtlUserThreadStart`调用的函数的堆栈的起始地址。我知道有点绕，大家先好好消化下，如果等不及的可以先看后面的一张调用图。

这里看到函数`_RtlUserThreadStart`的`EBP`的值是`0089ffec`，下一个函数（假定为B吧）的`EBP`的值是`0089ffdc`。

那么按照上面的逻辑，如果下一个函数还会继续调用函数C的话，那么C的`EBP`肯定也就会指向B的`EBP`，所以我们现在要做的是，在堆栈上搜索`0089ffdc`，找到C的`EBP`保存的地址。然后向下看4个字节，就是函数C在堆栈上的返回地址：
{% asset_img callstacksecondframe.png 找到下一个调用函数 %}

可以看到`_RtlUserThreadStart`又调用了一次`_RtlUserThreadStart`，然后第二个`_RtlUserThreadStart`的`EBP`的值就是`0089ff94`。

好了，下面我就依次类推了，我把每一步的内容的上下文都记录下来了：
```asm
...
0089f884  0089fcfc
0089f888  004113fe CallStack!CallFunc3+0x1e [d:\workspaces\c++\callstack\callstack\main.cpp @ 6]
...
0089fcfc  0089fdd0
0089fd00  00411443 CallStack!CallFunc2+0x23 [d:\workspaces\c++\callstack\callstack\main.cpp @ 13]
...
0089fdd0  0089fea8
0089fdd4  00411498 CallStack!CallFunc1+0x28 [d:\workspaces\c++\callstack\callstack\main.cpp @ 17]
...
0089fea8  0089ff80
0089feac  004114e5 CallStack!ThreadProc+0x25 [d:\workspaces\c++\callstack\callstack\main.cpp @ 24]
...
0089ff80  0089ff94
0089ff84  76bc3744 kernel32!BaseThreadInitThunk+0x24
...
0089ff94  0089ffdc
0089ff98  77259e54 ntdll!__RtlUserThreadStart+0x2f
...
0089ffdc  0089ffec
0089ffe0  77259e1f ntdll!_RtlUserThreadStart+0x1b
```

然后我们找到了`0089f884`之后，在往上找就没有了。那么到现在为止，大家可以看下是不是和`kbn`得到的堆栈内容一致？（忽略掉异常处理的那些东西）恩，所以说到现在为止线程调用堆栈已经恢复完成了。当然如果真的出现了堆栈破坏，用这种方法也只能恢复部分的调用堆栈，当然也不妨碍大家了解一下的，嘿嘿O(∩_∩)O~

好了，解决了调用堆栈之后，我们还需要知道函数调用的参数。要知道参数压栈的时机实在返回地址压栈之前，所以我们在每个`EBP`地址往下看8个字节，可以看到下面的内容：
```asm
...
0089f884  0089fcfc
0089f888  004113fe CallStack!CallFunc3+0x1e [d:\workspaces\c++\callstack\callstack\main.cpp @ 6]
0089f88c  00000023
...
0089fcfc  0089fdd0
0089fd00  00411443 CallStack!CallFunc2+0x23 [d:\workspaces\c++\callstack\callstack\main.cpp @ 13]
0089fd04  0089fea8
...
0089fdd0  0089fea8
0089fdd4  00411498 CallStack!CallFunc1+0x28 [d:\workspaces\c++\callstack\callstack\main.cpp @ 17]
0089fdd8  0041573c CallStack!`string'
...
0089fea8  0089ff80
0089feac  004114e5 CallStack!ThreadProc+0x25 [d:\workspaces\c++\callstack\callstack\main.cpp @ 24]
0089feb0  0000001e
...
0089ff80  0089ff94
0089ff84  76bc3744 kernel32!BaseThreadInitThunk+0x24
0089ff88  00000000
...
0089ff94  0089ffdc
0089ff98  77259e54 ntdll!__RtlUserThreadStart+0x2f
0089ff9c  00000000
...
0089ffdc  0089ffec
0089ffe0  77259e1f ntdll!_RtlUserThreadStart+0x1b
0089ffe4  ffffffff
```

这里看到`CallStack!ThreadProc+0x25`下面的内容是`0000001e`，也就是代码`CallFunc1(30);`中的`30`。

再好好想想看这里发生了什么，代码运行到了`ThreadProc`中，然后准备调用`CallFunc1`了，那么就会做下面几个操作：
1. 首先就要把参数`30`压栈
2. 然后把返回地址压栈
3. 然后把`EBP`压栈
4. 然后跳转到目标函数`CallFunc1`

所以现在大家应该明白为什么这里的`0000001e`就是那个`CallFunc1`的参数了吧！

其他的参数解析就留着大家去分析吧，这里注意一点就是，字符串需要通过`da`或`du`来查看内容

好了，『`dump`下的世界』第一章就这么结束了，想想其实也没啥内容，主要是准备得仓促了点，下次一定好好准备！