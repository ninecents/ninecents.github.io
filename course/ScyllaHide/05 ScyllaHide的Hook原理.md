# ScyllaHide的Hook原理

Hook是通过修改程序代码或数据，达到改变程序逻辑的目的。Hook种类很多，ScyllaHide主要使用的是inline hook，也就是在运行的流程中插入跳转指令（call/jmp）来抢夺程序运行流程的一个方法。

Hook根据系统版本（win7、win8、win10等）、cpu架构（x86、x64）、子系统类型（是否是wow64进程），实现由细微差别。本节只讨论win7、x64下32位进程（即wow64进程）的情况，其他情况请大家自己分析。

下面是针对目标环境（win7、x64下32位进程），我们将ScyllaHide所有API的hook类型进行统计，结果如下图（hook类型统计图）所示：

![hook类型统计](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/hook类型统计.png)


## ScyllaHide究竟做了什么

抛开源码，我们先通过工具查看下ScyllaHide对程序做了什么，然后再分析它怎么实现的。实验和代码结合，相辅相成，更容易理解其原理。

首先，我们打开OD，然后打开Plugins菜单的ScyllaHide的选项框。点击右上角的“Create new profile...”按钮，随便起一个名字（我命名为khz了），其效果如下图所示：

![ScyllaHide配置](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/ScyllaHide配置.png)

然后，我们通过OD，打开任意32进程，待程序运行起来后，打开PCHunter，查看被调试进程（我这里的被调试进程名字是MyTestAntiDebuger.exe）的进程钩子，如下图所示：

![PCHunter查看进程钩子](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/PCHunter查看进程钩子.png)

最后我们简单分析下，从本文章第一幅图“hook类型统计”中可以看出，被标记为HOOK的钩子共有6个，这6个钩子正好对应PCHunter扫描出来的前6个inline hook，可以看出它们是一种类型的钩子。

除了这6个钩子，其他钩子都以Nt开头（BlockInput函数虽然不是Nt开头，其实也属于Nt系列的），而PCHunter扫描出来的只剩下wow64cpu.dll的一个inline hook，难道这么多的Nt系列函数公用一个钩子就完成了所有的任务？具体原理暂且不说，我们到此可以看出ScyllaHide的钩子是不止一种的，而且ScyllaHide的钩子可以给多个函数上钩子（hook）。


## ApplyHook函数调用结构图

下面是Hook的整体流程，非目标环境（win7、x64下32位进程）的函数直接忽略了。

![ApplyHook函数调用结构图](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/ApplyHook函数调用结构图.png)

从上图中，可以看到，hook根据模块分为3类，分别是：Ntdll、Kernel32（或者KernelBase）、Win32u（或者User32）。其实这个划分跟HOOK原理关系不大，真正有关的是定义的几个宏：HOOK、HOOK_NATIVE、HOOK_NATIVE_NOTRAMP，与之对应的是FREE_HOOK和RESTORE_JMP（ScyllaHide并没有完整的恢复hook相关内容，有兴趣的大家可以自己改造一下源码，保证hook的正常恢复）。下面是三个hook相关宏定义的实现：

    #define HOOK(name) hdd->d##name = (t_##name)DetourCreateRemote(hProcess,_##name, Hooked##name, true, &hdd->##name##BackupSize)
    #define HOOK_NATIVE(name) hdd->d##name = (t_##name)DetourCreateRemoteNative(hProcess,_##name, Hooked##name, true, &hdd->##name##BackupSize)
    #define HOOK_NATIVE_NOTRAMP(name) DetourCreateRemoteNative(hProcess,_##name, Hooked##name, false, &hdd->##name##BackupSize)

如上所见，我们从HOOK_NATIVE和HOOK_NATIVE_NOTRAMP的宏定义可以看出，它们都是调用了DetourCreateRemoteNative32函数，只是参数notUsed不同而已；而HOOK宏调用了函数DetourCreateRemote。由此可见ScyllaHide通过两种hook类型实现反调试的。下面我们将针对这两种类型的hook进行举例，分别对HOOK、HOOK_NATIVE进行详细解说。


