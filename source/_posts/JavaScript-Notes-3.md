title: "JavaScript杂记（三）"
date: 2015-04-28 13:40:35
categories:
- Front-End
- JavaScript
tags:
- JavaScript
---

阅读[刘同学的博客](http://liuyangzuo.me/)有感，笔记如下：
<!-- more -->
* 非常有用的本地图片预览，使用了[HTML5 FileReader](https://developer.mozilla.org/zh-CN/docs/Web/API/FileReader)
  ```js
  function previewFile() {
    var preview = document.querySelector('img');
    var file    = document.querySelector('input[type=file]').files[0];
    var reader  = new FileReader();

    reader.onloadend = function () {
      preview.src = reader.result;
    }

    if (file) {
      reader.readAsDataURL(file);
    } else {
      preview.src = "";
    }
  }
  ```
  浏览器支持情况：
  Firefox (Gecko) 3.6(1.9.2)+
  Chrome 7+
  Internet Explorer\* 10+
  Opera\*
  Safari
  *IE9有一个File API Lab，Opera从11.10开始部分支持该API.

* [对象原型继承](http://liuyangzuo.me/2015/01/30/JavaScript-Prototype-%E4%B8%80/)，我的看法：
  ```js
  // 更改了创建方式
  Object.prototype.create = function(){
    var F = function(){};
    F.prototype = this;
    return F;
  }
  // 可以直接从所有Object进行继承，不需要使用Object.create的方式传入
  var stooge = {
    "first-name":"liu",
    "last-name":"yangzuo"
  }
  var another1_stooge = stooge.create();
  another1_stooge["first-name"] = "zhang";
  stooge["first-name"];          // "liu"
  another2_stooge["first-name"]; // "zhang"
  ```
  这与[对象的深拷贝](http://liuyangzuo.me/2015/04/17/Deep-copy/)是不同的，深拷贝知识将属性和方法进行复制，并不更改原型链：
  ```js
  Object.prototype.clone = function(){

      var o = this.constructor === Array ? [] : {};
    for(var e in this){
                o[e] = typeof this[e] === "object" ? this[e].clone() : this[e];
    }
      return o;
  }
  ```
* 关于[事件冒泡与事件捕获](http://liuyangzuo.me/2015/02/04/JavaScript-%E4%BA%8B%E4%BB%B6/)
  > 一般浏览级事件传播默认是事件冒泡,我们可以设置事件传播的方式是冒泡方式,还是捕获方式。上面所讲的addEventListener函数的第三个参数就是选择事件传播的方式，传入true事件传播方式为事件捕获,传入false事件传播方式为事件冒泡。

  另附：[jQuery中return false,e.preventDefault(),e.stopPropagation()的区别](http://blog.csdn.net/woshixuye/article/details/7422985)

* [怎么让一个div元素保持适中水平,竖直居中?](http://liuyangzuo.me/2015/01/14/div%E6%B0%B4%E5%B9%B3%E7%AB%96%E7%9B%B4%E5%B1%85%E4%B8%AD/)，上面的方法只能用来解决定宽定高的容器居中问题，其实使用Table配合
  ```css
  text-align:center;verticle-align:middle;
  ```
  效果更佳。

* [jQuery对象转化DOM对象](http://liuyangzuo.me/2014/11/09/jQuery%E5%AF%B9%E8%B1%A1%E8%BD%AC%E5%8C%96DOM%E5%AF%B9%E8%B1%A1/)，仅需将jQuery对象当作数组即可，新技能get。

* [好玩的JavaScript(二)](http://liuyangzuo.me/2014/11/03/%E5%A5%BD%E7%8E%A9%E7%9A%84JavaScript-%E4%BA%8C/)
  ```js
  var SideBar = (function Module(){
    var SideBar = {
      num : 1,
      animation : function(){
        function f(){
          console.log(this == window);
        }
        console.log(this == SideBar);
        f();
      }
    }
    return SideBar;
  }());
  console.log(SideBar.num); // 1
  SideBar.animation();      // true -> this === Sidebar
                            // true -> this === window
  ```
  注意，倒数第二行的Sidebar是闭包中的Sidebar，而不是立即函数前的Sidebar。这涉及到[JavaScript中的this](http://liuyangzuo.me/2014/10/31/JavaScript%E4%B8%AD%E7%9A%84this/)：
  > 如果嵌套函数作为函数调用,其this值不是全局对象(非严格模式)就是undefined(严格模式)

* 近来发现JavaScript不是唯一拥有闭包特性的语言。PHP和Python等也有……（囧）

通篇略读完，受益匪浅。