---
title:  “Javascript小结”
mathjax: true
layout: post
date:   2018-01-05 08:00:12 +0800
categories: javascript
---

使用JavaScript也有一段时间了，结果看到一段underscore.string上的代码蒙了。什么是~~？？
{% highlight javascript %}
module.exports = function pad(str, length, padStr, type) {
  str = makeString(str);
  length = ~~length;

  var padlen = 0;

  if (!padStr)
    padStr = ' ';
  else if (padStr.length > 1)
    padStr = padStr.charAt(0);
  ...
}
{% endhighlight %}


# ~~ 和 !!
~对操作数按位取反, 假如有一个变量var a, ~a + a = -1。

~~的意思即是两次取反操作，其实是等于原数本身。好处是什么呢？假如var a是一个小数，
比如a = 3.4, 那么~a = -4， ~~a = 3。本质上等同于一个高效的Math.floor()

!表示逻辑非，!!两次对操作数做逻辑非。好处是什么呢? 可以强制将后面的表达式强制转换
为布尔类型数据，也就是说true或者false。JavaScript约定：
* false, undefined, null, 0, "" 均为false

（附：JavaScript的设计者希望用null表示一个空的值，而undefined表示值未定义。
事实证明，这并没有什么卵用，区分两者的意义不大。大多数情况下，我们都应该用null。
undefined仅仅在判断函数参数是否传递的情况下有用。）
* true， 非0数字，非空字符串，[Object] 均为true

假如定义var a，此时a默认是undefined；var b = !!a，b是一个布尔值false。

# == vs ===
绝大多数场合应该使用 ===, 因为==会自动转换数据类型再比较，很多时候，会得到非常诡异的结果。
===是严格运算符，如果数据类型不一致，返回false，如果一致，再比较。复合数据类型不是比较它们的
值是否相等，而是比较它们是否指向同一个对象。

记住一个特例NaN === NaN 返回false。

# 类和继承
ES6开始支持class和extends语法，在JavaScript中写类和继承变得简单了许多。但是想了解
JavaScript的类本质，还是需要看一下原型继承。

* 首先创建构造函数，用new生成一个对象
{% highlight javascript %}
function Student(name) {
    this.name = name;
    this.hello = function () {
        alert('Hello, ' + this.name + '!');
    }
}
var xiaoming = new Student('小明');
{% endhighlight %}
原型关系如下图：
![image01]({{site.baseurl}}/image/js-prototype.png)

* 让创建的对象共享一个hello函数
{% highlight javascript %}
function Student(name) {
    this.name = name;
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};
{% endhighlight %}
现在原型关系图变成：
![image02]({{site.baseurl}}/image/js-common-hello.png)

* PrimaryStudent继承Student
{% highlight javascript %}
function inherits(Child, Parent) {
    var F = function () {};
    F.prototype = Parent.prototype;
    Child.prototype = new F();
    Child.prototype.constructor = Child;
}

function PrimaryStudent(props) {
    Student.call(this, props);
    this.grade = props.grade || 1;
}

// 实现原型继承链:
inherits(PrimaryStudent, Student);

// 绑定其他方法到PrimaryStudent原型:
PrimaryStudent.prototype.getGrade = function () {
    return this.grade;
};
{% endhighlight %}
继承原型关系图：
![image03]({{site.baseurl}}/image/js-inherits.png)

## this大坑
{% highlight javascript %}
function getAge() {
    var y = new Date().getFullYear();
    return y - this.birth;
}

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: getAge
};

xiaoming.age(); // 25, 正常结果
getAge(); // NaN, this此时指向全局对象window

// 更坑爹的是这样也不行
var fn = xiaoming.age; // 先拿到xiaoming的age函数
fn(); // NaN
{% endhighlight %}

下面换一个写法
{% highlight javascript %}
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        function getAgeFromBirth() {
            var y = new Date().getFullYear();
            return y - this.birth;
        }
        return getAgeFromBirth();
    }
};

// xiaoming.age() 执行了一个内部定义的函数getAgeFromBirth
xiaoming.age(); // Uncaught TypeError: Cannot read property 'birth' of undefined
{% endhighlight %}

在ES6之前常见解决方案有2个：
* 用that变量先捕获this
{% highlight javascript %}
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var that = this; // 在方法内部一开始就捕获this
        function getAgeFromBirth() {
            var y = new Date().getFullYear();
            return y - that.birth; // 用that而不是this
        }
        return getAgeFromBirth();
    }
};

xiaoming.age(); // 25
{% endhighlight %}

* 用apply传入this参数
{% highlight javascript %}
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function() {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
};

xiaoming.age(); // 25

var fn = xiaoming.age();
fn.apply(xiaoming, []); // 25, this指向xiaoming, 参数为空
{% endhighlight %}

参考:  
[廖雪峰的JavaScript教程][lxf-javascript]

[lxf-javascript]:https://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000
