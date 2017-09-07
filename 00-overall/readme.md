jQuery 总体架构

###1,jQuery 内部结构图

```
var jQuery = function(){
  return new jQuery.prototype.init();
}
jQuery.fn = jQuery.prototype = {
  ...
}
```

至于为什么要用 fn 指向 prototype？你不觉得 fn 比 prototype 好写多了吗。

网上的一张图：


从这张图中可以看出，window 对象上有两个公共的接口，分别是 $ 和 jQuery：

```
window.jQuery = window.$ = jQuery;
```
jQuery.extend 方法是一个对象拷贝的方法，包括深拷贝，后面会详细讲解源码，暂时先放一边。

下面的关系可能会有些乱，但是仔细看了前面的介绍，应该能看懂。fn 就是 prototype，所以 jQuery 的 fn 和 prototype 属性指向 fn 对象，而 init 函数本身就是 jQuery.prototype 中的方法，且 init 函数的 prototype 原型指向 fn。

###2，链式调用

链式调用的好处，就是写出来的代码非常简洁，而且代码返回的都是同一个对象，提高代码效率。

前面已经说了，在没有返回值的原型函数后面添加 return this：
```
var jQuery = function(){
  return new jQuery.fn.init();
}
jQuery.fn = jQuery.prototype = {
  constructor: jQuery,
  init: function(){
    this.jquery = 3.0;
    return this;
  },
  each: function(){
    console.log('each');
    return this;
  }
}
jQuery.fn.init.prototype = jQuery.fn;
jQuery().each().each();
// 'each'
// 'each'
```

###extend

jQuery 中一个重要的函数便是 extend，既可以对本身 jQuery 的属性和方法进行扩张，又可以对原型的属性和方法进行扩展。

先来说下 extend 函数的功能，大概有两种，如果参数只有一个 object，即表示将这个对象扩展到 jQuery 的命名空间中，也就是所谓的 jQuery 的扩展。如果函数接收了多个 object，则表示一种属性拷贝，将后面多个对象的属性全拷贝到第一个对象上，这其中，还包括深拷贝，即非引用拷贝，第一个参数如果是 true 则表示深拷贝。

```
jQuery.extend(target);// jQuery 的扩展
jQuery.extend(target, obj1, obj2,..);//浅拷贝 
jQuery.extend(true, target, obj1, obj2,..);//深拷贝 
```
一下是 jQuery 3 之后的 extend 函数源码，自己做了注释：

```
jQuery.extend = jQuery.fn.extend = function () {
  var options, name, src, copy, copyIsArray, clone, target = arguments[0] || {},
    i = 1,
    length = arguments.length,
    deep = false;

  // 判断是否为深拷贝
  if (typeof target === "boolean") {
    deep = target;

    // 参数后移
    target = arguments[i] || {};
    i++;
  }

  // 处理 target 是字符串或奇怪的情况，isFunction(target) 可以判断 target 是否为函数
  if (typeof target !== "object" && !jQuery.isFunction(target)) {
    target = {};
  }

  // 判断是否 jQuery 的扩展
  if (i === length) {
    target = this; // this 做一个标记，可以指向 jQuery，也可以指向 jQuery.fn
    i--;
  }

  for (; i < length; i++) {

    // null/undefined 判断
    if ((options = arguments[i]) != null) {

      // 这里已经统一了，无论前面函数的参数怎样，现在的任务就是 target 是目标对象，options 是被拷贝对象
      for (name in options) {
        src = target[name];
        copy = options[name];

        // 防止死循环，跳过自身情况
        if (target === copy) {
          continue;
        }

        // 深拷贝，且被拷贝对象是 object 或 array
        // 这是深拷贝的重点
        if (deep && copy && (jQuery.isPlainObject(copy) || (copyIsArray = Array.isArray(copy)))) {
          // 说明被拷贝对象是数组
          if (copyIsArray) {
            copyIsArray = false;
            clone = src && Array.isArray(src) ? src : [];
          // 被拷贝对象是 object
          } else {
            clone = src && jQuery.isPlainObject(src) ? src : {};
          }

          // 递归拷贝子属性
          target[name] = jQuery.extend(deep, clone, copy);

          // 常规变量，直接 =
        } else if (copy !== undefined) {
            target[name] = copy;
        }
      }
    }
  }

  // Return the modified object
  return target;
}
```
extend 函数符合 jQuery 中的参数处理规范，算是比较标准的一个。jQuery 对于参数的处理很有一套，总是喜欢错位来使得每一个位置上的变量和它们的名字一样，各司其职。比如 target 是目标对象，如果第一个参数是 boolean 型的，就对 deep 赋值 target，并把 target 向后移一位；如果参数对象只有一个，即对 jQuery 的扩展，就令 target 赋值 this，当前指针 i 减一。

这种方法逻辑虽然很复杂，但是带来一个非常大的优势：后面的处理逻辑只需要一个就可以。target 就是我们要拷贝的目标，options 就是要拷贝的对象，逻辑又显得非常的清晰。

extend 函数还需要主要一点，jQuery.extend = jQuery.fn.extend，不仅 jQuery 对象又这个函数，连原型也有，那么如何区分对象是扩展到哪里了呢，又是如何实现的？

其实这一切都要借助与 javascript 中 this 的动态性，target = this，代码就放在那里，谁去执行，this 就会指向谁，就会在它的属性上扩展。
