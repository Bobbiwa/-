---
title: "TypeScript 中 any、unknown 和 never 有什么区别？"
category: "TypeScript"
tags:
  - any
  - unknown
  - never
  - type-safety
  - boundary-validation
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "TypeScript Handbook - Everyday Types: any"
    url: "https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any"
    verified_at: 2026-08-11
  - title: "TypeScript Handbook - More on Functions: unknown and never"
    url: "https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown"
    verified_at: 2026-08-11
  - title: "TypeScript Handbook - Narrowing: The never type"
    url: "https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-never-type"
    verified_at: 2026-08-11
---

# TypeScript 中 any、unknown 和 never 有什么区别？

## 问题

`any`、`unknown` 和 `never` 分别表示什么？它们在可赋值性、允许的操作和典型使用场景上有什么区别？接收接口响应、解析 JSON 或处理异常时应该优先选哪个？

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p><code>any</code> 是类型检查的逃生口：对它读属性、调用或赋给其他类型通常都不会报错，风险会继续传播。<code>unknown</code> 也能接收任意值，但使用前必须通过 <code>typeof</code>、<code>instanceof</code>、类型守卫等方式收窄，所以更适合接口输入、JSON 和异常这类不可信边界。<code>never</code> 表示不可能出现的值，常用于永不正常返回的函数和联合类型的穷尽检查。业务代码应默认用具体类型；边界暂时未知时优先用 <code>unknown</code>，只有明确接受放弃检查时才局部使用 <code>any</code>。</p>
<h2>详细原理</h2>
<h3>1. <code>any</code> 会跳过检查并传播风险</h3>
<p>当一个值是 <code>any</code> 时，TypeScript 通常允许读取任意属性、调用它、参与运算，也允许它流入其他类型。编译器因此无法继续证明后续代码安全。<code>noImplicitAny</code> 只负责阻止编译器悄悄推断出隐式 <code>any</code>，不会禁止显式写出的 <code>any</code>；项目还可以通过 ESLint 的 <code>no-explicit-any</code> 进一步约束。</p>
<p><code>any</code> 适合极少数渐进迁移、缺失类型声明或确实需要绕过检查的局部边界。使用时应缩小作用域并尽快转换为可验证的类型，不能把它当作“万能泛型”。</p>
<h3>2. <code>unknown</code> 表示未知但仍要求证明</h3>
<p>任意值都可以赋给 <code>unknown</code>，但 <code>unknown</code> 不能直接当作某个具体类型使用。代码必须先检查它，控制流分析才会在对应分支中收窄类型。例如：</p>
<ul><li><code>typeof value === &quot;string&quot;</code> 收窄原始类型；</li><li><code>Array.isArray(value)</code> 收窄数组；</li><li><code>value instanceof Error</code> 收窄类实例；</li><li>自定义类型谓词或校验库验证对象结构。</li></ul>
<p>因此 <code>unknown</code> 很适合作为系统边界的输入类型。不过，写一句 <code>value as User</code> 只是告诉编译器“相信我”，不会生成运行时校验；真正不可信的数据仍需检查结构。</p>
<h3>3. <code>never</code> 表示空集合和不可达状态</h3>
<p><code>never</code> 不是“任何值”，恰恰相反，它表示正常情况下不存在的值。始终抛错或永不结束的函数可以返回 <code>never</code>；联合类型经过控制流收窄、所有成员都被排除后，剩余值也会是 <code>never</code>。</p>
<p>这个特性可用于穷尽检查：把 <code>switch</code> 的剩余值交给只接收 <code>never</code> 的函数。以后联合类型新增成员而分支没有同步处理，编译器就会报错。</p>
<h3>4. 三者的选择顺序</h3>
<p>可以把类型看成值的集合：<code>unknown</code> 位于安全的“顶部”，可以容纳所有值但不能直接消费；<code>never</code> 位于“底部”，没有正常值；<code>any</code> 则绕开了这套安全约束。实际选择顺序通常是：具体类型优先，其次在不可信边界使用 <code>unknown</code>，最后才是受控且局部的 <code>any</code>。</p>
<h2>TypeScript 示例（TypeScript 5.9，Node.js 或浏览器）</h2>
<pre><code class="language-typescript">interface ImportedWorkflow {&#10;  name: string;&#10;  steps: unknown[];&#10;}&#10;&#10;function isRecord(value: unknown): value is Record&lt;string, unknown&gt; {&#10;  return typeof value === &quot;object&quot; &amp;&amp; value !== null &amp;&amp; !Array.isArray(value);&#10;}&#10;&#10;function parseWorkflow(text: string): ImportedWorkflow {&#10;  // JSON.parse 的标准库返回值会失去具体类型信息，主动收口为 unknown。&#10;  const parsed: unknown = JSON.parse(text);&#10;&#10;  if (&#10;    !isRecord(parsed)&#10;    || typeof parsed.name !== &quot;string&quot;&#10;    || !Array.isArray(parsed.steps)&#10;  ) {&#10;    throw new Error(&quot;无效的工作流 JSON&quot;);&#10;  }&#10;&#10;  return {&#10;    name: parsed.name,&#10;    steps: parsed.steps,&#10;  };&#10;}&#10;&#10;function errorMessage(error: unknown): string {&#10;  return error instanceof Error ? error.message : &quot;未知错误&quot;;&#10;}&#10;&#10;function fail(message: string): never {&#10;  throw new Error(message);&#10;}&#10;&#10;try {&#10;  console.log(parseWorkflow(&#39;{&quot;name&quot;:&quot;演示流程&quot;,&quot;steps&quot;:[]}&#39;));&#10;  fail(&quot;模拟失败&quot;);&#10;} catch (error: unknown) {&#10;  console.log(errorMessage(error));&#10;}</code></pre>
<p>示例只验证了 <code>name</code> 和 <code>steps</code> 的最外层结构，尚未证明每个步骤都符合完整业务协议。真实导入功能还应继续验证数组元素，或者使用经过维护的 schema 校验方案。</p>
<h2>常见追问</h2>
<ol><li><strong><code>unknown</code> 和 <code>any</code> 都能接收任意值，核心差异是什么？</strong> <code>any</code> 允许未经证明就继续操作并会传播不安全；<code>unknown</code> 在使用前强制收窄。</li><li><strong>类型断言能把 <code>unknown</code> 安全地变成业务类型吗？</strong> 不能。断言只影响静态检查，不验证运行时数据；外部输入需要真实校验。</li><li><strong><code>void</code> 和 <code>never</code> 有什么区别？</strong> <code>void</code> 表示调用方不应使用返回结果，函数仍可正常结束；<code>never</code> 表示函数不能正常完成或该分支不可能到达。</li></ol>
<h2>易错点</h2>
<ul><li>认为 <code>any</code> 是“所有类型的联合”；它的关键特征是绕过检查，不等同于安全的联合类型。</li><li>为了消除报错把接口响应直接断言成目标类型，忽略网络、文件和存储数据在运行时不受 TypeScript 约束。</li><li>认为使用 <code>unknown</code> 后什么都不能做；完成可靠收窄后就可以正常使用。</li><li>把 <code>never</code> 和 <code>void</code> 混为一谈，或者给一个可能正常返回的函数标注 <code>never</code>。</li><li>只验证对象非空就断言成完整协议，未逐层检查必要字段。</li></ul>
<h2>项目关联</h2>
<p>以下结论基于 2026-08-11 对 <code>my-ipaas</code> 当前代码的只读检查，只代表仓库存在对应实现，不代表老公已经掌握：</p>
<ul><li><strong>已实现：导入边界收窄。</strong> <code>src/components/layout/WorkflowSider.tsx</code> 把 <code>JSON.parse</code> 的结果显式接为 <code>unknown</code>，用 <code>isRecord</code>、<code>Array.isArray</code> 和字段检查后再构造 <code>FlowExport</code>。它只完成有限结构检查，不能描述成完整 schema 校验。</li><li><strong>已实现：本地存储状态收窄。</strong> <code>src/stores/slices/debugSlice.ts</code> 将 <code>sessionStorage</code> 中解析出的值视为 <code>unknown</code>，并用 <code>isRunStatus</code> 只接受有限的调试状态字符串。</li><li><strong>仍有缺口：<code>any</code> 尚未清理。</strong> 当天运行 <code>npm run check</code> 共报告 18 个 lint 错误，其中多处是显式 <code>any</code>；lint 失败后 build 未执行。因此不能声称项目已经建立严格完整的类型边界。</li></ul>
<h2>相关题目</h2>
<ul><li><a href="类型守卫-可辨识联合与穷尽检查.md">类型守卫-可辨识联合与穷尽检查</a>：<code>unknown</code> 要靠收窄才能安全使用，<code>never</code> 可验证分支是否完整。</li><li><a href="泛型约束-keyof-与类型关系.md">泛型约束-keyof-与类型关系</a>：泛型用于保留输入输出关系，不能用 <code>any</code> 冒充泛型。</li><li><a href="../JavaScript/原始值与对象的赋值和传参.md">../JavaScript/原始值与对象的赋值和传参</a>：TypeScript 静态类型不会改变 JavaScript 的运行时传值语义。</li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any">TypeScript Handbook - Everyday Types: any</a>，核验于 2026-08-11。</li><li><a href="https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown">TypeScript Handbook - More on Functions: unknown and never</a>，核验于 2026-08-11。</li><li><a href="https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-never-type">TypeScript Handbook - Narrowing: The never type</a>，核验于 2026-08-11。</li></ul>
</details>