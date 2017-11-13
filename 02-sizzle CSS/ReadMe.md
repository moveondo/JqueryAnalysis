### Sizzle 选择器

Sizzle 原本是 jQuery 中用来当作 DOM 选择器的，后来被 John Resig 单独分离出去，成为一个单独的项目，可以直接导入到项目中使用。

   我们使用 jQuery 当作选择器，选定一些 #id 或 .class，使用 document.getElementById 或 document.getElemensByClassName 就可以很快锁定 DOM 所在的位置，然后返回给 jQuery 当作对象。但有时候会碰到一些比较复杂的选择 div div.hot>span 这类肯定用上面的函数是不行的，首先考虑到的是 Element.querySelectorAll() 函数，但这个函数存在严重的兼容性问题MDN querySelectorAll。这个时候 sizzle 就派上用场了。

init 函数介绍中已经说明白，没有介绍 find 函数，其本质上就是 Sizzle 函数在 jQuery 中的表现。这个函数在 jQuery 中两种存在形式，即原型和属性上分别有一个，先来看下 jQuery.fn.find:
```
jQuery.fn.find = function (selector) {
  var i, ret, len = this.length,
    self = this;
  // 这段话真不知道是个什么的
  if (typeof selector !== "string") {
    // fn.pushStack 和 jquery.merge 很像，但是返回一个 jquery 对象，且
    // jquery 有个 prevObject 属性指向自己
    return this.pushStack(jQuery(selector).filter(function () {
      for (i = 0; i < len; i++) {
        // jQuery.contains(a, b) 判断 a 是否是 b 的父代
        if (jQuery.contains(self[i], this)) {
            return true;
        }
      }
    }));
  }

  ret = this.pushStack([]);

  for (i = 0; i < len; i++) {
    // 在这里引用到 jQuery.find 函数
    jQuery.find(selector, self[i], ret);
  }
  // uniqueSort 去重函数
  return len > 1 ? jQuery.uniqueSort(ret) : ret;
}

jQuery.fn.find 的用法一般在 $('.test').find("span")，所以此时的 this 是指向 $('.test') 的，懂了这一点，后面的东西自然而然就好理解了。

然后就是 jQuery.find 函数，本章的重点讨论部分。先来看一个正则表达式：

var rquickExpr = /^(?:#([\w-]+)|(\w+)|\.([\w-]+))$/;
rquickExpr.exec('#id') //["#id", "id", undefined, undefined]
rquickExpr.exec('div') //["div", undefined, "div", undefined]
rquickExpr.exec('.test') //[".test", undefined, undefined, "test"]
rquickExpr.exec('div p')// null
你可能会疑惑，rquickExpr 的名字已经出现过一次了。实际上 Sizzle 是一个闭包，这个 rquickExpr 变量是在 Sizzle 闭包内的，不会影响到 jQuery 全局。这个正则的作用主要是用来区分 tag、id 和 class，而且从返回的数组也有一定的规律，可以通过这个规律来判断 selector 具体是哪一种。

jQuery.find = Sizzle;

function Sizzle(selector, context, results, seed) {
  var m, i, elem, nid, match, groups, newSelector, newContext = context && context.ownerDocument,

  // nodeType defaults to 9, since context defaults to document
  nodeType = context ? context.nodeType : 9;

  results = results || [];

  // Return early from calls with invalid selector or context
  if (typeof selector !== "string" || !selector || nodeType !== 1 && nodeType !== 9 && nodeType !== 11) {

    return results;
  }

  // Try to shortcut find operations (as opposed to filters) in HTML documents
  if (!seed) {

    if ((context ? context.ownerDocument || context : preferredDoc) !== document) {
      // setDocument 函数其实是用来将 context 设置成 document，考虑到浏览器的兼容性
      setDocument(context);
    }
    context = context || document;
    // true
    if (documentIsHTML) {

      // match 就是那个有规律的数组
      if (nodeType !== 11 && (match = rquickExpr.exec(selector))) {

        // selector 是 id 的情况
        if ((m = match[1])) {

          // Document context
          if (nodeType === 9) {
            if ((elem = context.getElementById(m))) {

              if (elem.id === m) {
                  results.push(elem);
                  return results;
              }
            } else {
              return results;
            }

          // 非 document 的情况
          } else {

            if (newContext && (elem = newContext.getElementById(m)) && contains(context, elem) && elem.id === m) {

                results.push(elem);
                return results;
            }
          }

        // selector 是 tagName 情况
        } else if (match[2]) {
            // 这里的 push：var push = arr.push
            push.apply(results, context.getElementsByTagName(selector));
            return results;

        // selector 是 class 情况
        } else if ((m = match[3]) && support.getElementsByClassName && context.getElementsByClassName) {

            push.apply(results, context.getElementsByClassName(m));
            return results;
        }
      }

      // 如果浏览器支持 querySelectorAll
      if (support.qsa && !compilerCache[selector + " "] && (!rbuggyQSA || !rbuggyQSA.test(selector))) {

        if (nodeType !== 1) {
          newContext = context;
          newSelector = selector;

          // qSA looks outside Element context, which is not what we want
          // Support: IE <=8,还是要考虑兼容性
        } else if (context.nodeName.toLowerCase() !== "object") {

          // Capture the context ID, setting it first if necessary
          if ((nid = context.getAttribute("id"))) {
            nid = nid.replace(rcssescape, fcssescape);
          } else {
            context.setAttribute("id", (nid = expando));
          }

          // Sizzle 词法分析的部分
          groups = tokenize(selector);
          i = groups.length;
          while (i--) {
            groups[i] = "#" + nid + " " + toSelector(groups[i]);
          }
          newSelector = groups.join(",");

          // Expand context for sibling selectors
          newContext = rsibling.test(selector) && testContext(context.parentNode) || context;
        }

        if (newSelector) {
          try {
            push.apply(results, newContext.querySelectorAll(newSelector));
            return results;
          } catch(qsaError) {} finally {
            if (nid === expando) {
                context.removeAttribute("id");
            }
          }
        }
      }
    }
  }

  // All others，select 函数和 tokenize 函数后文再谈
  return select(selector.replace(rtrim, "$1"), context, results, seed);
}
```
整个分析过程由于要考虑各种因素，包括效率和浏览器兼容性等，所以看起来非常长，但是逻辑一点都不难：先判断 selector 是否是非 string，然后正则 rquickExpr 对 selector 进行匹配，获得数组依次考虑 id、tagName 和 class 情况，这些都很简单，都是单一的选择，一般用浏览器自带的函数 getElement 即可解决。遇到复杂一点的，比如 div div.show p,先考虑 querySelectorAll 函数是否支持，然后考虑浏览器兼容 IE<8。若不支持，即交给 select 函数（下章）。

