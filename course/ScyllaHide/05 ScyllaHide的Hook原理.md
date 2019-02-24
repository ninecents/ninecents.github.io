# ScyllaHide的Hook原理

Hook是通过修改程序代码或数据，达到改变程序逻辑的目的。Hook种类很多，ScyllaHide主要使用的是inline hook，也就是在运行的流程中插入跳转指令（call/jmp）来抢夺程序运行流程的一个方法。

Hook根据系统版本（win7、win8、win10等）、cpu架构（x86、x64）、子系统类型（是否是wow64进程），实现由细微差别。本节只讨论win7、x64下32位进程（即wow64进程）的情况，其他情况请大家自己分析。


## ScyllaHide究竟做了什么

抛开源码，我们先通过工具查看下ScyllaHide对程序做了什么，然后再分析它怎么实现的。实验和代码结合，相辅相成，更容易理解其原理。

首先，我们打开OD，然后打开Plugins菜单的ScyllaHide的选项框。点击右上角的“Create new profile...”按钮，随便起一个名字（我命名为khz了），其效果如下图所示：

![ScyllaHide配置](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/ScyllaHide配置.png)

然后，我们通过OD，打开任意32进程，待程序运行起来后，打开PCHunter，查看被调试进程（我这里的被调试进程名字是Calc.exe）的进程钩子，如下图所示：

![PCHunter查看进程钩子](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/PCHunter查看进程钩子.png)

最后我们简单分析下，ntdll有两个inline hook，其余的都是KernelBase.dll中的inline hook，后面的文章会统计所有钩子的类型，我们可以看到KernelBase.dll使用的钩子是一一对应的，那么其他Nt系列函数的钩子都跑哪里了呢，难道只有最上面的两个钩子就完成了所有的任务？具体原理暂且不说，我们到此可以看出ScyllaHide的钩子是不止一种的。


## ApplyHook函数调用结构图

下面是Hook的整体流程，非目标环境（win7、x64下32位进程）的函数直接忽略了。

![ApplyHook函数调用结构图](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/ApplyHook函数调用结构图.png)

从上图中，可以看到，hook根据模块分为3类，分别是：Ntdll、Kernel32（或者KernelBase）、Win32u（或者User32）。其实这个划分跟HOOK原理关系不大，真正有关的是定义的几个宏：HOOK、HOOK_NATIVE、HOOK_NATIVE_NOTRAMP，与之对应的是FREE_HOOK和RESTORE_JMP。ScyllaHide并没有完整的恢复hook相关内容，有兴趣的大家可以自己改造一下源码。

下面是针对目标环境（win7、x64下32位进程）所有API的hook类型统计。

![hook类型统计](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理/hook类型统计.png)

如上所见，Kernel32.dll使用的是HOOK宏，除了NtSetDebugFilterState使用HOOK_NATIVE_NOTRAMP以外的其它函数，都是使用的HOOK_NATIVE宏。而HOOK_NATIVE和HOOK_NATIVE_NOTRAMP从宏定义我们可以看出，它们都是调用了DetourCreateRemoteNative32函数，只是参数notUsed不同而已。下面我们举两个例子，对HOOK、HOOK_NATIVE分别进行描述。


## OutputDebugStringA的Hook流程

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

## NtSetInformationThread的Hook流程




## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。


## 参考文档
- [WINDOWS上的 API HOOK 技术](https://segmentfault.com/a/1190000012993626)
- [DEEP HOOKS: MONITORING NATIVE EXECUTION IN WOW64 APPLICATIONS](https://www.sentinelone.com/blog/deep-hooks-monitoring-native-execution-wow64-applications-part-1/) 及[译文](https://xz.aliyun.com/t/3311)
