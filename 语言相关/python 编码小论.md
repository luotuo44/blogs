**假定:** 本文所描述的都是针对`python 2`版本，对于`python 3`版本会有所不同。

# 编码

编码是什么呢？首先我们来回忆一下`ascii`是什么。简单来说就是一个映射关系：将字符映射成某个数字。比如将大写字母`A`映射成数字65。为何需要这个映射？因为内存只能存储二进制，也就是数字。

`ascii`只有8比特，因此最多也就只能表示`256`个不同的字符。因此也就出现了其他的编码，比如`GBK`和`UNICODE`。同一个字符对于不同的编码格式，拥有不同的数值。换言之，解析数值的时候需要知道编码格式。

# 何时需要编码

1. 保存文件到磁盘时，需要选定一种编码格式。也就是将文件内容编码成字节流保存到磁盘
2. `python`读取文件内容时，需要以某个特定的编码格式解析文件的字节流
3. `python`内部以何种编码格式表示字符串，注意这里是字符串，不是字节流
4. `python`将字符串转换成字节流时，采用何种编码格式。`print`往往需要这种转换
5. 终端要显示字节流时，以何种编码格式解析字节流。

# 场景

## 保存文件

`Windows`的可视编辑器，一般是可以选定一种编码格式保存文件的。一般来说，都应该保存为`UTF-8`。在`VIM`环境，可以使用`:set encoding`查看`VIM`使用编码格式，这个格式是指你键入的内容在内存中所采用的编码格式。命令`:set fileencoding`则 表示当前文件所采用的编码格式。而`:set encoding=utf-8`表示设置`VIM`内部使用的编码格式为`UTF-8`，命令`:set fileencoding=utf-8`表示将文件保存格式设置为`UTF-8`格式。

**建议：** 所有的文件都保存为`UTF-8`格式，后面的论述也是以这个编码格式保存文件。

## python读取文件

`python`默认使用`ascii`格式解析`py`文件的，因此当`py`文件含有中文的时候会报错。

比如文件

```python
#!/bin/python

ss = "中文"
print ss
```

在执行时会报错

```text
File "./t.py", line 3
SyntaxError: Non-ASCII character '\xe4' in file ./t.py on line 3, but no encoding declared; see http://python.org/dev/peps/pep-0263/ for details
```

这个错误的原因应该不难理解，因为`ascii`编码无法表示汉字。解决方法也很简单：告知`python`以`UTF-8`编码格式解析文件。告知方式相信也很常见，就是在上面脚本的第二行输入`#-*- coding=utf-8 -*-`即可。



## python内部编码格式

`python`内部是以`UNICODE`方式表示字符串的。

在`python 2`中，有两种字符序列类型：`str`和`unicode`。下面代码的输出

```python
#!/bin/python
#-*- coding=utf-8 -*-

s = "中文"
print "s type is ", type(s)

ss = u"中文"
print "ss type is ", type(ss)
```

分别为:

```text
s type is  <type 'str'>
ss type is  <type 'unicode'>
```

简单来说，`str`类型表示的是字节数组(类似C语言的`char`数组)，具体是什么编码并不重要，而`unicode`类型则是`UNICODE`字符数组。

### 显式转换

由于`UNICODE`的字符集强大并且在`python`内部采用的类型。因此由`UNICODE`转换为其他编码格式，用**编码**这个词，而由其他编码转换为`UNICODE`时则用**解码**这个词。因此有下面这种写法

```python
 #!/bin/python
 #-*- coding=utf-8 -*-
 
 s = "中文"
 t1 = s.decode("utf-8")
 print "t1 type is ", type(t1)
 t2 = t1.encode("gbk")
 print "t2 type is ", type(t2)
```

输出结果为：

```text
t1 type is  <type 'unicode'>
t2 type is  <type 'str'>
```



上面代码在编解码时指定编码格式才是标准的用法，像下面这种用法有点山寨。

```python
s = "test"
ss = unicode(s)
sss = str(ss)
```

这代码在只有英文字符的情况下，也能正常跑。但如果变量`s`的内容含有中文，那么会报下面错误:

```text
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 2: ordinal not in range(128)
```

可以观察到，是`DecodeError`解码错误。究其原因，是因为`unicode(ss)`这行代码在不指定`s`的编码格式的情况下，默认采用了`sys.getdefaultencoding()`指向的编码格式解码，而该函数的返回值一般是`ascii`。知道了这点，改起来也就有方向了。