Sizzle 的优势

Sizzle 使用的是从右向左的选择方式，这种方式效率更高。

浏览器在处理 html 的时候，先生成一个 DOM tree，解析完 css 之后，然后更加 css 和 DOM tess 生成一个 render tree。render tree 用于渲染，不是一一对应，如 display:none 的 DOM 就不会出现在 render tree 中。

如果从左到右的匹配方式，div div.show p，

找到 div 节点，
从 1 的子节点中找到 div 且 class 为 show 的 DOM，找不到则返回上一步
从 2 的子节点中找到 p 元素，找不到则返回上一步
如果有一步找不到，向上回溯，直到遍历所有的 div，效率很低。

如果从右到左的方式，

先匹配到所有的 p 节点，
对 1 中的结果注意判断，若其父节点顺序出现 div.show 和 div，则保留，否则丢弃
因为子节点可以有若干个，而父节点只有一个，故从右向左的方式效率很高。

衍生的函数

jQuery.fn.pushStack

jQuery.fn.pushStack是一个类似于 jQuery.merge 的函数，它接受一个参数，把该参数（数组）合并到一个 jQuery 对象中并返回，源码如下：
```
jQuery.fn.pushStack = function (elems) {

  // Build a new jQuery matched element set
  var ret = jQuery.merge(this.constructor(), elems);

  // Add the old object onto the stack (as a reference)
  ret.prevObject = this;

  // Return the newly-formed element set
  return ret;
}
```
jQuery.contains

