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
可知，返回的结果是 groups，tokensize 就学完了。


对于一个复杂的 selector，其 tokensize 的过程远比上面介绍的要复杂，上面的例子有点简单（其实也比较复杂了），后面的内容更精彩。

### select 函数

前面已经介绍了 tokensize 函数的功能，已经生成了一个 tokens 数组，而且对它的组成我们也做了介绍，下面就是介绍对这个 tokens 数组如何处理。

DOM 元素之间的连接关系大概有 > + ~ 几种，包括空格，而 tokens 数组中是 type 是有 tag、attr 和连接符之分的，区分它们 Sizzle 也是有一套规则的，比如上一章我们所讲的 Expr 对象，它真的非常重要：
```
Expr.relative = {
  ">": { dir: "parentNode", first: true },
  " ": { dir: "parentNode" },
  "+": { dir: "previousSibling", first: true },
  "~": { dir: "previousSibling" }
};
```
Expr.relative 标记用来将连接符区分，对其种类又根据目录进行划分。

现在我们再来理一理 tokens 数组，这个数组目前是一个多重数组，现在不考虑逗号的情况，暂定只有一个分支。如果我们使用从右向左的匹配方式的话，div > div.seq h2 ~ p，会先得到 type 为 TAG 的 token，而对于 type 为 ~ 的 token 我们已经可以用 relative 对象来判断，现在来介绍 Expr.find 对象：
```
Expr.find = {};
Expr.find['ID'] = function( id, context ) {
  if ( typeof context.getElementById !== "undefined" && documentIsHTML ) {
    var elem = context.getElementById( id );
    return elem ? [ elem ] : [];
  }
};
Expr.find["CLASS"] = support.getElementsByClassName && function( className, context ) {
  if ( typeof context.getElementsByClassName !== "undefined" && documentIsHTML ) {
    return context.getElementsByClassName( className );
  }
};
Expr.find["TAG"] = function(){...};
```
实际上 jQuery 的源码还考虑到了兼容性，这里以 find["ID"] 介绍：
```
if(support.getById){
  Expr.find['ID'] = function(){...}; // 上面
}else{
  // 兼容 IE 6、7
  Expr.find["ID"] = function( id, context ) {
    if ( typeof context.getElementById !== "undefined" && documentIsHTML ) {
      var node, i, elems,
        elem = context.getElementById( id );

      if ( elem ) {

        // Verify the id attribute
        node = elem.getAttributeNode("id");
        if ( node && node.value === id ) {
          return [ elem ];
        }

        // Fall back on getElementsByName
        elems = context.getElementsByName( id );
        i = 0;
        while ( (elem = elems[i++]) ) {
          node = elem.getAttributeNode("id");
          if ( node && node.value === id ) {
            return [ elem ];
          }
        }
      }

      return [];
    }
  };
}
```
可以对 find 对象进行简化：
```
Expr.find = {
  "ID": document.getElementById,
  "CLASS": document.getElementsByClassName,
  "TAG": document.getElementsByTagName
}
```
以后还会介绍 Expr.filter。

select 源码

