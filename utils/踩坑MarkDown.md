# 踩坑MarkDown


## github的MarkDown常见问题
#### 链接地址不加.md后缀

| 正确示例 | 错误示例 |
| ------------- | ------------- |
| [00 简单介绍和使用](https://ninecents.github.io/course/ScyllaHide/00%20简单介绍和使用) | [00 简单介绍和使用](https://ninecents.github.io/course/ScyllaHide/00%20简单介绍和使用.md) |

#### 表格使用例子（注意前后有空行，否则github不认识）

| Header Cell | Header Cell |
| ------------- | ------------- |
| Content Cell | Content Cell |

#### 注释使用示例

[//]: <> (This may be the most platform independent comment
    ## Todo:
2019-2-11
MarkDown注释；
xmind文件提交；
html静态文本测试-ascii。)

#### Markdown编辑表格时如何输入竖线
http://www.itboth.com/d/6v2qAb/markdown

```|```

`|`

<code>|</code>

| a | r | 
| ----- | ----- | 
| `a += x;` | r1 | 
| a &#124;= y; | r2 |