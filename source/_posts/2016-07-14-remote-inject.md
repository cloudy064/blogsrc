---
title: 云064与你一起玩注入（一）
date: 2016-07-14 00:58:16
categories: Windows
tags: [windows,c++,inject]
---

## 导言
写这个系列文章主要是为了学习下Windows中了解但是还不是很熟悉的知识及其原理，并且因为这两年的工作大部分都体现在注入上了。说起注入，这个真的是病毒、杀软、监控等不可或缺的一门技术。知道你的Windows为什么这么慢么？你以为加内存就可以了？你以为仅仅是Windows卡死了？呵呵。。。

为了能让大家能有目的地去接触注入这门技术，这个系列我准备最终做一个监控其他模块注入的小工具吧。至于技术上能不能实现出来，那就是另外一回事了。另外，我提供的代码会用`Windows Native API`先实现一版；然后再对代码进行优化，但是过程中不会用第三方库，包括`STL`；最后提供利用第三方库实现的版本。所有代码都可以在我的[github](https://github.com/cloudy064/BlogCodes.git)上找到。好了，闲话不多说，开始今天的主题吧。
<!-- more -->

## 一些废话
本着从简单到复杂的原则，我决定这一章只说明一部分原理。（你根本只是想偷懒啊喂！

注入的方式有很多种，目前我知道的有如下几种方式：
1. Windows提供的API:`SetWindowsHookEx`
2. 远程线程注入
3. 消息钩子

而我这一章讲的是『远程线程注入』。因为我觉得这个是最简单的，也是最好理解的，它的原理我可以简单一句话描述：『在目标进程中创建一个线程，让它只做一件事：加载我们想要注入的dll』。

那么这句话描述的功能中，我们需要推敲几个点：
1. 我在其他进程中，怎么样才能让目标进程为我创建一个线程？
2. 目标进程怎么知道我要注入的dll在哪里？
3. 就算它知道我的dll路径，它怎么知道`LoadLibrary`的地址？

我知道大家现在肯定很多疑问，比如『为什么注入的模块一定是dll』，这些以后解答吧，因为我也还存在一些疑问。所以这里你看到的只有代码，这里你只需要跟着我的代码一直敲下去看效果就行了。

在做注入之前，我们首先需要一个被注入的进程，我们的老朋友`Hello World`登场，源码如下：
```cpp
int main()
{
    system("pause"); //只是为了让进程不退出，能够让我们注入
}
```

接着我们需要准备一个想要注入进去的dll（怎么创建工程自己百度去），代码也很简单：
```cpp
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        printf("i am injected");
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

它在注入到目标进程之后，会在控制台输出一句话。但是现在它和目标进程没有任何关系。

## 基础准备
所以，接下来就是今天需要重点说明的远程注入所需要用到的`API`:

### 创建线程
远程注入的基础是要能够在目标进程中运行代码，那么首当其中的就是创建注入线程的`API`:
```cpp
HANDLE
WINAPI
CreateRemoteThread(
    _In_ HANDLE hProcess,
    _In_opt_ LPSECURITY_ATTRIBUTES lpThreadAttributes,
    _In_ SIZE_T dwStackSize,
    _In_ LPTHREAD_START_ROUTINE lpStartAddress,
    _In_opt_ LPVOID lpParameter,
    _In_ DWORD dwCreationFlags,
    _Out_opt_ LPDWORD lpThreadId
    );
```

这个函数的具体描述可以参考`MSDN`，这里我只挑选重点关注的进行讲解：
1. hProcess: 这个是你需要注入的进程的句柄，你可以通过`OpenProcess`来获得这个句柄。然后你在`MSDN`会看到这样一句话：
> The handle must have the PROCESS_CREATE_THREAD, PROCESS_QUERY_INFORMATION, PROCESS_VM_OPERATION, PROCESS_VM_WRITE, and PROCESS_VM_READ access rights, and may fail without these rights on certain platforms.
   你需要在`OpenProcess`时指定上面提到的这些权限，否则`CreateRemoteThread`会失败
2. lpStartAddress: 这个是你线程开始运行的地址。这里你需要注意，你并不能传当前进程的函数地址过去，因为每个进程都有自己的虚拟内存，且互相看不见对方有啥（就目前来说），就32位系统来说，虚拟内存的地址范围都是0~4G。也就是说你这边的`0x23333333`是函数`A`，而到了目标进程中`0x23333333`地址就可能是函数`B`了。所以你这里需要在目标进程中申请一块内存，并且向这块内存中写入线程的起始地址才行，而这就需要用到另外一个知名`API`了:`VirtualAllocEx`。
3. lpParameter: 这个是线程运行时传递给`lpStartAddress`的一个参数，额，用过`CreateThread`的肯定还是知道的，就和那个一模一样。

所以现在我们可以先写出一部分代码，如下：
```cpp
int main()
{
    HANDLE hProcess = ::OpenProcess(
        PROCESS_CREATE_THREAD | PROCESS_OPERATION | PROCESS_QUERY_INFORMATION | PROCESS_VM_READ | PROCESS_VM_WRITE,
        FALSE,
        6916/*进程ID，这里因人而异，可以打开任务管理器看*/
    );

    if (hProcess == NULL)
    {
        return -1;
    }

    HANDLE hThread = ::CreateRemoteThread(
        hProcess,
        NULL,
        0,
        /*这里是要执行的函数的起始地址*/,
        /*传给执行函数的参数地址*/,
        0,
        NULL
    );

    if (hThread == NULL)
    {
        ::CloseHandle(hProcess);
        return -1;
    }

    ::WaitForSingleObject(hThread, INFINITE); //这里是为了确保注入线程正常退出，然后做资源回收

    ::CloseHandle(hThread);
    ::CloseHandle(hProcess);

    return 0;
}
```

那所以说现在还差两个东西，就是函数的起始地址和传递给函数的参数。继续往下看，你会找到答案。

### LoadLibrary
我们先用[ProcExp](https://technet.microsoft.com/en-us/sysinternals/bb896653/)来看下各个进程中`kernel32.dll`的地址分布吧：
{% asset_img explorer_kernel32_address.jpg Explorer.exe中kernel32的地址 %}
{% asset_img devenv_kernel32_address.jpg VS中kernel32的地址 %}

可以看出来，`kernel32.dll`在进程中的起始地址都是固定的，大家也可以自己验证下其他进程。而`LoadLibraryW`在`kernel32.dll`中的偏移地址也是一定的，所以如果我们想要知道目标进程中`LoadLibraryW`的地址，其实就可以获取当前进程中`LoadLibraryW`的地址。

所以获取`LoadLibraryW`的地址的代码如下：
```cpp
HMODULE hKnl = ::LoadLibraryW(L"kernel32.dll");
if (hKnl == NULL)
{
    return -1;
}

LPTHREAD_START_ROUTINE pLoadLibrary = (LPTHREAD_START_ROUTINE)::GetProcAddress(hKnl, "LoadLibraryW");
if (pLoadLibrary == NULL)
{
    return -1;
}
```

### 申请内存
现在我们还缺一个传递给`LoadLibraryW`的参数，也就是我们需要注入的dll的地址。但是由于我们创建的线程是在目标进程中跑的，而目标进程无法读取到我们当前进程中的内存，我们需要创建一块目标进程中的内存才行。所以自然地就引出了下面这个`API`:
```cpp
LPVOID
WINAPI
VirtualAllocEx(
    _In_ HANDLE hProcess,
    _In_opt_ LPVOID lpAddress,
    _In_ SIZE_T dwSize,
    _In_ DWORD flAllocationType,
    _In_ DWORD flProtect
    );
```

这个函数可以向目标进程申请一块内存，并且这块内存会被初始化为0。函数的具体参数解释如下：
1. hProcess: 目标进程的句柄，不做解释
2. lpAddress: 内存的起始地址，这里传NULL就行了
3. dwSize: 申请的内存的大熊啊
4. flAllocationType: 这个是申请内存的类型，以后会细讲，这里传`MEM_COMMIT`就行了
5. flProtect: 这个是你对这块内存索取的权限，以后细讲，这里传`PAGE_READWRITE`即可

我们现在有了目标进程中的一块内存，但是我们还要借助另外一个`API`来实现向这块内存中写入数据:
```cpp
BOOL
WINAPI
WriteProcessMemory(
    _In_ HANDLE hProcess,
    _In_ LPVOID lpBaseAddress,
    _In_reads_bytes_(nSize) LPCVOID lpBuffer,
    _In_ SIZE_T nSize,
    _Out_opt_ SIZE_T * lpNumberOfBytesWritten
    );
```

比如我们需要注入的`dll`的路径是`D:\RemoteInjectDll.dll`，那么代码如下:
```cpp
LPCWSTR lpszDllPath = LR"(D:\RemoteInjectDll.dll)";
SIZE_T dwMemSize = static_cast<SIZE_T>((wcslen(lpszDllPath) + 1) * sizeof(WCHAR));
LPVOID lpRemoteMemory = ::VirtualAllocEx(hProcess, NULL, dwMemSize, MEM_COMMIT, PAGE_READWRITE);
if (lpRemoteMemory == NULL)
{
    return -1;
}

SIZE_T sizeWritten = 0;
if (!::WriteProcessMemory(hProcess, lpRemoteMemory, (LPCVOID)lpszDllPath, dwMemSize, &sizeWritten))
{
    ::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
    return -1;
}

//...

::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
```

## 最后组装
现在所有的条件都具备了，需要把这几块都组装一下，形成的最终代码如下：
```cpp
// RemoteInjectExe.cpp : Defines the entry point for the console application.
//

int main()
{
    HANDLE hProcess = ::OpenProcess(
        PROCESS_CREATE_THREAD | PROCESS_OPERATION | PROCESS_QUERY_INFORMATION | PROCESS_VM_READ | PROCESS_VM_WRITE,
        FALSE,
        6916
    );

    if (hProcess == NULL)
    {
        return -1;
    }

    LPCWSTR lpszDllPath = LR"(D:\workspaces\c++\RemoteInjectExe\Debug\RemoteInjectDll.dll)"; //我改了下地址
    SIZE_T dwMemSize = static_cast<SIZE_T>((wcslen(lpszDllPath) + 1) * sizeof(WCHAR));
    LPVOID lpRemoteMemory = ::VirtualAllocEx(hProcess, NULL, dwMemSize, MEM_COMMIT, PAGE_READWRITE);

    if (lpRemoteMemory == NULL)
    {
        return -1;
    }

    SIZE_T sizeWritten = 0;
    if (!::WriteProcessMemory(hProcess, lpRemoteMemory, (LPCVOID)lpszDllPath, dwMemSize, &sizeWritten))
    {
        ::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
        ::CloseHandle(hProcess);
        return -1;
    }

    HMODULE hKnl = ::LoadLibraryW(L"kernel32.dll");
    if (hKnl == NULL)
    {
        ::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
        ::CloseHandle(hProcess);
        return -1;
    }

    LPTHREAD_START_ROUTINE pLoadLibrary = (LPTHREAD_START_ROUTINE)::GetProcAddress(hKnl, "LoadLibraryW");
    if (pLoadLibrary == NULL)
    {
        ::FreeLibrary(hKnl);
        ::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
        ::CloseHandle(hProcess);
        return -1;
    }

    DWORD dwThreadId;
    HANDLE hThread = ::CreateRemoteThread(
        hProcess,
        NULL,
        0,
        pLoadLibrary,
        lpRemoteMemory,
        0,
        &dwThreadId);

    if (hThread == NULL)
    {
        ::FreeLibrary(hKnl);
        ::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
        ::CloseHandle(hProcess);
        return -1;
    }

    ::WaitForSingleObject(hThread, INFINITE);

    ::FreeLibrary(hKnl);
    ::VirtualFreeEx(hProcess, lpRemoteMemory, dwMemSize, MEM_RELEASE);
    ::CloseHandle(hProcess);
    return 0;
}
```

## 测试
我们现在把`HelloWorld.exe`运行起来，然后把`OpenProcess`函数中的最后一个参数改为`HelloWorld.exe`的进程ID，编译运行即可。可以看到最终运行结果如下：
{% asset_img inject_result.png 运行结果 %}

到这里就完全讲完了，所以注入也不是一个神奇的事情，完全就是`API`的事情，知道了就行了。至于原理，做完了之后可以慢慢研究。

## 代码优化
1. 你看代码里面很多那种资源回收的代码，而且是完全相同的，是不是可以有一个统一回收的机制呢？比如说我们利用`C++`中析构函数会在对象生命周期结束时被调用，因为这里我们仅仅是需要在main函数结束的时候能够把句柄和内存释放掉。所以就有了下面两个类：
```cpp
class ScopeHandle
{
public:
    ScopeHandle(HANDLE h)
        : m_h(h)
    {

    }

    ScopeHandle(const ScopeHandle&) = delete;
    
    ~ScopeHandle()
    {
        Close();
    }

public:
    void Attach(HANDLE h)
    {
        Close();
        m_h = h;
    }

    HANDLE Detach()
    {
        HANDLE h = m_h;
        m_h = NULL;
        return h;
    }

public:
    operator HANDLE()
    {
        return m_h;
    }

    ScopeHandle& operator=(HANDLE h)
    {
        Close();
        m_h = h;
        return (*this);
    }

    ScopeHandle& operator=(const ScopeHandle&) = delete;

private:
    void Close()
    {
        if (m_h != NULL || m_h != INVALID_HANDLE_VALUE)
        {
            ::CloseHandle(m_h);
            m_h = NULL;
        }
    }

private:
    HANDLE m_h;
};

class ScopeVirtualMemoryEx
{
public:
    ScopeVirtualMemoryEx(LPVOID lpMemory, HANDLE hProcess, SIZE_T memSize)
        : m_lpMemory(lpMemory)
        , m_hProcess(hProcess)
        , m_memSize(memSize)
    {

    }

    ScopeVirtualMemoryEx(const ScopeVirtualMemoryEx&) = delete;

    ~ScopeVirtualMemoryEx()
    {
        Close();
    }

public:
    void Attach(LPVOID lpMemory, HANDLE hProcess, SIZE_T memSize)
    {
        Close();
        m_lpMemory = lpMemory;
        m_hProcess = hProcess;
        m_memSize = memSize;
    }

    LPVOID Detach()
    {
        LPVOID lpMemory = m_lpMemory;
        Close();

        return lpMemory;
    }

public:
    operator LPVOID()
    {
        return m_lpMemory;
    }

    ScopeVirtualMemoryEx& operator=(const ScopeVirtualMemoryEx&) = delete;

private:
    void Close()
    {
        if (m_lpMemory == NULL)
        {
            return;
        }

        if (m_hProcess == NULL)
        {
            return;
        }

        ::VirtualFreeEx(m_hProcess, m_lpMemory, m_memSize, MEM_RELEASE);

        m_hProcess = NULL;
        m_lpMemory = NULL;
        m_memSize = 0;
    }

private:
    LPVOID m_lpMemory;
    HANDLE m_hProcess;
    SIZE_T m_memSize;
};
```

有的人是不是会觉得得不偿失啊？我为了优化不到20行代码，平白无故多出来100多行代码。这个的话，看个人喜好吧。

那上面的是不是优化结束了呢？那万一我还有10个其他自定义的对象也需要这种自动释放的机制，那是不是还要在写100个类了？

所以，我们继续观察，可以发现其实上面两个类有非常多的相似之处，如果排除掉成员变量的因素，那就一模一样了！那么还能怎么抽取呢？我们观察下上面两个类，可以发现无非就是需要做两件事情：
1. 判断代理的目标对象是否有效
2. 释放代理的目标对象

也就是说我们其实可以再在这两个类上抽象一层，比如就叫`AutoReleaseObject`吧，然后传递一个判断是否有效以及释放的方法。恩，想法是很美好的，那么问题又来了，如果只有一个参数我们还能用一个模板来消除差异性，现在它们释放的函数签名完全不同，这你让我怎么玩？别急啊，办法总是有的。其中心思想就是把数据处理与流程分离（也可以理解为机制与策略分离，详见《unix编程思想》），那么我后来就写出了下面的代码（妈蛋，怎么越来越麻烦了！:
```cpp
template <typename T>
class AutoReleaseDelegate
{
public:
    AutoReleaseDelegate(T obj)
        : m_obj(obj)
    {

    }

    virtual ~AutoReleaseDelegate()
    {

    }

public:
    virtual bool Validate() const = 0;
    virtual void Release() = 0;

public:
    operator T()
    {
        return m_obj;
    }

protected:
    T m_obj;
};

class HandleDelegate
    : public AutoReleaseDelegate<HANDLE>
{
    using super = AutoReleaseDelegate<HANDLE>;

public:
    HandleDelegate(HANDLE h)
        : AutoReleaseDelegate(h)
    {

    }

    ~HandleDelegate()
    {

    }

public:
    virtual bool Validate() const override
    {
        return (super::m_obj != NULL);
    }

    virtual void Release() override
    {
        ::CloseHandle(super::m_obj);
        super::m_obj = NULL;
    }
};

class VirtualMemoryExDelegate
    : public AutoReleaseDelegate<LPVOID>
{
    using super = AutoReleaseDelegate<LPVOID>;

public:
    VirtualMemoryExDelegate(LPVOID lpMemory, HANDLE hProcess, SIZE_T memSize)
        : AutoReleaseDelegate(lpMemory)
        , m_hProcess(hProcess)
        , m_memSize(memSize)
    {

    }

    ~VirtualMemoryExDelegate()
    {

    }

public:
    virtual bool Validate() const override
    {
        if (super::m_obj == NULL)
            return false;

        if (m_hProcess == NULL)
            return false;

        return true;
    }

    virtual void Release() override
    {
        ::VirtualFreeEx(m_hProcess, super::m_obj, m_memSize, MEM_RELEASE);
        m_hProcess = NULL;
        super::m_obj = NULL;
        m_memSize = 0;
    }

private:
    HANDLE m_hProcess;
    SIZE_T m_memSize;
};

template <typename T>
class AutoReleaseObj
{
    using DelegateType = AutoReleaseDelegate<T>;

public:
    AutoReleaseObj(DelegateType* pDelegate)
        : m_pDelegate(pDelegate)
    {

    }
    
    AutoReleaseObj(const AutoReleaseObj&) = delete;

    ~AutoReleaseObj()
    {
        assert(m_pDelegate != nullptr);
        if (m_pDelegate->Validate())
        {
            m_pDelegate->Release();
        }
    }

public:
    operator T()
    {
        assert(m_pDelegate != nullptr);
        return T(*m_pDelegate);
    }

    AutoReleaseObj<T>& operator=(AutoReleaseObj<T>& rhs) = delete;

private:
    DelegateType* m_pDelegate;
};
```

这样一来我们要接入新的需要`AutoRelease`特性的对象时，只需要继承`AutoReleaseDelegate`即可，而整个流程不在需要你去关系，交给`AutoReleaseObj`就行了。当然上面的代码这样写不免有些过度设计之嫌，还是那句话，用你你喜欢或者你习惯的方式就好，不必追求什么设计。

我自己的优化已经结束了，如果大家觉得还有哪里可以进行优化的话，欢迎提出来，我努力实现。还有这里肯定会有人问为什么指针不判空，我想说，请分清楚库代码和工作代码之间的区别。

这次没有`ATL`优化版本，`ATL`中只提供了`CHandle`对`HANDLE`进行封装，其他并没有做任何封装（或者我没有找到），所以注入的第一篇就到这里结束啦，大家好好消化下吧。
