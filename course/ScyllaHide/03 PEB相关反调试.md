# [ScyllaHide] 文章列表-看雪地址：

- [00 简单介绍和使用](https://bbs.pediy.com/thread-249151.htm)
- [01 项目概览](https://bbs.pediy.com/thread-249305.htm)
- [02 InjectorCLI源码分析](https://bbs.pediy.com/thread-249306.htm)
- [03 PEB相关反调试](https://bbs.pediy.com/thread-249374.htm)
- [04 ScyllaHide配置报错原因定位](https://bbs.pediy.com/thread-249524.htm)
- [05 ScyllaHide的Hook原理](https://bbs.pediy.com/thread-249721.htm)


# PEB相关反调试

反调试，接触算是有四五年了，每次遇到问题，都是一头雾水，不知如何下手，总是通过换调试器、找插件进行各种测试。翻看过很多文章，却又总是蜻蜓点水，希望通过ScyllaHide的源码分析，能对反调试有进一步的了解吧。

先说下，最近翻看资料对反调试的理解吧。

首先，要说下进程。进程是一个具有一定独立功能的程序关于某个数据集合的一次运行活动。进程的概念主要有两点：第一，每一个进程都有它自己的地址空间，一般情况下，包括文本区域（text region）、数据区域（data region）和堆栈（stack region）；第二，进程是一个“执行中的程序”。从上面的描述，我们可以猜测，调试相关的信息也肯定在进程的**地址空间**内，系统检测进程的一些方法就是通过对进程的地址空间（数据区域）进行检测的。

其次，数据结构。任何程序都是数据结构和算法（逻辑）的结合，小到helloworld，大到操作系统都逃不出这两样东西。了解Windows就要了解其数据结构，在这里，我们要说的就是PEB（Process Envirorment Block Structure）结构体，“人”如其名，它是进程在ring3层的核心数据块，很多进程相关的操作都是通过它进行的。
发现一篇不错的文章，大家可以了解下PEB在进程中的作用：[从TEB到PEB再到SEH（一）](https://bbs.pediy.com/thread-223816.htm)。

然后，知识点。PEB中修改的每个值都是一个知识点，不要将其看作一个点，这样囫囵吞枣，会错过很多好东西；也不要看到东西太多而不知所措，每一个点都要了解到位，要相信，时间花费掉一定会有收获的。

最后，想说的就是**读书破万卷**。反调试有很多厉害的书籍，以前总是觉得很难，或者觉得没必要学，又去各种百度谷歌，查无结果，不断的迷茫中度日。其实很多知识都已经放在那里了，不断的啃下去，知识就一点一点的累积出来了。废话有点多了，其实就想说张银奎老师的《软件调试》真的很好，只要详细的品读，就一定会有收获的，比如本节中关于堆的操作就有详细的讲解。


## ApplyPEBPatch函数调用结构图
回顾上节内容，我们可以知道ApplyPEBPatch是对PEB进行反调试处理的核心函数，其调用结构如下所示：

![ApplyPEBPatch函数调用结构图](https://ninecents.github.io/course/ScyllaHide/03%20PEB相关反调试/ApplyPEBPatch函数调用结构图.png)

从图中我们可以看出，ApplyPEBPatch函数最终调用了读写进程内存操作（wow64使用的是wow64相关的读写api），将被调试进程的PEB、RTL_USER_PROCESS_PARAMETERS等信息读取到InjectorCLI进程，修改调试相关属性，再写回被调试进程。这里其实设计到以下5种情况：
- 32位系统，只能运行32位的Scylla程序，只需要执行scl::SetPeb函数。
- 64位系统，使用32位的Scylla程序，注入32位被调试进程，由于目标进程是wow64进程，所以既需要执行scl::SetPeb函数，又需要执行scl::Wow64SetPeb64函数。通过调试，可以发现，wow64进程（即64位系统下的32位被调试进程）有两个PEB块，**一个peb，一个peb64**，需要将他们相关的调试信息都抹去才能做到反调试。
- 64位系统，使用32位的Scylla程序，注入64位被调试进程。通过调试发现，scl::GetPeb执行失败，因此不会执行scl::SetPeb函数；而scl::IsWow64Process(hProcess)返回false，也不会执行scl::Wow64SetPeb64函数。所以该方案是失败的。
- 64位系统，使用64位的Scylla程序，注入32位被调试进程。通过调试发现，scl::GetPeb执行成功，但是数据读出来的是32位的PEB，64位的程序使用的64位PEB，对PEB数据进行操作，其偏移是错误的，执行结果也不会正确；而因为宏定义《#ifndef _WIN64》的存在，scl::Wow64SetPeb64系列函数不会被编译进程序，也不会执行。所以该方案是失败的。
- 64位系统，使用64位的Scylla程序，注入32位被调试进程。通过调试发现，scl::GetPeb执行成功，执行结果也正确；而因为宏定义《#ifndef _WIN64》的存在，scl::Wow64SetPeb64系列函数不会被编译进程序，也不会执行。所以该方案是成功的。

综上，被调试进程是64位，必须使用64位的Scylla程序；被调试进程是32位，则必须使用32位的Scylla程序。下面相关讨论，只考虑32位程序，不考虑wow64和64位程序的情况，相关情形请大家自己分析。

## 获取指定进程PEB地址
PEB包含了调试相关信息，通过修改被调试进程的相关值就可以实现反反调试功能，那么怎么获得被调试进程的PEB地址就成了PEB反调试的关键。

    scl::PEB *scl::GetPebAddress(HANDLE hProcess)
    {
        ::PROCESS_BASIC_INFORMATION pbi = { 0 };

        auto status = NtQueryInformationProcess(hProcess, ProcessBasicInformation, &pbi, sizeof(pbi), nullptr);

        return NT_SUCCESS(status) ? (PEB *)pbi.PebBaseAddress : nullptr;
    }

从源码可以看出，GetPebAddress逻辑很简单，windows导出了NtQueryInformationProcess函数，可以获得进程基本信息（ProcessBasicInformation），该信息中包含了peb的基址。

## peb之BeingDebugged
这个真的没啥说的，就是一个标志位；不过看了[180306 逆向-反调试技术（1）BeingDebugged](https://blog.csdn.net/whklhhhh/article/details/79656200)之后，感觉没那么简单了，当某进程（如calc.exe）被OD附加，系统会将BeingDebugged设置为True，同时将其他一些信息（如NtGlobalFlag等）修改，只有在合适的时机将其修改才能真正的起到反反调试的作用，具体内容这里不再讨论。

## peb之NtGlobalFlag
程序没有被调试时，NtGlobalFlag成员值为0，如果进程被调试这个成员通常值为0x70（代表下述标志被设置）：

    FLG_HEAP_ENABLE_TAIL_CHECK(0X10)
    FLG_HEAP_ENABLE_FREE_CHECK(0X20)
    FLG_HEAP_VALIDATE_PARAMETERS(0X40)

所以执行**《peb->NtGlobalFlag &= ~0x70;》**语句，将上述三个标记清除。

## scl::PebPatchProcessParameters
进程参数信息的数据结构是RTL_USER_PROCESS_PARAMETERS，该结构的指针在PEB结构中保存，和BeingDebugged一样，先将参数信息通过ReadProcessMemory读取到Scylla进程，然后修改参数，再将其通过WriteProcessMemory写回被调试进程。

从源码的注释中，我们可以看出，该函数是Scylla对StartUpInfo信息的修改，反汇编windows api GetStartupInfoW，可以看出，就是通过PEB的ProcessParameters成员变量获取出来的。

Scylla将结构体RTL_USER_PROCESS_PARAMETERS中的下面成员置为0，之后执行<code> rupp.Flags |= (ULONG)0x4000;</code>语句，没有查到相关资料，不太明白这个语句什么用途，```求大神指点```。

    struct RTL_USER_PROCESS_PARAMETERS {
        ....
        ULONG StartingX;
        ULONG StartingY;
        ULONG CountX;
        ULONG CountY;
        ULONG CountCharsX;
        ULONG CountCharsY;
        ULONG FillAttribute;
        ....
    }


## scl::PebPatchHeapFlags
PEB中包含堆信息数据，相关变量如下：

    template <typename T, typename NGF, int A>
    struct _PEB_T
    {
        ....
        DWORD NumberOfHeaps;
        DWORD MaximumNumberOfHeaps;
        T ProcessHeaps;
        ....
    }

NumberOfHeaps表示当前堆的个数，ProcessHeaps为堆地址数组。堆地址首部对应结构体_HEAP（《软件调试》23.4节内容），结构体_HEAP中包含成员变量flags和force_flags，这两个值就是反反调试的核心值。

当进程被调试的时候，flags的HEAP_GROWABLE位被清除；ForceFlags被设置为非0值。（参考文章：[Anti-Debug Protection Techniques: Implementation and Neutralization](https://www.codeproject.com/Articles/1090943/Anti-Debug-Protection-Techniques-Implementation-an?msg=5242911)中描述）

进程创建的时候会创建一个堆，被称为默认堆，即ProcessHeaps的第一个元素，默认堆的flags与其它堆的flags处理方式不一样，详细参考源码。

        if (i == 0)             // 进程默认堆
        {
            // Default heap.
            *flags &= HEAP_GROWABLE;
        }
        else
        {
            // Flags from RtlCreateHeap/HeapCreate.
            *flags &= (HEAP_GROWABLE | HEAP_GENERATE_EXCEPTIONS | HEAP_NO_SERIALIZE | HEAP_CREATE_ENABLE_EXECUTE);
        }

非默认堆flags为何设置了其它三个标志，未能找到答案，```求大神指点```。


[//]: <> (## Anti-Debug测试)

[//]: <> ( - 将被调试进程调试标志位设置为1，附加调试器，检测该标志位如果为0，则表示被ScyllaHide《或者别的调试器》修改了该标志位，通过该方法即可检测到调试器。 )

[//]: <> (- 堆数据结构操作，自己创建一块有特殊属性的堆，如果被Scylla或者其他调试器修改了，就表示检测到调试器。)

## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。

## 参考文档
- [180306 逆向-反调试技术（1）BeingDebugged](https://blog.csdn.net/whklhhhh/article/details/79656200)
- [调试器检测技术 - 2、PEB.NtGlobalFlag , Heap.HeapFlags, Heap.ForceFlags](https://blog.csdn.net/zhoujiaxq/article/details/23169587)
- [Windows下反反调试技术汇总](https://www.imuo.com/a/b578b307f41f49216d09a4b02a7fb3b056559669d6b4cdba78827de49856cb0d)
- [Anti-Debug Protection Techniques: Implementation and Neutralization](https://www.codeproject.com/Articles/1090943/Anti-Debug-Protection-Techniques-Implementation-an?msg=5242911)
- 张银奎《软件调试》
