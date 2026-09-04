---
title: "JavaScript 中 this 的绑定规则是什么？箭头函数有什么不同？"
category: "JavaScript"
tags:
  - this
  - function
  - arrow-function
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Language Specification - EvaluateCall"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluatecall"
    verified_at: 2026-08-10
  - title: "ECMAScript Language Specification - Arrow Function Definitions"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-arrow-function-definitions"
    verified_at: 2026-08-10
  - title: "MDN - this"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this"
    verified_at: 2026-08-10
  - title: "MDN - Arrow function expressions"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions"
    verified_at: 2026-08-10
---

# JavaScript 中 this 的绑定规则是什么？箭头函数有什么不同？

## 问题

普通函数中的 `this` 是如何确定的？请说明默认绑定、隐式绑定、显式绑定和构造调用的区别，以及箭头函数为什么不能用 `call`、`apply` 或 `bind` 改变 `this`。

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> 普通函数的 `this` 主要由调用方式决定，而不是由函数写在哪里决定。直接调用在严格模式下得到 `undefined`；通过 `obj.fn()` 调用时是点号左侧的接收者；`call`、`apply` 和 `bind` 可以显式指定；用 `new` 调用时则指向新创建的实例。常见优先级可以记为：构造调用高于 `bind`，`bind` 高于 `call`、`apply`，再高于隐式和默认绑定。箭头函数没有自己的 `this`，它从定义时所在的外层词法环境取得 `this`，所以这些 API 和 `new` 都不能改变它。
>
> ## 详细原理
>
> ### 普通函数看调用点
>
> 对普通函数，先区分以下调用方式：
>
> 1. **默认绑定**：`fn()` 没有接收者。严格模式下 `this` 是 `undefined`；对非严格普通函数，这次裸调用传入的 `undefined` 会被替换为 `globalThis`。更一般地说，通过 `call` 或 `apply` 传入的 `null`、`undefined` 也会被替换，原始值则会被装箱。现代代码不应依赖这些非严格模式转换。
> 1. **隐式绑定**：`obj.fn()` 会以 `obj` 作为这次调用的接收者，因此函数体中的 `this === obj`。链式成员访问只看最终调用时紧邻函数的基对象，例如 `a.b.fn()` 的接收者是 `a.b`。
> 1. **显式绑定**：`fn.call(value, ...args)` 和 `fn.apply(value, args)` 立即调用函数；`fn.bind(value, ...args)` 创建一个绑定了 `this` 和可选前置参数的新函数。
> 1. **构造调用**：`new Fn()` 通过函数的 `[[Construct]]` 内部方法创建实例。对普通基类构造函数，构造期间的 `this` 通常是新实例；构造函数显式返回对象时是一个重要例外，详见 [原型链-new-与-class](原型链-new-与-class.md)。
>
> 所谓“丢失 `this`”，通常不是 `this` 被改变了，而是调用表达式改变了：
>
> ```javascript
> 'use strict';
>
> const user = {
>   name: '小杜',
>   getName() {
>     return this.name;
>   },
> };
>
> console.log(user.getName()); // 小杜：以 user 为接收者
>
> const getName = user.getName;
> // getName() 是直接调用；严格模式下 this 为 undefined，读取 name 会抛错。
> ```
>
> 对象并不会永久“拥有”某个普通函数。将方法赋给变量、作为回调传递，或者改由另一个对象调用，都可能改变调用点和 `this`。
>
> ### 绑定优先级需要带条件理解
>
> 对可构造的普通函数，面试中可用下面的常见顺序判断：
>
> `new` 构造调用 > `bind` 创建的绑定函数 > `call` / `apply` 显式调用 > 成员访问的隐式调用 > 默认绑定。
>
> - 对绑定函数再次使用 `call` 或 `apply`，不能覆盖它已经绑定的 `this`。
> - 对绑定函数使用 `new` 时，绑定的 `thisArg` 会被忽略，但 `bind` 预置的参数仍然有效。
> - 这套口诀不适用于箭头函数，因为箭头函数的 `[[ThisMode]]` 是 lexical；它根本不执行普通函数的 `this` 绑定流程。
> - 代理对象、自定义 `Symbol.hasInstance`、内置构造器等还可能有额外语义，不能只靠口诀推导所有规范细节。
>
> ### 箭头函数使用词法 this
>
> 箭头函数没有自己的 `this` 绑定。执行箭头函数中的 `this` 表达式时，会沿词法环境向外寻找最近一个提供 `this` 的环境。因此，“定义时绑定”更准确的说法是“从定义位置的外层环境捕获”，并不是创建箭头函数时把某个对象值复制进函数。
>
> ```javascript
> const account = {
>   name: '小杜',
>   makeReader() {
>     return () => this.name;
>   },
> };
>
> const readName = account.makeReader();
> console.log(readName.call({ name: '其他人' })); // 小杜
> ```
>
> 这里的箭头函数捕获的是调用 `account.makeReader()` 时该方法中的 `this`。如果把 `makeReader` 也拆出来直接调用，那么箭头函数最终得到的 `this` 也会随外层调用环境变化。箭头函数还没有自己的 `arguments`、`super` 和 `new.target`，也没有 `[[Construct]]`，所以不能用 `new` 调用；这些特征使它适合需要保留外层上下文的回调，但通常不适合充当依赖动态接收者的对象方法。
>
> ### 严格模式和模块边界
>
> - ECMAScript Module 和 `class` 的代码天然运行在严格模式下，普通函数直接调用时不会把 `this` 自动替换成全局对象。
> - 浏览器经典脚本的顶层 `this` 通常是 `window`，浏览器 ES Module 的顶层 `this` 是 `undefined`。
> - Node.js CommonJS 会用模块包装器执行文件，顶层 `this` 通常是 `module.exports`；它不等同于浏览器脚本的顶层规则。判断输出题时必须先确认运行环境、脚本类型和严格模式。
>
> ## JavaScript 示例（Node.js 23 或现代浏览器）
>
> ```javascript
> 'use strict';
>
> function describe(prefix) {
>   return `${prefix}:${this.name}`;
> }
>
> const first = { name: 'first', describe };
> const second = { name: 'second' };
>
> console.log(first.describe('implicit')); // implicit:first
> console.log(describe.call(second, 'call')); // call:second
>
> const bound = describe.bind(first, 'bound');
> console.log(bound.call(second)); // bound:first，call 不能覆盖 bind
>
> const owner = {
>   name: 'owner',
>   makeArrow() {
>     return () => this.name;
>   },
> };
>
> const readOwner = owner.makeArrow();
> console.log(readOwner.call(second)); // owner，call 不能改变箭头函数的 this
>
> function Person(name) {
>   this.name = name;
> }
>
> const ignoredThis = { name: 'ignored' };
> const BoundPerson = Person.bind(ignoredThis);
> const person = new BoundPerson('instance');
>
> console.log(person.name); // instance
> console.log(ignoredThis.name); // ignored
> ```
>
> 这个示例只依赖 ECMAScript 运行时语义，可直接保存后用 `node` 执行，也可在现代浏览器控制台运行。
>
> ## 常见追问
>
> 1. **为什么把 `obj.method` 作为回调传入后经常丢失 `this`？** 传递的是函数值，之后若以 `callback()` 调用，就不再保留原来的成员访问接收者。可以按场景使用 `bind`、包装箭头函数，或让 API 显式传递上下文。
> 1. **事件监听器中的 `this` 一定是什么？** 不一定。浏览器 `addEventListener` 调用普通函数监听器时，`this` 通常是 `currentTarget`；箭头函数仍使用外层 `this`。React 函数组件事件回调不应套用 DOM 监听器的 `this` 结论。
> 1. **`bind` 返回的函数还能作为构造函数吗？** 如果原函数可构造，绑定函数通常也可被 `new` 调用；此时绑定的 `thisArg` 被忽略，预置参数仍参与调用。箭头函数不可构造，绑定后也不会变得可构造。
>
> ## 易错点
>
> - 说“`this` 永远指向定义函数的对象”。普通函数看调用方式，箭头函数才从外层词法环境取 `this`。
> - 认为 `obj.fn.call(other)` 的 `this` 仍是 `obj`。显式绑定在这次普通调用中选择了 `other`。
> - 忘记方法拆出后，`const fn = obj.fn; fn()` 已不再是隐式绑定。
> - 用 `call`、`apply` 或 `bind` 试图修改箭头函数的 `this`，或用 `new` 调用箭头函数。
> - 不说明严格模式、ES Module、浏览器经典脚本和 Node.js CommonJS 的差异，就直接断言顶层或默认 `this` 一定是 `window`。
> - 把优先级口诀当成完整规范，忽略绑定函数构造调用、构造函数返回对象等边界。
>
> ## 项目关联
>
> 弱关联（**已实现代码**）：`/Users/dubobo/Desktop/my-ipaas/db/repository.ts` 使用 `Object.prototype.hasOwnProperty.call(next, key)`。这里借用 `Object.prototype` 上的方法，并通过 `call` 把本次调用的 `this` 显式设为待检查对象，避免依赖该对象自身是否继承或覆盖了 `hasOwnProperty`。
>
> 这只是语言 API 的局部使用。当前项目没有自然的自定义 `this` 绑定或自定义继承设计案例，也不能据此判断小杜已经掌握本题。
>
> ## 相关题目
>
> - [原型链-new-与-class](原型链-new-与-class.md)
> - [闭包与词法作用域](闭包与词法作用域.md)
> - [var-let-const-作用域提升与暂时性死区](var-let-const-作用域提升与暂时性死区.md)
>
> ## 参考资料
>
> - [ECMAScript Language Specification - EvaluateCall](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-evaluatecall)，核验于 2026-08-10。
> - [ECMAScript Language Specification - Arrow Function Definitions](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-arrow-function-definitions)，核验于 2026-08-10。
> - [MDN - this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)，核验于 2026-08-10。
> - [MDN - Arrow function expressions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)，核验于 2026-08-10。
>