---
title: "var、let、const 在作用域、提升和暂时性死区上有什么区别？"
category: "JavaScript"
tags:
  - scope
  - hoisting
  - temporal-dead-zone
  - variable-declaration
difficulty: medium
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Variable Statement"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-variable-statement"
    verified_at: 2026-08-10
  - title: "ECMAScript Let and Const Declarations"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-let-and-const-declarations"
    verified_at: 2026-08-10
  - title: "MDN - var"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var"
    verified_at: 2026-08-10
  - title: "MDN - let"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let"
    verified_at: 2026-08-10
  - title: "MDN - const"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const"
    verified_at: 2026-08-10
---

# var、let、const 在作用域、提升和暂时性死区上有什么区别？

## 问题

请比较 `var`、`let`、`const` 的作用域、声明初始化、重复声明和重新赋值规则。它们是否都会“提升”？什么是暂时性死区（Temporal Dead Zone，TDZ），为什么 `typeof` 有时也会抛出 `ReferenceError`？

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> `var` 是函数作用域或全局作用域，不受普通代码块限制；进入对应执行上下文时，它的绑定就会被创建并初始化为 `undefined`，所以声明前读取通常得到 `undefined`。`let` 和 `const` 是块级作用域，进入作用域时绑定已经存在，但执行到声明完成初始化前处于 TDZ，读取会抛 `ReferenceError`。`let` 可以重新赋值，`const` 不可以重新赋值且声明时通常必须初始化，但 `const` 对象的属性仍可修改。与其只背“谁会提升”，更准确的是说明三者绑定何时创建、何时初始化。
>
> ## 详细原理
>
> ### 1. 作用域边界
>
> - `var` 的声明作用域是包含它的函数、静态初始化块或脚本全局环境，普通 `{}`、`if` 和 `for` 代码块不会为它建立块级绑定。
> - `let`、`const` 属于 lexical declaration，以块、函数体、模块等词法结构为边界。
> - 在 ES module 中，顶层声明都是模块作用域。浏览器经典 `<script>` 的顶层 `var` 与顶层 `let`/`const` 还存在全局对象属性差异，不能把该结论无条件套到 ES module 或 Node.js 模块包装环境。
>
> ### 2. “提升”背后的创建和初始化
>
> “提升”是常用理解术语，不是把源码声明真的移动到顶部。更准确的执行模型是：进入作用域时先进行声明实例化，再逐句执行代码。
>
> - `var` 绑定会提前创建并初始化为 `undefined`。所以赋值语句执行前能读取该绑定，只是还没有得到后面的赋值。
> - `let`、`const` 绑定也会在代码执行到声明前参与当前词法作用域和遮蔽，但初始状态是未初始化。此时从作用域开始位置到初始化完成之间就是 TDZ。
> - 执行到 `let` 声明时绑定被初始化；执行到 `const` 声明时必须以初始化值建立不可重新赋值的绑定。`for-in`、`for-of` 头部是语法上的特殊初始化位置。
>
> 因此，回答“`let` 和 `const` 会不会提升”时不宜只答会或不会：它们的绑定在声明前已经存在并产生遮蔽，但不能在初始化前访问。
>
> ### 3. TDZ 与 `typeof`
>
> TDZ 的起点是进入声明所在的词法作用域，而不是代码文本中的声明行。读取、写入或使用 `typeof` 访问同一作用域内尚未初始化的 lexical binding 都会抛 `ReferenceError`。
>
> `typeof completelyUndeclared` 在找不到任何绑定时返回字符串 `"undefined"`；但 `typeof value` 若解析到了 TDZ 中的 `value` 绑定，仍然会抛错。这两个场景不能混为一谈。
>
> ### 4. 重复声明、重新赋值与循环绑定
>
> - 同一作用域中的多个 `var` 声明通常允许共存；`var` 绑定也可以重新赋值。
> - 同一词法作用域不能用 `let`/`const` 重复声明同名绑定，也不能与会发生声明冲突的同名声明并存；这类问题通常是解析阶段的 `SyntaxError`。
> - `let` 可以重新赋值；`const` 不能重新赋值，但若其值是对象，对象属性仍可能被修改。
> - `for` 循环头中的 `let` 会为每次迭代建立对应绑定，回调能读到各自那一轮的值；`var` 通常由所有回调共享同一个函数级绑定。
>
> 工程代码默认优先 `const`，确实需要重新赋值时使用 `let`；这是一条可读性建议，不是语言规范的强制要求。现代业务代码一般避免 `var`，但维护旧代码时仍需理解它的语义。
>
> ## JavaScript 示例（Node.js 23）
>
> ```javascript
> "use strict";
>
> function declarationDemo() {
>   console.log(beforeVar); // undefined
>   var beforeVar = "var initialized";
>
>   try {
>     console.log(beforeLet);
>   } catch (error) {
>     console.log(error.name); // ReferenceError
>   }
>   let beforeLet = "let initialized";
>
>   if (true) {
>     var functionScoped = "visible outside block";
>     const blockScoped = "only inside block";
>     console.log(blockScoped); // only inside block
>   }
>   console.log(functionScoped); // visible outside block
>
>   try {
>     console.log(blockScoped);
>   } catch (error) {
>     console.log(error.name); // ReferenceError
>   }
> }
>
> declarationDemo();
>
> const callbacksWithVar = [];
> for (var i = 0; i < 3; i += 1) {
>   callbacksWithVar.push(() => i);
> }
> console.log(callbacksWithVar.map((callback) => callback())); // [3, 3, 3]
>
> const callbacksWithLet = [];
> for (let j = 0; j < 3; j += 1) {
>   callbacksWithLet.push(() => j);
> }
> console.log(callbacksWithLet.map((callback) => callback())); // [0, 1, 2]
>
> let outer = "outside";
> {
>   try {
>     console.log(typeof outer);
>   } catch (error) {
>     console.log(error.name); // ReferenceError：解析到下面的 TDZ 绑定
>   }
>   let outer = "inside";
>   console.log(outer); // inside
> }
> console.log(typeof neverDeclared); // undefined：没有找到任何同名绑定
> ```
>
> ## 常见追问
>
> 1. **`let` 和 `const` 到底算不算提升？** 如果“提升”指声明前绑定已经影响名字解析和遮蔽，可以说有类似提升的行为；但它们不会像 `var` 一样提前初始化为 `undefined`，初始化前处于 TDZ。面试时直接解释创建与初始化阶段最准确。
> 1. **为什么 `typeof` TDZ 变量会报错，而未声明变量不会？** 前者已经解析到一个存在但未初始化的 lexical binding，访问被禁止；后者根本找不到绑定，`typeof` 对这一情况有返回 `"undefined"` 的特殊规则。
> 1. **浏览器中顶层声明都会成为 `window` 属性吗？** 不是。经典 script 中符合条件的顶层 `var` 会建立全局对象属性，而顶层 `let`/`const` 不会；ES module、Node.js 模块及其他宿主环境不能直接套用这一说法。
>
> ## 易错点
>
> - 把“提升”理解为 JavaScript 引擎改写并移动了源码，而不解释声明实例化。
> - 说 `let`/`const` 在声明前“不存在”；它们已经参与名字解析，只是尚未初始化。
> - 认为 TDZ 从声明行开始；它实际覆盖进入该词法作用域到初始化前的区间。
> - 认为 `typeof` 访问任何未初始化名字都安全，忽略 TDZ 会触发 `ReferenceError`。
> - 把 `const` 误解成深层不可变，或者用 `var`/`let`/`const` 的差异解释对象属性是否可变。
> - 忽略运行环境，宣称所有顶层 `var` 都是 `globalThis` 属性。
>
> ## 项目关联
>
> **状态：仅可作为基础练习。** 当前没有必要把变量声明规则包装成 `my-ipaas` 的项目亮点。可以在本地写循环回调、块级遮蔽和 TDZ 小实验，并在走读项目时解释为何默认使用 `const`、何时必须使用 `let`。项目采用这些声明并不证明小杜已经掌握其执行语义，需能脱离原文件复现上述行为后再标记掌握。
>
> ## 相关题目
>
> - [原始值与对象的赋值和传参](原始值与对象的赋值和传参.md)：`const` 限制的是绑定，不能据此判断对象是否可变。
> - [闭包与词法作用域](闭包与词法作用域.md)：循环回调的结果取决于闭包所引用的绑定，以及 `let` 的逐次迭代环境。
>
> ## 参考资料
>
> - [ECMAScript Variable Statement](https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-variable-statement)，核验于 2026-08-10。
> - [ECMAScript Let and Const Declarations](https://tc39.es/ecma262/multipage/ecmascript-language-statements-and-declarations.html#sec-let-and-const-declarations)，核验于 2026-08-10。
> - [MDN - var](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var)，核验于 2026-08-10。
> - [MDN - let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let)，核验于 2026-08-10。
> - [MDN - const](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const)，核验于 2026-08-10。
>




# FAQ

## 1.结合JavaScript引擎执行模型理解TDZ

很多人对暂时性死区（TDZ）感到困惑，就是因为只背了“let 不断提升”这句过于简化的教学口诀。

要彻底搞懂 TDZ，我们必须像 JavaScript 引擎一样思考。文档中指出，更准确的执行模型并不是简单地把代码移动到顶部，而是分为两个阶段：**先进行声明实例化，再逐句执行代码**。  

我们通过这个底层模型来一步步拆解 TDZ 的形成：

### 第一阶段：声明实例化（引擎的“战前扫描”）

当 JavaScript 执行到一个新的作用域（比如进入一个函数或一个 `{}` 块）时，它不会立刻执行第一行代码，而是先快速扫描整个作用域里的所有变量声明。

- **对于 `var`：** 引擎会提前把这个变量创建出来，并且顺手给它塞一个初始值 `undefined`。  
- **对于 `let` 和 `const`：** 引擎同样会发现它们，并且让它们参与到当前作用域的名字解析和遮蔽中（也就是说，引擎已经登记了这个名字），但**初始状态被严格标记为“未初始化”**。  

### 第二阶段：逐句执行代码（真正的运行期）

扫描完毕后，代码才开始真正从上往下逐句执行。

- 如果执行到声明行之前你去读取 `var`，因为它在第一阶段已经被赋予了 `undefined`，所以引擎允许访问。  
- 如果执行到声明行之前你去读取 `let` 或 `const`，引擎一查状态：“我知道这个作用域里有这个变量，但它目前还没完成初始化，禁止触碰！” 于是直接抛出一个 `ReferenceError` 报错。当代码最终执行到 `let` 声明所在的那一行时，绑定才算真正被初始化。  

### 什么是真正的“暂时性死区 (TDZ)”？

结合上面两个阶段，TDZ 的本质就浮出水面了： **TDZ 并不是代码文本中的某几行物理空间，而是一个“时间窗口”。** 它的起点是引擎进入该词法作用域的一瞬间，终点是代码真正执行到变量声明并完成初始化的那一刻。在这个时间段内，任何读取或写入该变量的操作都会触发死区警报。  

### `typeof` 的“翻车”逻辑

理解了这个模型，文档中提到的 `typeof` 报错现象就极其合理了：

- **真正不存在的变量：** `typeof completelyUndeclared` 会安全地返回字符串 `"undefined"`，因为引擎在第一阶段扫描时根本没见过它，找不到任何绑定，于是触发了 `typeof` 的特殊豁免规则。  
- **处于 TDZ 的变量：** `typeof value` 会报错，因为引擎在第一阶段已经把 `value` 登记在册了（解析到了 TDZ 中的绑定）。引擎明确知道你想通过 `typeof` 试探一个已知但还没初始化的词法绑定，所以无情拦截并抛出错误。  

总结一下，与其说 `let` 和 `const` 不会提升，不如说它们“只提升了存在感（提前创建绑定），但没提升使用权（拒绝提前初始化）”。 



## 2. let和const 预扫描阶段，到底什么是“参与遮蔽 (Shadowing)”？

“遮蔽”的字面意思就是“挡住了”**。在编程里，指的是**内层作用域的变量，挡住了外层同名变量的视线。

文档中提到 `let` 和 `const` 绑定在声明前就已经参与了遮蔽。我们来看一段极其经典的“翻车”代码，你立刻就能明白：  

JavaScript

```
let name = "外层的杜老板"; // 外层变量

function test() {
  console.log(name); // 猜猜这里会输出什么？
  let name = "内层的小杜"; // 内层变量
}

test();
```

按照直觉，代码执行到 `console.log(name)` 时，内层的 `name` 还没执行到，那它不应该去外面找，打印出 `"外层的杜老板"` 吗？

**结果是：直接报错 `ReferenceError: Cannot access 'name' before initialization`。**

**为什么会报错？这就是“参与遮蔽”在发威！**

结合我们前面聊过的“引擎两阶段模型”：

1. **预扫描阶段：** 引擎进入 `test` 函数，扫了一圈，发现了 `let name = "内层的小杜"`。引擎心想：“好，这个函数块里有一个本地的 `name` 变量。从现在开始，这个函数里所有对 `name` 的访问，**都归我管，外面的 `name` 被我遮蔽（挡住）了**。”
2. **执行阶段：** 引擎走到 `console.log(name)`。它问作用域：“有 `name` 吗？”作用域说：“有啊，内层自己的那个就是（因为它把外层遮蔽了），但是它还没走到声明那一行，还处于暂时性死区（TDZ）里呢！”于是，无情报错。

所以，“参与遮蔽”的意思就是：**虽然我还没出生（没初始化），但我已经提前把位置占了，外面的同名兄弟谁也别想进来插手。**



## 3. 什么是“词法作用域（静态作用域）”？

你对这句话有印象非常棒！“词法作用域”的英文是 Lexical Scope，"Lexical" 翻译过来就是“词法的、文本的”。

所谓“代码写在哪，作用域就在哪，跟在哪调用无关”，文档中也明确指出了 `let` 和 `const` 属于 lexical declaration（词法声明），它们的边界是由你敲代码时的文本结构（比如大括号 `{}`、函数体）直接划定的。  

我们再用一段代码来实战感受一下它的威力：

JavaScript

```
let food = "苹果"; // 全局作用域的 food

// 定义函数 A
function eat() {
  // eat 函数在定义的时候，它的词法作用域就已经写死了
  // 它往外看，只能看到全局的 food = "苹果"
  console.log("我在吃：" + food); 
}

// 定义函数 B
function run() {
  let food = "香蕉"; // run 函数内部的 food
  eat(); // 在这里调用 eat()
}

run(); // 猜猜输出什么？
```

很多人会误以为输出是 `"我在吃：香蕉"`，因为 `eat()` 是在 `run()` 里面被调用的，而 `run` 里面明明有个香蕉啊！

**实际输出是：`"我在吃：苹果"`。**

**这就是词法作用域（静态作用域）的核心灵魂：**

`eat` 函数在被**定义（写下代码）\**的那一刻，它的作用域链就已经被死死地“焊住”了。当时它的眼睛只看到了全局的 `"苹果"`。后来无论你把它带到天涯海角去\**调用**（哪怕是在充满 `"香蕉"` 的 `run` 函数里），它依然只认它出生时看到的那个 `"苹果"`。

**总结一下：**

- 引擎不关心你是**顺着哪条线索找过来执行我的（调用栈）**。
- 引擎只关心你**这行代码物理上是写在哪个大括号里的（词法文本）**。

搞懂了词法作用域，以后遇到再复杂的“闭包”面试题，你只需要像看物理地图一样，死死盯住那个函数是**在哪一行被声明（写下）的**，就能轻松找出它能访问哪些变量，绝对不会再被它在哪调用给绕晕了。