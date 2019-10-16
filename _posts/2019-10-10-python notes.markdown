---
title:  “廖雪峰python笔记”
date:   2019-10-10 08:00:12 +0800
categories: deeplearning.ai
---

一直在用Tensorflow/Jupyter写机器学习相关例子，但是大部分都是比较简单的代码，基本不超过200行。最近要开始碰一个Python项目，连语法都快要忘光了。翻出多年前看过廖老师的Python教程再过一遍吧，顺便把一些常用语法和细节记录一下。


# Python基础

- 格式化字符串里面有%是一个普通字符，用%%转义

    ```python
    print('growth rate: %d%%' % 7)
    # 'growth rate: 7%'
    ```

    或者用format方法就不用转义
    ```python
    print('Hello, {0}成绩提升了{1:.1f}%'.format('小明', 17.125))
    # 'Hello, 小明成绩提升了17.1%'
    ```

- 三元表达式: `value = 1 if condition else 0`

- `(1,)`表示只有一个元素的`tuple`

- `dict`的key必须是不可变对象，因此`list`不能作为key；`set`也不可放入可变对象

# 函数

- 可以用内置函数`isinstance()`检查参数类型

- **默认参数必须指向不变对象!**

    `def add_end(L=[])`这样的函数定义会出问题

- 可变参数在前面加一个`*`

    定义：`def calc(*number)`，在函数内部，参数`numbers`接受到的是一个tuple

    调用：如果想把list或者tuple的元素变成可变参数传入，也是在前面加一个`*`号
    ```python
    nums = [1, 2, 3]
    calc(*nums)
    ```

- 关键字参数在前面加一个`**`

    定义：`def person(name, age, **kw)`，在函数内部，参数`kw`接受到的是一份dict的拷贝

    调用：如果想把dict的key-value对转换为关键字参数传入，在前面加一个`**`号
    ```python
    extra = {'city': 'Beijing', 'job': 'Engineer'}
    person('Jack', 24, **extra)
    ```

- 命名关键字参数

    限制关键字参数的名字，放在特殊分隔符`*`之后，例如`def person(name, age, *, city='Beijing', job)`；或者跟在可变参数之后，例如`def person(name, age, *args, city, job)`。

    **参数定义顺序是：必选参数，默认参数，可变参数，命名关键参数，关键字参数**。尽量别用太多的组合，否则函数接口可读性很差。

- 尾递归调用(标准Python解释器并没有针对尾递归做优化)

    递归调用的次数过多，会导致栈溢出。尾递归是指，函数返回的时候，调用函数本身，return语句不能包含表达式；这样无论调用多少次，解释器优化后都只占用一个栈帧。

# 高级特性

- 切片

    除了常见的用法`lt[1:3]`，还可以这样调用`lt[slice(1,3)]`，`slice(1,3)`是一个slice对象。

- 迭代

    `for`循环作用于`Iterable`对象，包括`list`，`tuple`，`dict`，`set`，`str`和`generator`。

    迭代list, tuple或者set:
    ```python
    for x in list(range(10)):
        print(x)
    
    # 内置的enumerate函数把一个list变成索引-元素对
    for i, value in enumerate(['A', 'B', 'C']):
        print(i, value)
    ```

    dict默认迭代key:
    ```python
    for key in d:
        print(key)

    for value in d.values():
        print(value)

    for k, v in d.items():
        print(k, v)
    ```

    `next()`函数只作用于`Iterator`对象，`generator`是`Iterator`对象。`Iterator`的计算是惰性的，只有在需要返回下一个数据时它才会计算。通过函数`iter()`可以把`list`，`dict`，`str`等变成`Iterator`对象。

- 列表生成式
    
    例子：
    ```python
    # 偶数平方
    [x * x for x in range(1, 11) if x % 2 == 0]

    # 双层循环
    [m + n for m in 'ABC' for n in 'XYZ']

    # os.listdir列出文件和目录
    [d for d in os.listdir('.')]
    ```

- 生成器`generator`

    ```python
    g = (x * x for x in range(10))
    for n in g:
        print(g)
    ```

    第二种方式，如果一个函数定义中包含`yield`关键字，它就不再是普通函数，而是一个`generator`。每次调用`next()`的时候执行，遇到`yield`返回，再次执行时从上次返回的`yield`语句处继续执行。
    ```python
    def fib(max):
        n, a, b = 0, 0 ,1
        while n < max:
            yield b
            a, b = b, a + b
            n = n + 1
        # 如果想要拿到返回值，必须通过调用next()，捕获StopIteration错误
        return 'done'

    f = fib(6)
    
    while True:
        try:
            x = next(f)
            ...
        except StopIteration as e:
            ...
    ```

# 函数式编程