这个函数是对 DOM 判断是否是父子关系，源码如下：
```
jQuery.contains = function (context, elem) {
  // 考虑到兼容性，设置 context 的值
  if ((context.ownerDocument || context) !== document) {
    setDocument(context);
  }
  return contains(context, elem);
}

// contains 是内部函数，判断 DOM_a 是否是 DOM_b 的
var contains =  function (a, b) {
  var adown = a.nodeType === 9 ? a.documentElement : a,
  bup = b && b.parentNode;
  return a === bup || !!(bup && bup.nodeType === 1 && (
  adown.contains ? adown.contains(bup) : a.compareDocumentPosition && a.compareDocumentPosition(bup) & 16));
}
jQuery.uniqueSort

jQuery 的去重函数，但这个去重职能处理 DOM 元素数组，不能处理字符串或数字数组，来看看有什么特别的：

jQuery.uniqueSort = function (results) {
  var elem, duplicates = [],
    j = 0,
    i = 0;

  // hasDuplicate 是一个判断是否有相同元素的 flag，全局
  hasDuplicate = !support.detectDuplicates;
  sortInput = !support.sortStable && results.slice(0);
  results.sort(sortOrder);

  if (hasDuplicate) {
    while ((elem = results[i++])) {
      if (elem === results[i]) {
          j = duplicates.push(i);
      }
    }
    while (j--) {
      // splice 用于将重复的元素删除
      results.splice(duplicates[j], 1);
    }
  }

  // Clear input after sorting to release objects
  // See https://github.com/jquery/sizzle/pull/225
  sortInput = null;

  return results;
}
sortOrder 函数如下，需要将两个函数放在一起理解才能更明白哦：

var sortOrder = function (a, b) {

  // 表示有相同的元素，设置 flag 为 true
  if (a === b) {
    hasDuplicate = true;
    return 0;
  }

  // Sort on method existence if only one input has compareDocumentPosition
  var compare = !a.compareDocumentPosition - !b.compareDocumentPosition;
  if (compare) {
    return compare;
  }

  // Calculate position if both inputs belong to the same document
  compare = (a.ownerDocument || a) === (b.ownerDocument || b) ? a.compareDocumentPosition(b) :

  // Otherwise we know they are disconnected
  1;

  // Disconnected nodes
  if (compare & 1 || (!support.sortDetached && b.compareDocumentPosition(a) === compare)) {

    // Choose the first element that is related to our preferred document
    if (a === document || a.ownerDocument === preferredDoc && contains(preferredDoc, a)) {
        return -1;
    }
    if (b === document || b.ownerDocument === preferredDoc && contains(preferredDoc, b)) {
        return 1;
    }

    // Maintain original order
    return sortInput ? (indexOf(sortInput, a) - indexOf(sortInput, b)) : 0;
  }

  return compare & 4 ? -1 : 1;
}
```
这只是对 Sizzle 开个头，任重而道远！下面就会看看 Sizzle 中的 tokens 和 select 函数。

### Tokens 词法分析

其实词法分析是汇编里面提到的词汇，把它用到这里感觉略有不合适，但 Sizzle 中的 tokensize函数干的就是词法分析的活。

上面我们已经讲到了 Sizzle 的用法，实际上就是 jQuery.find 函数，只不过还涉及到 jQuery.fn.find。jQuery.find 函数考虑的很周到，对于处理 #id、.class 和 TagName 的情况，都比较简单，通过一个正则表达式 rquickExpr 将内容给分开，如果浏览器支持 querySelectorAll，那更是最好的。

比较难的要数这种类似于 css 选择器的 selector，div > div.seq h2 ~ p , #id p，如果使用从左向右的查找规则，效率很低，而从右向左，可以提高效率。

本章就来介绍 tokensize 函数，看看它是如何将复杂的 selector 处理成 tokens 的。

我们以 div > div.seq h2 ~ p , #id p 为例，这是一个很简单的 css，逗号 , 将表达式分成两部分。css 中有一些基本的符号，这里有必要强调一下，比如 ,、space、>、+、～：
```
div,p , 表示并列关系，所有 div 元素和 p 元素；
div p 空格表示后代元素，div 元素内所有的 p 元素；
div>p > 子元素且相差只能是一代，父元素为 div 的所有 p 元素；
div+p + 表示紧邻的兄弟元素，前一个兄弟节点为 div 的所有 p 元素；
div~p ~ 表示兄弟元素，所有前面有兄弟元素 div 的所有 p 元素。
除此之外，还有一些 a、input 比较特殊的：

a[target=_blank] 选择所有 target 为 _blank 的所有 a 元素；
a[title=search] 选择所有 title 为 search 的所有 a 元素；
input[type=text] 选择 type 为 text 的所有 input 元素；
p:nth-child(2) 选择其为父元素第二个元素的所有 p 元素；
```
Sizzle 都是支持这些语法的，如果我们把这一步叫做词法分析，那么词法分析的结果是一个什么东西呢？

