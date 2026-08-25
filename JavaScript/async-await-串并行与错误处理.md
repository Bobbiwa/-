---
title: "async/await 如何控制串行、并发与错误处理？"
category: "JavaScript"
tags:
  - async-await
  - concurrency
  - error-handling
  - race-condition
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Language Specification - Async Function Definitions"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-async-function-definitions"
    verified_at: 2026-08-10
  - title: "ECMAScript Language Specification - Await"
    url: "https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-await"
    verified_at: 2026-08-10
  - title: "MDN - async function"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function"
    verified_at: 2026-08-10
  - title: "MDN - Promise.all()"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all"
    verified_at: 2026-08-10
---

# async/await 如何控制串行、并发与错误处理？

## 问题

`async` 函数和 `await` 的底层行为是什么？连续 `await` 为什么可能造成串行请求，怎样正确并发？被 await 的 Promise 拒绝时如何处理，又怎样防止旧异步结果覆盖新状态？

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p>调用 <code>async</code> 函数会立即执行到第一个 <code>await</code>，并且总是返回 Promise。<code>await</code> 会暂停当前 async 函数，等被采用的 Promise settled 后，再异步恢复后续代码；即使等待的是普通值，后续也不会在当前同步栈里继续。先 <code>await</code> 第一个请求、再创建第二个请求是串行；先创建多个 Promise，再用 <code>Promise.all</code> 等待才是并发。awaited Promise 拒绝时会像在该位置抛错，可用 <code>try/catch</code> 处理。UI 还要用 request ID、Effect 清理标志或 <code>AbortController</code> 防止过期结果写回。</p>
<h2>详细原理</h2>
<h3>1. <code>async</code> 函数总是返回 Promise</h3>
<p>调用 <code>async</code> 函数时，函数体先同步执行，直到遇到第一个真正暂停执行的 <code>await</code>、正常返回或抛错：</p>
<ul><li><code>return value</code> 会让调用方得到一个以 <code>value</code> fulfilled 的 Promise；</li><li><code>return promise</code> 会让外层 Promise 采用该 Promise 的最终状态；</li><li><code>throw error</code> 会让调用方得到 rejected Promise。</li></ul>
<p>因此调用方不能用同步 <code>try/catch</code> 捕获一个未等待的 async 调用产生的 rejection，必须 <code>await</code>、返回该 Promise，或挂接 rejection handler。</p>
<h3>2. <code>await</code> 暂停的是当前 async 函数，不会阻塞线程</h3>
<p><code>await expression</code> 会取得表达式的值并按 Promise 语义处理。当前 async 函数把控制权交还给调用方；值可用或错误产生后，其 continuation 通过 Promise job 异步恢复。这不会阻塞线程，因此其他工作可以在合适的调度时机运行；但一次 <code>await</code> 不保证跨越到下一个 task，也不保证浏览器获得渲染机会。</p>
<p>即使表达式是普通值或已经 fulfilled 的 Promise，<code>await</code> 后面的代码也会延后执行，而不是继续留在当前同步调用栈。</p>
<h3>3. 串行还是并发，由任务的创建时机决定</h3>
<pre><code class="language-javascript">const a = await requestA();&#10;const b = await requestB();</code></pre>
<p>这里 <code>requestB()</code> 要等 A 完成后才被调用，属于串行。若 B 依赖 A 的结果、接口要求严格顺序或需要控制速率，这正是正确写法。</p>
<pre><code class="language-javascript">const promiseA = requestA();&#10;const promiseB = requestB();&#10;const [a, b] = await Promise.all([promiseA, promiseB]);</code></pre>
<p>这里两个任务先后被创建，但在等待前都已启动，属于并发。JavaScript 单线程不等于所有 I/O 串行；网络和 timer 由宿主处理，Promise 负责表达最终结果。<code>Promise.all</code> 的结果数组保持输入顺序，不按完成顺序排列。</p>
<p>如果允许部分失败，使用 <code>Promise.allSettled</code> 并逐项处理结果。无论 <code>all</code> 还是 <code>allSettled</code>，都不会替你启动任务；真正的并发来自先调用任务函数，得到多个 Promise。</p>
<h3>4. 错误处理要覆盖正确的 Promise 边界</h3>
<p>awaited Promise rejected 时，效果类似在 <code>await</code> 位置抛出 rejection reason，因此同一 async 函数内的 <code>try/catch</code> 可以处理它。<code>catch</code> 返回普通值或最终 fulfilled 的 thenable 时，外层 Promise 会恢复为 fulfilled；再次 <code>throw</code> 或返回最终 rejected 的 thenable 时仍向上传播，返回一直 pending 的 thenable 时外层也会一直 pending。</p>
<p>在 <code>try</code> 内直接 <code>return task()</code> 时，async 函数的返回 Promise 会采用 <code>task()</code> 的状态，但 <code>task()</code> 之后发生的 rejection 不会在当前函数体中重新执行，局部 <code>catch</code> 捕获不到。需要由当前 <code>catch</code> 处理时应写 <code>return await task()</code>。因此“永远不要写 <code>return await</code>”不是可靠规则，应根据错误边界和调用栈需求决定。</p>
<p>对有意不等待的任务，可以写 <code>void task().catch(reportError)</code>；<code>void</code> 只表达“忽略返回值”的意图，本身不会处理 rejection。</p>
<h3>5. 正确的等待仍不能自动解决 UI 竞态</h3>
<p>多个请求可能按 A 先发、B 后发、B 先回、A 后回的顺序完成。单纯使用 <code>await</code> 不会判断哪一个结果仍然有效。常见策略是：</p>
<ul><li>单调递增 request ID，只允许最新请求写状态；</li><li>React Effect 清理时设置本轮 <code>cancelled</code> 标志，忽略旧回调；</li><li>使用 <code>AbortController</code> 尽量取消不再需要的 <code>fetch</code>；</li><li>必须按顺序提交写操作时使用队列、版本号或服务端并发控制。</li></ul>
<p>“忽略旧结果”和“真正取消请求”不同：前者保护本地状态，底层请求仍可能继续消耗资源或产生服务端副作用。</p>
<h2>JavaScript 示例</h2>
<p>运行环境：现代浏览器或 Node.js 23.11.1。timer 的实际耗时可能受宿主调度影响；稳定结论是任务启动位置和 <code>Promise.all</code> 结果顺序。</p>
<pre><code class="language-javascript">function request(label, delayMs) {&#10;  console.log(`start ${label}`);&#10;  return new Promise((resolve) =&gt; {&#10;    setTimeout(() =&gt; {&#10;      console.log(`finish ${label}`);&#10;      resolve(label);&#10;    }, delayMs);&#10;  });&#10;}&#10;&#10;async function serial() {&#10;  const a = await request(&#39;A&#39;, 30);&#10;  const b = await request(&#39;B&#39;, 10);&#10;  return [a, b];&#10;}&#10;&#10;async function concurrent() {&#10;  const promiseA = request(&#39;A&#39;, 30);&#10;  const promiseB = request(&#39;B&#39;, 10);&#10;  return Promise.all([promiseA, promiseB]);&#10;}&#10;&#10;(async () =&gt; {&#10;  console.log(&#39;--- serial ---&#39;);&#10;  console.log(await serial());&#10;&#10;  console.log(&#39;--- concurrent ---&#39;);&#10;  console.log(await concurrent());&#10;})().catch(console.error);</code></pre>
<p>典型输出：</p>
<pre><code class="language-text">--- serial ---&#10;start A&#10;finish A&#10;start B&#10;finish B&#10;[ &#39;A&#39;, &#39;B&#39; ]&#10;--- concurrent ---&#10;start A&#10;start B&#10;finish B&#10;finish A&#10;[ &#39;A&#39;, &#39;B&#39; ]</code></pre>
<p>串行阶段只有 A 完成后才启动 B；并发阶段先启动 A 和 B。即使 B 先完成，<code>Promise.all</code> 仍按输入顺序返回 <code>[&amp;#39;A&amp;#39;, &amp;#39;B&amp;#39;]</code>。</p>
<h2>常见追问</h2>
<ol><li><strong><code>await</code> 会阻塞主线程吗？</strong> 不会。它暂停当前 async 函数并让出控制权；但 <code>await</code> 前后的大量同步计算仍会阻塞主线程。</li><li><strong>为什么在循环里 <code>await</code> 有时是问题、有时是正确的？</strong> 独立任务逐个等待会损失并发性；若后一次依赖前一次结果、需要限流或必须保序，串行才符合业务语义。</li><li><strong><code>Promise.all</code> 一个任务失败后，其他任务会被取消吗？</strong> 不会。返回的组合 Promise 会拒绝，但其他已启动任务继续运行；需要显式取消或等待 <code>allSettled</code> 收集结果。</li></ol>
<h2>易错点</h2>
<ul><li>把“写了多个 <code>await</code>”等同于并发；关键是 Promise 在何时创建。</li><li>认为 <code>await</code> 阻塞整个 JavaScript 线程，忽略它只暂停当前 async 函数。</li><li>在 <code>Array.prototype.forEach</code> 中传 async 回调并期待外层等待；<code>forEach</code> 不消费回调返回的 Promise。串行用 <code>for...of</code>，并发可用 <code>map</code> 后 <code>Promise.all</code>。</li><li>用 <code>void task()</code> 假装处理错误；没有 <code>.catch</code> 的 rejection 仍可能成为未处理 rejection。</li><li>把 <code>Promise.all</code> 的尽快拒绝误解为自动取消其他任务。</li><li>只用 <code>try/catch</code> 处理网络异常，却忘记 <code>fetch</code> 的 HTTP 4xx/5xx 通常需要手动检查 <code>response.ok</code>。</li><li>认为 <code>await</code> 天然避免过期响应；它不提供“最后一次请求获胜”的身份判断。</li></ul>
<h2>项目关联</h2>
<ul><li><strong>已实现：有依赖的串行请求。</strong> <code>my-ipaas/src/hooks/useWorkflowNavigation.ts</code> 的 <code>createWorkflow</code> 依次读取连接器列表、触发器详情、创建流程，再加载新流程；<code>duplicateWorkflow</code> 也先取详情、再创建副本。后一步依赖前一步结果，因此串行是业务要求，不应盲目改成 <code>Promise.all</code>。</li><li><strong>已实现：过期结果保护。</strong> <code>my-ipaas/src/stores/slices/flowSlice.ts</code> 的 <code>loadFlow</code> 使用递增序号，让快速切换 A -&gt; B 时 A 的旧响应不能覆盖 B；<code>src/hooks/useWorkflowNavigation.ts</code> 也用 request ID 保护列表与 pending 状态。<code>src/hooks/useVariableSources.ts</code> 用 Effect 内的 <code>cancelled</code> 标志忽略已失效实例的回调，但不会取消共享请求。</li><li><strong>已实现：并发且允许部分失败。</strong> <code>my-ipaas/src/hooks/useVariableSources.ts</code> 先为多个上游步骤创建任务，再用 <code>Promise.allSettled</code> 收集成功与失败。</li><li><strong>已实现：模拟流程串行推进。</strong> <code>my-ipaas/src/utils/debugSimulator.ts</code> 在循环中 <code>await waitForStep()</code>，因此调试节点按顺序推进；这只是前端调试模拟，不是真实执行引擎。</li><li><strong>仅可作为改进练习：自动保存写入顺序。</strong> <code>my-ipaas/src/stores/flowStore.ts</code> 已用 timer 防抖，但当前未见在途 PUT 的串行队列、版本检查或过期响应保护。可以练习让最新快照最终获胜，但不能宣称该竞态已经解决。</li><li><strong>学习边界：</strong> 上述实现均需小杜能画出请求时序、解释串并行理由并补测试后，才能作为已掌握的项目亮点。</li></ul>
<h2>相关题目</h2>
<ul><li><a href="Promise-链与错误传播.md">Promise-链与错误传播</a></li><li><a href="浏览器事件循环与任务调度.md">浏览器事件循环与任务调度</a></li><li><a href="闭包与词法作用域.md">闭包与词法作用域</a></li><li><a href="../React/useEffect-依赖清理与StrictMode.md">../React/useEffect-依赖清理与StrictMode</a>：Effect 中的异步任务还需处理 cleanup、过期结果和底层取消的边界。</li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-async-function-definitions">ECMAScript Language Specification - Async Function Definitions</a>，核验于 2026-08-10。</li><li><a href="https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-await">ECMAScript Language Specification - Await</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function">MDN - async function</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all">MDN - Promise.all()</a>，核验于 2026-08-10。</li></ul>
</details>