1. 设置默认的编码格式

   执行下面代码即可

   ```python
   reload(sys)
   sys.setdefaultencoding("utf-8")
   ```

   但这种方式过于粗暴，不仅仅改变了你写代码的默认编码方式，还会影响第三方库。涉及面过广，难于把控，不建议使用。具体可以阅读[Why sys.setdefaultencoding() will break code](https://anonbadger.wordpress.com/2015/06/16/why-sys-setdefaultencoding-will-break-code/)。

2. 解码时指定编码格式

   使用下面代码即可

   ```python
   unicode(s, encoding="utf-8")
   ```

   ​

**PS：** 对于下面代码

```python
s = "中文"
s.encode("gbk")
```

看起来好像不对劲，因为`s`本身就是`str`类型，不应该还能`encode`的。实际上上面代码相当于

```python
s = "中文"
s.decode(sys.getdefaultencoding()).encode("gbk")
```



### 隐式转换

除了前面提及的显式转换会有编码问题，下面代码也会有编码问题

```python
#!/bin/python
#-*- coding=utf-8 -*-

s = "中文"
ss = u"中文"
sss = s + ss #UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0

sss1 = "join %s" % ss #正常
sss2 = "测试 join %s" % ss #UnicodeDecodeError: 'ascii' codec can't decode byte 0xe6 in position 0
sss3 = u"join %s" % s #UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0
```



明显，这里是混用了`str`和`unicode`这两个类型。类似C语言的不同内建类型的二元操作，都是需要先提升某个类型然后才操作的。这里也差不多，由于`python`内部是用`UNICODE`字符集的，因此`python`需要先将`str`类型转换成`unicode`类型。这就涉及到解码操作了，也就是将`str`类型变量解码成`unicode`类型。这里不指定编码格式，那么就会使用`sys.getdefaultencoding()`指明的编码格式进行解码。



#### JSON

`dict`里面免不了存在汉字，并且有时候需要将一个`dict`用`json.dumps`之后存放到`mysql`。这里可能会出现难于阅读的`\u`串。对于这些`\u`串真的看着心烦，特别是存放到`mysql`时，`select`出来都不知道是什么文字。不得不谷歌一下“unicode转中文”。如下代码:

```python
#!/bin/python
#-*- coding=utf-8 -*-

import json

a = {"first name": "张三", "second name": u"张三"}
print "json.dumps(a) type ", type(json.dumps(a))
print json.dumps(a)
```

输出为:

```text
json.dumps(a) type  <type 'str'>
{"first name": "\u5f20\u4e09", "second name": "\u5f20\u4e09"}
```

对于上面代码，可能会有两点疑问：

1. 上面代码同时出现了`str`和`unicode`类型，为什么没有问题？

   这是因为`json.dumps`有一个默认的参数`encoding=utf-8`。因此能正确解码`str`类型。

2. 为什么会有这些`\u`?

   这涉及到`json.dumps`的另外一个参数`ensure_ascii=True`。也就是`json`保证能用`ascii`编码格式能够cover住结果，所以不得不将`UNICODE`字符集都转换成`\u`形式。

如果将`ensure_ascii`设置为`False`呢？对于上面的代码将会报错`UnicodeDecodeError: 'ascii' codec can't decode byte 0xe5 in position 1: ordinal not in range(128)`。这是因为当`ensure_ascii=True`时，`json`内部是先将所有`str`类型解码成`unicode`类型，然后再拼接。如果`ensure_ascii=False`那么就是直接拼接，显然又回到了前面那个隐式转换问题上。
**PS:** 如果是一个层次很深的dict，设置了`ensure_ascii=True`，但`dumps`的时候还报错，那么可以尝试将dict变量print一下，看看里面是否混杂着`str`和`unicode`两种类型。

**注意：** 当`ensure_ascii=False`时，如果含有`unicode`类型，那么`json.dumps`返回的是`unicode`类型；如果不含有`unicode`类型，那么`json.dumps`返回的是`str`类型。



**最佳实践**

1. 保证要被`json.dumps`的字符类型都是`str`，并且将`ensure_ascii`设置成`False`，然后把`dumps`的结果就直接`print`或者传给`mysql`
2. 保证要被`json.dumps`的字符类型都是`unicode`，然后对`dumps`的结果再次`encode("utf-8")`，得到的结果就可以`print`或者传给`mysqwl`了



很可惜的是，无论`dumps`出来的结果是`str`还是`unicode`类型，用`json.loads`之后都是`unicode`类型。



## 交互

使用`python`免不了向终端输出东西，在`Linux`里下面代码会报错：

```python
#!/bin/python
#-*- coding=utf-8 -*-

s = "中文"
print s #正常输出结果
ss = u"中文"
print ss #UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-1
```

`print`语句对于`str`类型，是直接送给标准输出的。因为它本身就是字节流了，可以直接传输。

对于`unicode`类型，需要先将之转换成字节流，也就是需要编码。显然这里也会采用一种默认的编码格式，这个编码是由`sys.stdout.encoding`指定的。`Linux`下往往就是`ascii`，显然`ascii`无法编码汉字，从而报错。

解决方法也很简单，直接手动先将`ss`用`utf-8`编码再打印即可：`print ss.encode("utf-8")`



## 终端

像`secureCRT`这类终端，它需要识别网络里面接收到的字节流的编码格式。如果我们统一使用`UTF-8`，那么直接将`secureCRT`的`Character encoding`选定为`UTF-8`即可。



# 参考

[python2 json的大坑](https://shangliuyan.github.io/2016/06/15/python2-json%E7%9A%84%E5%A4%A7%E5%9D%91/)





