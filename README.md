# 欢迎来到九分杂谈


## ScyllaHide系列教程

- [00 简单介绍和使用](https://ninecents.github.io/course/ScyllaHide/00%20简单介绍和使用)
- [01 项目概览](https://ninecents.github.io/course/ScyllaHide/01%20项目概览)
- [02 InjectorCLI源码分析](https://ninecents.github.io/course/ScyllaHide/02%20InjectorCLI源码分析)
- [03 PEB相关反调试](https://ninecents.github.io/course/ScyllaHide/03%20PEB相关反调试)
- [04 ScyllaHide配置报错原因定位](https://ninecents.github.io/course/ScyllaHide/04%20ScyllaHide配置报错原因定位)
- [05 ScyllaHide的Hook原理](https://ninecents.github.io/course/ScyllaHide/05%20ScyllaHide的Hook原理)


## frida系列教程
- [00_简单介绍和使用](https://ninecents.github.io/course/frida/00_简单介绍和使用)


## Qt逆向系列教程
- [00_Qt资源解析](https://ninecents.github.io/course/Qt/00_Qt资源解析 )


## 开源库
- [Blackbone](https://github.com/DarthTon/Blackbone)
- [Detect-It-Easy](https://github.com/horsicq/Detect-It-Easy)


## 常用工具
- [ASCII码对照表](https://ninecents.github.io/utils/ASCII码对照表.html)


## 杂项
- [打包Qt应用程序](https://ninecents.github.io/utils/打包Qt应用程序)
- [踩坑MarkDown](https://ninecents.github.io/utils/踩坑MarkDown)
- [有趣的字节](https://ninecents.github.io/utils/interesting/有趣的字节)
- [chrome的console代码集锦](https://ninecents.github.io/utils/interesting/chrome的console代码集锦)
- IDA使用技巧：
  - 函数（尤其是导入表、Qt Creator创建的exe）解析：Options▶Demangled Names▶Assume GCC；设置完成后，连函数调用都被正常解析了（GCC编译的函数，堆栈在函数入口即被分配完成，函数传参，直接通过赋值[esp+0xFF]实现）。  参考文章https://blog.csdn.net/hgy413/article/details/50589942
  - 字符串解析为UTF8：Options▶ASCII string style▶change encoding/Set default encodings。  参考文章https://blog.csdn.net/C147258hong/article/details/52808663
- QT窗口程序逆向思路：
  - 关键函数：QComboBox::insertItem、QString::text() const等
  - 函数调用传参数
  - 想知道某个值是怎么生成的，通过CE和OD，不断下断点搜索内存，定位关键函数。如果可以确认是WindowsAPI或库函数，可以直接通过IDA查看函数调用关系，加快定位速度。



## TODO：
- 反调试测试之IsBadStringPtr系列
- 九分工具箱（补充吾爱破解工具包2.0）：
  - 导入导出函数：Depends.exe、（C++函数解析）
  - C++函数解析：demumble.exe
  - 其他：查看进程各种数据结构r0、r3 [xntsv](https://github.com/horsicq/xntsv)


## 关于作者

[邮箱](mailto:3357427767@qq.com)

qq：3357427767

如果您觉得我的文章对您有用，请随意打赏（微信），您的打赏是我持续下去的动力，谢谢。

![微信收款码](微信收款码.jpg)


copyright@2019~2019
 
