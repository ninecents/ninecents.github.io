# ScyllaHide的Hook原理

Hook是通过修改程序代码或数据，达到改变程序逻辑的目的。Hook种类很多，ScyllaHide主要使用的是inline hook，也就是在运行的流程中插入跳转指令（call/jmp）来抢夺程序运行流程的一个方法。

Hook根据系统版本（win7、win8、win10等）、cpu架构（x86、x64）、子系统类型（是否是wow64进程），实现由细微差别。本节只讨论win7、x64下32位进程（即wow64进程）的情况，其他情况请大家自己分析。


## ApplyHook函数调用结构图

下面是Hook的整体流程，非目标环境（win7、x64下32位进程）的函数直接忽略了。

![ApplyHook函数调用结构图](https://ninecents.github.io/course/ScyllaHide/03%20PEB相关反调试/ApplyHook函数调用结构图.png)


## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。


## 参考文档
- [180306 逆向-反调试技术（1）BeingDebugged](https://blog.csdn.net/whklhhhh/article/details/79656200)
- [调试器检测技术 - 2、PEB.NtGlobalFlag , Heap.HeapFlags, Heap.ForceFlags](https://blog.csdn.net/zhoujiaxq/article/details/23169587)
- [Windows下反反调试技术汇总](https://www.imuo.com/a/b578b307f41f49216d09a4b02a7fb3b056559669d6b4cdba78827de49856cb0d)
- [Anti-Debug Protection Techniques: Implementation and Neutralization](https://www.codeproject.com/Articles/1090943/Anti-Debug-Protection-Techniques-Implementation-an?msg=5242911)
- 张银奎《软件调试》