div > div.seq h2 ~ p , #id p 经过 tokensize(selector) 会返回一个数组，改数组在函数中称为 groups，该数组有两个元素，分别是 tokens0 和 tokens1，代表选择器的两部分。tokens 也是数组，它的每一个元素都是一个 token 对象。

token 对象结构如下所说：

token: {
  value: matched, // 匹配到的字符串
  type: type, //token 类型
  matches: match //去除 value 的正则结果数组
}
Sizzle 中 type 的种类有下面几种：ID、CLASS、TAG、ATTR、PSEUDO、CHILD、bool、needsContext，这几种有几种我也不知道啥意思，child 表示 nth-child、even、odd 这种子选择器。这是针对于 matches 存在的情况，对于 matches 不存在的情况，其 type 就是 value 的 trim() 操作，后面会谈到。

tokensize 函数对 selector 的处理，连空格都不放过，因为空格也属于 type 的一种，而且还很重要，div > div.seq h2 ~ p 的处理结果：
```
tokens: [
  [value:'div', type:'TAG', matches:Array[1]],
  [value:' > ', type:'>'],
  [value:'div', type:'TAG', matches:Array[1]],
  [value:'.seq', type:'CLASS', matches:Array[1]],
  [value:' ', type:' '],
  [value:'h2', type:'TAG', matches:Array[1]],
  [value:' ~ ', type:'~'],
  [value:'p', type:'TAG', matches:Array[1]],
]
```
这个数组会交给 Sizzle 的下一个流程来处理，今天暂不讨论。

tokensize 源码

照旧，先来看一下几个正则表达式。
```
var rcomma = /^[\x20\t\r\n\f]*,[\x20\t\r\n\f]*/;
rcomma.exec('div > div.seq h2 ~ p');//null
rcomma.exec(' ,#id p');//[" ,"]
```
rcomma 这个正则，主要是用来区分 selector 是否到下一个规则，如果到下一个规则，就把之前处理好的 push 到 groups 中。这个正则中 [\x20\t\r\n\f] 是用来匹配类似于 whitespace 的，主体就一个逗号。
```
var rcombinators = /^[\x20\t\r\n\f]*([>+~]|[\x20\t\r\n\f])[\x20\t\r\n\f]*/;
rcombinators.exec(' > div.seq h2 ~ p'); //[" > ", ">"]
rcombinators.exec(' ~ p'); //[" ~ ", "~"]
rcombinators.exec(' h2 ~ p'); //[" ", " "]
```
是不是看来 rcombinators 这个正则表达式，上面 tokens 那个数组的内容就完全可以看得懂了。

其实，如果看 jQuery 的源码，rcomma 和 rcombinators 并不是这样来定义的，而是用下面的方式来定义：
```
var whitespace = "[\\x20\\t\\r\\n\\f]";
var rcomma = new RegExp( "^" + whitespace + "*," + whitespace + "*" ),
  rcombinators = new RegExp( "^" + whitespace + "*([>+~]|" + whitespace + ")" + whitespace + "*" ),
  rtrim = new RegExp( "^" + whitespace + "+|((?:^|[^\\\\])(?:\\\\.)*)" + whitespace + "+$", "g" ),
```
有的时候必须得要佩服 jQuery 中的做法，该简则简，该省则省，每一处代码都是极完美的。