源码之前，来看几个正则表达式。
```
var runescape = /\\([\da-f]{1,6}[\x20\t\r\n\f]?|([\x20\t\r\n\f])|.)/gi
//这个正则是用来对转义字符特殊处理，带个反斜杠的 token
runescape.exec('\\ab'); //["\ab", "ab", undefined]
var rsibling = /[+~]/; //匹配 +、~

matchExpr['needsContext'] = /^[\x20\t\r\n\f]*[>+~]|:(even|odd|eq|gt|lt|nth|first|last)(?:\([\x20\t\r\n\f]*((?:-\d)?\d*)[\x20\t\r\n\f]*\)|)(?=[^-]|$)/i
//needsContext 用来匹配不完整的 selector
matchExpr['needsContext'].test(' + p')//true
matchExpr['needsContext'].test(':first-child p')//true
```
//这个不完整，可能是由于抽调 #ID 导致的
而对于 runescape 正则，往往都是配合 replace 来使用：
```
var str = '\\ab';
str.replace(runescape, funescape);
var funescape = function (_, escaped, escapedWhitespace) {
  var high = "0x" + escaped - 0x10000;
  // NaN means non-codepoint
  // Support: Firefox<24
  // Workaround erroneous numeric interpretation of +"0x"
  return high !== high || escapedWhitespace ? escaped : high < 0 ?
  // BMP codepoint
  String.fromCharCode(high + 0x10000) :
  // Supplemental Plane codepoint (surrogate pair)
  String.fromCharCode(high >> 10 | 0xD800, high & 0x3FF | 0xDC00);
}



var select = Sizzle.select = function (selector, context, results, seed) {
  var i, tokens, token, type, find, compiled = typeof selector === "function" && selector,
    match = !seed && tokenize((selector = compiled.selector || selector));

  results = results || [];

  // 长度为 1，即表示没有逗号，Sizzle 尝试对此情况优化
  if (match.length === 1) {
    tokens = match[0] = match[0].slice(0);
    // 第一个 TAG 为一个 ID 选择器，设置快速查找
    if (tokens.length > 2 && (token = tokens[0]).type === "ID" && context.nodeType === 9 && documentIsHTML && Expr.relative[tokens[1].type]) {
      //将新 context 设置成那个 ID
      context = (Expr.find["ID"](token.matches[0].replace(runescape, funescape), context) || [])[0];
      if (!context) {
        // 第一个 ID 都找不到就直接返回
        return results;

      // 此时 selector 为 function，应该有特殊用途
      } else if (compiled) {
        context = context.parentNode;
      }

      selector = selector.slice(tokens.shift().value.length);
    }

    // 在没有 CHILD 的情况，从右向左，仍然是对性能的优化
    i = matchExpr["needsContext"].test(selector) ? 0 : tokens.length;
    while (i--) {
      token = tokens[i];

      // 碰到 +~ 等符号先停止
      if (Expr.relative[(type = token.type)]) {
        break;
      }
      if ((find = Expr.find[type])) {
        // Search, expanding context for leading sibling combinators
        if ((seed = find(
        token.matches[0].replace(runescape, funescape), rsibling.test(tokens[0].type) && testContext(context.parentNode) || context))) {
          // testContext 是判断 getElementsByTagName 是否存在
          // If seed is empty or no tokens remain, we can return early
          tokens.splice(i, 1);
          selector = seed.length && toSelector(tokens);
          //selector 为空，表示到头，直接返回
          if (!selector) {
            push.apply(results, seed);
            return results;
          }
          break;
        }
      }
    }
  }

  // Compile and execute a filtering function if one is not provided
  // Provide `match` to avoid retokenization if we modified the selector above
  (compiled || compile(selector, match))(
  seed, context, !documentIsHTML, results, !context || rsibling.test(selector) && testContext(context.parentNode) || context);
  return results;
}
toSelector 函数是将 tokens 除去已经选择的将剩下的拼接成字符串：

function toSelector(tokens) {
  var i = 0,
    len = tokens.length,
    selector = "";
  for (; i < len; i++) {
    selector += tokens[i].value;
  }
  return selector;
}
```
在最后又多出一个 compile 函数，是 Sizzle 的编译函数，下章讲。

到目前为止，该优化的都已经优化了，selector 和 context，还有 seed，而且如果执行到 compile 函数，这几个变量的状态：

selector 可能已经不上最初那个，经过各种去头去尾；
match 没变，仍是 tokensize 的结果；
seed 事种子集合，所有等待匹配 DOM 的集合；
context 可能已经是头（#ID）；
results 没变。
可能，你也发现了，其实 compile 是一个异步函数 compile()()。

总结

select 大概干了几件事，

将 tokenize 处理 selector 的结果赋给 match，所以 match 实为 tokens 数组；
在长度为 1，且第一个 token 为 ID 的情况下，对 context 进行优化，把 ID 匹配到的元素赋给 context；
若不含 needsContext 正则，则生成一个 seed 集合，为所有的最右 DOM 集合；
最后事 compile 函数，参数真多...

