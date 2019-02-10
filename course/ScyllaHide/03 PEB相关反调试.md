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

## 获取指定进程PEB地址
PEB包含了调试相关信息，通过修改被调试进程的相关值就可以实现反反调试功能，那么怎么获得被调试进程的PEB地址就成了PEB反调试的关键。

    scl::PEB *scl::GetPebAddress(HANDLE hProcess)
    {
        ::PROCESS_BASIC_INFORMATION pbi = { 0 };

        auto status = NtQueryInformationProcess(hProcess, ProcessBasicInformation, &pbi, sizeof(pbi), nullptr);

        return NT_SUCCESS(status) ? (PEB *)pbi.PebBaseAddress : nullptr;
    }
<注释> 关于Wow64，暂时没有研究忽略
NtQueryInformationProcess
Wow64QueryInformationProcess64

## isDebugged

## 

## 堆，默认堆


## Anti-Debug测试
- 将被调试进程调试标志位设置为1，附加调试器，检测该标志位如果为0，则表示被ScyllaHide(或者别的调试器)修改了该标志位，通过该方法即可检测到调试器。

- 堆数据结构操作，自己创建一块有特殊属性的堆，如果被Scylla或者其他调试器修改了，就表示检测到调试器。