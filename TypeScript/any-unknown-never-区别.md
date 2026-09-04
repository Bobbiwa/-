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

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> `any` 是类型检查的逃生口：对它读属性、调用或赋给其他类型通常都不会报错，风险会继续传播。`unknown` 也能接收任意值，但使用前必须通过 `typeof`、`instanceof`、类型守卫等方式收窄，所以更适合接口输入、JSON 和异常这类不可信边界。`never` 表示不可能出现的值，常用于永不正常返回的函数和联合类型的穷尽检查。业务代码应默认用具体类型；边界暂时未知时优先用 `unknown`，只有明确接受放弃检查时才局部使用 `any`。
>
> ## 详细原理
>
> ### 1. `any` 会跳过检查并传播风险
>
> 当一个值是 `any` 时，TypeScript 通常允许读取任意属性、调用它、参与运算，也允许它流入其他类型。编译器因此无法继续证明后续代码安全。`noImplicitAny` 只负责阻止编译器悄悄推断出隐式 `any`，不会禁止显式写出的 `any`；项目还可以通过 ESLint 的 `no-explicit-any` 进一步约束。
>
> `any` 适合极少数渐进迁移、缺失类型声明或确实需要绕过检查的局部边界。使用时应缩小作用域并尽快转换为可验证的类型，不能把它当作“万能泛型”。
>
> ### 2. `unknown` 表示未知但仍要求证明
>
> 任意值都可以赋给 `unknown`，但 `unknown` 不能直接当作某个具体类型使用。代码必须先检查它，控制流分析才会在对应分支中收窄类型。例如：
>
> - `typeof value === "string"` 收窄原始类型；
> - `Array.isArray(value)` 收窄数组；
> - `value instanceof Error` 收窄类实例；
> - 自定义类型谓词或校验库验证对象结构。
>
> 因此 `unknown` 很适合作为系统边界的输入类型。不过，写一句 `value as User` 只是告诉编译器“相信我”，不会生成运行时校验；真正不可信的数据仍需检查结构。
>
> ### 3. `never` 表示空集合和不可达状态
>
> `never` 不是“任何值”，恰恰相反，它表示正常情况下不存在的值。始终抛错或永不结束的函数可以返回 `never`；联合类型经过控制流收窄、所有成员都被排除后，剩余值也会是 `never`。
>
> 这个特性可用于穷尽检查：把 `switch` 的剩余值交给只接收 `never` 的函数。以后联合类型新增成员而分支没有同步处理，编译器就会报错。
>
> ### 4. 三者的选择顺序
>
> 可以把类型看成值的集合：`unknown` 位于安全的“顶部”，可以容纳所有值但不能直接消费；`never` 位于“底部”，没有正常值；`any` 则绕开了这套安全约束。实际选择顺序通常是：具体类型优先，其次在不可信边界使用 `unknown`，最后才是受控且局部的 `any`。
>
> ## TypeScript 示例（TypeScript 5.9，Node.js 或浏览器）
>
> ```typescript
> interface ImportedWorkflow {
>   name: string;
>   steps: unknown[];
> }
>
> function isRecord(value: unknown): value is Record<string, unknown> {
>   return typeof value === "object" && value !== null && !Array.isArray(value);
> }
>
> function parseWorkflow(text: string): ImportedWorkflow {
>   // JSON.parse 的标准库返回值会失去具体类型信息，主动收口为 unknown。
>   const parsed: unknown = JSON.parse(text);
>
>   if (
>     !isRecord(parsed)
>     || typeof parsed.name !== "string"
>     || !Array.isArray(parsed.steps)
>   ) {
>     throw new Error("无效的工作流 JSON");
>   }
>
>   return {
>     name: parsed.name,
>     steps: parsed.steps,
>   };
> }
>
> function errorMessage(error: unknown): string {
>   return error instanceof Error ? error.message : "未知错误";
> }
>
> function fail(message: string): never {
>   throw new Error(message);
> }
>
> try {
>   console.log(parseWorkflow('{"name":"演示流程","steps":[]}'));
>   fail("模拟失败");
> } catch (error: unknown) {
>   console.log(errorMessage(error));
> }
> ```
>
> 示例只验证了 `name` 和 `steps` 的最外层结构，尚未证明每个步骤都符合完整业务协议。真实导入功能还应继续验证数组元素，或者使用经过维护的 schema 校验方案。
>
> ## 常见追问
>
> 1. **`unknown` 和 `any` 都能接收任意值，核心差异是什么？** `any` 允许未经证明就继续操作并会传播不安全；`unknown` 在使用前强制收窄。
> 1. **类型断言能把 `unknown` 安全地变成业务类型吗？** 不能。断言只影响静态检查，不验证运行时数据；外部输入需要真实校验。
> 1. **`void` 和 `never` 有什么区别？** `void` 表示调用方不应使用返回结果，函数仍可正常结束；`never` 表示函数不能正常完成或该分支不可能到达。
>
> ## 易错点
>
> - 认为 `any` 是“所有类型的联合”；它的关键特征是绕过检查，不等同于安全的联合类型。
> - 为了消除报错把接口响应直接断言成目标类型，忽略网络、文件和存储数据在运行时不受 TypeScript 约束。
> - 认为使用 `unknown` 后什么都不能做；完成可靠收窄后就可以正常使用。
> - 把 `never` 和 `void` 混为一谈，或者给一个可能正常返回的函数标注 `never`。
> - 只验证对象非空就断言成完整协议，未逐层检查必要字段。
>
> ## 项目关联
>
> 以下结论基于 2026-08-11 对 `my-ipaas` 当前代码的只读检查，只代表仓库存在对应实现，不代表老公已经掌握：
>
> - **已实现：导入边界收窄。** `src/components/layout/WorkflowSider.tsx` 把 `JSON.parse` 的结果显式接为 `unknown`，用 `isRecord`、`Array.isArray` 和字段检查后再构造 `FlowExport`。它只完成有限结构检查，不能描述成完整 schema 校验。
> - **已实现：本地存储状态收窄。** `src/stores/slices/debugSlice.ts` 将 `sessionStorage` 中解析出的值视为 `unknown`，并用 `isRunStatus` 只接受有限的调试状态字符串。
> - **仍有缺口：`any` 尚未清理。** 当天运行 `npm run check` 共报告 18 个 lint 错误，其中多处是显式 `any`；lint 失败后 build 未执行。因此不能声称项目已经建立严格完整的类型边界。
>
> ## 相关题目
>
> - [类型守卫-可辨识联合与穷尽检查](类型守卫-可辨识联合与穷尽检查.md)：`unknown` 要靠收窄才能安全使用，`never` 可验证分支是否完整。
> - [泛型约束-keyof-与类型关系](泛型约束-keyof-与类型关系.md)：泛型用于保留输入输出关系，不能用 `any` 冒充泛型。
> - [../JavaScript/原始值与对象的赋值和传参](../JavaScript/原始值与对象的赋值和传参.md)：TypeScript 静态类型不会改变 JavaScript 的运行时传值语义。
>
> ## 参考资料
>
> - [TypeScript Handbook - Everyday Types: any](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any)，核验于 2026-08-11。
> - [TypeScript Handbook - More on Functions: unknown and never](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown)，核验于 2026-08-11。
> - [TypeScript Handbook - Narrowing: The never type](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-never-type)，核验于 2026-08-11。
>