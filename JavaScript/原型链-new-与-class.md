---
title: "JavaScript 的原型链、new 和 class 是什么关系？"
category: "JavaScript"
tags:
  - prototype
  - prototype-chain
  - new
  - class
difficulty: hard
priority: P0
status: new
score: null
review_count: 0
last_reviewed: null
next_review: null
updated: 2026-08-25
sources:
  - title: "ECMAScript Language Specification - Ordinary Object Internal Methods and Internal Slots"
    url: "https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinary-object-internal-methods-and-internal-slots"
    verified_at: 2026-08-10
  - title: "ECMAScript Language Specification - The new Operator"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-new-operator"
    verified_at: 2026-08-10
  - title: "ECMAScript Language Specification - Class Definitions"
    url: "https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-class-definitions"
    verified_at: 2026-08-10
  - title: "MDN - Inheritance and the prototype chain"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain"
    verified_at: 2026-08-10
  - title: "MDN - new operator"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new"
    verified_at: 2026-08-10
  - title: "MDN - Classes"
    url: "https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes"
    verified_at: 2026-08-10
---

# JavaScript 的原型链、new 和 class 是什么关系？

## 问题

请解释对象的 `[[Prototype]]`、构造函数的 `prototype` 属性、`new` 运算符和 `class` 之间的关系。为什么说 `class` 基于原型机制，但不能简单断言它“只是语法糖”？

