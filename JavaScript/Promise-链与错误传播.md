---
title: "Promise 链中的值和错误如何传播？"
category: "JavaScript"
tags:
  - promise
  - error-handling
  - asynchronous
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Language Specification - Promise Objects"
    url: "https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-objects"
    verified_at: 2026-08-10
  - title: "MDN - Promise.prototype.then()"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then"
    verified_at: 2026-08-10
  - title: "MDN - Promise.prototype.finally()"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally"
    verified_at: 2026-08-10
  - title: "MDN - Promise.allSettled()"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled"
    verified_at: 2026-08-10
---

# Promise 链中的值和错误如何传播？

## 问题

`then`、`catch` 和 `finally` 分别会返回什么？回调中 `return` 普通值、返回另一个 Promise、抛出异常或什么都不返回时，后续 Promise 的状态和值如何变化？为什么有时 `catch` 之后还能继续进入成功分支？

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p><code>then</code>、<code>catch</code> 和 <code>finally</code> 都会返回一个新的 Promise。<code>then</code> 的回调返回普通值时，下一个 Promise 以该值 fulfilled；返回 Promise 或 thenable 时，下一个 Promise 会采用它的最终状态；抛出异常时，下一个 Promise rejected。没有提供对应处理函数时，原来的值或错误会继续向后传。<code>catch</code> 本质上是 <code>then(undefined, onRejected)</code>；它返回普通值或最终 fulfilled 的 thenable 时，错误才算恢复，抛错或返回最终 rejected 的 thenable 时仍走失败分支。<code>finally</code> 通常只做清理并保持原来的值或错误，但若它抛错或返回 rejected Promise，会用新的错误覆盖原结果。</p>
<h2>详细原理</h2>
<h3>1. 每次链式调用都会创建新的 Promise</h3>
<p><code>promise.then(...)</code> 不会修改原 Promise，而是立即返回一个新的、最初处于 pending 状态的 Promise。即使原 Promise 已经 settled，处理函数也不会同步执行，而会由宿主调度 Promise reaction job。</p>
<p>处理函数执行后的结果决定新 Promise：</p>
<table><thead><tr><th>处理函数结果</th><th>新 Promise 的结果</th></tr></thead><tbody><tr><td><code>return 42</code></td><td>fulfilled，值为 <code>42</code></td></tr><tr><td>没有 <code>return</code></td><td>fulfilled，值为 <code>undefined</code></td></tr><tr><td><code>return anotherPromise</code></td><td>采用该 Promise 的最终状态和值或原因</td></tr><tr><td><code>return thenable</code></td><td>按 Promise resolution procedure 处理该 thenable</td></tr><tr><td><code>throw error</code></td><td>rejected，原因为 <code>error</code></td></tr></tbody></table>
<p>不能让一个 Promise 用它自身完成，否则会以 <code>TypeError</code> 拒绝。业务中更常见的问题是漏写 <code>return</code>：外层链不会等待回调中启动的异步任务，其 rejection 也不会进入外层的 <code>catch</code>。</p>
<h3>2. 缺失的处理函数会让结果继续传播</h3>
<p>如果 <code>then</code> 没有传 <code>onFulfilled</code>，fulfilled 值会原样传给后续；如果没有传 <code>onRejected</code>，rejection 原因也会继续向后传播。因此可以把一个 <code>catch</code> 放在链尾，集中处理此前没有被恢复的错误。</p>
<p><code>catch(onRejected)</code> 等价于调用 <code>then(undefined, onRejected)</code>。<code>catch</code> 回调返回普通值或最终 fulfilled 的 thenable 时，其返回的新 Promise 才是 fulfilled，后面的成功分支会执行；若再次 <code>throw</code>，或返回最终 rejected 的 thenable，失败会继续传播。若返回的 thenable 一直 pending，整条后续链也会一直 pending。</p>
<h3>3. <code>finally</code> 默认透明，但自身也可能失败</h3>
<p><code>finally</code> 的回调不接收前一阶段的值或错误，它适合关闭 loading、释放资源等与结果无关的清理。它返回普通值时不会替换原结果；若返回 Promise 或 thenable，链会等待：最终 fulfilled 才透传原结果，一直 pending 则后续链也一直 pending。只有 <code>finally</code> 抛出异常，或返回最终 rejected 的 thenable，链才会改为使用这个新错误。</p>
<h3>4. 组合 Promise 时要先确定失败策略</h3>
<ul><li><code>Promise.all</code> 适合“全部成功才算成功”，任一输入 rejected 时，返回的 Promise 会尽快 rejected；其他已启动任务不会因此自动取消。</li><li><code>Promise.allSettled</code> 会等待所有输入 settled，并为每项保留 <code>fulfilled</code> 或 <code>rejected</code> 结果，适合允许部分成功的场景。</li><li><code>Promise.race</code> 采用第一个 settled 的输入；<code>Promise.any</code> 采用第一个 fulfilled 的输入，全部失败时才以 <code>AggregateError</code> 拒绝。</li></ul>
<p>“尽快拒绝”描述的是组合 Promise 的结果，不等于中止底层网络请求。真正取消请求通常需要 <code>AbortController</code> 等额外机制。</p>
<h3>5. 未处理 rejection 属于宿主行为</h3>
<p>ECMAScript 定义 Promise 与 reaction job，但浏览器如何报告 <code>unhandledrejection</code>、Node.js 如何报告未处理 rejection，属于宿主环境行为。面试时不应把某一版本 Node.js 的警告或退出策略说成 JavaScript 语言的固定规则。</p>
<h2>JavaScript 示例</h2>
<p>运行环境：现代浏览器控制台或 Node.js 23.11.1。</p>
<pre><code class="language-javascript">console.log(&#39;script start&#39;);&#10;&#10;const result = Promise.resolve(2)&#10;  .then((value) =&gt; {&#10;    console.log(&#39;step 1:&#39;, value);&#10;    return value * 3;&#10;  })&#10;  .then((value) =&gt; {&#10;    throw new Error(`bad:${value}`);&#10;  })&#10;  .catch((error) =&gt; {&#10;    console.log(&#39;caught:&#39;, error.message);&#10;    return 10; // 正常返回，表示已经把失败恢复为成功。&#10;  })&#10;  .finally(() =&gt; {&#10;    console.log(&#39;cleanup&#39;);&#10;    return 999; // 普通返回值不会覆盖前面的 10。&#10;  });&#10;&#10;result.then((value) =&gt; console.log(&#39;result:&#39;, value));&#10;console.log(&#39;script end&#39;);</code></pre>
<p>输出：</p>
<pre><code class="language-text">script start&#10;script end&#10;step 1: 2&#10;caught: bad:6&#10;cleanup&#10;result: 10</code></pre>
<p>下面的写法漏掉了 <code>return</code>，外层链不会等待 <code>save()</code>：</p>
<pre><code class="language-javascript">doSomething()&#10;  .then(() =&gt; {&#10;    save(); // 错误：save 返回的 Promise 没有接入当前链。&#10;  })&#10;  .catch(reportError);&#10;&#10;// 应写成：&#10;doSomething()&#10;  .then(() =&gt; save())&#10;  .catch(reportError);</code></pre>
<h2>常见追问</h2>
<ol><li><strong>为什么 <code>catch</code> 后面的 <code>then</code> 通常会执行？</strong> 因为 <code>catch</code> 返回普通值或最终 fulfilled 的 thenable 时，它产生的新 Promise 是 fulfilled；再次抛错或返回最终 rejected 的 thenable 会继续走失败分支，返回始终 pending 的 thenable 则不会进入后续分支。</li><li><strong><code>Promise.all</code> 中一个请求失败后，其他请求会停止吗？</strong> 不会。<code>Promise.all</code> 只会尽快拒绝自己的结果，已经启动的任务仍会继续，除非显式取消。</li><li><strong><code>finally(() =&gt; 123)</code> 和 <code>then(() =&gt; 123)</code> 有什么不同？</strong> 前者通常透传原结果，后者用 <code>123</code> 完成新 Promise；但 <code>finally</code> 自身失败时仍会覆盖原结果。</li></ol>
<h2>易错点</h2>
<ul><li>认为 <code>.then</code> 会修改并返回原 Promise；实际上每次调用都返回新的 Promise。</li><li>在 <code>.then</code> 中启动异步任务却忘记 <code>return</code>，导致执行顺序、错误传播和 loading 状态都脱离当前链。</li><li>认为 <code>catch</code> 只记录错误、不改变状态，或认为回调没有同步抛错就一定恢复成功；实际结果还取决于它返回的 thenable 最终状态。</li><li>认为 <code>finally</code> 的普通返回值会替换业务结果，或忽略了 <code>finally</code> 自己也可能抛错。</li><li>认为 <code>fetch</code> 收到 HTTP 4xx/5xx 会自动 rejected；<code>fetch</code> 通常只在网络失败等情况下拒绝，业务代码仍需检查 <code>response.ok</code>。</li></ul>
<h2>项目关联</h2>
<ul><li><strong>已实现：Promise 请求缓存与失败失效。</strong> <code>my-ipaas/src/hooks/useConnectors.ts</code> 的 <code>cachePromise</code> 会让多个 Hook 复用同一次请求，并在 rejection 后清空缓存；<code>src/utils/connectorActions.ts</code> 和 <code>src/hooks/useVariableSources.ts</code> 也按 <code>connectorId</code> 缓存 Promise，失败时删除对应项，避免永久复用 rejected Promise。</li><li><strong>已实现：允许部分成功。</strong> <code>my-ipaas/src/hooks/useVariableSources.ts</code> 使用 <code>Promise.allSettled</code> 并行加载多个上游连接器的输出协议；某一项失败时仍保留其他 fulfilled 结果，只有没有可用来源且存在失败时才显示整体错误。</li><li><strong>学习边界：</strong> 以上只表示当前代码中存在相应实现，不表示小杜已经能独立解释或重写。可练习画出“缓存命中、首次请求、失败失效、手动重试”四条链路，并为它们补测试。</li></ul>
<h2>相关题目</h2>
<ul><li><a href="浏览器事件循环与任务调度.md">浏览器事件循环与任务调度</a></li><li><a href="async-await-串并行与错误处理.md">async-await-串并行与错误处理</a></li><li><a href="闭包与词法作用域.md">闭包与词法作用域</a></li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-objects">ECMAScript Language Specification - Promise Objects</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then">MDN - Promise.prototype.then()</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally">MDN - Promise.prototype.finally()</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled">MDN - Promise.allSettled()</a>，核验于 2026-08-10。</li></ul>
</details>