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

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> 调用 `async` 函数会立即执行到第一个 `await`，并且总是返回 Promise。`await` 会暂停当前 async 函数，等被采用的 Promise settled 后，再异步恢复后续代码；即使等待的是普通值，后续也不会在当前同步栈里继续。先 `await` 第一个请求、再创建第二个请求是串行；先创建多个 Promise，再用 `Promise.all` 等待才是并发。awaited Promise 拒绝时会像在该位置抛错，可用 `try/catch` 处理。UI 还要用 request ID、Effect 清理标志或 `AbortController` 防止过期结果写回。
>
> ## 详细原理
>
> ### 1. `async` 函数总是返回 Promise
>
> 调用 `async` 函数时，函数体先同步执行，直到遇到第一个真正暂停执行的 `await`、正常返回或抛错：
>
> - `return value` 会让调用方得到一个以 `value` fulfilled 的 Promise；
> - `return promise` 会让外层 Promise 采用该 Promise 的最终状态；
> - `throw error` 会让调用方得到 rejected Promise。
>
> 因此调用方不能用同步 `try/catch` 捕获一个未等待的 async 调用产生的 rejection，必须 `await`、返回该 Promise，或挂接 rejection handler。
>
> ### 2. `await` 暂停的是当前 async 函数，不会阻塞线程
>
> `await expression` 会取得表达式的值并按 Promise 语义处理。当前 async 函数把控制权交还给调用方；值可用或错误产生后，其 continuation 通过 Promise job 异步恢复。这不会阻塞线程，因此其他工作可以在合适的调度时机运行；但一次 `await` 不保证跨越到下一个 task，也不保证浏览器获得渲染机会。
>
> 即使表达式是普通值或已经 fulfilled 的 Promise，`await` 后面的代码也会延后执行，而不是继续留在当前同步调用栈。
>
> ### 3. 串行还是并发，由任务的创建时机决定
>
> ```javascript
> const a = await requestA();
> const b = await requestB();
> ```
>
> 这里 `requestB()` 要等 A 完成后才被调用，属于串行。若 B 依赖 A 的结果、接口要求严格顺序或需要控制速率，这正是正确写法。
>
> ```javascript
> const promiseA = requestA();
> const promiseB = requestB();
> const [a, b] = await Promise.all([promiseA, promiseB]);
> ```
>
> 这里两个任务先后被创建，但在等待前都已启动，属于并发。JavaScript 单线程不等于所有 I/O 串行；网络和 timer 由宿主处理，Promise 负责表达最终结果。`Promise.all` 的结果数组保持输入顺序，不按完成顺序排列。
>
> 如果允许部分失败，使用 `Promise.allSettled` 并逐项处理结果。无论 `all` 还是 `allSettled`，都不会替你启动任务；真正的并发来自先调用任务函数，得到多个 Promise。
>
> ### 4. 错误处理要覆盖正确的 Promise 边界
>
> awaited Promise rejected 时，效果类似在 `await` 位置抛出 rejection reason，因此同一 async 函数内的 `try/catch` 可以处理它。`catch` 返回普通值或最终 fulfilled 的 thenable 时，外层 Promise 会恢复为 fulfilled；再次 `throw` 或返回最终 rejected 的 thenable 时仍向上传播，返回一直 pending 的 thenable 时外层也会一直 pending。
>
> 在 `try` 内直接 `return task()` 时，async 函数的返回 Promise 会采用 `task()` 的状态，但 `task()` 之后发生的 rejection 不会在当前函数体中重新执行，局部 `catch` 捕获不到。需要由当前 `catch` 处理时应写 `return await task()`。因此“永远不要写 `return await`”不是可靠规则，应根据错误边界和调用栈需求决定。
>
> 对有意不等待的任务，可以写 `void task().catch(reportError)`；`void` 只表达“忽略返回值”的意图，本身不会处理 rejection。
>
> ### 5. 正确的等待仍不能自动解决 UI 竞态
>
> 多个请求可能按 A 先发、B 后发、B 先回、A 后回的顺序完成。单纯使用 `await` 不会判断哪一个结果仍然有效。常见策略是：
>
> - 单调递增 request ID，只允许最新请求写状态；
> - React Effect 清理时设置本轮 `cancelled` 标志，忽略旧回调；
> - 使用 `AbortController` 尽量取消不再需要的 `fetch`；
> - 必须按顺序提交写操作时使用队列、版本号或服务端并发控制。
>
> “忽略旧结果”和“真正取消请求”不同：前者保护本地状态，底层请求仍可能继续消耗资源或产生服务端副作用。
>
> ## JavaScript 示例
>
> 运行环境：现代浏览器或 Node.js 23.11.1。timer 的实际耗时可能受宿主调度影响；稳定结论是任务启动位置和 `Promise.all` 结果顺序。
>
> ```javascript
> function request(label, delayMs) {
>   console.log(`start ${label}`);
>   return new Promise((resolve) => {
>     setTimeout(() => {
>       console.log(`finish ${label}`);
>       resolve(label);
>     }, delayMs);
>   });
> }
>
> async function serial() {
>   const a = await request('A', 30);
>   const b = await request('B', 10);
>   return [a, b];
> }
>
> async function concurrent() {
>   const promiseA = request('A', 30);
>   const promiseB = request('B', 10);
>   return Promise.all([promiseA, promiseB]);
> }
>
> (async () => {
>   console.log('--- serial ---');
>   console.log(await serial());
>
>   console.log('--- concurrent ---');
>   console.log(await concurrent());
> })().catch(console.error);
> ```
>
> 典型输出：
>
> ```text
> --- serial ---
> start A
> finish A
> start B
> finish B
> [ 'A', 'B' ]
> --- concurrent ---
> start A
> start B
> finish B
> finish A
> [ 'A', 'B' ]
> ```
>
> 串行阶段只有 A 完成后才启动 B；并发阶段先启动 A 和 B。即使 B 先完成，`Promise.all` 仍按输入顺序返回 `['A', 'B']`。
>
> ## 常见追问
>
> 1. **`await` 会阻塞主线程吗？** 不会。它暂停当前 async 函数并让出控制权；但 `await` 前后的大量同步计算仍会阻塞主线程。
> 1. **为什么在循环里 `await` 有时是问题、有时是正确的？** 独立任务逐个等待会损失并发性；若后一次依赖前一次结果、需要限流或必须保序，串行才符合业务语义。
> 1. **`Promise.all` 一个任务失败后，其他任务会被取消吗？** 不会。返回的组合 Promise 会拒绝，但其他已启动任务继续运行；需要显式取消或等待 `allSettled` 收集结果。
>
> ## 易错点
>
> - 把“写了多个 `await`”等同于并发；关键是 Promise 在何时创建。
> - 认为 `await` 阻塞整个 JavaScript 线程，忽略它只暂停当前 async 函数。
> - 在 `Array.prototype.forEach` 中传 async 回调并期待外层等待；`forEach` 不消费回调返回的 Promise。串行用 `for...of`，并发可用 `map` 后 `Promise.all`。
> - 用 `void task()` 假装处理错误；没有 `.catch` 的 rejection 仍可能成为未处理 rejection。
> - 把 `Promise.all` 的尽快拒绝误解为自动取消其他任务。
> - 只用 `try/catch` 处理网络异常，却忘记 `fetch` 的 HTTP 4xx/5xx 通常需要手动检查 `response.ok`。
> - 认为 `await` 天然避免过期响应；它不提供“最后一次请求获胜”的身份判断。
>
> ## 项目关联
>
> - **已实现：有依赖的串行请求。** `my-ipaas/src/hooks/useWorkflowNavigation.ts` 的 `createWorkflow` 依次读取连接器列表、触发器详情、创建流程，再加载新流程；`duplicateWorkflow` 也先取详情、再创建副本。后一步依赖前一步结果，因此串行是业务要求，不应盲目改成 `Promise.all`。
> - **已实现：过期结果保护。** `my-ipaas/src/stores/slices/flowSlice.ts` 的 `loadFlow` 使用递增序号，让快速切换 A -> B 时 A 的旧响应不能覆盖 B；`src/hooks/useWorkflowNavigation.ts` 也用 request ID 保护列表与 pending 状态。`src/hooks/useVariableSources.ts` 用 Effect 内的 `cancelled` 标志忽略已失效实例的回调，但不会取消共享请求。
> - **已实现：并发且允许部分失败。** `my-ipaas/src/hooks/useVariableSources.ts` 先为多个上游步骤创建任务，再用 `Promise.allSettled` 收集成功与失败。
> - **已实现：模拟流程串行推进。** `my-ipaas/src/utils/debugSimulator.ts` 在循环中 `await waitForStep()`，因此调试节点按顺序推进；这只是前端调试模拟，不是真实执行引擎。
> - **仅可作为改进练习：自动保存写入顺序。** `my-ipaas/src/stores/flowStore.ts` 已用 timer 防抖，但当前未见在途 PUT 的串行队列、版本检查或过期响应保护。可以练习让最新快照最终获胜，但不能宣称该竞态已经解决。
> - **学习边界：** 上述实现均需小杜能画出请求时序、解释串并行理由并补测试后，才能作为已掌握的项目亮点。
>
> ## 相关题目
>
> - [Promise-链与错误传播](Promise-链与错误传播.md)
> - [浏览器事件循环与任务调度](浏览器事件循环与任务调度.md)
> - [闭包与词法作用域](闭包与词法作用域.md)
> - [../React/useEffect-依赖清理与StrictMode](../React/useEffect-依赖清理与StrictMode.md)：Effect 中的异步任务还需处理 cleanup、过期结果和底层取消的边界。
>
> ## 参考资料
>
> - [ECMAScript Language Specification - Async Function Definitions](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-async-function-definitions)，核验于 2026-08-10。
> - [ECMAScript Language Specification - Await](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-await)，核验于 2026-08-10。
> - [MDN - async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)，核验于 2026-08-10。
> - [MDN - Promise.all()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)，核验于 2026-08-10。
>