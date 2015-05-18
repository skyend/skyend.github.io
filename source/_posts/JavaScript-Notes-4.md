title: "JavaScript杂记（四）"
date: 2015-05-18 19:28:45
categories:
- Front-End
- JavaScript
tags:
- JavaScript
---
![](/thumbnails/t6.jpg)
以下摘自《JavaScript 权威指南（第6版）》
<!-- more -->
* null和undefined: 可以将null认为是一个特殊的对象，含义是“非对象”，事实上通常认为null是它自有类型的唯一一个成员，表示数字、字符串和对象是“无值”的；undefined是预定义的全局变量（它和null不一样，它不是关键字），它的值就是未定义。null和undefined都不包含任何方法。

* JavaScript通过调用new Object(x)的方式将原始值（Number、String、Boolean……）进行包装，所以表面上可以对基本类型使用属性的引用，如"x".length。但是一旦属性引用结束，这个新创建的对象就会销毁。例如：
  ```js
  var s = "test";
  s.len = 4;
  var t = s.len;              // => undefined
  ```

* JavaScript中的原始值与对象有着根本区别，原始值是不可变的。JavaScript中字符串中的所有方法实际上返回的都是一个新的字符串值。例如：
  ```js
  var s = "hello";
  s.toUpperCase();
  s                           // => "hello"
  ```

* 对象的比较并非值的比较，即使两个对象包含相同的属性和相同的值，他们也是不相等的，各个索引元素完全相等的两个数组也不相等。

* 任何对象和数组（包括空对象/数组）到布尔值的转换都是true。
  ```js
  if({} && [])
    console.log(1);           // => 1
  ```

* if语句将undefined转换为false，但“==”运算符从不试着将其操作数转换为布尔值。

* !!x 相当于Boolean(x)

* new Boolean(false)是一个对象而不是原始值，它将转换为true：
  ```js
  Boolean(new Boolean(false)) // => true
  ```

* JavaScript的提前声明特性：JavaScript函数里声明的所有变量（但不涉及赋值）都被“提前”至函数体的顶部，示例如下：
  ```js
  var scope = "global";
  function f() {
    console.log(scope);       // => undefined
    var scope = "local";      // 变量在这里赋初值，但变量本身在函数体内的任何地方均是有定义的
    console.log(scope);       // => "local"
  }
  ```

* 对于嵌套函数来说，每次调用外部函数时，内部函数又会重新定义一遍。因为每次调用外部函数的时候，作用域链都是不同的。