- 内建高阶函数

    `map`接受两个参数，一个是函数，一个是`Iterable`，返回一个`Iterator`:
    ```python
    list(map(lambda x: x * x, [1, 2, 3, 4, 5]))
    ```

    `reduce`做累积计算，要求函数`f`必须接受两个参数:
    ```python
    reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)
    ```

    `filter`保留传入函数返回值是`True`的元素，返回也是一个`Iterator`:
    ```python
    def is_odd(n):
        return n % 2 == 1
    
    list(filter(is_odd, [1, 2, 4, 5, 6, 9, 10, 15]))
    ```

    `sorted`函数对`Iterable`对象排序，返回是一个新的`list`；参数`key`实现自定义排序，参数`reverse`控制升序还是降序:
    ```python
    sorted(['bob', 'about', 'Zoo', 'Credit'], key=str.lower, reverse=True)
    ```

    对`list`去重的一个简单方法是`list(set(lt))`，但是这样会改变原来的顺序；解决方案是通过index排序。
    ```python
    lt = [1, 2, 4, 9, 5, 6, 5, 2, 4, 7, 8, 0]

    # [0, 1, 2, 4, 5, 6, 7, 8, 9]
    list(set(lt))

    # [1, 2, 4, 9, 5, 6, 7, 8, 0]
    sorted(set(lt), key = lt.index)
    ```

- 闭包

    如果外部函数中定义了内部函数，并且内部函数引用了外部函数的参数和局部变量，当外部函数返回内部函数时，相关参数和变量都保存在返回的函数中，这种程序结构称为"闭包(Closure)"。每次调用外部函数返回都是一个新的函数。

    **闭包需要牢记的一点：返回的内部函数不要引用任何外部函数的循环变量，或者后续会发生变化的变量，因为返回的函数并没有立刻执行。**
    ```python
    def count():
        fs = []
        for i in range(1, 4):
            def f():
                return i * i
            fs.append(f)
        return fs


    def count():
        # 如果一定要引用循环变量，方法是再创建一个函数，用该函数的参数绑定循环变量当前的值。无论该循环变量后续如何更改，已绑定到函数参数的值不变
        def f(j):
            def g():
                return j*j
            return g
        fs = []
        for i in range(1, 4):
            fs.append(f(i)) # f(i)立刻被执行，因此i的当前值被传入f()
        return fs
    ```

- 装饰器

    用我自己的话解释，装饰器就是用闭包实现切片编程(AOP)的效果。例子:
    ```python
    def log(text):
        def decorator(func):
            @functools.wraps(func)
            def wrapper(*args, **kw):
                print('%s %s():' % (text, func.__name__))
                return func(*args, **kw)
            return wrapper
        return decorator

    # 注解等于于调用now = log('execute')(now)
    @log('execute')
    def now():
        print('2015-3-25')
    ```

# 模块

- 命令行运行模块文件时，Python解释器把一个特殊变量__name__置为__main__，而如果在其他地方导入该模块时，if判断将失败，因此，这种if测试可以让一个模块通过命令行运行时执行一些额外的代码，最常见的就是运行测试。

    标准模块文件格式：
    ```python
    #!/usr/bin/env python3
    # -*- coding: utf-8 -*-

    ' a test module '

    __author__ = 'Michael Liao'

    import sys

    def test():
        args = sys.argv
        ...

    if __name__=='__main__':
        test()
    ```

- 正常的函数和变量名是公开的，如`abc`, `PI`；类似__xxx__这样的变量是特殊变量，可以被直接引用，但是有特殊用途，如`__author__`, `__name__`；类似_xxx和__xxx这样的函数或变量就是非公开的，如`_abc`, `__abc`。

# 面向对象编程

- 如果没有合适的继承类，就使用`object`，这是所有类最终都会继承的类

- 访问限制

    实例的变量名如果以__开头，就变成了一个私有变量（private），只有内部可以访问，外部不能访问。但其实Python解释器对外把__name变量改成了_类名__name。Python本身没有任何机制阻止你干坏事，一切全靠自觉。

    对于以一个下划线开头的实例变量名，比如_name，这样的实例变量外部是可以访问的，但是，按照约定俗成的规定，当你看到这样的变量时，意思就是，“虽然我可以被访问，但是，请把我视为私有变量，不要随意访问"。

- 鸭子类型

    对于静态语言来说，如果需要传入`Animal`类型，则传入的对象必须是`Animal`类型或者它的子类，否则将无法调用其中的`run()`方法。对于Python这样的动态语言来说，则不一定需要传入`Animal`类型。我们只需要保证传入的对象有一个`run()`方法就可以了。
    ```python
    def run_twice(animal):
        animal.run()
        animal.run()

    class Timer(object):
        def run(self):
            print('Start...')

    # OK
    run_twice(Timer())
    ```

