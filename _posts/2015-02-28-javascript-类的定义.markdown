---
layout: post
title:  "javascript-类的定义"
date:   2015-02-28 20:24:19 +0800
categories: 编程
---

一直以来习惯了面向对象语言如`c++`、`python`等，于是理所当然地觉得高级一点的语言应该得有`class`关键字吧，或者对`OO`的态度多少也得亲和一些吧，然后今天学习了一下`javascript`，居然木有`class`关键字！可他有`object`耶！
 
不对，肯定有可以定义类的方法，就算没有`class`关键字。于是google了一下，查到了w3school的一篇教程[ECMAScript 定义类或对象](http://www.w3school.com.cn/js/pro_js_object_defining.asp)，找到了几种创建对象以及模拟类行为的方法。

如果平时想快速得到一个对象的话，就直接定义即可。

1.直接用花括号定义：
{% highlight javascript %}
var o = {name: "...", age: 22, learn: function() { alert("get"); } }
{% endhighlight %}
2.使用`new`关键字，调用构造函数：
{% highlight javascript %}
var o = new Object; // or: var o = new Object();
o.name = '...';
o.age = 22;
o.learn = function() { alert("get"); }
{% endhighlight %}

但这样每次只能生成一个对象，如果想要批量生成对象要怎么办呢？总不能一个个构造吧？

于是，我们会想寻找`class`关键字，有了它就方便了，这也是我一开始的想法，可遗憾的是，__木有__！
所以我们只能通过其它方法来批量生成对象。

那么要找到`javascript`能模拟建造类的方法，进行`OOP`，首先得知道什么是`OOP`要求什么。

`OOP`要求的4种基本能力是：

> 1. 封装
> 2. 聚合
> 3. 继承
> 4. 多态

因为`javascript`是能满足这4种要求的，我们可以找到适当的方法。

接下来记录一下我今天学到的这5种方法，这5种方法都来自于前文所提到的w3school教程。

----

#### 1.__工厂函数__（直接定义对象）

容易想到的是工厂模式，比较简单直接po代码：

{% highlight javascript %}
var createObject = function() {
    var o = {}; // 每次调用工厂函数都会生成一个新的内部对象
    o.name = '...'; // 并做相应赋值
    o.age = 22;
    o.learn = function() {
        alert("get");
    }
    return o; // 最终返回这个内部对象的引用
}

var a = createObject(); // a 得到了内部对象的引用
var b = createObject(); // b 得到了另外一个引用，a与b都是独立的
{% endhighlight %}

__优点__：
对象独立，对`a`的修改不会导致`b`也被改动。

__缺点__：
因为每个对象的某个方法功能是一样的（如例子中的`learn`），事实上只需要同一个函数（同一个引用），所以工厂函数法的明显缺陷就是，`function learn() { ... }` 会执行多次，导致重复定义方法；其次，没有适当的方法来区分这个对象的类别；

其实缺点中的函数重复定义也可以避免：

{% highlight javascript %}
function _learn() {
    alert("get");
}

var createObject = function() {
    var o = {}; // 每次调用工厂函数都会生成一个新的内部对象
    o.name = '...'; // 并做相应赋值
    o.age = 22;
    o.learn = _learn; // 引用外边的，这样就函数只需要定义一次咯
    // o.learn = function() {
    //     alert("get");
    // }
    return o; // 最终返回这个内部对象的引用
}
{% endhighlight %}

__缺点__：

不够美观，代码上不够“聚集”，分散。（←_←）话是这么说，但`golang`貌似也这样，好不好见仁见智吧～

----

#### 2.__构造函数__

`构造函数 constructor`与`new`的出现，是我目前为止觉得比较突兀的一件事。可能是受`c++`的影响吧，我至今还觉得有点不习惯。在`c++`中，函数就干函数的事情，没绑定的就是普通函数，绑定的就变成类方法，该初始化就初始化，该析构就析构，这个在`python`里也是一样的，然后`javascript`里，哦，居然还能生成对象？！这不是类干的事情吗？！好吧，看来是因为没有类，所以就只能靠函数了，感觉真的不够美。

而且还跟`new`结合，有`new`就起构造函数作用，能产生对象，没`new`就是普通函数职能。当然，我想这仅仅是理解层面上的区分，如果偏偏直接以普通函数来调用构造函数呢？其实也可以。举个例子：

{% highlight javascript %}
function Constructor() { // 本意是构造函数
    this.x = 1; // this指的是调用该构造函数的对象
}

Constructor(); // 由于当前在浏览器控制台下，所以里边的this指的是global对象，即window对象

alert(x); // 这时window对象就多了一个x属性了，所以调用构造函数时，务必记得new关键字
{% endhighlight %}

所以有使用`new`关键字的话，就会先创建一个对象，然后再执行构造函数的内容来为新对象赋值属性与方法，最终返回这个构造完的新对象。

我没有去深究`new`究竟做了神马，但在这里觉得，好像只是 __语法糖__？
又或者说，直接构造法`var o = { ... }` 中的`{ }`其实是 __语法糖__ ？

好吧这个以后再研究哈，继续话题～

构造函数法的代码如下：

{% highlight javascript %}
function People() {
    this.name = '...';
    this.age = 22;
    this.learn = function() { // 没错，这里还是会重复定义方法
        alert("get");
    }
}

var o = new People;
{% endhighlight %}

通过构造函数的方法，结果其实也跟工厂函数法一样啊，无非就是利用了`new`关键字而已。
所以 __优缺点__ 同工厂函数法。

不过一定要说 __优点__ 的话其实还是有的：

1.构造函数中，用了`this`关键字看起来比较`OO`；
2.能结合`new`关键字，看起来也比较`OO`，例如`var o = new People();`（当然前提是别告诉我`People`这货是函数= =！）
3.可以用`instanceof`来区分类别，跟`python`中的`isinstance(obj, cls)`一样，写起来更`OO`；

----

#### 3.__原型__

{% highlight javascript %}
function People() { // 先定义一个空的构造函数
}
// 为构造函数设置原型
People.prototype.name = '...';
People.prototype.age = 22;
People.prototype.learn = function() {
    alert("get");
}

var o = new People;
{% endhighlight %}

原型是我一直觉得头大的一个概念，以前真心没接触过，目前我的理解可能也比较单薄，我现在的想法是，原型也是达到`OO`继承的一种途径。

在`python`中，我们有`class`来描述定义`object`的属性与方法，当然也有`metaclass`来描述`class`，类是处在更高一层来定义对象的，并且这些`class`的继承行为可以构成一棵树，然后，继承类的对象就是在这棵树上边查找属性与方法，在`python`中的查找规则是宽度优先搜索（老版本是深度优先）。

在`javascript`中，我们没有`class`，但有`prototype`，原型并没有在更高一层定义对象。生动点说，类是上帝视角，而原型是平凡人。上帝描述了元信息，然后造人；而原型可以想象成第一个人（对象），然后充当模板，可以被其他对象套用。原型可以构成原型链，对象的属性方法沿着原型链进行查找，这与`python`的类继承树有异曲同工之妙。

并且，要能实现真正的继承，那么子类的对象的属性必须是独立的，`javascript`与`python`也具有一样的策略，如果只是读取属性的话，那么便沿着 原型链/继承树 查找，然后相应地执行；一旦赋值属性，就在当前的对象中赋值出独立的属性，而不是直接修改原型的属性。从而确保所有对象都是独立的，不会相互干扰。 __但是__ ！在`javascript`中，如果是属性是对象，例如是数组`[1, 2, 3, ]`，那么在继承的对象中对这个属性进行push的话，由于引用的关系，导致原型中的属性也将被修改，因为所有对象引用同一个实体。

用刚才的例子，改为：

{% highlight javascript %}
function People() { // 先定义一个空的构造函数
}
// 为构造函数设置原型
People.prototype.name = '...'; // str 是不可变的，所以可以放在原型中
People.prototype.age = 22; // number 也是不可变的
People.prototype.learn = function() { // 函数是对象，如果没有利用到它的可变性质的话，可以认为不变，所以可以放在原型中
    alert("get");
}
People.prototype.hobby = ["study", ]; // Array对象是可变的，会出问题！

var a = new People;
var b = new People;

a.hobby.push("football"); // ["study", "football", ]
b.hobby.push("eat"); // ["study", "football", "eat", ]
{% endhighlight %}

那么，这说明了什么问题？是`javascript`不好吗？不，其实这是变量引用导致的，`python`中也有类似的问题，只是出现在别的情况（例如默认参数为对象的问题）。

所以，这意味着，原型的 __缺点__ 便是：如果属性是可变对象的话，引用机制不可避免将导致以上属性“不独立”的问题；而这更多是因为错用了原型！

对于可以共享引用的不可变属性或者方法，可以放进原型中；而对于不可共享引用的属性或方法，应当放在构造函数中定义。（这其实就是第4种方法， __混合构造函数与原型__ ）
或者，在构造函数中记得覆盖原型中的属性。（这在继承中是有必要的，因为通过设置原型的方式来继承，完全可能面临这个问题）

举个例子：

{% highlight javascript %}
function People() {
    this.hobby = ["study", ]; // 不可共享引用的属性放进了构造函数中
}
People.prototype.name = "..."; // 可共享引用的属性放进了原型中
// ...继续其他属性

var o = new People; // 到这一步完全没有问题，ok！

// 如果 o 有幸被当成原型呢？

function SuperMan() {
}
SuperMan.prototype = o;
// 那么…
var t1 = new SuperMan;
var t2 = new SuperMan;
t1.hobby.push("save people"); // 到这一步，t1.hobby = ["study", "save people", ]
// ...
t2.hobby.push("stop war"); // 现在，t2.hobby = ["study", "save people", "stop war", ]
{% endhighlight %}

可见，尽管对象`o`工作得很好，但是当直接被当成原型的时候，就可能导致问题。所以此时应该在`SuperMan`的构造函数中，定义`hobby`，从而进行覆盖。

关于原型的理解先写到这儿，接下来的学习中，如果有新的想法，我会再更新或者重新写一篇。
这里推荐 __BYVoid__ 的[JavaScript對象與原型](https://www.byvoid.com/blog/javascript-object-prototype)，里边的原型链图有助于理解，我就不盗图咯。

----

#### 4.__混合构造函数与原型__

由于在上边的第三种已经提到，这里只放简单代码：
{% highlight javascript %}
function People(name, age) { // 1. 能带参数，发挥构造函数优势
    this.name = name;
    this.age = age;
    this.hobby = []; // 2. 可以确保每个对象都有独立的不可共享的引用实体属性
}
People.prototype.learn = function() { // 3. 利用了原型的优势，函数只需要定义1次
    alert("get");
}

var o = new People;
{% endhighlight %}

__优点__：
能结合构造函数与原型的优点；

__缺点__：
代码还是有点分散哦！强迫症受不鸟= =...

----

#### 5.__动态原型__（方法4的美化版，__推荐__）

木错！这个方法纯粹是为了解除强迫症患者的苦痛！

{% highlight javascript %}
function People(name, age) { // 1. 能带参数，发挥构造函数优势
    this.name = name;
    this.age = age;
    this.hobby = []; // 2. 可以确保每个对象都有独立的不可共享的引用实体属性
    // 把外边的方法定义移到里边，并且通过设置flag，只会定义一次哦！
    if(!People._initialized) {
        People.prototype.learn = function() { // 3. 利用了原型的优势，函数只需要定义1次
            alert("get");
        }
    People._initialized = true;
    }
}
// People.prototype.learn = function() { // 3. 利用了原型的优势，函数只需要定义1次
//     alert("get");
// }

var o = new People;
{% endhighlight %}

----

其实还有第6种方法，称为 _混合工厂_ （混合了工厂函数法与构造函数法），因为我看来看去看不出什么特殊作用，先不写啦～（感兴趣的可以回最上边的那个链接里看哦）