## HOOK实现原理（OutputDebugStringA的hook流程）

对于HOOK类型的钩子，我们只分析一个熟悉的函数OutputDebugStringA，其他函数类似的原理。依照惯例，我们先上函数调用结构图：

![OutputDebugStringA的Hook流程](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/OutputDebugStringA的Hook流程.png)

从图中我们可以看出，ScyllaHide先获得了两个函数地址：钩子函数HookedOutputDebugStringA和目标函数OutputDebugStringA。

钩子函数HookedOutputDebugStringA是通过函数GetDllFunctionAddressRVA解析PE，获得导出函数地址，再加上模块（HookLibraryx86.dll）基地址构成。

而目标函数OutputDebugStringA直接通过GetProcAddress获得。有些人肯定会问，GetProcAddress获得的是本进程（本文中就是OD.exe）函数地址，怎么可以拿来当做被调试进程的地址呢？其实这个很容易理解，Windows系统有上百个进程，每个进程的很多dll都是相同的（尤其是windows自带的dll，如ntdll、kernel32、user32等），如果ntdll.dll在所有进程的地址都是一样的，那么对应的物理地址也就是一样的了，无论是内存开销还是内存管理上都会简单很多。除此之外，我们还需要知道一点：如果被调试进程包含kernelbase.dll模块，则获得kernelbase.dll的OutputDebugStringA函数，否则获得kernel32.dll的OutputDebugStringA函数，因为kernel32.dll最终会调用kernelbase.dll。

有了上面两个地址，直接使用宏HOOK进行挂钩，我们将HOOK宏展开，如下所示：

    HOOK(OutputDebugStringA);
    #define HOOK(name) hdd->d##name = (t_##name)DetourCreateRemote(hProcess,_##name, Hooked##name, true, &hdd->##name##BackupSize)

    hdd->dOutputDebugStringA = (t_OutputDebugStringA)DetourCreateRemote(hProcess,_OutputDebugStringA, HookedOutputDebugStringA, true, &hdd->OutputDebugStringABackupSize)

    void * DetourCreateRemote(void * hProcess, void * lpFuncOrig, void * lpFuncDetour, bool createTramp, unsigned long * backupSize);

现在我们分析函数DetourCreateRemote。
- 首先，ScyllaHide将被调试进程的OutputDebugStringA函数地址（lpFuncOrig）的50个字节读出来保存到局部变量里面originalBytes。
- 然后，执行**int detourLen = GetDetourLen(originalBytes, minDetourLen);**，获得detour指令（调整指令覆盖的所有指令）的长度。minDetourLen为常量6，包含一个nop，一个jmp，和一个4字节的地址。GetDetourLen调用了LengthDisassemble，LengthDisassemble是封装了反汇编引擎distorm而实现的获得指令长度的函数。函数GetDetourLen逐条分析指令，获得每条指令的长度，最终获得一个大于等于常量minDetourLen（即6）的值。
- 接着，如果参数createTramp为true，创建trampoline（蹦床），trampoline的内容包含两部分：detourLen个字节的函数OutputDebugStringA原始内容，和JMP 
<OutputDebugStringA + detourLen>。
- 最后，将被调试进程的函数OutputDebugStringA的内容改为JMP <HookedOutputDebugStringA>

我们将上述结果画图表示如下：

![ScyllaHide inline hook实现原理](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/ScyllaHide%20inline%20hook实现原理.png)

我们可以看出，ScyllaHide直接将OutputDebugStringA跳转到了HookedOutputDebugStringA，虽然也创建了Trampoline，但是根本没有使用到。对照下面OutputDebugStringA的汇编代码，我们清晰的看到Trampoline将OutputDebugStringA前两条语句备份了一下，然后添加了一条JMP回<HookedOutputDebugStringA+0x0a>的指令。

    ; OutputDebugStringA原始汇编代码
    76C15A60    68 34020000     PUSH 234
    76C15A65    68 90CCCC76     PUSH 76CCCC90
    76C15A6A    E8 B5410300     CALL 76C49C24

为了大家能方便的理解Trampoline的作用，大家可以看下面图片内容：

