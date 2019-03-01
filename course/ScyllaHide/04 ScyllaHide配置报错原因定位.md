# [ScyllaHide] 文章列表-看雪地址：

- [00 简单介绍和使用](https://bbs.pediy.com/thread-249151.htm)
- [01 项目概览](https://bbs.pediy.com/thread-249305.htm)
- [02 InjectorCLI源码分析](https://bbs.pediy.com/thread-249306.htm)
- [03 PEB相关反调试](https://bbs.pediy.com/thread-249374.htm)
- [04 ScyllaHide配置报错原因定位](https://bbs.pediy.com/thread-249524.htm)
- [05 ScyllaHide的Hook原理](https://bbs.pediy.com/thread-249721.htm)


# ScyllaHide配置报错原因定位

使用OD进行调试的时候，一直报错，如下图所示：

![OD报错截图](https://ninecents.github.io/course/ScyllaHide/04%20ScyllaHide配置报错原因定位/OD报错截图.png)

下面我们一步一步的开始分析错误原因，分享下解决问题的思路，希望对大家有一定的帮助。

## 定位问题

通过关键字“[ScyllaHide] NtUser* API Addresses are missing!”定位到下面代码：

    scl::NtApiLoader api_loader;
    auto res = api_loader.Load(file);
    if (!res.first)
    {
        g_log.LogError(L"Failed to load NT API addresses: %s", res.second);
        return;
    }

    hde->NtUserQueryWindowRVA = (DWORD)api_loader.get_fun(L"user32.dll", L"NtUserQueryWindow");
    hde->NtUserBuildHwndListRVA = (DWORD)api_loader.get_fun(L"user32.dll", L"NtUserBuildHwndList");
    hde->NtUserFindWindowExRVA = (DWORD)api_loader.get_fun(L"user32.dll", L"NtUserFindWindowEx");

    g_log.LogInfo(L"Loaded RVA for user32.dll!NtUserQueryWindow = 0x%p", hde->NtUserQueryWindowRVA);
    g_log.LogInfo(L"Loaded RVA for user32.dll!NtUserBuildHwndList = 0x%p", hde->NtUserBuildHwndListRVA);
    g_log.LogInfo(L"Loaded RVA for user32.dll!NtUserFindWindowEx = 0x%p", hde->NtUserFindWindowExRVA);

    if (!hde->NtUserQueryWindowRVA || !hde->NtUserBuildHwndListRVA || !hde->NtUserFindWindowExRVA)
    {
        g_log.LogError(
            L"NtUser* API Addresses are missing!\n"
    ...
    
从上面的代码可以看出，api_loader.get_fun失败导致该报错信息。通过第二节[02 InjectorCLI源码分析](https://ninecents.github.io/course/ScyllaHide/02%20InjectorCLI源码分析)的分析，结合调试，我们可以发现，Scylla没有找到配置文件NtApiCollection.ini中的“6.1.3.0.1.9.x86”section(节区)。

配置文件NtApiCollection.ini，是由PDBReaderx86.exe生成的，打开生成的配置文件，我们可以看到下面内容：

    [6.1.1.0.1.9.x86]
    user32.dll!1b6fa!NtUserBuildHwndList=193f6
    user32.dll!1b6fa!NtUserFindWindowEx=167dd
    user32.dll!1b6fa!NtUserQueryWindow=16915

我们发现，配置文件中只有一个section(节区)：[6.1.1.0.1.9.x86]。显然节区内容不匹配，咋整呢，先调试下，定位节区产生的地址吧。

## 调试ScyllaHide

调试ScyllaHide，其实就是调试OD2，我们按照下面流程进行调试：
- 将编译好的Debug版本ScyllaHideOlly2Plugin.dll放置到OD插件目录（我用的OD2.0，所以使用的ScyllaHideOlly2Plugin.dll，使用OD1.x的同学，需要用ScyllaHideOlly1Plugin.dll）。
- 启动OD，并使用VS附加。
- 在函数ReadNtApiInformation入口下断点。
- 使用OD打开任意x86进程，我这里使用的是calc.exe

这时VS会断到函数ReadNtApiInformation入口，我们逐语句（F11）调试，很快能定位到节区【6.1.3.0.1.9.x86】产生的地方，如下图所示堆栈：

![section获取堆栈](https://ninecents.github.io/course/ScyllaHide/04%20ScyllaHide配置报错原因定位/section获取堆栈.png)

其中字符串【6.1.3.0.1.9.x86】中的3就是osVerInfo->wServicePackMajor对应的数字，查看MSDN可以知道，3是系统的补丁号。

使用命令行winver查看系统版本信息，如下图：

![winver查看系统版本](https://ninecents.github.io/course/ScyllaHide/04%20ScyllaHide配置报错原因定位/winver查看系统版本.png)

系统版本分明是1，这难道是系统出问题了？记得有位前辈说过“计算机绝对真实的执行自己的任务”，肯定还有未考虑到的地方，难道是OD做了什么手脚？针对这个假想，我继续开始调试GetVersionExW函数，堆栈信息如下图：

![VS汇编调试GetVersionExW函数](https://ninecents.github.io/course/ScyllaHide/04%20ScyllaHide配置报错原因定位/VS汇编调试GetVersionExW函数.png)

我们知道GetVersionExW是Kernel32.dll的导出函数，而截图中，调用的却是AcLayers.dll。这究竟是怎么回事儿？网上搜了下，发现它是兼容性dll（[文章链接](https://www.file.net/process/aclayers.dll.html)）。


## 兼容性问题定位

本以为已经定位到问题了，可是打开OD2的属性页面，并没有发现兼容性设置。怎么办，怎么办？迷茫中，通过Explorer.exe打开了一次OD2，发现没有出现错误提示。忽然想到一直以来都是通过“吾爱破解工具箱”打开的OD2，会不会是因为“吾爱破解工具箱”的原因呢？赶快打开“工具箱”的属性页，发现的确被设置为了XP-SP3，如下图所示：

![吾爱破解工具箱兼容性](https://ninecents.github.io/course/ScyllaHide/04%20ScyllaHide配置报错原因定位/吾爱破解工具箱兼容性.png)

将兼容性设置取消，重新打开“吾爱破解工具箱”，通过“工具箱”打开OD2，再打开calc.exe，没有提示错误对话框，一切安静了。

## 总结

- 源码之前了无密码，其实即使没有源码，我们通过逆向找到问题答案。本节就是结合源码和汇编一起定位问题的。
- ScyllaHide的NtApiCollection.ini配置信息是通过系统的版本号进行section区分的。
- 兼容性设置是会遗传的。


## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。
