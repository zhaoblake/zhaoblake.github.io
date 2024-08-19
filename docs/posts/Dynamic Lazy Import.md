---
date: 2024-08-18
categories:
  - Python
  - LangChain
  - API Design
comments: true
authors: [ Blake ]
---

# Dynamic Lazy Import

在看 LangChain 代码的时候，我注意到它在包的 `__init__.py` 中采用了一种特别的导入设计。起初，我并未完全理解这种设计的用意，直到最近，在设计内部框架的回调接口时，遇到了一种场景，才明白这种导入设计的目的。

<!-- more -->

## 设计回调接口时遇到的问题
先看下我们代码库结构。newbee（代称） 是我们框架目录，biz 是我们业务目录。

```shell
├─newbee
│  ├─callback.py
│  ...
├─biz
   ├─a_task.py
   ├─b_task.py
   └─c_task.py
   ...
```

可以简单把 newbee 理解成一个任务执行器：请求任务，并根据映射，找到对应的任务对象，并执行并返回任务结果。而任务对象，就存放在
biz 目录下的每个任务文件中。

下面是我们的回调注册接口，借鉴了 Flask 回调接口的设计。

```python
# Flask 中的回调注册接口
from flask import Flask

app = Flask(__name__)


@app.before_request
def foo():
    ...


@app.after_response
def bar():
    ...
```

```python
# newbee 的回调注册接口
from newbee import callback


@callback.before_task
def foo():
    """任务执行前执行"""


@callback.after_task
def bar():
    """任务执行后执行"""
```

回调接口设计好了，假设我要注册一个 before_task 的回调，这个回调函数会在每个任务执行前执行。那么下面这段注册的代码，要放在哪里？

```python
from newbee import callback


@callback.before_task
def foo():
    """任务执行前执行"""
```

首先，回调是业务相关，注册的回调的代码应该放在 biz 目录中，这个没有问题。问题是如果将注册的代码放在某一个任务代码文件中，如
`a_task.py` 中，只有在执行 `a_task.py` 中任务的时候，这个回调才会生效。要想实现“这个回调函数会在每个任务执行前执行”的效果，就得在每个任务文件中都注册一遍，这显然是不合理的。

为了解决重复注册的问题，我给 biz 目录下增加一个 `__init_.py` 文件，将所有的任务都导入到 `__init__.py` 文件中，统一出口。newbee
通过 `from biz import xTask` 来导入任务，这样不管执行的什么任务，都会加载 `__init__.py` 中的代码，进行回调注册。


```shell
├─newbee
│  ├─callback.py
│  ...
├─biz
   ├─__init__.py
   ├─a_task.py
   ├─b_task.py
   └─c_task.py
   ...
```

```python
# biz/__init__.py
from a_task import ATask
from b_task import Btask
from c_task import Ctask

...

from newbee import callback


@callback.before_task
def foo():
    ...
```

biz 目录存放着几百个任务，由我们十几个同事共同维护，且每天的变动十分频繁。多人协助 + 频繁的变动 = 出现 bug 的概率变高。

将所有任务导入到一个文件中 (`biz/__init_py`) ，就像是把一个鸡蛋放在一个篮子里面，一但一个代码文件异常了，所有的任务都无法正常执行，这是我们无法接受的。

既然 bug 不可避免，那就将 bug 的影响范围降到最小。我们希望导入 ATask 时发生的异常，只会影响 ATask 任务，其他任务还可以正常导入和执行。

于是，我们给每个导入，都加了异常捕获。看似简单粗暴，却也解决了实际问题。

```python
# biz/__init__.py
try:
    from a_task import ATask
except Exception as e:
    logger.error(e)

try:
    from b_task import BTask
except Exception as e:
    logger.error(e)

try:
    from c_task import CTask
except Exception as e:
    logger.error(e)
...
```

但是，正如我上面所说，我们拢共几百个任务、几十个代码文件，为每个导入都加入这样的异常处理，既繁琐也不优雅。

## LangChain 和 Dynamic Lazy Import

[LangChain]([GitHub - langchain-ai/langchain: 🦜🔗 Build context-aware reasoning applications](https://github.com/langchain-ai/langchain))
是一个大模型应用开发框架，它提供了用于集成大模型的工具和组件，帮助开发者更方便地将大模型与外部数据源、数据库、API
等集成，构建出更复杂和实用的应用。

LangChain 中的文档加载器（document loader）用来加载各种格式、各种数据源的数据。如 `TextLoader`
是用来加载本地文本数据，`WikipediaLoader` 用来加载（爬取）维基百科的数据。截住目前，LangChain 一共支持将近 200 种文档加载器。

在 Python 中，将接口导入到 `__init__.py` 文件中并对外暴露是一种常见的做法，LangChain 也不例外。

如此众多数量的文档加载器，就面临和我们框架一样的问题：在 `document_loaders/__init__.py` 中，一个文档加载器 import 失败，会导致其他文档加载器也不可用。

LangChain 用了一种我称之为 Dynamic Lazy Import 的方法进行导入，巧妙❓地解决了这个问题。

LangChain 并没有在 `document_loaders/__init__.py`  直接导入 loader, 而是通过定义一个 `__getattr__` 配合 `importlib`  动态地懒导入 loader。

当 `from langchain_community.ducoment_loaders import WikipediaLoader` 时，会触发

`ducoment_loaders/__init__.py` 中的 `__getattr__` 方法导入 `WikipediaLoader` 。这样就实现了，仅当我需要使用某个 loader
时，才导入某个 loader，不会将所有的 loader 都导入到 `document_loaders/__init__.py` 中。既统一了出口，又避免了 loader 之间的影响。

```python
# langchain_community/document_loaders/__init__.py
_module_lookup = {
    "BiliBiliLoader": "langchain_community.document_loaders.bilibili",
    "CSVLoader": "langchain_community.document_loaders.csv_loader",
    "TextLoader": "langchain_community.document_loaders.text",
    "WikipediaLoader": "langchain_community.document_loaders.wikipedia",
    # ...
}


def __getattr__(name: str) -> Any:
    if name in _module_lookup:
        module = importlib.import_module(_module_lookup[name])
        return getattr(module, name)
    raise AttributeError(f"module {__name__} has no attribute {name}")
```

## Simple is better than complex!
看到这里，你可能会认为，我会借鉴 LangChain 的这种导入设计，来解决上面提到的内部框架的问题。
实际上，我还是维持了最初的解决方案：为每个导入语句做异常捕获。LangChain 的这种设计，在我看来，副作用大于其疗效。

LangChain 为了解决因为动态导入，导致 mypy 检查不通过的问题，又在 `document_loaders/__init__.py` 中 的 `if TYPE_CHECKING` 语句下，又将所有的 loader 直接导入了。加之该文件中又维护了 `__all__`，也就是说，新增一个 loader：
1. 要在 `_module_lookup` 字典中维护导入路径；
2. 要在 `if TYPE_CHECKING` 语句下进行直接导入；
3. 要在 `__all__` 中维护 loader 的名称。

不仅变得更加繁琐，而且增加了代码复杂度，降低了可读性，这也是为什么一个 `__init__.py` 文件，代码行数逼近 1000 行的原因。