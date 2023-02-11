---
title: UML 类图
date: 2022-06-07
comment: false
---

UML  的各种线条属实让人有点迷糊。

<!--more-->



![](https://s2.loli.net/2022/06/07/VAIjKL9sTeqMSwZ.png)

简单说明：

最上边的*动物*矩形框：

- 第一行表示类名，如果是抽象类，则用斜体显示
- 第二行表示类的属性
- 第三行表示类的方法

> ‘+’表示public，‘-’表示private，‘#’表示protected。



接口的两种表示方法：

1. 左下角的*飞翔* 矩形框：
   1. 矩形表示法
2. 最下方的 *唐老鸭* 表示的也是接口：
   1. 棒棒糖表示法



鸟与翅膀：

- 合成（Composition，也有翻译成‘组合’的）是一种强的‘拥有’关系，体现了严格的部分和整体的关系，部分和整体的生命周期一样



大雁与雁群：

- 聚合表示一种弱的‘拥有’关系，体现的是A对象可以包含B对象，但B对象不是A对象的一部分



剩下的图中都有很详细的说明，内容来自《大话设计模式》这本书，作为设计模式入门教材还是很不错的。



无奈的就是自己没有详细接触过一门面向对象编程语言，只能看懂大致的语法，没办法从面向对象的角度出发进行思考。



包括到现在，自己写的代码也都是面向过程的，一个函数套另一个函数，更多的像是函数式编程。



Go 中没有**类**的概念，关于结构体也是仁者见仁智者见智，我们可以把它类比成类，为结构体增加的方法类比成类中的方法。



将类的继承类比成接口的组合。



虽然实际接触过，但也了解到继承这个特性经常被人诟病。父类繁多、难以维护、阅读。相比之下，大家更推崇基于组合的方式，这正是 Go 中的实现方式。



自己对这块也是不太了解，只能是看着 JAVA 的代码想着 Go 的实现。。