- 对象信息

    * `type()`返回对象类型，基本数据类型有`int`，`str`，`list`等等，其他包括types模块中定义的常量`types.FunctionType`，`types.BuiltinFunctionType`，`types.LambdaType`，`types.GeneratorType`等等。

    * `isinstance`可以判断一个变量是否是某些类型中的一种，如`isinstance([1, 2, 3], (list, tuple))`。

    * `dir()`返回一个对象的所有属性和方法；`getattr()`、`setattr()`以及`hasattr()`，可以直接操作一个对象的状态。

# 面向对象高级编程

- __slots__

    Python允许在程序运行的过程中动态给类或者实例绑定方法。如果想要限制实例允许添加的属性，则需要在类里定义一个特殊的__slots__变量。__slots__仅对当前类的实例起作用，对继承的子类是不起作用的。
    ```python
    class Student(object):
        __slots__ = ('name', 'age') # 用tuple定义允许绑定的属性名称
    ```

- @property装饰器

    内置的@property装饰器负责把一个方法变成属性调用。
    ```python
    class Student(object):
        @property
        def score(self):
            return self._score

        @score.setter
        def score(self, value):
            if not isinstance(value, int):
                raise ValueError('score must be an integer!')
            if value < 0 or value > 100:
                raise ValueError('score must between 0 ~ 100!')
            self._score = value

    s = Student()
    s.score = 60 # OK，实际转化为s.set_score(60)
    s.score # OK，实际转化为s.get_score()
    ```

- MixIn

    设计继承关系时，通常主线都是单一继承下来。但是如果需要”混入“额外的功能，则通过多重继承实现，这种设计通常称之为MixIn。例如：
    ```python
    class Dog(Mammal, RunnableMixIn, CarnivorousMixIn):
        pass
    ```

- 定制类

    __len__() : 让class作用于`len()`函数
    __str__() : 返回用户看到的字符串
    __repr__() : 返回程序开发者看到的字符串
    __iter__() : 返回一个迭代对象，class可以作用于`for ... in`循环
    __next__() : 循环不断调用该方法拿到下一个值，直到遇到StopIteration错误时退出
    __getitem__() : 按照下标取出元素，传入的参数可能是一个int，也可能是一个切片对象`slice`
    __getattr__() : 在没有找到属性的情况下，会在`__getattr__`中查找
    __call__() : 直接对实例进行调用；通过`callable()`函数，可以判断一个对象是否是”可调用“对象

- 枚举类

    ```python
    from enum import Enum

    Month = Enum('Month', ('Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'))
    ```

    上面定义了一个`Month`类型的枚举类，可以直接使用`Month.Jan`引用一个常量，默认从1开始计数。也可以从`Enum`派生自定义类：
    ```python
    from enum import Enum, unique

    @unique #检查保证没有重复值
    class Weekday(Enum):
        Sun = 0 # Sun的value被设定为0
        Mon = 1
        Tue = 2
        Wed = 3
        Thu = 4
        Fri = 5
        Sat = 6

    for name, member in Weekday.__members__.items():
        print(name, '=>', member)
    ```

- 元类

    * 动态创建类

        `type()`函数可以查看一个类型或变量的类型，当传入一个类比如`Student`，它的类型就是`type`。同时`type()`函数也可以创建出新的类型，三个参数依次为：
        1. class名字
        2. 继承父类tuple
        3. class的方法名称与函数绑定

        ```python
        def fn(self, name='world'): # 先定义函数
            print('Hello, %s.' % name)

        Hello = type('Hello', (object,), dict(hello=fn)) # 动态创建Hello类
        ```

    * metaclass

        在Python中，类同时也是对象；当遇到关键词class的时候，Python就会自动执行产生一个对象。metaclass就是用来创造“类对象”的类.它是“类对象”的“类”，函数type就是python内置的特殊metaclass。

        ![image01]({{site.baseurl}}/image/20191010/python_metaclass.png)

        当你写下`class Foo(Bar)`的时候，类对象Foo还没有在内存中生成。Python会在类Foo和父类中中寻找`__metaclass__`。如果找到了，Python就会使用这个`__metaclass__`来创造类对象Foo。如果没找到，如果python由下往上遍历父类也都没有找不到__metaclass__，它就会在模块(module)中去寻找`__metaclass__`；如果还没有找到，Python就使用`type`来创造Foo。

        自定义metaclass例子：
        ```python
        # 自定义metaclass继承type类
        class ModelMetaclass(type):
            # __new__()方法接收到的参数依次是：当前准备创建的类的对象，类的名字，类继承的父类集合，类的方法集合。
            def __new__(cls, name, bases, attrs):
                ...
                return type.__new__(cls, name, bases, attrs)

        class Model(dict, metaclass=ModelMetaclass):
            def __init__(self, **kw):
                super(Model, self).__init__(**kw)

        ```
        
        使用metaclass的主要目的，是为了能够在创建类的时候，自动地修改类。99%的情况可能都用不到，最常见的场景是写ORM框架。

