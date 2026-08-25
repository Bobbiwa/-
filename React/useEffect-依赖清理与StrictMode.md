---
title: "React useEffect 的适用边界、依赖、清理与 Strict Mode 行为是什么？"
category: "React"
tags:
  - useEffect
  - dependencies
  - cleanup
  - strict-mode
  - synchronization
difficulty: hard
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "React 19.2 - Synchronizing with Effects"
    url: "https://react.dev/learn/synchronizing-with-effects"
    verified_at: 2026-08-11
  - title: "React 19.2 - Lifecycle of Reactive Effects"
    url: "https://react.dev/learn/lifecycle-of-reactive-effects"
    verified_at: 2026-08-11
  - title: "React 19.2 - StrictMode"
    url: "https://react.dev/reference/react/StrictMode"
    verified_at: 2026-08-11
---

# React useEffect 的适用边界、依赖、清理与 Strict Mode 行为是什么？

## 问题

`useEffect` 应该解决什么问题，哪些逻辑不应放进去？依赖数组如何确定？cleanup 在什么时候执行？为什么 Strict Mode 开发环境会额外执行 Effect，异步请求又该怎样避免过期结果？

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p><code>useEffect</code> 用于把 React 当前的 props 和 State 与外部系统同步，例如订阅、DOM API、网络连接或第三方组件。纯派生数据应在渲染中计算，明确由点击触发的操作应放事件处理器，不要为了“代码运行一次”就用 Effect。依赖不是手选的优化项：Effect 中读取的响应式值都要声明，React 用 <code>Object.is</code> 比较依赖。重新同步前先执行上一轮 cleanup，卸载时也会清理。Strict Mode 在开发环境额外执行 setup 和 cleanup，是为了暴露不纯渲染或缺失清理；正确的 Effect 应能安全地建立、撤销并再次建立同步。</p>
<h2>详细原理</h2>
<h3>1. Effect 的职责是外部同步</h3>
<p>渲染代码负责根据 props 和 State 纯计算 UI；事件处理器负责由具体交互直接触发的动作；Effect 则处理“组件出现在屏幕上后，需要让某个外部系统与当前状态一致”的逻辑。典型外部系统包括事件监听器、timer、网络连接、浏览器 API 和非 React 控件。</p>
<p>如果一个值能从现有 props 或 State 计算出来，直接在渲染中计算或在确有成本时使用 memo，而不是先渲染旧值、再用 Effect 写入派生 State。购买、提交等由一次点击引起的动作也应放在对应事件中，否则重新挂载可能错误地再次执行。</p>
<h3>2. 依赖由 Effect 读取的响应式值决定</h3>
<p>组件 props、State 以及组件函数体中计算的变量都可能随渲染变化，属于响应式值。Effect 读取它们，就必须列入依赖；React 在每次提交后用 <code>Object.is</code> 比较新旧依赖，变化时重新同步。</p>
<p>不能为了阻止 Effect 重跑而删依赖或关闭 lint。应该改变代码结构：把非响应式常量移到组件外，把只由事件触发的逻辑移到事件里，稳定真正需要稳定的函数，或者拆开不相关的同步过程。每个 Effect 最好只代表一个独立同步过程。</p>
<h3>3. cleanup 与 setup 必须对称</h3>
<p>Effect 返回的 cleanup 用于撤销本轮 setup：移除监听器、清除 timer、断开连接或取消请求。在依赖变化并执行新 setup 前，React 会先运行上一轮 cleanup；组件卸载时也会运行最后一次 cleanup。</p>
<p>异步任务有两种不同策略：</p>
<ul><li><code>AbortController</code> 等真正取消底层工作，适合独占请求；</li><li>用布尔标志或 request ID 忽略过期结果，只阻止旧回调写状态，不等于取消网络请求。</li></ul>
<p>共享请求缓存可能不能由某一个组件随意中止，此时忽略当前订阅者的过期结果反而更合理，但要明确资源仍在运行。</p>
<h3>4. Strict Mode 的额外执行只用于开发检查</h3>
<p>在 React 19.2 中，<code>&lt;StrictMode&gt;</code> 会启用开发期额外检查，其中包括额外重新渲染，以及额外执行一次 Effect 和 ref callback 的建立/清理流程。目的不是模拟生产次数，而是更快暴露缺失 cleanup 和不纯逻辑。</p>
<p>不能把现象简单背成“<code>useEffect</code> 永远执行两次”。是否启用 Strict Mode、它包裹的树位置以及开发或生产构建都会影响观察结果。正确目标是让用户无法区分一次 setup 与 setup -&gt; cleanup -&gt; setup。</p>
<h2>TypeScript 示例（React 19.2，浏览器 TSX）</h2>
<pre><code class="language-tsx">import { useEffect, useState } from &quot;react&quot;;&#10;&#10;export function NetworkStatus() {&#10;  const [online, setOnline] = useState&lt;boolean&gt;(() =&gt; navigator.onLine);&#10;&#10;  useEffect(() =&gt; {&#10;    function updateStatus(): void {&#10;      setOnline(navigator.onLine);&#10;    }&#10;&#10;    window.addEventListener(&quot;online&quot;, updateStatus);&#10;    window.addEventListener(&quot;offline&quot;, updateStatus);&#10;&#10;    return () =&gt; {&#10;      window.removeEventListener(&quot;online&quot;, updateStatus);&#10;      window.removeEventListener(&quot;offline&quot;, updateStatus);&#10;    };&#10;  }, []);&#10;&#10;  return &lt;p&gt;网络状态：{online ? &quot;在线&quot; : &quot;离线&quot;}&lt;/p&gt;;&#10;}</code></pre>
<p>这个 Effect 同步浏览器网络事件：setup 注册两个监听器，cleanup 使用相同函数引用撤销它们。它没有读取会变化的组件内响应式值，因此依赖数组为空。Strict Mode 额外建立和清理时也不会残留重复监听器。</p>
<h2>常见追问</h2>
<ol><li><strong>空依赖数组是不是表示“整个应用只执行一次”？</strong> 不是。它表示该组件某次挂载期间不因响应式依赖变化而重新同步；组件重新挂载仍会执行，开发期 Strict Mode 还会额外检查。</li><li><strong>为什么不能自己挑依赖？</strong> 依赖描述 Effect 实际读取的数据。漏掉依赖会让同步逻辑继续使用旧闭包，形成 stale closure。</li><li><strong>cleanup 中设置 <code>cancelled = true</code> 是否取消了请求？</strong> 没有。它只能让回调忽略结果；真正取消需底层 API 支持，例如 <code>AbortController</code>。</li></ol>
<h2>易错点</h2>
<ul><li>把 Effect 当作生命周期回调或通用“代码运行器”，没有先说明要同步哪个外部系统。</li><li>为了只运行一次而删除依赖、禁用 <code>exhaustive-deps</code>，留下旧闭包。</li><li>在一个 Effect 中混合多个无关流程，使任一依赖变化都触发全部逻辑。</li><li>cleanup 使用了新的函数引用，导致原监听器没有被移除。</li><li>把 Strict Mode 的开发检查描述成生产环境固定执行两次，或通过全局标志掩盖缺失清理。</li><li>只忽略异步结果，却声称网络、timer 或共享 Promise 已被取消。</li></ul>
<h2>项目关联</h2>
<p>以下结论基于 2026-08-11 对 <code>my-ipaas</code> 当前代码的只读检查，只代表仓库存在对应实现：</p>
<ul><li><strong>已实现：过期结果隔离。</strong> <code>src/hooks/useVariableSources.ts</code> 的每轮 Effect 建立独立 <code>cancelled</code> 绑定，cleanup 将旧轮次标记为失效，后续 Promise 回调据此跳过状态写入。共享请求仍会继续完成，这不是底层取消。</li><li><strong>已实现：开发期 Strict Mode。</strong> <code>src/main.tsx</code> 用 <code>&lt;StrictMode&gt;</code> 包裹 <code>&lt;App /&gt;</code>，因此开发环境会启用 React 19.2 对渲染、Effect 和 ref cleanup 的额外检查。</li><li><strong>已发现质量问题。</strong> 当天 <code>npm run check</code> 的 18 个 lint 错误中，包括 <code>ActionTab.tsx</code> 在 Effect 内同步设置本地 State，以及 <code>VirtualCanvas.tsx</code> 渲染期间读取 ref 等 React Hooks 规则问题；lint 失败后 build 未执行。不能声称项目 Effect 设计已经全部通过检查。</li></ul>
<h2>相关题目</h2>
<ul><li><a href="State-快照批处理与函数式更新.md">State-快照批处理与函数式更新</a>：Effect 与事件回调读取的值都属于某次渲染快照。</li><li><a href="key-组件身份与状态保留.md">key-组件身份与状态保留</a>：<code>key</code> 变化会导致组件重新挂载，从而触发旧 Effect 清理和新 Effect 建立。</li><li><a href="../JavaScript/闭包与词法作用域.md">../JavaScript/闭包与词法作用域</a>：依赖遗漏造成旧值问题的基础是回调闭合了某次渲染的词法环境。</li><li><a href="../JavaScript/async-await-串并行与错误处理.md">../JavaScript/async-await-串并行与错误处理</a>：Effect 中的异步任务还需处理并发、错误和过期结果。</li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://react.dev/learn/synchronizing-with-effects">React 19.2 - Synchronizing with Effects</a>，核验于 2026-08-11。</li><li><a href="https://react.dev/learn/lifecycle-of-reactive-effects">React 19.2 - Lifecycle of Reactive Effects</a>，核验于 2026-08-11。</li><li><a href="https://react.dev/reference/react/StrictMode">React 19.2 - StrictMode</a>，核验于 2026-08-11。</li></ul>
</details>