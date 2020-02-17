---
title: "Python开发之几个内联函数的介绍"
date: 2014-07-28T20:00:00+08:00
tags: ["devops", "python"]
categories: ["python"]
comments: false
showMeta: false
showAction: false
---

Python本身有很多内置的函数供开发人员使用，其中有几个感觉挺有学习价值，在这里记录一下。

<!--more-->

* lambda
* zip
* filter
* map
* reduce

#### lambda

Python允许单行快速定义一个最小函数，这便是lambda函数。比如，原本使用常规的函数去定义，它可能是如下形式：

	def add(x,y):
    	return x+y

而使用lambda函数的语法形式，它可以轻松简化为单行代码：

	add_by_lambda = lambda x,y: x+y
    
其中，x,y便是函数入口参数，而后面则是具体的函数体，默认lambda可以返回一个函数对象，相应的调用效果类似：

	print add(1, 1)
    print add_by_lambda(1, 1)

甚至还可以直接在后面追加实参来直接获取返回值，比如`lambda x,y : x+y, 1, 1`返回结果就是2

lambda函数使得Python代码可以变得更加优雅、简洁。

#### zip

Python内置的zip函数接受任意多个list作为参数，并把相同索引的元素组合成tuple，最后形成一个新的list，新的list长度以传入参数的最小值为准。比如：

```
x = [ 1, 2, 3]
y = [ 'a', 'b', 'aa', 'bb']
z = [ 'A', 'B', 'C']

# zip them!
print zip(x,y,z)

# (*) 是 zip的反函数
print zip(*zip(x,y,z))
print zip(*([(1, 'a', 'aa'), (2, 'b', 'bb'), (3, 'c', 'cc')]))
```

#### filter

filter函数接受两个参数，func和list，而经过过滤后返回一个list，其中func函数对象只能有一个传入参数。原理便是根据列表list中所有元素作为参数传递给函数func，返回可以令func返回真的元素的列表，如果func为None，那么会使用默认的Python内置的identity函数直接判断元素的True or False。

例如：
```
a = [1,2,3,4,5,6,7]
b=filter(lambda x:x>2, a)
print b

#过滤奇数集
a = [1,2,3,4,5,6,7]
b=filter(lambda x:x%2, a)
print b
```

#### map

map函数是一个很强大的一个映射函数，其传入两个参数，一个是func，一个是list，而功效便是func作用于给定序列的每个元素，并用一个列表来提供返回值。例如：

```
a=[0,1,2,3,4,5,6,7]
map(lambda x:x+3, a)

a=[1,2,3]
b=[4,5,6]
map(lambda x,y:x+y, a,b)
```

```
#my_map函数实现
def my_map(func, *args):
	   return [ func(arg) for arg in args ]
```

上面的代码假使通过for循环方式实现的话，不仅代码行数变多，而且可能多层if判断，结构冗余，完全没有map方式来得简洁、优雅。

#### reduce

reduce函数传入参数为func和list，其遍历list元素，并调用func函数实现累积，具体效果便是：

`reduce(f, [x1, x2, x3, x4]) = f ( f ( f ( x1,  x2 ),  x3 ), x4 ) `

使用范例如下：
```
#str to int
def str2int(s):
       return reduce(lambda x,y: x*10+y, map(int, s))
```

#### 一些妙用

以上简单介绍了五个强大简洁的Python内置函数的用法，下面附上一些小巧精妙的例子，以飨读者。

```
#两个list，取(x - y) + (y - x)
x=[{'a': 1, 'b': 2}, {'c': 3}, {'d': 4}]
y=[{'a': 1}, {'c': 3}, {'e': 5}]

filter(lambda z: (x+y).count(z)<2, (x+y))
```

```
#flatten out nested sublist
#result: [ 1, 2, 3, 4, 5 ]
import operator
reduce( operator.concat, [ [ 1, 2 ], [ 3, 4 ], [ ], [ 5 ] ], [ ] ) 
```

```
#多项式求和
import operator
def evaluate (a, x):
       xi = map( lambda i: x**i, range( 0, len(a)))
	   axi = map(operator.mul, a, xi)
       return reduce( operator.add, axi, 0 )
```

```
#数据库SQL
reduce( max, map( Camera.pixels, filter( 
        lambda c: c.brand() == "Nikon", cameras ) ) )

#maybe equals
SELECT max(pixels) 
FROM cameras
WHERE brand = “Nikon”

#There.
#cameras is a sequence
#where clause is a filter
#pixels is a map
#max is a reduce
```

```
#一行并发
import urllib2
from multiprocessing.dummy import Pool as ThreadPool

urls = [
           'http://www.python.org',
           'http://www.google.com',
           'http://www.baidu.com',
           'http://www.python.org/community/',
           'http://www.saltstack.com'
           ]

#pool = ThreadPool()
pool = ThreadPool(4) # Sets the pool size to 4

result = pool.map(urllib2.urlopen, urls)
pool.close()
pool.join()
```

上述的代码实现相较普通的for循环等实现方式是不是特别的简洁、优雅呢? 尝试着使用它们吧!

========================================================

参考： [MIT Lecture](http://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-005-elements-of-software-construction-fall-2011/lecture-notes/MIT6_005F11_lec15.pdf) 、[Python 的一行并发](http://www.oschina.net/translate/python-parallelism-in-one-line)

========================================================
