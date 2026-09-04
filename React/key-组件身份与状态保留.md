---
title: "React 的 key 如何决定组件身份、状态保留与重置？"
category: "React"
tags:
  - key
  - identity
  - reconciliation
  - state-preservation
  - lists
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "React 19.2 - Rendering Lists"
    url: "https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key"
    verified_at: 2026-08-11
  - title: "React 19.2 - Preserving and Resetting State"
    url: "https://react.dev/learn/preserving-and-resetting-state"
    verified_at: 2026-08-11
  - title: "React Versions"
    url: "https://react.dev/versions"
    verified_at: 2026-08-11
---

# React 的 key 如何决定组件身份、状态保留与重置？

## 问题

React 为什么要求列表项提供 `key`？什么样的 `key` 才稳定？使用数组索引或随机数会有什么问题？`key` 如何影响组件 State 的保留和主动重置？

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> `key` 是 React 在同一个父节点下识别子项身份的依据之一。列表插入、删除或重排时，稳定的业务 ID 能让 React 把新旧元素正确对应，从而保留属于同一项的 State 和 DOM。`key` 只需在兄弟节点之间唯一，但在这项数据的生命周期内必须稳定；动态列表不要用索引，也不要在渲染时生成随机数。类型或 `key` 改变时，React 会把它视为不同身份，旧组件卸载、State 重置；这个行为也可以用于有意清空表单。`key` 不会作为普通 prop 传给组件。
>
> ## 详细原理
>
> ### 1. State 绑定在渲染树中的身份位置
>
> React 保存 State 时，需要判断本次渲染的组件是否仍是上一次的那个组件。公开语义上，组件类型、父节点中的位置以及 `key` 共同影响身份。相同身份通常保留 State；类型或 `key` 改变则建立新身份，旧 State 被丢弃。
>
> `key` 并不只适用于列表。在同一位置切换两个相同类型的组件时，可以用不同 `key` 明确告诉 React 它们代表不同实体；表单切换联系人时便可借此主动重置草稿。
>
> ### 2. 列表 key 必须稳定且在兄弟间唯一
>
> 列表发生排序、插入和删除时，位置会变化。稳定业务 ID 让 React 仍能识别“移动的是同一项”。唯一性范围是当前兄弟列表，不要求整个应用全局唯一；不同数组可以复用同一个值。
>
> `key` 应来自数据，在数据创建时生成并持久保存。渲染期间调用 `Math.random()` 或每次生成 UUID 会让每轮身份都变化，导致子树反复卸载、创建，丢失输入和局部 State。
>
> ### 3. 数组索引只适用于不会变化的静态顺序
>
> 若用索引作为 `key`，删除第一项后，后面的数据会继承前一个位置对应的组件身份。无状态的简单文本可能暂时看不出问题，但带输入框、焦点、动画或请求状态时容易出现状态“串行”。
>
> 只有列表内容和顺序确定不变、项目没有稳定 ID 时，索引才可能是可接受的退让。一旦支持插入、删除、筛选、排序或拖拽，就应使用稳定 ID。
>
> ### 4. `key` 是 React 提示，不是组件参数
>
> React 消费 `key` 来匹配元素，组件内部不能通过 `props.key` 读取它。组件若也需要 ID，必须显式传入 `id={item.id}`。当 `map` 返回多个并列元素时，`key` 要放在数组直接返回的最外层元素上；短语法 Fragment 不能接收 `key`，需要使用 `<Fragment key={...}>`。
>
> 面试中可以解释身份与结果，不应把某个具体 diff 步骤或复杂度细节包装成 React 对所有版本都承诺的公开实现。
>
> ## TypeScript 示例（React 19.2，浏览器 TSX）
>
> ```tsx
> import { useState } from "react";
>
> interface Task {
>   id: string;
>   title: string;
> }
>
> function TaskRow({ task }: { task: Task }) {
>   const [draft, setDraft] = useState(task.title);
>
>   return (
>     <label>
>       {task.id}
>       <input
>         value={draft}
>         onChange={(event) => setDraft(event.currentTarget.value)}
>       />
>     </label>
>   );
> }
>
> export function TaskBoard() {
>   const [tasks, setTasks] = useState<Task[]>([
>     { id: "task-a", title: "读取流程" },
>     { id: "task-b", title: "修改节点" },
>   ]);
>
>   return (
>     <section>
>       <button
>         type="button"
>         onClick={() => setTasks((current) => [...current].reverse())}
>       >
>         反转顺序
>       </button>
>
>       {tasks.map((task) => (
>         <TaskRow key={task.id} task={task} />
>       ))}
>     </section>
>   );
> }
> ```
>
> 输入草稿后反转列表，稳定的 `task.id` 会让草稿跟随对应任务。如果改用数组索引，位置变化后草稿可能与错误任务对应。若业务希望任务切换时主动清空子组件 State，可以有意让目标组件的 `key` 随实体 ID 改变。
>
> ## 常见追问
>
> 1. **`key` 必须全局唯一吗？** 不需要，只需在当前父节点的兄弟元素之间唯一；但同一数据项跨渲染必须保持稳定。
> 1. **为什么不能每次渲染调用 `crypto.randomUUID()`？** 新旧 `key` 永远匹配不上，React 会重建组件和 DOM，局部 State、焦点和未提交输入都会丢失。
> 1. **什么时候可以主动修改 `key`？** 当业务明确要求把同一位置视为新实体并重置整棵子树状态时，例如切换不同收件人的编辑表单；这应是有意的身份设计。
>
> ## 易错点
>
> - 只要求 `key` 唯一，却忽略稳定性。
> - 动态列表使用数组索引，插入、删除或拖拽后让局部 State 对错数据。
> - 在渲染时生成随机 `key`，造成每次都卸载重建。
> - 把 `key` 写在子组件内部，而不是 `map` 直接返回的元素上。
> - 尝试读取 `props.key`，没有另传业务 ID。
> - 为“强制刷新”随意改 `key`，掩盖真实的数据流或 Effect 清理问题。
>
> ## 项目关联
>
> 以下结论基于 2026-08-11 对 `my-ipaas` 当前代码的只读检查，只代表仓库存在对应实现：
>
> - **已实现：递归步骤稳定身份。** `src/pages/canvas/StepTree.tsx` 对普通、分支和循环步骤使用 `step.id`，对分支列使用 `path.pathId`，适合步骤插入、删除和跨容器拖拽后的身份匹配。
> - **已实现：有意重建拖拽 Provider。** `src/pages/canvas/Index.tsx` 把 `moveCount` 作为 `DragDropProvider` 的 `key`；`flowSlice.ts` 在成功移动节点后递增它，注释明确这是为了重建 Provider、清理放置目标缓存。这里是有目的的重置，不是普通列表 key 的写法。
> - **学习边界：稳定 key 不等于整个渲染正确。** 仍需本人用带输入 State 的列表复现索引 key 错位，再解释项目为何使用步骤 ID。当天 `npm run check` 有 18 个 lint 错误且 build 未执行，不能据此声称画布已通过完整质量验证。
>
> ## 相关题目
>
> - [State-快照批处理与函数式更新](State-快照批处理与函数式更新.md)：同一身份保留 State 后，setter 再决定这份 State 如何更新。
> - [useEffect-依赖清理与StrictMode](useEffect-依赖清理与StrictMode.md)：`key` 变化触发重新挂载，也会触发 Effect 的 cleanup 与新 setup。
> - [../TypeScript/类型守卫-可辨识联合与穷尽检查](../TypeScript/类型守卫-可辨识联合与穷尽检查.md)：步骤的 `role` 决定数据成员，步骤的 `id` 作为 `key` 决定渲染身份。
>
> ## 参考资料
>
> - [React 19.2 - Rendering Lists](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key)，核验于 2026-08-11。
> - [React 19.2 - Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)，核验于 2026-08-11。
> - [React Versions](https://react.dev/versions)，核验于 2026-08-11；当日文档最新版为 React 19.2。
>