![标准inline hook实现原理](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/标准inline%20hook实现原理.png)

至此，ScyllaHide的HOOK宏方式hook全部结束，下面分析下HOOK_NATIVE宏的原理。

## HOOK_NATIVE实现原理（NtClose的hook流程）

#### SSDT和ntdll调用机制

要了解一个hook实现nt系列函数的hook功能，不得不说下ntdll的原理。我们先看下ntdll究竟干了什么：

![Win7_x64_Wow64进程Nt系列函数汇编代码示例](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/Win7_x64_Wow64进程Nt系列函数汇编代码示例.png)

    B8 0C000000         mov     eax, 0C                          ;  ZwClose
    33C9                xor     ecx, ecx
    8D5424 04           lea     edx, dword ptr [esp+4]
    64:FF15 C0000000    call    dword ptr fs:[C0]
    83C4 04             add     esp, 4
    C2 0400             retn    4

从图中我们可以看出，Nt系列函数的实现都很像，图中是NtSetInformationThread、NtSetEvent、NtClose函数的汇编内容，我们可以看出他们都设置了eax、ecx、edx，然后调用了fs:[C0]。其中eax是索引，ecx是子索引，edx是参数列表地址。这个eax的索引又是什么呢？我们先看下面截图：

![WIN64AST的SSDT截图](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/WIN64AST的SSDT截图.png)

对比上面两张图，我们发现这个索引刚好就是SSDT的索引，从r3进入r0层就是通过这个索引判断调用的哪个函数。由于Nt系列函数实现相似，根据索引实现的系统函数调用，所以，我们修改他们通用的调用地址，然后hook掉，再根据不同的eax和ecx值，就可以实现Nt系列函数的hook了。这个通用地址就是fs:[C0]。

Windows操作系统中，fs寄存器用于记录线程环境块TEB，根据TEB结构体定义可以看出0xC0偏移处的定义为：

    PVOID WOW32Reserved;  // 0xC0