> [!note]- 查看答案
>
> ## 30 秒口述答案
>
> 每个普通对象都有一个内部 `[[Prototype]]`，属性查找会从对象自身沿这条链一直查到 `null`。构造函数的公开 `prototype` 属性是另一个对象；执行 `new F()` 时，新实例的 `[[Prototype]]` 通常会指向 `F.prototype`，所以实例能共享其中的方法。`class` 的实例方法同样放在 `Class.prototype` 上，`extends` 也会建立原型关系，因此它基于原型模型。但类构造器必须用 `new`、类体默认严格模式、方法默认不可枚举，并且还有 `super`、私有字段等额外语义，所以不能把它粗暴等同于普通构造函数的纯语法替换。
>
> ## 详细原理
>
> ### `[[Prototype]]` 不等于 `.prototype`
>
> - `[[Prototype]]` 是对象的内部槽，值为另一个对象或 `null`。可以用 `Object.getPrototypeOf(obj)` 读取，用 `Object.create(proto)` 在创建对象时指定。
> - 可构造的普通函数通常具有自有的 `.prototype` 普通公开属性；箭头函数、对象方法和绑定函数等并不因此自动拥有该属性。构造函数的 `.prototype` 通常参与决定新实例的 `[[Prototype]]`。
> - 当 `F` 使用普通构造流程、`F.prototype` 是对象且构造函数没有返回替代对象时，典型关系是 `Object.getPrototypeOf(new F()) === F.prototype`，而不是“实例拥有一个等于构造函数的 `prototype` 属性”。
> - `__proto__` 是用于访问原型的历史兼容访问器，不应把它当成规范内部槽本身。新代码优先使用 `Object.getPrototypeOf`、`Object.create`；也应避免在热路径中动态调用 `Object.setPrototypeOf`，因为改变既有对象的原型可能破坏引擎优化。
>
> 当读取 `obj.key` 时，普通对象先检查自己的属性；若没有，再对其 `[[Prototype]]` 重复相同查找，直到找到属性或原型为 `null`。因此：
>
> - `Object.hasOwn(obj, key)` 只判断自有属性；`key in obj` 会包含原型链上的属性。
> - 给实例赋值通常会创建或更新自有属性，遮蔽原型上的同名属性，而不是直接修改原型属性。
> - 多个实例可以通过共同的原型共享方法，避免每次构造都创建一份同功能函数。
>
> ### `new` 触发构造语义
>
> `new F(...args)` 并不只是手写几行对象操作，它会检查目标是否具有 `[[Construct]]`，并调用规范的构造流程。对常见的普通基类构造函数，可以用下面的近似步骤理解：
>
> 1. 创建一个新对象。若 `F.prototype` 是对象，就将其作为新对象的 `[[Prototype]]`；否则回退到对应 realm 的 `%Object.prototype%`。
> 1. 以新对象作为构造期间的 `this`，传入参数执行 `F`。
> 1. 如果 `F` 显式返回一个对象，则 `new` 表达式采用该对象；如果返回原始值或没有显式返回，则采用最初创建的新对象。
>
> 这是帮助理解普通构造函数的模型，不是对所有构造器的等价实现。内置构造器、Proxy 和派生类都有额外的内部语义；派生类构造器必须先调用 `super()` 才能访问 `this`，因为实例由父类构造流程创建并初始化。
>
> `F.prototype` 被替换后，只影响之后以它为原型创建的实例；旧实例仍指向原来的原型对象。反过来，若只是修改原有 `F.prototype` 对象上的属性，已经存在的实例也可能通过原型链看到变化。
>
> ### `class` 建立在原型模型之上
>
> ```javascript
> class Parent {
>   speak() {
>     return 'parent';
>   }
> }
>
> class Child extends Parent {}
> ```
>
> 上例会建立两条相关但不同的继承链：
>
> - 实例方法链：`Object.getPrototypeOf(Child.prototype) === Parent.prototype`。
> - 构造器静态成员链：`Object.getPrototypeOf(Child) === Parent`。
>
> `Child` 的实例以 `Child.prototype` 为直接原型，并继续沿链访问 `Parent.prototype` 上的 `speak`。普通实例方法定义在 `.prototype` 上，静态方法定义在类构造器本身；实例字段则在构造期间初始化到每个实例上。
>
> 因此，说 `class` **基于原型和构造机制提供了更高层语法**是准确的；说它“只是语法糖、与普通构造函数完全一样”则会遗漏可观察差异：
>
> - 类构造器不带 `new` 调用会抛出 `TypeError`。
> - 类体始终使用严格模式。
> - 类声明的绑定具有类似暂时性死区的限制，不能像函数声明那样在初始化前调用。
> - 类语法定义的原型方法默认不可枚举，而直接赋值 `F.prototype.method = ...` 创建的属性默认可枚举。
> - 派生构造器受 `super()` 和 `this` 初始化规则约束。
> - 私有字段使用私有名称和品牌检查，不是原型链上的普通字符串属性，不能用简单的构造函数赋值完整模拟其语义。
>
> ### `instanceof` 不是类型的绝对证明
>
> 当右侧是普通非绑定构造函数且未自定义 `Symbol.hasInstance` 时，`value instanceof F` 会检查 `F.prototype` 是否出现在 `value` 的原型链中。绑定函数会把默认判断委托给它的目标函数，其他对象也可以通过 `Symbol.hasInstance` 自定义行为。不同 realm（例如不同 iframe）拥有各自的内建构造器和原型对象，所以跨 realm 的数组可能不满足当前 realm 的 `Array` 的 `instanceof`；这类判断应优先考虑 `Array.isArray` 等专用 API。
>
> ## JavaScript 示例（Node.js 23 或现代浏览器）
>
> ```javascript
> class Base {
>   speak() {
>     return this.name;
>   }
> }
>
> class User extends Base {
>   constructor(name) {
>     super();
>     this.name = name;
>   }
> }
>
> const user = new User('小杜');
>
> console.log(Object.getPrototypeOf(user) === User.prototype); // true
> console.log(Object.getPrototypeOf(User.prototype) === Base.prototype); // true
> console.log(Object.getPrototypeOf(User) === Base); // true
> console.log(Object.prototype.hasOwnProperty.call(user, 'name')); // true
> console.log(Object.prototype.hasOwnProperty.call(user, 'speak')); // false
> console.log(user.speak()); // 小杜：从 Base.prototype 找到方法
> console.log(Object.keys(Base.prototype)); // []：class 方法默认不可枚举
>
> function ReturnsObject() {
>   this.created = true;
>   return { replacement: true };
> }
>
> function ReturnsPrimitive() {
>   this.created = true;
>   return 1;
> }
>
> console.log(new ReturnsObject()); // { replacement: true }
> console.log(new ReturnsPrimitive()); // ReturnsPrimitive { created: true }
> ```
>
> 最后一行的对象展示格式可能因控制台不同而不同，但它的 `created` 属性为 `true`，且原型是 `ReturnsPrimitive.prototype`。
>
> ## 常见追问
>
> 1. **替换 `Constructor.prototype` 后，已有实例会怎样？** 已有实例仍指向旧原型对象；新实例通常指向新原型。若只修改原原型对象的属性，已有实例仍能沿原型链观察到修改。
> 1. **`Object.create(proto)` 与 `new Constructor()` 有什么区别？** 前者直接创建以 `proto` 为原型的对象，不执行构造函数；后者执行目标的 `[[Construct]]` 流程，包含参数传递、初始化和构造函数返回值规则。
> 1. **实例方法、静态方法和实例字段分别放在哪里？** 普通实例方法在 `Class.prototype` 上，静态方法在类构造器对象上，实例字段在每个实例自身；私有字段还有独立的私有名称与品牌检查语义。
>
> ## 易错点
>
> - 混淆实例的 `[[Prototype]]` 和构造函数的 `.prototype`，或认为实例直接拥有构造函数的 `prototype` 属性。
> - 手写 `new` 时忘记：目标必须可构造、`prototype` 非对象时需要回退，以及构造函数显式返回对象会替换默认实例。
> - 认为箭头函数可以作为构造器。箭头函数没有 `[[Construct]]`，也通常没有用于构造实例的自有 `prototype` 属性。
> - 把 `class` 粗暴描述成“纯语法糖”，忽略严格模式、不可直接调用、方法可枚举性、`super` 和私有字段等可观察语义。
> - 认为 `instanceof` 检查的是不可变的“类型标签”，忽略原型可变、`Symbol.hasInstance` 和跨 realm 场景。
> - 为了继承在运行时频繁修改对象原型；这既难以维护，也可能影响引擎优化。
>
> ## 项目关联
>
> 弱关联（**已实现代码**）：`/Users/dubobo/Desktop/my-ipaas/db/repository.ts` 使用 `Object.prototype.hasOwnProperty.call(next, key)`。它直接从 `Object.prototype` 取得方法再借给目标对象使用，体现了“方法可从原型对象取得，但调用接收者可以另行指定”。这种写法也不依赖目标对象是否把 `Object.prototype` 放在自己的原型链上。
>
> 当前项目没有自定义原型链或自定义类继承设计，该代码也不构成复杂面向对象架构案例；它只适合作为理解原型方法借用的弱关联，不能据此判断小杜已经掌握本题。
>
> ## 相关题目
>
> - [this-绑定规则与箭头函数](this-绑定规则与箭头函数.md)
> - [原始值与对象的赋值和传参](原始值与对象的赋值和传参.md)
> - [var-let-const-作用域提升与暂时性死区](var-let-const-作用域提升与暂时性死区.md)
>
> ## 参考资料
>
> - [ECMAScript Language Specification - Ordinary Object Internal Methods and Internal Slots](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinary-object-internal-methods-and-internal-slots)，核验于 2026-08-10。
> - [ECMAScript Language Specification - The new Operator](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-new-operator)，核验于 2026-08-10。
> - [ECMAScript Language Specification - Class Definitions](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-class-definitions)，核验于 2026-08-10。
> - [MDN - Inheritance and the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain)，核验于 2026-08-10。
> - [MDN - new operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)，核验于 2026-08-10。
> - [MDN - Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)，核验于 2026-08-10。
>