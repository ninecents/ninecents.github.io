# InjectorCLI源码分析

## 项目依赖关系
Scylla项目不算复杂，项目间的依赖关系也很简单。基础库依赖于底层库，其他所有可执行文件项目依赖于基础库，具体描述如下所示：

	底层库：distorm.lib（反汇编引擎）
	基础库：Scylla.lib（Scylla通用库）
	可执行文件：其他所有可执行文件项目

## 源码分析
先上一张总的函数调用图，如下：

![InjectorCLI源码分析总图](https://ninecents.github.io/course/ScyllaHide/02%20InjectorCLI源码分析/InjectorCLI源码分析总图.png)
