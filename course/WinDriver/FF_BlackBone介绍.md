# FF_BlackBone介绍

作为Windows开发人员，经常遇到枚举进程、枚举模块、读写进程内存的操作；Windows安全开发人员更是会涉及注入、hook、操作PE文件、编写驱动。每次都要翻各种资料制造轮子，那么有没有好的开源库解决这个问题呢，目前知道的就Blackbone了，而且它的代码写的很好，完全可以作为教科书来用。

PS：作者DarthTon在github上的头像很有意思，就粘了下来。

![作者头像](https://avatars2.githubusercontent.com/u/2190385?s=400&v=4)


# 名词解释
MSDIA：Debug Interface Access，[官网链接](https://docs.microsoft.com/en-us/visualstudio/debugger/debug-interface-access/debug-interface-access-sdk?view=vs-2019)

DEP ：数据执行保护的英文缩写，全称为Data Execution Prevention。数据执行保护(DEP) 是一套软硬件技术，能够在内存上执行额外检查以帮助防止在系统上运行恶意代码。


# 介绍
- [github网址](https://github.com/DarthTon/Blackbone)
- 最新版本只支持VS2019，如果要使用VS2017，需要选择分支：Branch_bd30dd1（2019/3/6 8:28:08）
- 库包含内容（详细参考github主页README.md）
- [Xenos](https://github.com/DarthTon/Xenos)（基于Blackbone的开源的Windows dll注入工具）


# 库包含内容
- Process interaction（进程交互）
- - 操作PEB32/PEB64
- - 通过WOW64 barrier管理进程
- Process Memory（进程内存）
- - 分配和释放虚拟内存
- - 改变内存保护属性
- - 读写虚拟内存
- Process modules（进程模块）
- - 枚举所有进程加载的模块 (32/64 bit)
- - 获得导出函数地址
- - 获得主模块
- - 从模块列表中去除模块信息
- - 注入、卸载模块(支持 pure IL images)
- - 在WOW64进程中注入64位模块
- - 操作PE
- Threads（线程）
- - 枚举线程
- - 创建和关闭线程
- - 获得线程退出码
- - 获得主线程
- - 管理 TEB32/TEB64
- - join线程（等待线程退出）
- - 暂停、恢复线程
- - 设置、清除硬件断点
- Pattern search
- - 特征码匹配（支持本进程和其他进程）
- Remote code execution
- - 在远程进程中执行函数
- - 组装自己的代码并远程执行（shellcode注入）
- - 支持各种调用方式： cdecl/stdcall/thiscall/fastcall 
- - 支持各种参数类型
- - 支持FPU
- - 支持在新线程或已经存在的线程中执行shellcode
- Remote hooking
- - 通过软硬断点，在远程进程中hook函数
- - Hook functions upon return（通过return挂钩函数）
- Manual map features
- - 隐藏注入，支持x86和x64，注入任意未保护的进程，支持导入表和延迟导入等等特性，不一一列举了。
- Driver features
- - 分配、释放、保护内存
- - 读写用户层、驱动层内存
- - 对于WOW64进程，禁用DEP
- - 修改进程保护标记
- - 修改句柄访问权限
- - 操作进程内存（如：将目标进程map到本进程等）
- - 隐藏已分配用户模式内存
- - 用户模式dll注入；手动mapping（手动加载模块）
- - 手动加载驱动


## 编译问题总结
- sdk、wdk版本要相同10.0.17763.0。
- cor.h文件及相关的mscoree.lib库找不到编译错误，需要安装C#模块。
- fatal error LNK1104: 无法打开文件“msvcprtd.lib”；安装Spectre组件
- fatal error LNK1104: 无法打开文件“atls.lib”；安装带Spectre的ATL
- vs2017-xp支持：https://docs.microsoft.com/en-us/cpp/build/configuring-programs-for-windows-xp?view=vs-2019
- 非类型模板参数中的 "auto" 最低要求为 "/std:c++17"；在《项目->属性-> C/C++ -> 语言 -> C++语言标准》中设置c++17或者c++latest都行


编译选项如下图所示：

![BlackBone开发环境设置](https://ninecents.github.io/course/WinDriver/FF_BlackBone介绍/BlackBone开发环境设置.png)

C++17编译错误修正（集成BlackBone到自己的工程会出现这个错误）：

![C++17编译错误](https://ninecents.github.io/course/WinDriver/FF_BlackBone介绍/C++17编译错误.png)


## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。


## 参考文档
