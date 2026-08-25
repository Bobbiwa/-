---
title: "var、let、const 在作用域、提升和暂时性死区上有什么区别？"
category: "JavaScript"
tags:
  - scope
  - hoisting
  - temporal-dead-zone
  - variable-declaration
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Variable Statement"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-variable-statement"
    verified_at: 2026-08-10
  - title: "ECMAScript Let and Const Declarations"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-let-and-const-declarations"
    verified_at: 2026-08-10
  - title: "MDN - var"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var"
    verified_at: 2026-08-10
  - title: "MDN - let"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let"
    verified_at: 2026-08-10
  - title: "MDN - const"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const"
    verified_at: 2026-08-10
---

# var、let、const 在作用域、提升和暂时性死区上有什么区别？

## 问题

请比较 `var`、`let`、`const` 的作用域、声明初始化、重复声明和重新赋值规则。它们是否都会“提升”？什么是暂时性死区（Temporal Dead Zone，TDZ），为什么 `typeof` 有时也会抛出 `ReferenceError`？

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p><code>var</code> 是函数作用域或全局作用域，不受普通代码块限制；进入对应执行上下文时，它的绑定就会被创建并初始化为 <code>undefined</code>，所以声明前读取通常得到 <code>undefined</code>。<code>let</code> 和 <code>const</code> 是块级作用域，进入作用域时绑定已经存在，但执行到声明完成初始化前处于 TDZ，读取会抛 <code>ReferenceError</code>。<code>let</code> 可以重新赋值，<code>const</code> 不可以重新赋值且声明时通常必须初始化，但 <code>const</code> 对象的属性仍可修改。与其只背“谁会提升”，更准确的是说明三者绑定何时创建、何时初始化。</p>
<h2>详细原理</h2>
<h3>1. 作用域边界</h3>
<ul><li><code>var</code> 的声明作用域是包含它的函数、静态初始化块或脚本全局环境，普通 <code>{}</code>、<code>if</code> 和 <code>for</code> 代码块不会为它建立块级绑定。</li><li><code>let</code>、<code>const</code> 属于 lexical declaration，以块、函数体、模块等词法结构为边界。</li><li>在 ES module 中，顶层声明都是模块作用域。浏览器经典 <code>&lt;script&gt;</code> 的顶层 <code>var</code> 与顶层 <code>let</code>/<code>const</code> 还存在全局对象属性差异，不能把该结论无条件套到 ES module 或 Node.js 模块包装环境。</li></ul>
<h3>2. “提升”背后的创建和初始化</h3>
<p>“提升”是常用教学术语，不是把源码声明真的移动到顶部。更准确的执行模型是：进入作用域时先进行声明实例化，再逐句执行代码。</p>
<ul><li><code>var</code> 绑定会提前创建并初始化为 <code>undefined</code>。所以赋值语句执行前能读取该绑定，只是还没有得到后面的赋值。</li><li><code>let</code>、<code>const</code> 绑定也会在代码执行到声明前参与当前词法作用域和遮蔽，但初始状态是未初始化。此时从作用域开始位置到初始化完成之间就是 TDZ。</li><li>执行到 <code>let</code> 声明时绑定被初始化；执行到 <code>const</code> 声明时必须以初始化值建立不可重新赋值的绑定。<code>for-in</code>、<code>for-of</code> 头部是语法上的特殊初始化位置。</li></ul>
<p>因此，回答“<code>let</code> 和 <code>const</code> 会不会提升”时不宜只答会或不会：它们的绑定在声明前已经存在并产生遮蔽，但不能在初始化前访问。</p>
<h3>3. TDZ 与 <code>typeof</code></h3>
<p>TDZ 的起点是进入声明所在的词法作用域，而不是代码文本中的声明行。读取、写入或使用 <code>typeof</code> 访问同一作用域内尚未初始化的 lexical binding 都会抛 <code>ReferenceError</code>。</p>
<p><code>typeof completelyUndeclared</code> 在找不到任何绑定时返回字符串 <code>&quot;undefined&quot;</code>；但 <code>typeof value</code> 若解析到了 TDZ 中的 <code>value</code> 绑定，仍然会抛错。这两个场景不能混为一谈。</p>
<h3>4. 重复声明、重新赋值与循环绑定</h3>
<ul><li>同一作用域中的多个 <code>var</code> 声明通常允许共存；<code>var</code> 绑定也可以重新赋值。</li><li>同一词法作用域不能用 <code>let</code>/<code>const</code> 重复声明同名绑定，也不能与会发生声明冲突的同名声明并存；这类问题通常是解析阶段的 <code>SyntaxError</code>。</li><li><code>let</code> 可以重新赋值；<code>const</code> 不能重新赋值，但若其值是对象，对象属性仍可能被修改。</li><li><code>for</code> 循环头中的 <code>let</code> 会为每次迭代建立对应绑定，回调能读到各自那一轮的值；<code>var</code> 通常由所有回调共享同一个函数级绑定。</li></ul>
<p>工程代码默认优先 <code>const</code>，确实需要重新赋值时使用 <code>let</code>；这是一条可读性建议，不是语言规范的强制要求。现代业务代码一般避免 <code>var</code>，但维护旧代码时仍需理解它的语义。</p>
<h2>JavaScript 示例（Node.js 23）</h2>
<pre><code class="language-javascript">&quot;use strict&quot;;&#10;&#10;function declarationDemo() {&#10;  console.log(beforeVar); // undefined&#10;  var beforeVar = &quot;var initialized&quot;;&#10;&#10;  try {&#10;    console.log(beforeLet);&#10;  } catch (error) {&#10;    console.log(error.name); // ReferenceError&#10;  }&#10;  let beforeLet = &quot;let initialized&quot;;&#10;&#10;  if (true) {&#10;    var functionScoped = &quot;visible outside block&quot;;&#10;    const blockScoped = &quot;only inside block&quot;;&#10;    console.log(blockScoped); // only inside block&#10;  }&#10;  console.log(functionScoped); // visible outside block&#10;&#10;  try {&#10;    console.log(blockScoped);&#10;  } catch (error) {&#10;    console.log(error.name); // ReferenceError&#10;  }&#10;}&#10;&#10;declarationDemo();&#10;&#10;const callbacksWithVar = [];&#10;for (var i = 0; i &lt; 3; i += 1) {&#10;  callbacksWithVar.push(() =&gt; i);&#10;}&#10;console.log(callbacksWithVar.map((callback) =&gt; callback())); // [3, 3, 3]&#10;&#10;const callbacksWithLet = [];&#10;for (let j = 0; j &lt; 3; j += 1) {&#10;  callbacksWithLet.push(() =&gt; j);&#10;}&#10;console.log(callbacksWithLet.map((callback) =&gt; callback())); // [0, 1, 2]&#10;&#10;let outer = &quot;outside&quot;;&#10;{&#10;  try {&#10;    console.log(typeof outer);&#10;  } catch (error) {&#10;    console.log(error.name); // ReferenceError：解析到下面的 TDZ 绑定&#10;  }&#10;  let outer = &quot;inside&quot;;&#10;  console.log(outer); // inside&#10;}&#10;console.log(typeof neverDeclared); // undefined：没有找到任何同名绑定</code></pre>
<h2>常见追问</h2>
<ol><li><strong><code>let</code> 和 <code>const</code> 到底算不算提升？</strong> 如果“提升”指声明前绑定已经影响名字解析和遮蔽，可以说有类似提升的行为；但它们不会像 <code>var</code> 一样提前初始化为 <code>undefined</code>，初始化前处于 TDZ。面试时直接解释创建与初始化阶段最准确。</li><li><strong>为什么 <code>typeof</code> TDZ 变量会报错，而未声明变量不会？</strong> 前者已经解析到一个存在但未初始化的 lexical binding，访问被禁止；后者根本找不到绑定，<code>typeof</code> 对这一情况有返回 <code>&quot;undefined&quot;</code> 的特殊规则。</li><li><strong>浏览器中顶层声明都会成为 <code>window</code> 属性吗？</strong> 不是。经典 script 中符合条件的顶层 <code>var</code> 会建立全局对象属性，而顶层 <code>let</code>/<code>const</code> 不会；ES module、Node.js 模块及其他宿主环境不能直接套用这一说法。</li></ol>
<h2>易错点</h2>
<ul><li>把“提升”理解为 JavaScript 引擎改写并移动了源码，而不解释声明实例化。</li><li>说 <code>let</code>/<code>const</code> 在声明前“不存在”；它们已经参与名字解析，只是尚未初始化。</li><li>认为 TDZ 从声明行开始；它实际覆盖进入该词法作用域到初始化前的区间。</li><li>认为 <code>typeof</code> 访问任何未初始化名字都安全，忽略 TDZ 会触发 <code>ReferenceError</code>。</li><li>把 <code>const</code> 误解成深层不可变，或者用 <code>var</code>/<code>let</code>/<code>const</code> 的差异解释对象属性是否可变。</li><li>忽略运行环境，宣称所有顶层 <code>var</code> 都是 <code>globalThis</code> 属性。</li></ul>
<h2>项目关联</h2>
<p><strong>状态：仅可作为基础练习。</strong> 当前没有必要把变量声明规则包装成 <code>my-ipaas</code> 的项目亮点。可以在本地写循环回调、块级遮蔽和 TDZ 小实验，并在走读项目时解释为何默认使用 <code>const</code>、何时必须使用 <code>let</code>。项目采用这些声明并不证明小杜已经掌握其执行语义，需能脱离原文件复现上述行为后再标记掌握。</p>
<h2>相关题目</h2>
<ul><li><a href="原始值与对象的赋值和传参.md">原始值与对象的赋值和传参</a>：<code>const</code> 限制的是绑定，不能据此判断对象是否可变。</li><li><a href="闭包与词法作用域.md">闭包与词法作用域</a>：循环回调的结果取决于闭包所引用的绑定，以及 <code>let</code> 的逐次迭代环境。</li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-variable-statement">ECMAScript Variable Statement</a>，核验于 2026-08-10。</li><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-let-and-const-declarations">ECMAScript Let and Const Declarations</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var">MDN - var</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let">MDN - let</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const">MDN - const</a>，核验于 2026-08-10。</li></ul>
</details>