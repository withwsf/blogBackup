title: Python在Windows环境下处理文件路径问题最佳实践
date: 2015-12-30 14:45:38
tags: [Python]
---
Windows中路径分隔符是反斜线'\'，而在Python中'\'又有转义符的作用，因而直接从windows资源管理器复制的路径在Python中是不能正常识别的。

* **最优实践：**
使用os.path.join来join不同的路径，比如
```Python
path=os.path.join(dirpath,filepath)
```
也可以使用os.sep，python会根据不同的系统自动选择合适的路径分隔符
```python
path=dirpath+os.sep+filepath
```
* **次优方案**
可以将所有路径都使用正斜线'/'，不管在Windows和Linux中都适用.
* **不要使用**
在引用的字符串前面加上'r'可以将转义字符串(escaped strings )转换为原始字符串(raw strings)，可以解决部分问题。比如：
```python
file=open('c:\myfile') #打开错误
file=open(r'c:\myfile')#可以正确打开
```
但是'r'是为了方便书写正则表达式而不是为了解决Windows下文件路径问题而设计的特性，所以会遇到一下问题

>file=open(r'c:\dir\'+'myfile')

上述代码是错误的，虽然'\'失去了转义作用，但仍然有保护作用，其后的‘'’并不会被视为closing delimiter。


由此可见，使用r不仅没有完全解决问题，还会引入新的问题，所以最好不要使用。

