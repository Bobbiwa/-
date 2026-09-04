---
title: "React State 的快照、批处理和函数式更新是什么？"
category: "React"
tags:
  - state
  - snapshot
  - batching
  - functional-update
  - hooks
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "React 19.2 - State as a Snapshot"
    url: "https://react.dev/learn/state-as-a-snapshot"
    verified_at: 2026-08-11
  - title: "React 19.2 - Queueing a Series of State Updates"
    url: "https://react.dev/learn/queueing-a-series-of-state-updates"
    verified_at: 2026-08-11
  - title: "React Versions"
    url: "https://react.dev/versions"
    verified_at: 2026-08-11
---

# React State 的快照、批处理和函数式更新是什么？

## 问题

为什么调用 State setter 后，当前事件处理函数里的变量不会立刻变化？React 如何批处理多个更新？连续更新同一个状态时，什么时候必须使用函数式更新？

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> React 每次渲染都会拿到一份固定的 State 快照，事件处理函数也闭合这次渲染的值。调用 setter 是把更新加入队列并请求下一次渲染，不会修改当前函数里的变量。React 会批处理一批更新，再计算新状态和渲染界面。若新状态依赖之前的状态，应传 updater，例如 `setCount(count => count + 1)`；React 会按顺序把队列中的结果交给下一个 updater。若只是用一个与旧值无关的新值替换状态，直接传值更清楚。
>
> ## 详细原理
>
> ### 1. 每次渲染看到固定快照
>
> 函数组件被调用时，React 会根据当次 props 和 State 计算 JSX、局部变量和事件处理函数。这个组合是一份当时的快照。setter 不会回头修改已经运行中的局部变量，而是请求 React 使用新状态再调用组件。
>
> 因此在同一个点击处理器中，即使执行了 `setCount(count + 1)`，后面的 `console.log(count)` 仍看到当前渲染的旧值。这不是 setter 异步返回一个 Promise，也不能通过 `await setCount(...)` 等待更新，因为 setter 没有这种契约。
>
> ### 2. 批处理减少不必要的中间渲染
>
> React 会先让当前事件处理代码执行完成，再处理队列中的 State 更新，避免每次 setter 都产生一个用户看不到的中间界面。不同的有意用户事件会分别处理，例如两次独立点击不会被当成同一次操作合并。
>
> 批处理描述的是更新何时统一处理，不代表多次更新会自动累加。同一快照中的三次 `setCount(count + 1)` 都根据同一个 `count` 计算，等价于连续请求替换成同一个值。
>
> ### 3. updater 按队列中的前一结果计算
>
> 传给 setter 的函数叫 updater。React 处理队列时，会把前一个更新得到的 pending state 传给下一个 updater，所以三次 `setCount(value => value + 1)` 能连续累加三次。
>
> 当下一个状态依赖旧状态、更新可能在同一批次发生、或回调不应依赖闭包中的旧快照时，应优先使用 updater。若直接设置为接口返回值或固定状态，例如 `setStatus("succeeded")`，传值即可。
>
> ### 4. 更新对象和数组要创建新值
>
> State 快照中的对象仍遵循 JavaScript 引用语义。直接修改旧对象不会向 React 表达一次可靠的状态替换，还会污染旧快照。应使用展开、`map`、`filter` 等方式创建新对象或数组；若基于旧集合修改，也应放在函数式 updater 中。
>
> ## TypeScript 示例（React 19.2，浏览器 TSX）
>
> ```tsx
> import { useState } from "react";
>
> interface Job {
>   id: string;
>   retries: number;
> }
>
> export function RetryPanel() {
>   const [count, setCount] = useState(0);
>   const [jobs, setJobs] = useState<Job[]>([
>     { id: "job-1", retries: 0 },
>   ]);
>
>   function addThree(): void {
>     setCount((value) => value + 1);
>     setCount((value) => value + 1);
>     setCount((value) => value + 1);
>   }
>
>   function retry(id: string): void {
>     setJobs((current) => current.map((job) => (
>       job.id === id
>         ? { ...job, retries: job.retries + 1 }
>         : job
>     )));
>   }
>
>   return (
>     <section>
>       <p>计数：{count}</p>
>       <button type="button" onClick={addThree}>连续加三</button>
>
>       {jobs.map((job) => (
>         <button key={job.id} type="button" onClick={() => retry(job.id)}>
>           {job.id}：重试 {job.retries} 次
>         </button>
>       ))}
>     </section>
>   );
> }
> ```
>
> `addThree` 的三个 updater 按队列依次计算。`retry` 同时使用函数式更新和不可变数组更新，既不依赖旧闭包，也不修改上一份 State 快照。
>
> ## 常见追问
>
> 1. **setter 是不是异步函数？** 不能这样概括。它会请求并排队更新，但不返回可等待本次渲染完成的 Promise；重点是 State 属于某次渲染快照。
> 1. **是否所有 setter 都必须写成函数式更新？** 不是。只有下一值依赖之前状态时 updater 才更合适；替换成独立的新值可直接传值。
> 1. **为什么直接 `array.push()` 后再把同一个数组传给 setter 有风险？** 它修改了旧快照且保留相同引用，破坏不可变更新假设，也可能让 React 或依赖引用比较的逻辑无法识别变化。
>
> ## 易错点
>
> - 调用 setter 后立刻读取当前变量，并期待它已经是新值。
> - 把批处理理解成“React 会自动合并业务运算”，忽略三次替换值与三次 updater 的区别。
> - 试图 `await setState(...)` 获取更新后的 State。
> - 在 updater 内产生请求、日志上报等副作用；updater 应保持纯函数，开发检查可能额外调用它来发现不纯逻辑。
> - 只浅拷贝最外层对象，却继续直接修改共享的嵌套对象。
>
> ## 项目关联
>
> 以下结论基于 2026-08-11 对 `my-ipaas` 当前代码的只读检查，只代表仓库存在对应实现：
>
> - **已实现：重试版本递增。** `src/hooks/useVariableSources.ts` 使用 `setRevision((value) => value + 1)`，下一值明确依赖队列中的前一值，适合函数式更新。
> - **已实现：列表基于旧值更新。** `src/hooks/useWorkflowNavigation.ts` 使用 `setFlows((current) => ...)` 插入、重命名和发布流程，避免异步回调依赖创建时的旧列表快照。
> - **学习边界：不能据此声称已掌握。** 需要本人解释闭包快照、手写连续 updater，并独立修改和验证一个状态更新场景。当天项目 `npm run check` 仍因 18 个 lint 错误中止，build 未执行。
>
> ## 相关题目
>
> - [useEffect-依赖清理与StrictMode](useEffect-依赖清理与StrictMode.md)：Effect 读取的 props 和 State 同样来自某次渲染，依赖决定何时重新同步。
> - [key-组件身份与状态保留](key-组件身份与状态保留.md)：State 更新决定值如何变化，组件身份决定这份 State 是否被保留。
> - [../JavaScript/闭包与词法作用域](../JavaScript/闭包与词法作用域.md)：事件处理器闭合当次渲染的变量，是理解旧快照问题的 JavaScript 基础。
> - [../TypeScript/泛型约束-keyof-与类型关系](../TypeScript/泛型约束-keyof-与类型关系.md)：`useState<T>` 通过泛型关联状态值与 setter 输入。
>
> ## 参考资料
>
> - [React 19.2 - State as a Snapshot](https://react.dev/learn/state-as-a-snapshot)，核验于 2026-08-11。
> - [React 19.2 - Queueing a Series of State Updates](https://react.dev/learn/queueing-a-series-of-state-updates)，核验于 2026-08-11。
> - [React Versions](https://react.dev/versions)，核验于 2026-08-11；当日文档最新版为 React 19.2。
>