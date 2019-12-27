---
title: JavaScript 高级程序设计
tags: JavaScript
key: JavaScript
---

记录《JavaScript 高级程序设计》学习过程的一些值得注意的地方。

<!--more-->

## chapter 4. 变量, 作用域和引用

### 动态的属性

JS 中没有对 Object 内容的显式定义，而是可以任意的扩展：

```js
var person = new Object();
person.name = "ckcd";
alert(person.name);   // ckcd
```

### 值传递和引用传递

> 老生常谈的问题...

基础类型是值传递，Object 是引用传递, 参数都是值传递.

```js
// 值传递
var num1 = 3;
var num2 = num1;
num1 = 5;
alert(num2);   // 3

// 引用传递
var person1 = new Object();
person1.name = "ckcd";
var person2 = person1;
person1.name = "ckchang";
alert(person2.name);   // ckchang
```

### 检测类型

```js
var s = "ckcd";
var b = true;
var i = 22;
var u;
var n = null;
var o = new Object();

alert(typeof s);        //string
alert(typeof i);        //number
alert(typeof b);        //boolean
alert(typeof u);        //undefined
alert(typeof n);        //object
alert(typeof o);        //object
```

### 没有块级作用域

```js
if (true) {
    var color = "blue";
}
alert(color);    //"blue", 而不是报错
```

### GC

曾经使用`引用计数`，现在主要使用 `标记清除`.

手动`解除引用`:

```js
function createPerson(name){
    var localPerson = new Object();
    localPerson.name = name;
}

var globalPerson = createPerson("Nicholas"); 

// 手工解除 globalPerson 的引用
globalPerson = null;
```

解除一个值的引用并不意味着自动回收该值所占用的内存。解除引用的真正作用是让值脱离`执行环境`，以便垃圾收集器下次运行时将其回收.

