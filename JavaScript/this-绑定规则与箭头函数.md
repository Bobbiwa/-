---
title: "JavaScript 中 this 的绑定规则是什么？箭头函数有什么不同？"
category: "JavaScript"
tags:
  - this
  - function
  - arrow-function
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Language Specification - EvaluateCall"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluatecall"
    verified_at: 2026-08-10
  - title: "ECMAScript Language Specification - Arrow Function Definitions"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-arrow-function-definitions"
    verified_at: 2026-08-10
  - title: "MDN - this"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this"
    verified_at: 2026-08-10
  - title: "MDN - Arrow function expressions"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions"
    verified_at: 2026-08-10
---

# JavaScript 中 this 的绑定规则是什么？箭头函数有什么不同？

## 问题

普通函数中的 `this` 是如何确定的？请说明默认绑定、隐式绑定、显式绑定和构造调用的区别，以及箭头函数为什么不能用 `call`、`apply` 或 `bind` 改变 `this`。

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p>普通函数的 <code>this</code> 主要由调用方式决定，而不是由函数写在哪里决定。直接调用在严格模式下得到 <code>undefined</code>；通过 <code>obj.fn()</code> 调用时是点号左侧的接收者；<code>call</code>、<code>apply</code> 和 <code>bind</code> 可以显式指定；用 <code>new</code> 调用时则指向新创建的实例。常见优先级可以记为：构造调用高于 <code>bind</code>，<code>bind</code> 高于 <code>call</code>、<code>apply</code>，再高于隐式和默认绑定。箭头函数没有自己的 <code>this</code>，它从定义时所在的外层词法环境取得 <code>this</code>，所以这些 API 和 <code>new</code> 都不能改变它。</p>
<h2>详细原理</h2>
<h3>普通函数看调用点</h3>
<p>对普通函数，先区分以下调用方式：</p>
<ol><li><strong>默认绑定</strong>：<code>fn()</code> 没有接收者。严格模式下 <code>this</code> 是 <code>undefined</code>；对非严格普通函数，这次裸调用传入的 <code>undefined</code> 会被替换为 <code>globalThis</code>。更一般地说，通过 <code>call</code> 或 <code>apply</code> 传入的 <code>null</code>、<code>undefined</code> 也会被替换，原始值则会被装箱。现代代码不应依赖这些非严格模式转换。</li><li><strong>隐式绑定</strong>：<code>obj.fn()</code> 会以 <code>obj</code> 作为这次调用的接收者，因此函数体中的 <code>this === obj</code>。链式成员访问只看最终调用时紧邻函数的基对象，例如 <code>a.b.fn()</code> 的接收者是 <code>a.b</code>。</li><li><strong>显式绑定</strong>：<code>fn.call(value, ...args)</code> 和 <code>fn.apply(value, args)</code> 立即调用函数；<code>fn.bind(value, ...args)</code> 创建一个绑定了 <code>this</code> 和可选前置参数的新函数。</li><li><strong>构造调用</strong>：<code>new Fn()</code> 通过函数的 <code>[[Construct]]</code> 内部方法创建实例。对普通基类构造函数，构造期间的 <code>this</code> 通常是新实例；构造函数显式返回对象时是一个重要例外，详见 <a href="原型链-new-与-class.md">原型链-new-与-class</a>。</li></ol>
<p>所谓“丢失 <code>this</code>”，通常不是 <code>this</code> 被改变了，而是调用表达式改变了：</p>
<pre><code class="language-javascript">&#39;use strict&#39;;&#10;&#10;const user = {&#10;  name: &#39;小杜&#39;,&#10;  getName() {&#10;    return this.name;&#10;  },&#10;};&#10;&#10;console.log(user.getName()); // 小杜：以 user 为接收者&#10;&#10;const getName = user.getName;&#10;// getName() 是直接调用；严格模式下 this 为 undefined，读取 name 会抛错。</code></pre>
<p>对象并不会永久“拥有”某个普通函数。将方法赋给变量、作为回调传递，或者改由另一个对象调用，都可能改变调用点和 <code>this</code>。</p>
<h3>绑定优先级需要带条件理解</h3>
<p>对可构造的普通函数，面试中可用下面的常见顺序判断：</p>
<p><code>new</code> 构造调用 &gt; <code>bind</code> 创建的绑定函数 &gt; <code>call</code> / <code>apply</code> 显式调用 &gt; 成员访问的隐式调用 &gt; 默认绑定。</p>
<ul><li>对绑定函数再次使用 <code>call</code> 或 <code>apply</code>，不能覆盖它已经绑定的 <code>this</code>。</li><li>对绑定函数使用 <code>new</code> 时，绑定的 <code>thisArg</code> 会被忽略，但 <code>bind</code> 预置的参数仍然有效。</li><li>这套口诀不适用于箭头函数，因为箭头函数的 <code>[[ThisMode]]</code> 是 lexical；它根本不执行普通函数的 <code>this</code> 绑定流程。</li><li>代理对象、自定义 <code>Symbol.hasInstance</code>、内置构造器等还可能有额外语义，不能只靠口诀推导所有规范细节。</li></ul>
<h3>箭头函数使用词法 this</h3>
<p>箭头函数没有自己的 <code>this</code> 绑定。执行箭头函数中的 <code>this</code> 表达式时，会沿词法环境向外寻找最近一个提供 <code>this</code> 的环境。因此，“定义时绑定”更准确的说法是“从定义位置的外层环境捕获”，并不是创建箭头函数时把某个对象值复制进函数。</p>
<pre><code class="language-javascript">const account = {&#10;  name: &#39;小杜&#39;,&#10;  makeReader() {&#10;    return () =&gt; this.name;&#10;  },&#10;};&#10;&#10;const readName = account.makeReader();&#10;console.log(readName.call({ name: &#39;其他人&#39; })); // 小杜</code></pre>
<p>这里的箭头函数捕获的是调用 <code>account.makeReader()</code> 时该方法中的 <code>this</code>。如果把 <code>makeReader</code> 也拆出来直接调用，那么箭头函数最终得到的 <code>this</code> 也会随外层调用环境变化。箭头函数还没有自己的 <code>arguments</code>、<code>super</code> 和 <code>new.target</code>，也没有 <code>[[Construct]]</code>，所以不能用 <code>new</code> 调用；这些特征使它适合需要保留外层上下文的回调，但通常不适合充当依赖动态接收者的对象方法。</p>
<h3>严格模式和模块边界</h3>
<ul><li>ECMAScript Module 和 <code>class</code> 的代码天然运行在严格模式下，普通函数直接调用时不会把 <code>this</code> 自动替换成全局对象。</li><li>浏览器经典脚本的顶层 <code>this</code> 通常是 <code>window</code>，浏览器 ES Module 的顶层 <code>this</code> 是 <code>undefined</code>。</li><li>Node.js CommonJS 会用模块包装器执行文件，顶层 <code>this</code> 通常是 <code>module.exports</code>；它不等同于浏览器脚本的顶层规则。判断输出题时必须先确认运行环境、脚本类型和严格模式。</li></ul>
<h2>JavaScript 示例（Node.js 23 或现代浏览器）</h2>
<pre><code class="language-javascript">&#39;use strict&#39;;&#10;&#10;function describe(prefix) {&#10;  return `${prefix}:${this.name}`;&#10;}&#10;&#10;const first = { name: &#39;first&#39;, describe };&#10;const second = { name: &#39;second&#39; };&#10;&#10;console.log(first.describe(&#39;implicit&#39;)); // implicit:first&#10;console.log(describe.call(second, &#39;call&#39;)); // call:second&#10;&#10;const bound = describe.bind(first, &#39;bound&#39;);&#10;console.log(bound.call(second)); // bound:first，call 不能覆盖 bind&#10;&#10;const owner = {&#10;  name: &#39;owner&#39;,&#10;  makeArrow() {&#10;    return () =&gt; this.name;&#10;  },&#10;};&#10;&#10;const readOwner = owner.makeArrow();&#10;console.log(readOwner.call(second)); // owner，call 不能改变箭头函数的 this&#10;&#10;function Person(name) {&#10;  this.name = name;&#10;}&#10;&#10;const ignoredThis = { name: &#39;ignored&#39; };&#10;const BoundPerson = Person.bind(ignoredThis);&#10;const person = new BoundPerson(&#39;instance&#39;);&#10;&#10;console.log(person.name); // instance&#10;console.log(ignoredThis.name); // ignored</code></pre>
<p>这个示例只依赖 ECMAScript 运行时语义，可直接保存后用 <code>node</code> 执行，也可在现代浏览器控制台运行。</p>
<h2>常见追问</h2>
<ol><li><strong>为什么把 <code>obj.method</code> 作为回调传入后经常丢失 <code>this</code>？</strong> 传递的是函数值，之后若以 <code>callback()</code> 调用，就不再保留原来的成员访问接收者。可以按场景使用 <code>bind</code>、包装箭头函数，或让 API 显式传递上下文。</li><li><strong>事件监听器中的 <code>this</code> 一定是什么？</strong> 不一定。浏览器 <code>addEventListener</code> 调用普通函数监听器时，<code>this</code> 通常是 <code>currentTarget</code>；箭头函数仍使用外层 <code>this</code>。React 函数组件事件回调不应套用 DOM 监听器的 <code>this</code> 结论。</li><li><strong><code>bind</code> 返回的函数还能作为构造函数吗？</strong> 如果原函数可构造，绑定函数通常也可被 <code>new</code> 调用；此时绑定的 <code>thisArg</code> 被忽略，预置参数仍参与调用。箭头函数不可构造，绑定后也不会变得可构造。</li></ol>
<h2>易错点</h2>
<ul><li>说“<code>this</code> 永远指向定义函数的对象”。普通函数看调用方式，箭头函数才从外层词法环境取 <code>this</code>。</li><li>认为 <code>obj.fn.call(other)</code> 的 <code>this</code> 仍是 <code>obj</code>。显式绑定在这次普通调用中选择了 <code>other</code>。</li><li>忘记方法拆出后，<code>const fn = obj.fn; fn()</code> 已不再是隐式绑定。</li><li>用 <code>call</code>、<code>apply</code> 或 <code>bind</code> 试图修改箭头函数的 <code>this</code>，或用 <code>new</code> 调用箭头函数。</li><li>不说明严格模式、ES Module、浏览器经典脚本和 Node.js CommonJS 的差异，就直接断言顶层或默认 <code>this</code> 一定是 <code>window</code>。</li><li>把优先级口诀当成完整规范，忽略绑定函数构造调用、构造函数返回对象等边界。</li></ul>
<h2>项目关联</h2>
<p>弱关联（<strong>已实现代码</strong>）：<code>/Users/dubobo/Desktop/my-ipaas/db/repository.ts</code> 使用 <code>Object.prototype.hasOwnProperty.call(next, key)</code>。这里借用 <code>Object.prototype</code> 上的方法，并通过 <code>call</code> 把本次调用的 <code>this</code> 显式设为待检查对象，避免依赖该对象自身是否继承或覆盖了 <code>hasOwnProperty</code>。</p>
<p>这只是语言 API 的局部使用。当前项目没有自然的自定义 <code>this</code> 绑定或自定义继承设计案例，也不能据此判断小杜已经掌握本题。</p>
<h2>相关题目</h2>
<ul><li><a href="原型链-new-与-class.md">原型链-new-与-class</a></li><li><a href="闭包与词法作用域.md">闭包与词法作用域</a></li><li><a href="var-let-const-作用域提升与暂时性死区.md">var-let-const-作用域提升与暂时性死区</a></li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluatecall">ECMAScript Language Specification - EvaluateCall</a>，核验于 2026-08-10。</li><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-arrow-function-definitions">ECMAScript Language Specification - Arrow Function Definitions</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this">MDN - this</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions">MDN - Arrow function expressions</a>，核验于 2026-08-10。</li></ul>
</details>