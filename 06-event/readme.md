### 由 JS 事件引入

事件是 JS DOM 中极具活力的内容，你可以随时监听 DOM 的变化，并对它们及时的做出反应，如果你不是太懂 JS 中的事件，建议你先去看一些相关介绍的文章，直接看 jQuery 中的事件委托头会头大的。

事件的处理顺序由两个环节，一个是捕获环节，一个是冒泡环节，借用别人的一张图：
![event](https://github.com/moveondo/JqueryLearn/blob/master/06-event/event.jpg)  

事件冒泡与捕获

如果把处理也算进的话，整个事件分为三个阶段，分别是捕获阶段，目标处理阶段和冒泡阶段。捕获阶段由外向内寻找 target，冒泡阶段由内向外直到根结点。这只是一个事件，当这三个阶段中又穿插着更多的事件时，还需要将事件的执行顺序考虑进去。

而 jQuery 事件委托的概念：事件目标自身不处理事件，而是将其委托给父元素或祖先元素或根元素，而借助事件的冒泡性质（由内向外）来达到最终处理事件。

jQuery 中的事件优化

首先必须要知道，绑定事件越多，浏览器内存占用越大，就会间接的影响性能。而且一旦出现 ajax，局部刷新导致重新绑定事件。

使用事件委托可以解决以上带来的问题，借助事件的冒泡，尤其当一个父元素的子元素过多，而且子元素绑定的事件非常多时，委托事件的作用就体现出来了。



在早期的 jQuery 版本，使用的是 .delegate()、.bind()、.live()等方法来实现事件监听，当然也包括.click()方法，随着 jQuery 的发展，像 live 方法已经明确从 jQuery 中删除，而其余的方法，比如 bind 方法也将在 3.0 之后的版本陆续删除，取而代之的是 .on()方法。而且剩下的其它方法都是通过 on 方法来间接实现的，如果介绍，只需要看 on 的源码即可。

on 函数在 jQuery 中的用法也很简单，.on( events [, selector ] [, data ], handler(eventObject) )events 表示绑定的事件，比如 "click" 或 "click mouseleave"，selector 和 data 是可选的，分别表示要绑定事件的元素和要执行的数据，handler 表示事件执行函数。

off 函数的用法 .off( events [, selector ] [, handler ] )，events 代表要移除的事件，selector 表示选择的 dom，handler 表示事件处理函数。还有更残暴的比如 .off()不接受任何参数，表示着移除所有 on 绑定的函数。

on off 函数源码

虽然我分析的源码时 jQuery 3.1.1，但这个时候 bind 和 delegate 函数并没有从源码中移除呢，先来看看它们怎么调用 on：
```
jQuery.fn.extend( {
  bind: function( types, data, fn ) {
    return this.on( types, null, data, fn );
  },
  unbind: function( types, fn ) {
    return this.off( types, null, fn );
  },
  delegate: function( selector, types, data, fn ) {
    return this.on( types, selector, data, fn );
  },
  undelegate: function( selector, types, fn ) {
    // ( namespace ) or ( selector, types [, fn] )
    return arguments.length === 1 ?
      this.off( selector, "**" ) :
      this.off( types, selector || "**", fn );
  }
} );
```
可以看得出来，全都被 on 和 off 这两个函数来处理了。
```
jQuery.fn.extend( {
  on: function (types, selector, data, fn) {
    // on 又依托于全局的 on 函数
    return on(this, types, selector, data, fn);
  }
} );
function on( elem, types, selector, data, fn, one ) {
  var origFn, type;

  // 支持 object 的情况
  if ( typeof types === "object" ) {

    // ( types-Object, selector, data )
    if ( typeof selector !== "string" ) {

      // ( types-Object, data )
      data = data || selector;
      selector = undefined;
    }
    // 一次执行 object 的每一个
    for ( type in types ) {
      on( elem, type, selector, data, types[ type ], one );
    }
    return elem;
  }
  // 参数为两个的情况
  if ( data == null && fn == null ) {

    // ( types, fn )
    fn = selector;
    data = selector = undefined;
  } else if ( fn == null ) {
    if ( typeof selector === "string" ) {

      // ( types, selector, fn )
      fn = data;
      data = undefined;
    } else {

      // ( types, data, fn )
      fn = data;
      data = selector;
      selector = undefined;
    }
  }
  if ( fn === false ) {
    // returnFalse 是一个返回 false 的函数
    fn = returnFalse;
  } else if ( !fn ) {
    return elem;
  }

  if ( one === 1 ) {
    origFn = fn;
    fn = function( event ) {

      // Can use an empty set, since event contains the info
      jQuery().off( event );
      return origFn.apply( this, arguments );
    };

    // Use same guid so caller can remove using origFn
    fn.guid = origFn.guid || ( origFn.guid = jQuery.guid++ );
  }
  return elem.each( function() {
    // 关键
    jQuery.event.add( this, types, fn, data, selector );
  } );
}
```
是的，你没有看错，这个全局的 on 函数，其实只是起到了校正参数的作用，而真正的大头是：
```
jQuery.event = {
  global = {},
  add: function(){...},
  remove: function(){...},
  dispatch: function(){...},
  handlers: function(){...},
  addProp: function(){...},
  fix: function(){...},
  special: function(){...}
}
off 函数：

jQuery.fn.off = function (types, selector, fn) {
  var handleObj, type;
  if (types && types.preventDefault && types.handleObj) {
    // ( event )  dispatched jQuery.Event
    handleObj = types.handleObj;
    jQuery(types.delegateTarget).off(
      handleObj.namespace ? handleObj.origType + "." + handleObj.namespace : handleObj.origType,
      handleObj.selector,
      handleObj.handler
    );
    return this;
  }
  if (typeof types === "object") {
    // ( types-object [, selector] )
    for (type in types) {
      this.off(type, selector, types[type]);
    }
    return this;
  }
  if (selector === false || typeof selector === "function") {
    // ( types [, fn] )
    fn = selector;
    selector = undefined;
  }
  if (fn === false) {
    fn = returnFalse;
  }
  return this.each(function() {
    // 关键
    jQuery.event.remove(this, types, fn, selector);
  });
}
```



可见 jQuery 对于参数的放纵导致其处理起来非常复杂，不过对于使用者来说，却非常大便利。

委托事件也带来了一些不足，比如一些事件无法冒泡，load、submit 等，会加大管理等复杂，不好模拟用户触发事件等。



### 一些遗留问题

前面介绍 bind、delegate 和它们的 un 方法的时候，经提醒，忘记提到一些内容，却是我们经常使用的。比如 $('body').click，$('body').mouseleave等，它们是直接定义在原型上的函数，不知道怎么，就把它们给忽略了。
```
jQuery.each( ( "blur focus focusin focusout resize scroll click dblclick " +
  "mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave " +
  "change select submit keydown keypress keyup contextmenu" ).split( " " ),
  function( i, name ) {

  // Handle event binding
  jQuery.fn[ name ] = function( data, fn ) {
    return arguments.length > 0 ?
      this.on( name, null, data, fn ) :
      this.trigger( name );
  };
} );
```
这个构造也是十分巧妙的，这些方法组成的字符串通过 split(" ") 变成数组，而后又通过 each 方法，在原型上对应每个名称，定义函数，这里可以看到，依旧是 on，还有 targger：
```
jQuery.fn.extend( {
  trigger: function(type, data){
    return this.each(function (){
      // // 依旧是 event 对象上的方法
      jQuery.event.trigger(type, data, this);
    })
  }
} )
```
还缺少一个 one 方法，这个方法表示绑定的事件同类型只执行一次，.one()：
```
jQuery.fn.extend( {
  one: function( types, selector, data, fn ) {
    // 全局 on 函数
    return on( this, types, selector, data, fn, 1 );
  },
} );
```
DOM 事件知识点

发现随着 event 源码的不断的深入，我自己出现越来越多的问题，比如没有看到我所熟悉的 addEventListener，还有一些看得很迷糊的 events 事件，所以我决定还是先来看懂 JS 中的 DOM 事件吧。

早期 DOM 事件

在 HTML 的 DOM 对象中，有一些以 on 开头的熟悉，比如 onclick、onmouseout 等，这些就是早期的 DOM 事件，它的最简单的用法，就是支持直接在对象上以名称来写函数：
```
document.getElementsByTagName('body')[0].onclick = function(){
  console.log('click!');
}
document.getElementsByTagName('body')[0].onmouseout = function(){
  console.log('mouse out!');
}
```
onclick 函数会默认传入一个 event 参数，表示触发事件时的状态，包括触发对象，坐标等等。

这种方式有一个非常大的弊端，就是相同名称的事件，会前后覆盖，后一个 click 函数会把前一个 click 函数覆盖掉：
```
var body = document.getElementsByTagName('body')[0];
body.onclick = function(){
  console.log('click1');
}
body.onclick = function(){
  console.log('click2');
}
// "click2"
body.onclick = null;
// 没有效果
```
#### DOM 2.0

随着 DOM 的发展，已经来到 2.0 时代，也就是我所熟悉的 addEventListener 和 attachEvent(IE)，JS 中的事件冒泡与捕获。这个时候和之前相比，变化真的是太大了，MDN addEventListener()。

变化虽然是变化了，但是浏览器的兼容却成了一个大问题，比如下面就可以实现不支持 addEventListener 浏览器：
```
(function() {
  // 不支持 preventDefault
  if (!Event.prototype.preventDefault) {
    Event.prototype.preventDefault=function() {
      this.returnValue=false;
    };
  }
  // 不支持 stopPropagation
  if (!Event.prototype.stopPropagation) {
    Event.prototype.stopPropagation=function() {
      this.cancelBubble=true;
    };
  }
  // 不支持 addEventListener 时候
  if (!Element.prototype.addEventListener) {
    var eventListeners=[];
    
    var addEventListener=function(type,listener /*, useCapture (will be ignored) */) {
      var self=this;
      var wrapper=function(e) {
        e.target=e.srcElement;
        e.currentTarget=self;
        if (typeof listener.handleEvent != 'undefined') {
          listener.handleEvent(e);
        } else {
          listener.call(self,e);
        }
      };
      if (type=="DOMContentLoaded") {
        var wrapper2=function(e) {
          if (document.readyState=="complete") {
            wrapper(e);
          }
        };
        document.attachEvent("onreadystatechange",wrapper2);
        eventListeners.push({object:this,type:type,listener:listener,wrapper:wrapper2});
        
        if (document.readyState=="complete") {
          var e=new Event();
          e.srcElement=window;
          wrapper2(e);
        }
      } else {
        this.attachEvent("on"+type,wrapper);
        eventListeners.push({object:this,type:type,listener:listener,wrapper:wrapper});
      }
    };
    var removeEventListener=function(type,listener /*, useCapture (will be ignored) */) {
      var counter=0;
      while (counter<eventListeners.length) {
        var eventListener=eventListeners[counter];
        if (eventListener.object==this && eventListener.type==type && eventListener.listener==listener) {
          if (type=="DOMContentLoaded") {
            this.detachEvent("onreadystatechange",eventListener.wrapper);
          } else {
            this.detachEvent("on"+type,eventListener.wrapper);
          }
          eventListeners.splice(counter, 1);
          break;
        }
        ++counter;
      }
    };
    Element.prototype.addEventListener=addEventListener;
    Element.prototype.removeEventListener=removeEventListener;
    if (HTMLDocument) {
      HTMLDocument.prototype.addEventListener=addEventListener;
      HTMLDocument.prototype.removeEventListener=removeEventListener;
    }
    if (Window) {
      Window.prototype.addEventListener=addEventListener;
      Window.prototype.removeEventListener=removeEventListener;
    }
  }
})();
```
虽然不支持 addEventListener 的浏览器可以实现这个功能，但本质上还是通过 attachEvent 函数来实现的，在理解 DOM 早期的事件如何来建立还是比较捉急的。

#### addEvent 库

addEvent库的这篇博客发表于 2005 年 10 月，所以这篇博客所讲述的 addEvent 方法算是经典型的，就连 jQuery 中的事件方法也是借鉴于此，故值得一提：
```
function addEvent(element, type, handler) {
  // 给每一个要绑定的函数添加一个标识 guid
  if (!handler.$$guid) handler.$$guid = addEvent.guid++;
  // 在绑定的对象事件上创建一个事件对象
  if (!element.events) element.events = {};
  // 一个 type 对应一个 handlers 对象，比如 click 可同时处理多个函数
  var handlers = element.events[type];
  if (!handlers) {
    handlers = element.events[type] = {};
    // 如果 onclick 已经存在一个函数，拿过来
    if (element["on" + type]) {
      handlers[0] = element["on" + type];
    }
  }
  // 防止重复绑定，每个对应一个 guid
  handlers[handler.$$guid] = handler;
  // 把 onclick 函数替换成 handleEvent
  element["on" + type] = handleEvent;
};
// 初始 guid
addEvent.guid = 1;

function removeEvent(element, type, handler) {
  // delete the event handler from the hash table
  if (element.events && element.events[type]) {
    delete element.events[type][handler.$$guid];
  }
  // 感觉后面是不是要加个判断，当 element.events[type] 为空时，一起删了
};

function handleEvent(event) {
  // grab the event object (IE uses a global event object)
  event = event || window.event;
  // 这里的 this 指向 element
  var handlers = this.events[event.type];
  // execute each event handler
  for (var i in handlers) {
    // 这里有个小技巧，为什么不直接执行，而是先绑定到 this 后执行
    // 是为了让函数执行的时候，内部 this 指向 element
    this.$$handleEvent = handlers[i];
    this.$$handleEvent(event);
  }
};
```
如果能将上面 addEvent 库的这些代码看懂，那么在看 jQuery 的 events 源码就明朗多了。

还有一个问题，所谓事件监听，是将事件绑定到父元素或 document 上，子元素来响应，如何实现？

要靠 event 传入的参数 e：
```
var body = document.getElementsByTagName('body')[0];
body.onclick = function(e){
  console.log(e.target.className);
}
```
这个 e.target 对象就是点击的那个子元素了，无论是捕获也好，冒泡也好，貌似都能够模拟出来。接下来，可能要真的步入正题了。

