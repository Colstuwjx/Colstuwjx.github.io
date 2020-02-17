---
title: "[ 三年后的杂谈 ] 测试之Mock"
date: 2017-10-14T20:00:00+08:00
tags: ["talk", "mock", "devops", "test"]
categories: ["test"]
comments: false
showMeta: false
showAction: false
---

笔者最近开始补全测试方面的一些基础知识，由于也算是个人职业技能建设的一部分，权且也算作"三年小结"系列的一篇罢。

<!--more-->

> TIPS：测试领域的水非常深，本人只是班门弄斧，欢迎测试领域的大佬们拍砖~

## Mock

### why mock

在软件测试的领域里，单元测试被视为一个项目代码覆盖率的重要衡量指标。我们在执行单元测试时，往往需要测试和关注的是自身组件代码的稳定性，而非外界依赖\服务的交互（这块的测试放到了集成测试的部分），也因此产生了伪造调用依赖/服务的需求。

通过mock，我们可以假装调用了xxx服务，或者xxx sdk的方法，甚至于可以借此测试一些复杂的场景，比如重试部分的代码是否符合预期等等。

### mock in Python

Python的mock库为我们提供了完善的功能，可以很好地实现伪造请求/调用的需求。

### MagicMock

我们一起来看看下面这个例子：

```
# a simple module with a simple class
class AClass(object):
    def method(self, an_argument):
        print "I typed {}".format(an_argument)
```

```
>>> from mock import MagicMock
>>> import amodule
>>> thing = amodule.AClass()

# method方法替换成MagicMock实例
# 这里传入参数指定了return_value，因此调用MagicMock伪造的method方法，将会返回3！
>>> thing.method = MagicMock(return_value=3)
>>> print thing.method('an_arg')
3

# assert_called_with 即验证，我们最近一次调用method方法，是传入了'others'来调用的
# 但是，显然不是:( 因此这行会报错
>>> print thing.method.assert_called_with('others')
...
AssertionError: Expected call: mock('others')
Actual call: mock('an_arg')
```

上面这个例子可以实现什么效果呢？`MagicMock`是mock模块的一个魔法类，它可以实现对现实方法的模拟，在本例子里，MagicMock即替代了`thing`的`method`方法，它会假装接受传入的参数，然后返回值设置为`3`，因此调用`thing.method`时实现的效果便是得到的返回值为`3`。

我们还注意到，这里有调用一个`assert_called_with`方法，那么这个方法的作用是什么呢？正如注释里面所说的，它会去验证最近一次对mock代替的方法的调用，传入的参数是预期的那样，否则会报错`AssertionError`。

当然，MagicMock还有其他的一些功能，比如`side_effect`实现调用该方法执行一些额外操作等。

### Patch

除了MagicMock外，mock模块里另外一个经常用到的是patch。

patch经常被用到的场景是结合with关键字使用，通常它可以用来在with作用域内mock一个或一组方法的调用。

下面是一个mock `requests.get`返回值的例子：
```
import mock
import requests
import json
import unittest


def get_request():
    req = requests.get("http://www.baidu.com")
    # do sth...
    return req.status_code == 200


class TestClient(unittest.TestCase):
    @mock.patch('requests.get')
    def test_get_request(self, mock_request):
        mock_request.return_value = mock.MagicMock(
            status_code=200, response=json.dumps({'key': 'value'})
        )
        self.assertEqual(get_request(), True)


if __name__ == '__main__':
    unittest.main()
```

```
# result
$ python request_get_mock.py
.
----------------------------------------------------------------------
Ran 1 test in 0.001s

OK
```

我们通过patch关键字成功mock了`requests.get`方法的调用！可以看到，使用patch的过程非常的Pythonic！

## 一个完整的单元测试

