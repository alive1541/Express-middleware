---
title: Express中间件原理详解
---
## 前言
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Express和Koa是目前最主流的基于node的web开发框架，他们的开发者是同一班人马。貌似现在Koa更加流行，但是仍然有大量的项目在使用Express，所以我想通过这篇文章说说Express中间件的原理。
## 中间件的功能和分类
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;中间件的本质就是一个函数，在收到请求和返回相应的过程中做一些我们想做的事情。Express文档中对它的作用是这么描述的：
> 执行任何代码。</br>
> 修改请求和响应对象。</br>
> 终结请求-响应循环。</br>
> 调用堆栈中的下一个中间件。</br>
#### 分类
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Express文档中把他们分为了五类，但是他们的原理相同，只是用法不同：
> 应用级中间件</br>
> 路由级中间件</br>
> 错误处理中间件</br>
> 内置中间件</br>
> 第三方中间件</br>

## 中间件的原理
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先我们看看中间件的用法：
``````
var express = require('express')
var app = express();
app.use('/user', function (req, res, next) {
  //TODO
  next();
});
app.listen(8080)

``````
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接下来我们对比看一下下源码：

![](https://user-gold-cdn.xitu.io/2018/3/10/1620f9dd0da58ed1?w=2068&h=1028&f=jpeg&s=297602)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;与中间件有关的有三部分：</br>
* express.js继承application.js并对外暴露接口
* application.js挂载了所有核心方法
* router文件夹处理路由逻辑

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;先看express.js的代码：

![](https://user-gold-cdn.xitu.io/2018/3/10/1620e095f05361e2?w=1420&h=1764&f=jpeg&s=261545)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这部分代码中最重要的是红色方框部分，<code>mixin</code>是一个第三方库。可以简单理解为继承(实际上它不是继承而是混合)。</br>
<!--&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实际上，express.js模块做了两件事，一是暴露对外接口；二是继承<code>application.js</code>导出的类(实际上它只是个对象)。。</br>-->
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;接下来我们看application.js：

![](https://user-gold-cdn.xitu.io/2018/3/10/1620e1afbd0d28ea?w=1644&h=1084&f=jpeg&s=183419)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我把文件下载下来并且删去了注释，通过这张图我们可以看出这个文件的作用是挂载了所有的方法（包括use等关键api）。</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这里面比较重要的是<code>use</code>方法，它的作用就是把我们用<code>app.use</code>注册的所有中间件和路由方法交给<code>Router</code>类来处理。</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;那我们再看看router文件夹类的结构：

![](https://user-gold-cdn.xitu.io/2018/3/10/1620e3eb527724d1?w=2054&h=524&f=jpeg&s=148712)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;index.js是入口文件，处理所有的路由；</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;layer.js中声明了<code>Layer</code>类，处理每一层路由中间件或者每一个子中间件；</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;router.js中声明了<code>Router</code>类，处理每一个子路由。</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这里面有一个子中间件的概念，对应Exprees文档中有这一句话：
> 另外，你还可以同时装载一系列中间件函数，从而在一个挂载点上创建一个子中间件栈。</br>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这句话的意思是说我们可以把代码写成下面这种形式：
``````
app.use('/user1', function fn1(req, res, next) {
  // TODO
  next();
}, function fn2(req, res, next) {
  //TODO
  next();
});
app.use('/user2', function fn3(req, res, next) {
  // TODO
  next();
}, function fn4(req, res, next) {
  //TODO
  next();
});
``````
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上面的代码给user1和user2分别创建了一个子中间件栈。这种语法的实现就是靠<code>Layer</code>类实现的。</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;画一张图来解释上面的代码：

![](https://user-gold-cdn.xitu.io/2018/3/10/1620ebc3bcac2d5a?w=1055&h=731&f=jpeg&s=61518)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;解释一下上面的代码和图：</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们写了两个路由/user1和/user2,每个路由给了两个处理函数。对于这段代码，Express是这样处理的：
1. 在index.js文件中，定义了一个<code>stack</code>数组，接下来会创建两个<code>Layer</code>放到这个<code>stack</code>中。
2. route.js模块会给/user1再创建一个<code>stack</code>和fn1、fn2两个<code>Layer</code>。
3. /user2同/user1
4. 最后，Express会从上往下执行每个<code>Layer</code>里的函数，对应到图上就是从上至下、从左至右的依次执行，顺序为fn1、fn2、fn3、fn4。

## 最后
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;以上就是Express中间件的简单原理，白话较多，如果写的不准确的地方，希望大家批评指正。

## 参考资料
[Express官网](http://www.expressjs.com.cn/guide/using-middleware.html)</br>
[Express的github仓库地址](http://www.expressjs.com.cn/guide/using-middleware.html)