compile

讲了这么久的 Sizzle，总感觉差了那么一口气，对于一个 selector，我们把它生成 tokens，进行优化，优化的步骤包括去头和生成 seed 集合。对于这些种子集合，我们知道最后的匹配结果是来自于集合中的一部分，似乎接下来的任务也已经明确：对种子进行过滤（或者称其为匹配）。

匹配的过程其实很简单，就是对 DOM 元素进行判断，而且弱是那种一代关系（>）或临近兄弟关系（+），不满足，就结束，若为后代关系（ ）或者兄弟关系（~），会进行多次判断，要么找到一个正确的，要么结束，不过仍需要考虑回溯问题。

比如div > div.seq h2 ~ p，已经对应的把它们划分成 tokens，如果每个 seed 都走一遍流程显然太麻烦。一种比较合理的方法就是对应每个可判断的 token 生成一个闭包函数，统一进行查找。

Expr.filter 是用来生成匹配函数的，它大概长这样：

Expr.filter = {
  "ID": function(id){...},
  "TAG": function(nodeNameSelector){...},
  "CLASS": function(className){...},
  "ATTR": function(name, operator, check){...},
  "CHILD": function(type, what, argument, first, last){...},
  "PSEUDO": function(pseudo, argument){...}
}
看两个例子，一切都懂了：

Expr.filter["ID"] = function( id ) {
  var attrId = id.replace( runescape, funescape );
  //这里返回一个函数
  return function( elem ) {
    return elem.getAttribute("id") === attrId;
  };
};
对 tokens 分析，让发现其 type 为 ID，则把其 id 保留，并返回一个检查函数，参数为 elem，用于判断该 DOM 的 id 是否与 tokens 的 id 一致。这种做法的好处是，编译一次，执行多次。

那么，是这样的吗？我们再来看看其他例子：

Expr.filter["TAG"] = function( nodeNameSelector ) {
  var nodeName = nodeNameSelector.replace( runescape, funescape ).toLowerCase();
  return nodeNameSelector === "*" ?
    //返回一个函数
    function() { return true; } :
    // 参数为 elem
    function( elem ) {
      return elem.nodeName && elem.nodeName.toLowerCase() === nodeName;
    };
};

Expr.filter["ATTR"] = function( name, operator, check ) {
  // 返回一个函数
  return function( elem ) {
    var result = Sizzle.attr( elem, name );

    if ( result == null ) {
      return operator === "!=";
    }
    if ( !operator ) {
      return true;
    }

    result += "";

    return operator === "=" ? result === check :
      operator === "!=" ? result !== check :
      operator === "^=" ? check && result.indexOf( check ) === 0 :
      operator === "*=" ? check && result.indexOf( check ) > -1 :
      operator === "$=" ? check && result.slice( -check.length ) === check :
      operator === "~=" ? ( " " + result.replace( rwhitespace, " " ) + " " ).indexOf( check ) > -1 :
      operator === "|=" ? result === check || result.slice( 0, check.length + 1 ) === check + "-" :
      false;
  };
},
最后的返回结果：

input[type=button] 属性 type 类型为 button；
input[type!=button] 属性 type 类型不等于 button；
input[name^=pre] 属性 name 以 pre 开头；
input[name*=new] 属性 name 中包含 new；
input[name$=ed] 属性 name 以 ed 结尾；
input[name=~=new] 属性 name 有用空格分离的 new；
input[name|=zh] 属性 name 要么等于 zh，要么以 zh 开头且后面有关连字符 -。
所以对于一个 token，即生成了一个闭包函数，该函数接收的参数为一个 DOM，用来判断该 DOM 元素是否是符合 token 的约束条件，比如 id 或 className 等等。如果将多个 token （即 tokens）都这么来处理，会得到一个专门用来判断的函数数组，这样子对于 seed 中的每一个元素，就可以用这个函数数组对其父元素或兄弟节点挨个判断，效率大大提升，即所谓的编译一次，多次使用。