# 测试

- 单元测试

    针对单元测试，Python自带`unittest`模块。继承`unittest.TestCase`，以test开头的方法就是测试方法。运行测试：`python -m unittest dict_test`。

    ```python
    # dict_test.py
    import unittest

    class TestXXX(unittest.TestCase):

        # 每一个测试方法前调用
        def setUp(self):
            print('setUp...')

        # 每一个测试方法后调用
        def tearDown(self):
            print('tearDown...')

        def test_attr(self):
            d = {'key' : 'value'}
            self.assertTrue('key' in d)
            self.assertEqual(d['key'], 'value')

        def test_keyerror(self):
            # 期待抛出指定类型的Error
            with self.assertRaises(KeyError):
                value = d['empty']
    ```

- 文档测试

    Python内置的“文档测试”（doctest）模块可以直接提取注释中的代码并执行测试。doctest严格按照Python交互式命令行的输入和输出来判断测试结果是否正确。只有测试异常的时候，可以用`...`表示中间一大段烦人的输出。

    ```python
    # mydict.py
    class Dict(dict):
        '''
        Simple dict but also support access as x.y style.

        >>> d1 = Dict()
        >>> d1['x'] = 100
        >>> d1.x
        100
        >>> d1.y = 200
        >>> d1['y']
        200
        >>> d2 = Dict(a=1, b=2, c='3')
        >>> d2.c
        '3'
        >>> d2['empty']
        Traceback (most recent call last):
            ...
        KeyError: 'empty'
        >>> d2.empty
        Traceback (most recent call last):
            ...
        AttributeError: 'Dict' object has no attribute 'empty'
        '''
        def __init__(self, **kw):
            super(Dict, self).__init__(**kw)
    ```

# I/O

- 文件读写

    Python引入了with语句来自动帮我们调用close()方法。

    ```python
    with open('/path/to/file', 'r', encoding='utf8', errors='ignore') as f:
        for line in f.readlines(): # 一次读取所有内容并按行返回list
            print(line.strip()) # 把末尾的'\n'删掉

    with open('/Users/michael/test.txt', 'w') as f:
        f.write('Hello, world!')
    ```

- 文件和目录

    `os.path.abspath('.')` - 查看当前目录绝对路径
    `os.path.join('/User/shizy', 'testdir')` - 合成路径
    `os.path.split('/Users/shizy/testdir/file.txt')` - 拆分路径，后一部分总是最后级别的目录或文件名
    `os.path.splitext('/path/to/file.txt')` - 后一部分是文件扩展名`.txt`
    `os.mkdir('/User/shizy/testdir')` - 创建目录
    `os.rmdir('/User/shizy/testdir')` - 删除目录
    `os.rename('test.txt', 'test.py')` - 文件重命名
    `os.remove('test.py')` - 删掉文件
    `os.listdir('.')` - 目录和文件列表
    `os.path.isdir('/User/shizy/testdir')` - 判断是否是目录
    `os.path.isfile('/User/shizy/testdir')` - 判断是否是文件

# 异步I/O

- 协程

    协程(Coroutine)在一个线程中执行，但是执行子程序A的过程中，可以中断去去执行子程序B，适当的时候再回来执行A。协程不是函数调用，也不存在线程切换。Python对协程的支持是通过generator实现的。

- asyncio

    用下面`hello world`的例子来说明, `@asyncio.coroutine`把一个generator标记为coroutine类型。`hello()`会首先打印出`Hello world!`，然后，`yield from`语法可以让我们方便地调用另一个`generator`。由于`asyncio.sleep()`也是一个`coroutine`，所以线程不会等待`asyncio.sleep()`，而是直接中断并执行下一个消息循环。当`asyncio.sleep()`返回时，线程就可以从`yield from`拿到返回值（此处是None），然后接着执行下一行语句。实际场景会把`asyncio.sleep()`换成真正的IO操作，例如`asyncio.open_connection`，`reader.readline()`等。

    ```python
    import asyncio

    @asyncio.coroutine
    def hello():
        print("Hello world!")
        # 异步调用asyncio.sleep(1):
        r = yield from asyncio.sleep(1)
        print("Hello again!")

    # 获取EventLoop:
    loop = asyncio.get_event_loop()

    # 执行coroutine
    # loop.run_until_complete(hello())

    # 用task封装coroutine
    tasks = [hello(), hello()]
    loop.run_until_complete(asyncio.wait(tasks))
    loop.close()
    ```

    Python3.5开始引入新的语法`async/await`，让`coroutine`的代码更简洁易读。

    ```python
    async def hello():
        print("Hello world!")
        r = await asyncio.sleep(1)
        print("Hello again!")
    ```