还有两个对象，Expr 和 matchExpr，Expr 是一个非常关键的对象，它涵盖了几乎所有的可能的参数，比较重要的参数比如有：
```
Expr.filter = {
  "TAG": function(){...},
  "CLASS": function(){...},
  "ATTR": function(){...},
  "CHILD": function(){...},
  "ID": function(){...},
  "PSEUDO": function(){...}
}
Expr.preFilter = {
  "ATTR": function(){...},
  "CHILD": function(){...},
  "PSEUDO": function(){...}
}
这个 filter 和 preFilter 是处理 type=TAG 的关键步骤，包括一些类似于 input[type=text] 也是这几个函数处理，也比较复杂，我本人是看迷糊了。还有 matchExpr 正则表达式：

var identifier = "(?:\\\\.|[\\w-]|[^\0-\\xa0])+",
    attributes = "\\[" + whitespace + "*(" + identifier + ")(?:" + whitespace +
    // Operator (capture 2)
    "*([*^$|!~]?=)" + whitespace +
    // "Attribute values must be CSS identifiers [capture 5] or strings [capture 3 or capture 4]"
    "*(?:'((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\"|(" + identifier + "))|)" + whitespace +
    "*\\]",
    pseudos = ":(" + identifier + ")(?:\\((" +
    // To reduce the number of selectors needing tokenize in the preFilter, prefer arguments:
    // 1. quoted (capture 3; capture 4 or capture 5)
    "('((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\")|" +
    // 2. simple (capture 6)
    "((?:\\\\.|[^\\\\()[\\]]|" + attributes + ")*)|" +
    // 3. anything else (capture 2)
    ".*" +
    ")\\)|)",
    booleans = "checked|selected|async|autofocus|autoplay|controls|defer|disabled|hidden|ismap|loop|multiple|open|readonly|required|scoped";
var matchExpr = {
  "ID": new RegExp( "^#(" + identifier + ")" ),
  "CLASS": new RegExp( "^\\.(" + identifier + ")" ),
  "TAG": new RegExp( "^(" + identifier + "|[*])" ),
  "ATTR": new RegExp( "^" + attributes ),
  "PSEUDO": new RegExp( "^" + pseudos ),
  "CHILD": new RegExp( "^:(only|first|last|nth|nth-last)-(child|of-type)(?:\\(" + whitespace +
    "*(even|odd|(([+-]|)(\\d*)n|)" + whitespace + "*(?:([+-]|)" + whitespace +
    "*(\\d+)|))" + whitespace + "*\\)|)", "i" ),
  "bool": new RegExp( "^(?:" + booleans + ")$", "i" ),
  // For use in libraries implementing .is()
  // We use this for POS matching in `select`
  "needsContext": new RegExp( "^" + whitespace + "*[>+~]|:(even|odd|eq|gt|lt|nth|first|last)(?:\\(" +
    whitespace + "*((?:-\\d)?\\d*)" + whitespace + "*\\)|)(?=[^-]|$)", "i" )
}
```
matchExpr 作为正则表达式对象，其 key 的每一项都是一个 type 类型，将 type 匹配到，交给后续函数处理。

tokensize 源码如下：
```
var tokensize = function (selector, parseOnly) {
  var matched, match, tokens, type, soFar, groups, preFilters, cached = tokenCache[selector + " "];
  // tokenCache 表示 token 缓冲，保持已经处理过的 token
  if (cached) {
    return parseOnly ? 0 : cached.slice(0);
  }

  soFar = selector;
  groups = [];
  preFilters = Expr.preFilter;

  while (soFar) {

    // 判断一个分组是否结束
    if (!matched || (match = rcomma.exec(soFar))) {
      if (match) {
        // 从字符串中删除匹配到的 match
        soFar = soFar.slice(match[0].length) || soFar;
      }
      groups.push((tokens = []));
    }

    matched = false;

    // 连接符 rcombinators
    if ((match = rcombinators.exec(soFar))) {
      matched = match.shift();
      tokens.push({
        value: matched,
        type: match[0].replace(rtrim, " ")
      });
      soFar = soFar.slice(matched.length);
    }

    // 过滤，Expr.filter 和 matchExpr 都已经介绍过了
    for (type in Expr.filter) {
      if ((match = matchExpr[type].exec(soFar)) && (!preFilters[type] || (match = preFilters[type](match)))) {
        matched = match.shift();
        // 此时的 match 实际上是 shift() 后的剩余数组
        tokens.push({
          value: matched,
          type: type,
          matches: match
        });
        soFar = soFar.slice(matched.length);
      }
    }

    if (!matched) {
      break;
    }
  }

  // parseOnly 这个参数应该以后会用到
  return parseOnly ? 
    soFar.length : 
    soFar ? 
      Sizzle.error(selector) :
      // 存入缓存
      tokenCache(selector, groups).slice(0);
}
```
不仅数组，字符串也有 slice 操作，而且看源码的话，jQuery 中对字符串的截取，使用的都是 slice 方法。

如果此时 parseOnly 不成立，则返回结果需从 tokenCache 这个函数中来查找：
```
var tokenCache = createCache();
function createCache() {
  var keys = [];

  function cache( key, value ) {
    // Expr.cacheLength = 50
    if ( keys.push( key + " " ) > Expr.cacheLength ) {
      // 删，最不经常使用
      delete cache[ keys.shift() ];
    }
    // 整个结果返回的是 value
    return (cache[ key + " " ] = value);
  }
  return cache;
}
```
可知，返回的结果是 groups，tokensize 就学完了，下章将介绍 tokensize 的后续。


对于一个复杂的 selector，其 tokensize 的过程远比上面介绍的要复杂，上面的例子有点简单（其实也比较复杂了），后面的内容更精彩。