wow64中直接也拿这个保留位置用于进行32位64位环境切换的跳板（wow64调用逻辑可以参考文章[汇编里看Wow64的原理](https://bbs.pediy.com/thread-221236.htm)）。OD步进代码**call    dword ptr fs:[C0]**，就到了地址72B82320，该地址就是PCHunter中最后的那个hook地址了。下面我们分别看下hook前后内容：

![WOW32Reserved地址hook前汇编内容](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/WOW32Reserved地址hook前汇编内容.png)

![WOW32Reserved地址hook后汇编内容](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/WOW32Reserved地址hook后汇编内容.png)

这就可以看出，所有Nt系列函数都进入了HookLibraryx86.dll的内存空间，我们可以继续单步执行，调整到的内容就是HookedFunctions.cpp的HookedNativeCallInternal函数。

    void NAKED NTAPI HookedNativeCallInternal()
    {
    #ifndef _WIN64
        __asm
        {
            PUSHAD
            PUSH ECX
            PUSH EAX
            CALL HandleNativeCallInternal
            cmp eax, 0
            je NoHook
            POPAD
            ADD ESP,4
            PUSH ECX
            PUSH EAX
            CALL HandleNativeCallInternal
            jmp eax
            NoHook:
            POPAD
            jmp HookDllData.NativeCallContinue
        }
    #endif
    }


#### ScyllaHide实现分析

有了上面的内容，我们已经了解了ScyllaHide实现Nt系列函数的原理，下面我们从源码看下它究竟如何实现的。

我们先看下HOOK_NATIVE宏展开内容，如下所示：

    HOOK_NATIVE(NtClose);
    #define HOOK_NATIVE(name) hdd->d##name = (t_##name)DetourCreateRemoteNative(hProcess,_##name, Hooked##name, true, &hdd->##name##BackupSize)
    #define DetourCreateRemoteNative DetourCreateRemoteNative32
   
    hdd->dNtClose = (t_NtClose)DetourCreateRemoteNative32(hProcess,_NtClose, HookedNtClose, true, &hdd->NtCloseBackupSize)

我们可以看出HOOK_NATIVE最终调用了DetourCreateRemoteNative32函数，并将结果返回给了hdd->dNtClose，该值会被写入被调试进程，最红被HookedNtClose使用。

现在，我们可以看下函数DetourCreateRemoteNative32的调用图：

![DetourCreateRemoteNative32流程](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/DetourCreateRemoteNative32流程.png)

DetourCreateRemoteNative32函数相对简单点：

- 先读取NtClose的内存写入数组originalBytes中。
- 再通过函数GetSysCallIndex32(originalBytes)，获取系统调用索引值，即汇编中的eax值。（GetSysCallIndex32等黑边框灰底色的函数都是distrom反汇编引擎实现的，大家可以跟踪下代码看下逻辑，不详细解释了）
- 根据程序运行环境类型，如果是32位的操作系统，执行DetourCreateRemoteNative32Normal函数，相关逻辑不在本节讨论范围，大家自己分析，可以参考HOOK和HOOK_NATIVE原理。
- 如果是64位的操作系统，先通过GetEcxSysCallIndex32获取系统调用“子索引值”，即汇编中的ecx值，然后执行DetourCreateRemoteNativeSysWow64函数，实现hook。
- 最后，将返回值赋值给hdd->dNtClose，该值之后会在HookedNtClose函数中被调用。


接下来，我们再看下函数DetourCreateRemoteNativeSysWow64的调用逻辑，如下图所示：

![DetourCreateRemoteNativeSysWow64流程](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/DetourCreateRemoteNativeSysWow64流程.png)

函数DetourCreateRemoteNativeSysWow64执行流程为：

- 根据originalBytes，通过distorm获取NtClose函数的关键值funcSize、callOffset、callSize。
- 根据函数GetCallDestination获取sysWowSpecialJmpAddress值（即前文的WOW32Reserved，函数内部直接通过 **(DWORD)__readfsdword(0xC0)** 获取）。
- 备份sysWowSpecialJmpAddress地址的内存内容。
- 将NtClose函数备份，并修改call调用。
- 在sysWowSpecialJmpAddress地址写入JMP HookedNativeCallInternal，实现hook。
- 返回trampoline，即自实现的NtClose函数，用于HookedNtClose函数中被调用。

蓝底的两个跟sysWowSpecialJmpAddress变量有关的逻辑，只执行一次，根据全局变量onceNativeCallContinue实现的。

NtClose函数的备份，用于HookedNtClose函数中被调用，那么它的内容是什么呢？假设跟原始的NtClose函数一样，那么肯定会重新进入HookedNtClose函数，从而进入死循环。我们先看下ScyllaHide如何实现的：

![NtClose函数备份内容](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/NtClose函数备份内容.png)

从图中可以看出，备份代码中添加了**PUSH 180017**指令，并修改了CALL指令为**JMP FAR 0033:72B8271E**。一开始很不理解为何多了个PUSH，调试了下，发现，CALL指令就是PUSH和JMP，ScyllaHide巧妙的构造了该内容，保证了程序不会进入死循环。


最后，总结下Nt系列hook原理，如下图所示：

![Nt系列hook总结](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/Nt系列hook总结.png)


## 其它关键点

#### minDetourLen为何为常量6（包含一个nop，一个jmp，和一个4字节的地址）

为什么是6，为什么不是5，为什么要多一个nop？？？我们知道被hook的地方有的是jmp指令（**call    dword ptr fs:[C0]**），ScyllaHide会循环调用ReadProcessMemory函数，读取被调试进程的内存，部分代码通过读取第一个字节的指令内容来判断是否被hook过了，填充nop就是为了方便比较。


## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。


## 参考文档
- [WINDOWS上的 API HOOK 技术](https://segmentfault.com/a/1190000012993626)
- [DEEP HOOKS: MONITORING NATIVE EXECUTION IN WOW64 APPLICATIONS](https://www.sentinelone.com/blog/deep-hooks-monitoring-native-execution-wow64-applications-part-1/) 及[译文](https://xz.aliyun.com/t/3311)
- [汇编里看Wow64的原理（浅谈32位程序是怎样在windows 64上运行的？）](https://bbs.pediy.com/thread-221236.htm)
