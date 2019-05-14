# which原理

Linux下很多好用的命令行工具，各种复杂的操作，几个命令就完成了。本节将要说的就是which命令。

## which介绍及类似命令对比

- which 查看可执行文件的位置,从全局环境变量PATH里面查找对应的路径,默认是找 bash内所规范的目录

- whereis 查看文件的位置,配合参数-b，用于程序名的搜索，从linux数据库查找。

- locate 配合数据库查看文件位置。

- find 实际搜寻硬盘查询文件名称,效率低。

windows下也可以使用Linux命令，只要你安装了Cygwin等程序即可；当然，如果你安装了git，也可以找到很多Linux命令工具（我安装在D盘了，目录为D:\Program Files\Git\usr\bin\which.exe）。

下面演示下程序的使用：

```
C:\Users\Administrator>"D:\Program Files\Git\usr\bin\which.exe" cmd
/c/Windows/system32/cmd

C:\Users\Administrator>"D:\Program Files\Git\usr\bin\which.exe" adb
/usr/bin/which: no adb in (/c/Windows/system32:/c/Windows:/c/Windows/System32/Wbem:/c/Windows/System32/WindowsPowerShell/v1.0:/c/Program Files/Microsoft SQL Server/110/Tools/Binn:/cmd:/d/Program Files/TortoiseGit/bin:/c/Users/Administrator/AppData/Local/Microsoft/WindowsApps)
```

可以看出，如果能找到目标程序cmd，which将显示其全路径；如果没有找到目标程序，将提示出未找到，并打印搜索路径。

## which实现原理

从上面的例子可以猜测，which通过系统搜索规则，查找相关路径下是否有目标程序。

具体实现可以查看源码，源码地址为[http://mirrors.ustc.edu.cn/gnu/which/which-2.21.tar.gz]()，其中找到which.c可以查看详细实现。

## 另类实现

windows有多个相同名称应用的时候，经常遇到需要确认程序路径的情况，比如adb.exe，常常因为多个版本导致异常结果。一直想在windows下写个类似的which程序的功能，又觉得太过琐碎，实现起来麻烦。不过，今天忽发奇想，LoadLibrary函数加载PE文件的搜索路径和which搜索路径如出一辙，何不通过LoadLibrary函数实现which功能呢。

于是写了下面代码：
```
int main(int argc, char **argv)
{
	if (argc != 2)
	{
		std::cout << "必须有一个参数！\n";
		return -1;
	}

	HMODULE module = LoadLibraryA(argv[1]);
	printf("module = 0X%08X\n", (unsigned int)module);

	char szModuleFileName[MAX_PATH] = { 0 };
	GetModuleFileNameA(module, szModuleFileName, MAX_PATH);
	printf("%s 的全路径是 %s\n", argv[1], szModuleFileName);
	getchar();

	return 0;
}
```

设置adb环境变量前，运行结果如下图所示，不能正确查找到目标进程路径：

![设置adb环境变量前](https://ninecents.github.io/tools/which实现原理/设置adb环境变量前.png)

设置adb环境变量后，运行结果如下图所示，正确地将目标进程路径打印了出来：

![设置adb环境变量后](https://ninecents.github.io/tools/which实现原理/设置adb环境变量后.png)

相关源码已上传git：[https://github.com/ninecents/MyOpen/tree/master/tools/tools]()


## 广而告之
九分出品，欢迎吐槽。更多精彩，可以前往[博客地址](https://ninecents.github.io)。
