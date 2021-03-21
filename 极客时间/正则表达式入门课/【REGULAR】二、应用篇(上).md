# 【REGULAR】二、应用篇(上)



## 05 | 断言：如何用断言更好地实现替换重复出现的单词？

![](https://static001.geekbang.org/resource/image/b7/42/b78ecae648ee423a434f28814ca97d42.jpg)

今天我来和你聊聊正则断言（Assertion）。

什么是断言呢？简单来说，断言是指对匹配到的文本位置有要求。这么说你可能还是没理解，我通过一些例子来给你讲解。你应该知道 \d{11} 能匹配上 11 位数字，但这 11 位数字可能是 18 位身份证号中的一部分。再比如，去查找一个单词，我们要查找  tom，但其它的单词，比如 tomorrow 中也包含了  tom。

也就是说，在有些情况下，我们对要匹配的文本的位置也有一定的要求。为了解决这个问题，正则中提供了一些结构，只用于匹配位置，而不是文本内容本身，这种结构就是断言。常见的断言有三种：***单词边界、行的开始或结束以及环视***。

![](https://static001.geekbang.org/resource/image/df/db/df5f394cc3c0beaee306881704512cdb.png)

### 5.1 单词边界(Word Boundary)

在讲单词边界具体怎么使用前，我们先来看一下例子。我们想要把下面文本中的 tom 替换成 jerry。注意一下，在文本中出现了 tomorrow 这个单词，tomorrow 也是以 tom 开头的。

> tom asked me if I would go fishing with him tomorrow.
>
> 中文翻译：Tom 问我明天能否和他一同去钓鱼。

利用前面学到的知识，我们如果直接替换，会出现下面这种结果。

> 替换前：tom asked me if I would go fishing with him tomorrow.
>
> 替换后：jerry asked me if I would go fishing with him jerryorrow.

这显然是错误的，因为明天这个英语单词里面的 tom 也被替换了。

那正则是如何解决这个问题的呢？单词的组成一般可以用元字符 \w+ 来表示，\w 包括了大小写字母、下划线和数字（即 [A-Za-z0-9_]）。那如果我们能找出单词的边界，也就是当出现了\w  表示的范围以外的字符，比如引号、空格、标点、换行等这些符号，我们就可以在正则中使用\b 来表示单词的边界。 ***\b 中的 b  可以理解为是边界（Boundary）这个单词的首字母***。

![](https://static001.geekbang.org/resource/image/4d/11/4d6c0dc075aebb6023ebcd791e787d11.jpg)

> \b 匹配这样的位置：它的前一个字符和后一个字符不全是(一个是,一个不是或不存在) \w。
>
> ##### 什么是位置 ?
>
> It's a nice day today.
>
> 'I' 占一个位置，'t' 占一个位置，所有的单个字符（包括不可见的空白字符）都会占一个位置，这样的位置我给它取个名字叫“显式位置”。
>
> 字符与字符之间还有一个位置，例如 'I' 和 't' 之间就有一个位置（没有任何东西），这样的位置我给它取个名字叫“隐式位置”。
>
> ***“隐式位置”就是 \b 的关键！通俗的理解，\b 就是“隐式位置”。***
>
> 此时，再来理解一下这句话：
>
> *如果需要更精确的说法，\b 匹配这样的位置：**它的前一个字符和后一个字符不全是(一个是,一个不是或不存在) \w。***

根据刚刚学到的内容，在准确匹配单词时，我们使用 \b\w+\b 就可以实现了。

下面我们以 Python3 语言为例子，为你实现上面提到的 “tom 替换成 jerry”：

```python
>>> import re
>>> test_str = "tom asked me if I would go fishing with him tomorrow."
>>> re.sub(r'\btom\b', 'jerry', test_str)
'jerry asked me if I would go fishing with him tomorrow.'
```

### 5.2 行的开始或结束

和单词的边界类似，在正则中还有文本每行的开始和结束，如果我们要求匹配的内容要出现在一行文本开头或结尾，就可以使用 ^ 和 $  来进行位置界定。

我们先说一下行的结尾是如何判断的。你应该知道换行符号。在计算机中，回车（\r）和换行（\n）其实是两个概念，并且在不同的平台上，换行的表示也是不一样的。我在这里列出了 Windows、Linux、macOS 平台上换行的表示方式。

![](https://static001.geekbang.org/resource/image/e8/51/e8c52998873240d57a33b6dfedb3a551.jpg)

那你可能就会问了，匹配行的开始或结束有什么用呢？