说了这么多，让我们一起看一个完整的例子来系统了解下现实世界里mock的用法吧！下面代码是一个完整的用来测试博客`Blog`类的`post`方法的测试用例，覆盖了成功、失败、重试等完整逻辑功能，可以从[我的Github](https://github.com/Colstuwjx/python-examples/tree/master/test/testcase)找到可执行的源代码。

```
# coding=utf-8

import unittest
import retrying
import mock
import requests
import json


class RequestError(Exception):
    def __init__(self, server):
        self.message = "Request failed to {}.".format(server)


class Blog(object):
    '''
    Blog类包含一些博客文章相关的操作封装
    类在初始化后，post方法提供对指定文章链接内容的获取并返回JSON的格式
    '''
    def __init__(self, name):
        self.name = name

    @retrying.retry(stop_max_attempt_number=3, wait_fixed=100)
    def post(self, article_path):
        '''
        根据获取文章的内容
        '''
        response = requests.get("http://{name}/{path}".format(
            name=self.name,
            path=article_path))
        if response.status_code >= 200 and response.status_code < 300:
            return response.text
        else:
            raise RequestError(self.name)

    def __repr__(self):
        return '<Blog: {}>'.format(self.name)


class TestBlog(unittest.TestCase):
    '''
    我们需要测试哪些内容？根据K大的http://docs.python-guide.org/en/latest/writing/tests/
    我们关注本身的功能的覆盖，那么主要需要测试：
    1. 正常请求时是否能获得正确的JSON数据
    2. 异常返回时是否抛出RequestError异常
    3. 遇到错误是否会重试
    '''
    @mock.patch('requests.get')
    def test_blog_post(self, mock_get):
        mocked_content = json.dumps({
            'status': True, 'content': 'This is the post content'
        })
        mocked_success = mock.MagicMock(
                status_code=200,
                headers={'content-type': "application/json"},
                text=mocked_content)
        mock_get.return_value = mocked_success

        # new blog instance and call `post`.
        # confirm return value on succcess.
        b = Blog(name="devopstarter.info")
        content = b.post("three-years-later-test-mock")
        self.assertEqual(content, mocked_content)

        mock_get.reset_mock()  # reset mock, include called count etc.
        mocked_error = mock.MagicMock(
                status_code=500,
                headers={'content-type': "application/json"},
                text="Internal server error")
        # confirm raise exception on failure.
        # due to mock object return_value is always error, retry is also error.
        mock_get.return_value = mocked_error
        with self.assertRaises(RequestError):
            b.post("three-years-later-test-mock")
        # confirm the retry works in failure case, call_count exceeded 3!
        self.assertEqual(mock_get.call_count, 3)

        # confirm the retry works while ONLY one time failed.
        # `side_effect` helped us complete this goal!
        mock_get.reset_mock()  # reset mock, include called count etc.
        mock_get.side_effect = [
            requests.ConnectionError('Test error'),
            mocked_success
        ]
        content = b.post("three-years-later-test-mock")
        self.assertEqual(content, mocked_content)


if __name__ == '__main__':
    unittest.main()
```

关于测试用例如何覆盖，笔者强烈建议阅读k大的[guide](http://docs.python-guide.org/en/latest/writing/tests/)。

## mock模块的实现分析

### MonkeyPatch

这么棒的模块究竟是怎么实现的呢？带着这样的好奇，我们不妨先考虑一下mock模块本身的需求，然后再通过阅读源码来确认实现原理。

mock模块的需求，大致有下面这些：

* 能够模拟模块里的类和方法，得到的mock实例应该与原有模块的方法"动作"一样；
* 能够控制mock实例的返回值、调用情况、异常错误处理等；
* ...

在继续分析mock源码之前，我们先通过一个简单的例子讲解一下monkeypatch这一技巧。不妨假设我们有下面这样一个模块`mymodule`，里面实现了一个`Hello`方法。

```
# mymodule.py
def Hello():
    print "Hello, Jacky!"
```

假设我们现在有N个（N>>100）代码文件里均调用了这一方法，而我们现在有一个需求，需要让Hello的行为发生变化，在不改变上述源码内容的情况下，有什么办法实现吗？答案当然是有啦！

在动态语言里，例如Python，可以通过覆盖模块的方法对象来实现对方法的hack，不妨参考下面的代码：

```
# monkeypatch.py

import sys
import mymodule


# patched method `Hello`
def Hello():
    print "Hello, Monkey!"


# monkey patch
class monkeypatch(object):
    def __init__(self):
        mymod = self._get_global_module("mymodule")

        # Methods to replace
        self.methods = ('Hello', )
        # Store the original methods
        self.orig_methods = dict(
            (m, mymod.__dict__[m]) for m in self.methods)
        # Monkey patch
        g = globals()
        for m in self.methods:
            # mymod's method has been replaced!
            mymod.__dict__[m] = g[m]

    def _get_global_module(self, mod):
        return sys.modules[mod]

    def __enter__(self):
        return self

    def __exit__(self, *args):
        mymod = self._get_global_module("mymodule")
        # restore the method
        for m in self.methods:
            mymod.__dict__[m] = self.orig_methods[m]


if __name__ == '__main__':
    monkeypatch()

    # under this hood, mymodule's Hello has been monkeypatched.
    mymodule.Hello()
```

```
# actual call
$ python monkeypatch.py
Hello, Monkey!
```

我们看到了，最终我们在没有改动源码的情况下，实现了对Hello方法的hack！Success！

上述例子即是一个monkeypatch（猴子补丁）的示范（完整例子可以转到[Github](https://github.com/Colstuwjx/python-examples/tree/master/test/monkey_patch)），而通过实现`__enter__`,`__exit__`方法，monkeypatch的影响范围限制在本身实例方法的作用域（也让它可以用`with...`的方式调用）。

一般来说，monkeypatch适用于无法很好地修改当前调用某个类库的方法，而需要在上层调用时动手脚的场景。（在此，感谢毛毛同学让我第一次知道monkeypatch这个技巧~）

### \_get\_child_mock

如果读者尝试过执行MagicMock方法，会发现这样的效果：
```
In [6]: import mock

In [7]: m = mock.MagicMock()

In [8]: isinstance(m, mock.MagicMock)
Out[8]: True

In [9]: isinstance(m.m, mock.MagicMock)
Out[9]: True

In [10]: isinstance(m.m.m, mock.MagicMock)
Out[10]: True
```

啊咧，怎么会这样？！！
别着急，细想一下，如果我们需要方便的模拟类里面内部甚至嵌套的任何一处调用，黑魔法似的MagicMock似乎正好可以帮助我们实现这样的效果！

那么，它是怎么实现的呢？让我们正式开始阅读[mock源码](https://github.com/testing-cabal/mock/blob/master/mock/mock.py)的旅行吧！

我们不难发现，`MagicMock`类是继承`Mock`和`MagicMixin`。

继续八下去，我们可以看到它有重载`__call__`和`__get__`方法（`__get__`是Python里面元类的一个方法，class.xxx，均是调用此方法获取的，重载它即修改了class.xxx的调用结果），每次都会调用`create_mock `方法，后面又调用了`_get_child_mock`方法：

```
...
    def _get_child_mock(self, **kw):
        """Create the child mocks for attributes and return value.
        By default child mocks will be the same type as the parent.
        Subclasses of Mock may want to override this to customize the way
        child mocks are made.
        For non-callable mocks the callable variant will be used (rather than
        any custom subclass)."""
...
```

根据代码docstring即可知晓，它便是MagicMock类实现上述黑魔法的依仗（Mock类的`__getattr__`同样调用了该方法~）。

### CallableMixin

上述实现恐怕还不能够满足mock的需求，我们还需要实现模拟原模块方法的调用，那么`调用`怎么实现呢？

同样从源码里，我们可以发现`MagicMock`是部分继承自`Mock `的，`Mock`有一个很有意思的父类，叫做`CallableMixin `，它即是模拟调用的核心实现。

我们可以看到它有个[_mock_call](https://github.com/testing-cabal/mock/blob/master/mock/mock.py#L1065)的实现，其内部又是调用[_Call](https://github.com/testing-cabal/mock/blob/master/mock/mock.py#L2091)类来实现链式调用，具体的话，读者们可以细看下该类的完整实现~

### 一些黑科技

mock模块里其他一些利用到的黑科技还包括以下这些：

```
# https://github.com/testing-cabal/mock/blob/master/mock/mock.py#L1203
# _importer 方法实现的即动态根据"class.module"引入相应的原始模块

# https://github.com/testing-cabal/mock/blob/master/mock/mock.py#L1270
# patch是通过猴子补丁实现方法和类的mock，没错，就是前面介绍到的monkeypatch，这也是mock模块的主要实现思想！

# 其他一些代码也比较有趣，奈何功力有限，读的不是很清晰，就不误人子弟了，:)
...
```

总的来说，mock模块通过简单的2k+代码实现了强大完善的模块类和方法的模拟，棒棒哒！

## 结语

鄙人在这篇文章里介绍了mock的概念以及Python的mock模块提供的`MagicMock`、`patch`、`return_value`、`side_effect`等诸多方法，并简单分析了mock模块的源码实现。

测试的领域同样浩瀚无边，如何验证自己的代码在各种各样的业务场景下的健壮性和可维护性，是一门很深的学问。希望这篇文章能够让更多的人了解mock的用法、意义以及它的实现原理。

参考：

* http://docs.python-guide.org/en/latest/writing/tests/

* https://github.com/testing-cabal/mock/blob/master/mock/mock.py


引申阅读：

* [TDD](https://en.wikipedia.org/wiki/Test-driven_development)

* [mock用法](http://www.voidspace.org.uk/python/mock/mock.html)

* 系列介绍mock的好文两篇： [python-mocks-a-gentle-introduction-part-1](http://blog.thedigitalcatonline.com/blog/2016/03/06/python-mocks-a-gentle-introduction-part-1)、[python-mocks-a-gentle-introduction-part-2](http://blog.thedigitalcatonline.com/blog/2016/09/27/python-mocks-a-gentle-introduction-part-2)