### compile 源码

直接贴上 compile 函数代码，这里会有 matcherFromTokens 和 matcherFromGroupMatchers 这两个函数，也一并介绍了。
```
var compile = function(selector, match) {
  var i,setMatchers = [],elementMatchers = [],
    cached = compilerCache[selector + " "];
  // 判断有没有缓存，好像每个函数都会判断
  if (!cached) {
    if (!match) {
      // 判断 match 是否生成 tokens
      match = tokenize(selector);
    }
    i = match.length;
    while (i--) {
      // 这里将 tokens 交给了这个函数
      cached = matcherFromTokens(match[i]);
      if (cached[expando]) {
        setMatchers.push(cached);
      } else {
        elementMatchers.push(cached);
      }
    }

    // 放到缓存
    cached = compilerCache(
      selector,
      // 这个函数生成最终的匹配器
      matcherFromGroupMatchers(elementMatchers, setMatchers)
    );

    // Save selector and tokenization
    cached.selector = selector;
  }
  return cached;
};
```
编译 compile 函数貌似很简单，来看 matcherFromTokens：

```
function matcherFromTokens(tokens) {
  var checkContext,matcher,j,len = tokens.length,leadingRelative = Expr.relative[tokens[0].type],
    implicitRelative = leadingRelative || Expr.relative[" "],
    i = leadingRelative ? 1 : 0,
    // 确保元素都能找到
    // addCombinator 就是对 Expr.relative 进行判断
    /*
      Expr.relative = {
        ">": { dir: "parentNode", first: true },
        " ": { dir: "parentNode" },
        "+": { dir: "previousSibling", first: true },
        "~": { dir: "previousSibling" }
      };
     */
    matchContext = addCombinator(
      function(elem) {
        return elem === checkContext;
      },implicitRelative,true),
    matchAnyContext = addCombinator(
      function(elem) {
        return indexOf(checkContext, elem) > -1;
      },implicitRelative,true),
    matchers = [
      function(elem, context, xml) {
        var ret = !leadingRelative && (xml || context !== outermostContext) || ((checkContext = context).nodeType ? matchContext(elem, context, xml) : matchAnyContext(elem, context, xml));
        // Avoid hanging onto element (issue #299)
        checkContext = null;
        return ret;
      }
    ];

  for (; i < len; i++) {
    // 处理 "空 > ~ +"
    if (matcher = Expr.relative[tokens[i].type]) {
      matchers = [addCombinator(elementMatcher(matchers), matcher)];
    } else {
      // 处理 ATTR CHILD CLASS ID PSEUDO TAG，filter 函数在这里
      matcher = Expr.filter[tokens[i].type].apply(null, tokens[i].matches);

      // Return special upon seeing a positional matcher
      // 伪类会把selector分两部分
      if (matcher[expando]) {
        // Find the next relative operator (if any) for proper handling
        j = ++i;
        for (; j < len; j++) {
          if (Expr.relative[tokens[j].type]) {
            break;
          }
        }
        return setMatcher(
          i > 1 && elementMatcher(matchers),
          i > 1 && toSelector(
              // If the preceding token was a descendant combinator, insert an implicit any-element `*`
              tokens
                .slice(0, i - 1)
                .concat({value: tokens[i - 2].type === " " ? "*" : ""})
            ).replace(rtrim, "$1"),
          matcher,
          i < j && matcherFromTokens(tokens.slice(i, j)),
          j < len && matcherFromTokens(tokens = tokens.slice(j)),
          j < len && toSelector(tokens)
        );
      }
      matchers.push(matcher);
    }
  }

  return elementMatcher(matchers);
}
```
其中 addCombinator 函数用于生成 curry 函数，来解决 Expr.relative 情况：
```
function addCombinator(matcher, combinator, base) {
  var dir = combinator.dir, skip = combinator.next, key = skip || dir, checkNonElements = base && key === "parentNode", doneName = done++;

  return combinator.first ? // Check against closest ancestor/preceding element
    function(elem, context, xml) {
      while (elem = elem[dir]) {
        if (elem.nodeType === 1 || checkNonElements) {
          return matcher(elem, context, xml);
        }
      }
      return false;
    } : // Check against all ancestor/preceding elements
    function(elem, context, xml) {
      var oldCache, uniqueCache, outerCache, newCache = [dirruns, doneName];

      // We can't set arbitrary data on XML nodes, so they don't benefit from combinator caching
      if (xml) {
        while (elem = elem[dir]) {
          if (elem.nodeType === 1 || checkNonElements) {
            if (matcher(elem, context, xml)) {
              return true;
            }
          }
        }
      } else {
        while (elem = elem[dir]) {
          if (elem.nodeType === 1 || checkNonElements) {
            outerCache = elem[expando] || (elem[expando] = {});

            // Support: IE <9 only
            // Defend against cloned attroperties (jQuery gh-1709)
            uniqueCache = outerCache[elem.uniqueID] || (outerCache[elem.uniqueID] = {});

            if (skip && skip === elem.nodeName.toLowerCase()) {
              elem = elem[dir] || elem;
            } else if ((oldCache = uniqueCache[key]) && oldCache[0] === dirruns && oldCache[1] === doneName) {
              // Assign to newCache so results back-propagate to previous elements
              return newCache[2] = oldCache[2];
            } else {
              // Reuse newcache so results back-propagate to previous elements
              uniqueCache[key] = newCache;

              // A match means we're done; a fail means we have to keep checking
              if (newCache[2] = matcher(elem, context, xml)) {
                return true;
              }
            }
          }
        }
      }
      return false;
    };
}
```
其中 elementMatcher 函数用于生成匹配器：
```
function elementMatcher(matchers) {
  return matchers.length > 1 ? function(elem, context, xml) {
      var i = matchers.length;
      while (i--) {
        if (!matchers[i](elem, context, xml)) {
          return false;
        }
      }
      return true;
    } : matchers[0];
}
matcherFromGroupMatchers 如下：

function matcherFromGroupMatchers(elementMatchers, setMatchers) {
  var bySet = setMatchers.length > 0,
    byElement = elementMatchers.length > 0,
    superMatcher = function(seed, context, xml, results, outermost) {
      var elem,j,matcher,matchedCount = 0,i = "0",unmatched = seed && [],setMatched = [],
        contextBackup = outermostContext,
        // We must always have either seed elements or outermost context
        elems = seed || byElement && Expr.find["TAG"]("*", outermost),
        // Use integer dirruns iff this is the outermost matcher
        dirrunsUnique = dirruns += contextBackup == null ? 1 : Math.random() || 0.1,len = elems.length;

      if (outermost) {
        outermostContext = context === document || context || outermost;
      }

      // Add elements passing elementMatchers directly to results
      // Support: IE<9, Safari
      // Tolerate NodeList properties (IE: "length"; Safari: <number>) matching elements by id
      for (; i !== len && (elem = elems[i]) != null; i++) {
        if (byElement && elem) {
          j = 0;
          if (!context && elem.ownerDocument !== document) {
            setDocument(elem);
            xml = !documentIsHTML;
          }
          while (matcher = elementMatchers[j++]) {
            if (matcher(elem, context || document, xml)) {
              results.push(elem);
              break;
            }
          }
          if (outermost) {
            dirruns = dirrunsUnique;
          }
        }

        // Track unmatched elements for set filters
        if (bySet) {
          // They will have gone through all possible matchers
          if (elem = !matcher && elem) {
            matchedCount--;
          }

          // Lengthen the array for every element, matched or not
          if (seed) {
            unmatched.push(elem);
          }
        }
      }

      // `i` is now the count of elements visited above, and adding it to `matchedCount`
      // makes the latter nonnegative.
      matchedCount += i;

      // Apply set filters to unmatched elements
      // NOTE: This can be skipped if there are no unmatched elements (i.e., `matchedCount`
      // equals `i`), unless we didn't visit _any_ elements in the above loop because we have
      // no element matchers and no seed.
      // Incrementing an initially-string "0" `i` allows `i` to remain a string only in that
      // case, which will result in a "00" `matchedCount` that differs from `i` but is also
      // numerically zero.
      if (bySet && i !== matchedCount) {
        j = 0;
        while (matcher = setMatchers[j++]) {
          matcher(unmatched, setMatched, context, xml);
        }

        if (seed) {
          // Reintegrate element matches to eliminate the need for sorting
          if (matchedCount > 0) {
            while (i--) {
              if (!(unmatched[i] || setMatched[i])) {
                setMatched[i] = pop.call(results);
              }
            }
          }

          // Discard index placeholder values to get only actual matches
          setMatched = condense(setMatched);
        }

        // Add matches to results
        push.apply(results, setMatched);

        // Seedless set matches succeeding multiple successful matchers stipulate sorting
        if (outermost && !seed && setMatched.length > 0 && matchedCount + setMatchers.length > 1) {
          Sizzle.uniqueSort(results);
        }
      }

      // Override manipulation of globals by nested matchers
      if (outermost) {
        dirruns = dirrunsUnique;
        outermostContext = contextBackup;
      }

      return unmatched;
    };

  return bySet ? markFunction(superMatcher) : superMatcher;
}
```
这个过程太复杂了，请原谅我无法耐心的看完。。。

