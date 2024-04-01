---
draft: false 
date: 2024-03-31 
categories:
  - Python
comments: true
authors: [Blake]
---


#  我 Review 我自己
以下内容源于我在公司的内部分享，已做脱敏处理。

## 引入

我们之前说过我们每个月都会有一个 code review 的环节，准确来说这不算是一个代码 review，我总结了一些我写代码踩过的一些坑，拎出来跟大家一起讨论一下。
<!-- more -->

## 内容介绍

今天讲四个小点的内容，魔法方法，异常处理，移除代码和反射。

## 魔法方法

魔法方法，官方叫特殊方法（special methods），以双下划线开头双下线结尾的方法，常见的有 `__init__`、`__new__` 等。

对于 Python 内置的一些容器类型，字典，列表，我们是不是可以 `len` 方法来获取它的长度，就是元素个数。那如果我们自己实现了一个容器类型 ，我们也想让它支持通过 `len` 方法来获取它的长度，我们要怎么做呢？

我们先借助 `list` 来实现一个队列。

```python
 class Full(Exception):
    pass


class Empty(Exception):
    pass


class Queue:

    def __init(self, maxsize):
        self.maxsize = maxsize
        self._data = list()


def push(self, item):
    if len(self._data) == self.maxsize:
        raise Full
    self._data.append(item)


def get(self):
    if not self._data:
        raise Empty
    return self._data.pop(0)
```

测试我们实现的这个队列能不能工作。

```shell
>>> q = Queue(10)
>>> q.push(1)
>>> q.push(2)
>>> q.get()
1
>>> q.get()
2
>>> q.get()
Traceback (most recent call last):
...
__main__.Empty
```

回到我们刚刚那个需求，我们也想让它支持通过 `len`  方法来获取它的长度，我们要怎么做呢？

我们可以通过定一个 `__len__` 魔法方法来实现在这样的功能。

```python
def __len__(self):
  return len(self._data)
```

```shell
>>> q = Queue(10)  
>>> q.push(1)
>>> q.push(2)
>>> len(q)
2
```

刚刚我们通过一个例子，介绍了这个 `__len__` 魔法方法的一个功能。当然今天的重点不是教大家这些魔法方法怎么使用，我知道已经有一些同学对魔法方法已经很熟悉了，我今天想说的是魔法方法使用的注意事项。

我们看下 *Fluent Python* 中对魔法方法的一个讲述。它是这样说的。

> 关于魔法方法，你首先要知道的是它们是由 Python 解释器调用的，而不是你自己来调用的。

怎么理解这句话呢？我们再回到我能刚刚的例子里面。

也就是说我们定义的魔法方法，是让它支持 Python 的一些语法机制，我们给自定义的类型定义了 `__len__` 方法，目的是为了让它支持通过 `len` 方法获取长度，而不是说我们要自己调用 `__len__` 方法。

这时候，可能有的同学会问，直接调用调用魔法方法是不是更快？提问环节🙋‍

那我们直接代码实验一下。

```python
q = Queue(1000)
start = time.perf_counter()
for _ in range(10000):
    len(q)
end = time.perf_counter()
print(f"len: {end - start}")  

start = time.perf_counter()
for _ in range(10000):
    q.__len__()
end = time.perf_counter()
print(f"__len__: {end - start}")
```

运行结果如下 👇

```
len: 0.002637699999999993
__len__: 0.0023554999999999965
```

可以看到直接调用魔法方法的确更快，但**性能优化第一部先是找到程序的性能瓶颈，再针对性能瓶颈进行优化**。所以，这里真的是你程序的性能瓶颈吗🤔？

## 异常处理

特别是对于我们 *** 程序，异常处理应该是我们在编码过程中高频遇到的一个场景。忽略异常，然后进行重试，我相信各位开发同学最熟悉不过了。

```python
try:
    ...
except:  # bad
    pass


try:
    ...
except Exception:  # bad
    pass

```

上面的这种代码出现在我之前写的的代码中。我们看下 Python 之禅中对异常处理是怎么描述的。

> Errors should never pass silently.



所以我们的结论就是：不要忽略异常！至少，记录下日志。

```python
try:
  ...
except Exception as e:  # at very least, log it
  logger.error(e)
```

## 移除代码

这段代码已经废弃了，是注释还是删除？提问环节🙋‍

*Refactoring* （重构）中有一个小节专门提到了对于无用代码（dead code）该如果处理，我们看下它的观点。

> 一旦代码不再被使用，我们就该立马删除它。有可能以后又会需要这段代码，但我从不担心这种情况；就算真的发生，我也可以从版本控制系统里再次将它翻找出来。

可能会有同学会问，万一这段代码我后面还会用怎么办？

会不会出现这种情况？确实会，但是非常少。至少在我的工作经历中，代码被删了，后面又用到的这种情况基本没有。所以你不必为了这种极少发生的情况，把不用的代码留下来，你把无用的代码留下来是有成本的。更何况，我们有版本控制系统，版本控制系统的存在，就是来记录的代码的改动的，所以我们把这个工作交给它做了就行了，你删了，还能通过版本控制系统找回来。

所以结论是不用代码的直接删，不要有丝毫犹豫，放心删，大胆删，遗留的无用代码（即使被注释掉）只会白白增加你代码的复杂度，增加阅读者的心智成本。



## 使用反射

Python 中有很多方法利用了反射机制，例如：`type`，`isinstance`，`callable`，`gettarr` 等，本章节的观点只针对 `getattr`，今天先只欺负 `getattr` 它一个。

先简单演示下 `getattr` 的用法。

```python
class Person:

    def __init__(self, name, gendr):
        self.name = name
        self.gender = gender

    @staticmethod
    def eat(food):
        print(f"eating {food}...")


>>> p = Person("Blake", "male")
>>> getattr(p, "name")
Blake
>>> getattr(p, "eat")(meat)
eating meat...
```

好我们现在已经知道了 `getattr` 的用法，当然如何使用不是我们今天的重点，我们更进一步，我们今天的重点是使用注意事项。



我们来看下谷歌 Python 代码风格指南中，对于反射等这一类 ”奇技淫巧“ 的使用提醒。

> 这些很酷（反射、元类等）的特性十分诱人, 但多数情况下没必要使用。包含奇技淫巧的代码难以阅读、理解和调试。 一开始可能还好(对原作者而言)，但以后回顾代码时，这种代码通常比那些长而直白的代码更加深奥。



我自己的业务代码也用到了 `getattr`，确实给我后面增加了一些阅读和调试上的成本，所以大家对与 `getattr` 还是要保持谨慎。

## 参考资料

- [Fluent Python: Clear, Concise, and Effective Programming](https://book.douban.com/subject/27028517/)

- [Refactoring: Improving the Design of Existing Code](https://book.douban.com/subject/30468597/)

- [PEP 20 – The Zen of Python | peps.python.org](https://peps.python.org/pep-0020/)

- [Avoiding Silent Failures in Python: Best Practices for Error Handling](https://pybit.es/articles/python-errors-should-not-pass-silently/)

- [Google Python Style Guide](https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/contents/)