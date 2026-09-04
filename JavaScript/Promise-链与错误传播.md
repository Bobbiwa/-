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

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> `then`、`catch` 和 `finally` 都会返回一个新的 Promise。`then` 的回调返回普通值时，下一个 Promise 以该值 fulfilled；返回 Promise 或 thenable 时，下一个 Promise 会采用它的最终状态；抛出异常时，下一个 Promise rejected。没有提供对应处理函数时，原来的值或错误会继续向后传。`catch` 本质上是 `then(undefined, onRejected)`；它返回普通值或最终 fulfilled 的 thenable 时，错误才算恢复，抛错或返回最终 rejected 的 thenable 时仍走失败分支。`finally` 通常只做清理并保持原来的值或错误，但若它抛错或返回 rejected Promise，会用新的错误覆盖原结果。
>
> ## 详细原理
>
> ### 1. 每次链式调用都会创建新的 Promise
>
> `promise.then(...)` 不会修改原 Promise，而是立即返回一个新的、最初处于 pending 状态的 Promise。即使原 Promise 已经 settled，处理函数也不会同步执行，而会由宿主调度 Promise reaction job。
>
> 处理函数执行后的结果决定新 Promise：
>
> | 处理函数结果 | 新 Promise 的结果 |
> | --- | --- |
> | `return 42` | fulfilled，值为 `42` |
> | 没有 `return` | fulfilled，值为 `undefined` |
> | `return anotherPromise` | 采用该 Promise 的最终状态和值或原因 |
> | `return thenable` | 按 Promise resolution procedure 处理该 thenable |
> | `throw error` | rejected，原因为 `error` |
>
> 不能让一个 Promise 用它自身完成，否则会以 `TypeError` 拒绝。业务中更常见的问题是漏写 `return`：外层链不会等待回调中启动的异步任务，其 rejection 也不会进入外层的 `catch`。
>
> ### 2. 缺失的处理函数会让结果继续传播
>
> 如果 `then` 没有传 `onFulfilled`，fulfilled 值会原样传给后续；如果没有传 `onRejected`，rejection 原因也会继续向后传播。因此可以把一个 `catch` 放在链尾，集中处理此前没有被恢复的错误。
>
> `catch(onRejected)` 等价于调用 `then(undefined, onRejected)`。`catch` 回调返回普通值或最终 fulfilled 的 thenable 时，其返回的新 Promise 才是 fulfilled，后面的成功分支会执行；若再次 `throw`，或返回最终 rejected 的 thenable，失败会继续传播。若返回的 thenable 一直 pending，整条后续链也会一直 pending。
>
> ### 3. `finally` 默认透明，但自身也可能失败
>
> `finally` 的回调不接收前一阶段的值或错误，它适合关闭 loading、释放资源等与结果无关的清理。它返回普通值时不会替换原结果；若返回 Promise 或 thenable，链会等待：最终 fulfilled 才透传原结果，一直 pending 则后续链也一直 pending。只有 `finally` 抛出异常，或返回最终 rejected 的 thenable，链才会改为使用这个新错误。
>
> ### 4. 组合 Promise 时要先确定失败策略
>
> - `Promise.all` 适合“全部成功才算成功”，任一输入 rejected 时，返回的 Promise 会尽快 rejected；其他已启动任务不会因此自动取消。
> - `Promise.allSettled` 会等待所有输入 settled，并为每项保留 `fulfilled` 或 `rejected` 结果，适合允许部分成功的场景。
> - `Promise.race` 采用第一个 settled 的输入；`Promise.any` 采用第一个 fulfilled 的输入，全部失败时才以 `AggregateError` 拒绝。
>
> “尽快拒绝”描述的是组合 Promise 的结果，不等于中止底层网络请求。真正取消请求通常需要 `AbortController` 等额外机制。
>
> ### 5. 未处理 rejection 属于宿主行为
>
> ECMAScript 定义 Promise 与 reaction job，但浏览器如何报告 `unhandledrejection`、Node.js 如何报告未处理 rejection，属于宿主环境行为。面试时不应把某一版本 Node.js 的警告或退出策略说成 JavaScript 语言的固定规则。
>
> ## JavaScript 示例
>
> 运行环境：现代浏览器控制台或 Node.js 23.11.1。
>
> ```javascript
> console.log('script start');
>
> const result = Promise.resolve(2)
>   .then((value) => {
>     console.log('step 1:', value);
>     return value * 3;
>   })
>   .then((value) => {
>     throw new Error(`bad:${value}`);
>   })
>   .catch((error) => {
>     console.log('caught:', error.message);
>     return 10; // 正常返回，表示已经把失败恢复为成功。
>   })
>   .finally(() => {
>     console.log('cleanup');
>     return 999; // 普通返回值不会覆盖前面的 10。
>   });
>
> result.then((value) => console.log('result:', value));
> console.log('script end');
> ```
>
> 输出：
>
> ```text
> script start
> script end
> step 1: 2
> caught: bad:6
> cleanup
> result: 10
> ```
>
> 下面的写法漏掉了 `return`，外层链不会等待 `save()`：
>
> ```javascript
> doSomething()
>   .then(() => {
>     save(); // 错误：save 返回的 Promise 没有接入当前链。
>   })
>   .catch(reportError);
>
> // 应写成：
> doSomething()
>   .then(() => save())
>   .catch(reportError);
> ```
>
> ## 常见追问
>
> 1. **为什么 `catch` 后面的 `then` 通常会执行？** 因为 `catch` 返回普通值或最终 fulfilled 的 thenable 时，它产生的新 Promise 是 fulfilled；再次抛错或返回最终 rejected 的 thenable 会继续走失败分支，返回始终 pending 的 thenable 则不会进入后续分支。
> 1. **`Promise.all` 中一个请求失败后，其他请求会停止吗？** 不会。`Promise.all` 只会尽快拒绝自己的结果，已经启动的任务仍会继续，除非显式取消。
> 1. **`finally(() => 123)` 和 `then(() => 123)` 有什么不同？** 前者通常透传原结果，后者用 `123` 完成新 Promise；但 `finally` 自身失败时仍会覆盖原结果。
>
> ## 易错点
>
> - 认为 `.then` 会修改并返回原 Promise；实际上每次调用都返回新的 Promise。
> - 在 `.then` 中启动异步任务却忘记 `return`，导致执行顺序、错误传播和 loading 状态都脱离当前链。
> - 认为 `catch` 只记录错误、不改变状态，或认为回调没有同步抛错就一定恢复成功；实际结果还取决于它返回的 thenable 最终状态。
> - 认为 `finally` 的普通返回值会替换业务结果，或忽略了 `finally` 自己也可能抛错。
> - 认为 `fetch` 收到 HTTP 4xx/5xx 会自动 rejected；`fetch` 通常只在网络失败等情况下拒绝，业务代码仍需检查 `response.ok`。
>
> ## 项目关联
>
> - **已实现：Promise 请求缓存与失败失效。** `my-ipaas/src/hooks/useConnectors.ts` 的 `cachePromise` 会让多个 Hook 复用同一次请求，并在 rejection 后清空缓存；`src/utils/connectorActions.ts` 和 `src/hooks/useVariableSources.ts` 也按 `connectorId` 缓存 Promise，失败时删除对应项，避免永久复用 rejected Promise。
> - **已实现：允许部分成功。** `my-ipaas/src/hooks/useVariableSources.ts` 使用 `Promise.allSettled` 并行加载多个上游连接器的输出协议；某一项失败时仍保留其他 fulfilled 结果，只有没有可用来源且存在失败时才显示整体错误。
> - **学习边界：** 以上只表示当前代码中存在相应实现，不表示小杜已经能独立解释或重写。可练习画出“缓存命中、首次请求、失败失效、手动重试”四条链路，并为它们补测试。
>
> ## 相关题目
>
> - [浏览器事件循环与任务调度](浏览器事件循环与任务调度.md)
> - [async-await-串并行与错误处理](async-await-串并行与错误处理.md)
> - [闭包与词法作用域](闭包与词法作用域.md)
>
> ## 参考资料
>
> - [ECMAScript Language Specification - Promise Objects](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-objects)，核验于 2026-08-10。
> - [MDN - Promise.prototype.then()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then)，核验于 2026-08-10。
> - [MDN - Promise.prototype.finally()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally)，核验于 2026-08-10。
> - [MDN - Promise.allSettled()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)，核验于 2026-08-10。
>