先留名，以后分析。。。

到此，其实已经可以结束了，但我本着负责的心态，我们再来理一下 Sizzle 整个过程。

Sizzle 虽然独立出去，单独成一个项目，不过在 jQuery 中的代表就是 jQuery.find 函数，这两个函数其实就是同一个，完全等价的。然后介绍 tokensize 函数，这个函数的被称为词法分析，作用就是将 selector 划分成 tokens 数组，数组每个元素都有 value 和 type 值。然后是 select 函数，这个函数的功能起着优化作用，去头去尾，并 Expr.find 函数生成 seed 种子数组。

后面的介绍就马马虎虎了，我本身看的也不少很懂。compile 函数进行预编译，就是对去掉 seed 后剩下的 selector 生成闭包函数，又把闭包函数生成一个大的 superMatcher 函数，这个时候就可用这个 superMatcher(seed) 来处理 seed 并得到最终的结果。

那么 superMatcher 是什么？

superMatcher

前面就已经说过，这才是 compile()() 函数的正确使用方法，而 compile() 的返回值即 superMatcher，无论是介绍 matcherFromTokens 还说介绍 matcherFromGroupMatchers，其结果都是为了生成超级匹配，然后处理 seed，这是一个考验的时刻，只有经得住筛选才会留下来。

总结

第一步

div > p + div.aaron input[type="checkbox"]
从最右边先通过 Expr.find 获得 seed 数组，在这里的 input 是 TAG，所以通过 getElementsByTagName() 函数。

第二步

重组 selector，此时除去 input 之后的 selector：

div > p + div.aaron [type="checkbox"]
第三步

此时通过 Expr.relative 将 tokens 根据关系分成紧密关系和非紧密关系，比如 [">", "+"] 就是紧密关系，其 first = true。而对于 [" ", "~"] 就是非紧密关系。紧密关系在筛选时可以快速判断。

matcherFromTokens 根据关系编译闭包函数，为四组：

div > 
p + 
div.aaron 
input[type="checkbox"]
编译函数主要借助 Expr.filter 和 Expr.relative。

第四步

将所有的编译闭包函数放到一起，生成 superMatcher 函数。
```
function( elem, context, xml ) {
    var i = matchers.length;
    while ( i-- ) {
        if ( !matchers[i]( elem, context, xml ) ) {
            return false;
        }
    }
    return true;
}
```
从右向左，处理 seed 集合，如果有一个不匹配，则返回 false。如果成功匹配，则说明该 seed 元素是符合筛选条件的，返回给 results。

