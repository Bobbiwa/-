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

<details>
<summary>查看答案</summary>
<h2>30 秒口述答案</h2>
<p>每个普通对象都有一个内部 <code>[[Prototype]]</code>，属性查找会从对象自身沿这条链一直查到 <code>null</code>。构造函数的公开 <code>prototype</code> 属性是另一个对象；执行 <code>new F()</code> 时，新实例的 <code>[[Prototype]]</code> 通常会指向 <code>F.prototype</code>，所以实例能共享其中的方法。<code>class</code> 的实例方法同样放在 <code>Class.prototype</code> 上，<code>extends</code> 也会建立原型关系，因此它基于原型模型。但类构造器必须用 <code>new</code>、类体默认严格模式、方法默认不可枚举，并且还有 <code>super</code>、私有字段等额外语义，所以不能把它粗暴等同于普通构造函数的纯语法替换。</p>
<h2>详细原理</h2>
<h3><code>[[Prototype]]</code> 不等于 <code>.prototype</code></h3>
<ul><li><code>[[Prototype]]</code> 是对象的内部槽，值为另一个对象或 <code>null</code>。可以用 <code>Object.getPrototypeOf(obj)</code> 读取，用 <code>Object.create(proto)</code> 在创建对象时指定。</li><li>可构造的普通函数通常具有自有的 <code>.prototype</code> 普通公开属性；箭头函数、对象方法和绑定函数等并不因此自动拥有该属性。构造函数的 <code>.prototype</code> 通常参与决定新实例的 <code>[[Prototype]]</code>。</li><li>当 <code>F</code> 使用普通构造流程、<code>F.prototype</code> 是对象且构造函数没有返回替代对象时，典型关系是 <code>Object.getPrototypeOf(new F()) === F.prototype</code>，而不是“实例拥有一个等于构造函数的 <code>prototype</code> 属性”。</li><li><code>__proto__</code> 是用于访问原型的历史兼容访问器，不应把它当成规范内部槽本身。新代码优先使用 <code>Object.getPrototypeOf</code>、<code>Object.create</code>；也应避免在热路径中动态调用 <code>Object.setPrototypeOf</code>，因为改变既有对象的原型可能破坏引擎优化。</li></ul>
<p>当读取 <code>obj.key</code> 时，普通对象先检查自己的属性；若没有，再对其 <code>[[Prototype]]</code> 重复相同查找，直到找到属性或原型为 <code>null</code>。因此：</p>
<ul><li><code>Object.hasOwn(obj, key)</code> 只判断自有属性；<code>key in obj</code> 会包含原型链上的属性。</li><li>给实例赋值通常会创建或更新自有属性，遮蔽原型上的同名属性，而不是直接修改原型属性。</li><li>多个实例可以通过共同的原型共享方法，避免每次构造都创建一份同功能函数。</li></ul>
<h3><code>new</code> 触发构造语义</h3>
<p><code>new F(...args)</code> 并不只是手写几行对象操作，它会检查目标是否具有 <code>[[Construct]]</code>，并调用规范的构造流程。对常见的普通基类构造函数，可以用下面的近似步骤理解：</p>
<ol><li>创建一个新对象。若 <code>F.prototype</code> 是对象，就将其作为新对象的 <code>[[Prototype]]</code>；否则回退到对应 realm 的 <code>%Object.prototype%</code>。</li><li>以新对象作为构造期间的 <code>this</code>，传入参数执行 <code>F</code>。</li><li>如果 <code>F</code> 显式返回一个对象，则 <code>new</code> 表达式采用该对象；如果返回原始值或没有显式返回，则采用最初创建的新对象。</li></ol>
<p>这是帮助理解普通构造函数的模型，不是对所有构造器的等价实现。内置构造器、Proxy 和派生类都有额外的内部语义；派生类构造器必须先调用 <code>super()</code> 才能访问 <code>this</code>，因为实例由父类构造流程创建并初始化。</p>
<p><code>F.prototype</code> 被替换后，只影响之后以它为原型创建的实例；旧实例仍指向原来的原型对象。反过来，若只是修改原有 <code>F.prototype</code> 对象上的属性，已经存在的实例也可能通过原型链看到变化。</p>
<h3><code>class</code> 建立在原型模型之上</h3>
<pre><code class="language-javascript">class Parent {&#10;  speak() {&#10;    return &#39;parent&#39;;&#10;  }&#10;}&#10;&#10;class Child extends Parent {}</code></pre>
<p>上例会建立两条相关但不同的继承链：</p>
<ul><li>实例方法链：<code>Object.getPrototypeOf(Child.prototype) === Parent.prototype</code>。</li><li>构造器静态成员链：<code>Object.getPrototypeOf(Child) === Parent</code>。</li></ul>
<p><code>Child</code> 的实例以 <code>Child.prototype</code> 为直接原型，并继续沿链访问 <code>Parent.prototype</code> 上的 <code>speak</code>。普通实例方法定义在 <code>.prototype</code> 上，静态方法定义在类构造器本身；实例字段则在构造期间初始化到每个实例上。</p>
<p>因此，说 <code>class</code> <strong>基于原型和构造机制提供了更高层语法</strong>是准确的；说它“只是语法糖、与普通构造函数完全一样”则会遗漏可观察差异：</p>
<ul><li>类构造器不带 <code>new</code> 调用会抛出 <code>TypeError</code>。</li><li>类体始终使用严格模式。</li><li>类声明的绑定具有类似暂时性死区的限制，不能像函数声明那样在初始化前调用。</li><li>类语法定义的原型方法默认不可枚举，而直接赋值 <code>F.prototype.method = ...</code> 创建的属性默认可枚举。</li><li>派生构造器受 <code>super()</code> 和 <code>this</code> 初始化规则约束。</li><li>私有字段使用私有名称和品牌检查，不是原型链上的普通字符串属性，不能用简单的构造函数赋值完整模拟其语义。</li></ul>
<h3><code>instanceof</code> 不是类型的绝对证明</h3>
<p>当右侧是普通非绑定构造函数且未自定义 <code>Symbol.hasInstance</code> 时，<code>value instanceof F</code> 会检查 <code>F.prototype</code> 是否出现在 <code>value</code> 的原型链中。绑定函数会把默认判断委托给它的目标函数，其他对象也可以通过 <code>Symbol.hasInstance</code> 自定义行为。不同 realm（例如不同 iframe）拥有各自的内建构造器和原型对象，所以跨 realm 的数组可能不满足当前 realm 的 <code>Array</code> 的 <code>instanceof</code>；这类判断应优先考虑 <code>Array.isArray</code> 等专用 API。</p>
<h2>JavaScript 示例（Node.js 23 或现代浏览器）</h2>
<pre><code class="language-javascript">class Base {&#10;  speak() {&#10;    return this.name;&#10;  }&#10;}&#10;&#10;class User extends Base {&#10;  constructor(name) {&#10;    super();&#10;    this.name = name;&#10;  }&#10;}&#10;&#10;const user = new User(&#39;小杜&#39;);&#10;&#10;console.log(Object.getPrototypeOf(user) === User.prototype); // true&#10;console.log(Object.getPrototypeOf(User.prototype) === Base.prototype); // true&#10;console.log(Object.getPrototypeOf(User) === Base); // true&#10;console.log(Object.prototype.hasOwnProperty.call(user, &#39;name&#39;)); // true&#10;console.log(Object.prototype.hasOwnProperty.call(user, &#39;speak&#39;)); // false&#10;console.log(user.speak()); // 小杜：从 Base.prototype 找到方法&#10;console.log(Object.keys(Base.prototype)); // []：class 方法默认不可枚举&#10;&#10;function ReturnsObject() {&#10;  this.created = true;&#10;  return { replacement: true };&#10;}&#10;&#10;function ReturnsPrimitive() {&#10;  this.created = true;&#10;  return 1;&#10;}&#10;&#10;console.log(new ReturnsObject()); // { replacement: true }&#10;console.log(new ReturnsPrimitive()); // ReturnsPrimitive { created: true }</code></pre>
<p>最后一行的对象展示格式可能因控制台不同而不同，但它的 <code>created</code> 属性为 <code>true</code>，且原型是 <code>ReturnsPrimitive.prototype</code>。</p>
<h2>常见追问</h2>
<ol><li><strong>替换 <code>Constructor.prototype</code> 后，已有实例会怎样？</strong> 已有实例仍指向旧原型对象；新实例通常指向新原型。若只修改原原型对象的属性，已有实例仍能沿原型链观察到修改。</li><li><strong><code>Object.create(proto)</code> 与 <code>new Constructor()</code> 有什么区别？</strong> 前者直接创建以 <code>proto</code> 为原型的对象，不执行构造函数；后者执行目标的 <code>[[Construct]]</code> 流程，包含参数传递、初始化和构造函数返回值规则。</li><li><strong>实例方法、静态方法和实例字段分别放在哪里？</strong> 普通实例方法在 <code>Class.prototype</code> 上，静态方法在类构造器对象上，实例字段在每个实例自身；私有字段还有独立的私有名称与品牌检查语义。</li></ol>
<h2>易错点</h2>
<ul><li>混淆实例的 <code>[[Prototype]]</code> 和构造函数的 <code>.prototype</code>，或认为实例直接拥有构造函数的 <code>prototype</code> 属性。</li><li>手写 <code>new</code> 时忘记：目标必须可构造、<code>prototype</code> 非对象时需要回退，以及构造函数显式返回对象会替换默认实例。</li><li>认为箭头函数可以作为构造器。箭头函数没有 <code>[[Construct]]</code>，也通常没有用于构造实例的自有 <code>prototype</code> 属性。</li><li>把 <code>class</code> 粗暴描述成“纯语法糖”，忽略严格模式、不可直接调用、方法可枚举性、<code>super</code> 和私有字段等可观察语义。</li><li>认为 <code>instanceof</code> 检查的是不可变的“类型标签”，忽略原型可变、<code>Symbol.hasInstance</code> 和跨 realm 场景。</li><li>为了继承在运行时频繁修改对象原型；这既难以维护，也可能影响引擎优化。</li></ul>
<h2>项目关联</h2>
<p>弱关联（<strong>已实现代码</strong>）：<code>/Users/dubobo/Desktop/my-ipaas/db/repository.ts</code> 使用 <code>Object.prototype.hasOwnProperty.call(next, key)</code>。它直接从 <code>Object.prototype</code> 取得方法再借给目标对象使用，体现了“方法可从原型对象取得，但调用接收者可以另行指定”。这种写法也不依赖目标对象是否把 <code>Object.prototype</code> 放在自己的原型链上。</p>
<p>当前项目没有自定义原型链或自定义类继承设计，该代码也不构成复杂面向对象架构案例；它只适合作为理解原型方法借用的弱关联，不能据此判断小杜已经掌握本题。</p>
<h2>相关题目</h2>
<ul><li><a href="this-绑定规则与箭头函数.md">this-绑定规则与箭头函数</a></li><li><a href="原始值与对象的赋值和传参.md">原始值与对象的赋值和传参</a></li><li><a href="var-let-const-作用域提升与暂时性死区.md">var-let-const-作用域提升与暂时性死区</a></li></ul>
<h2>参考资料</h2>
<ul><li><a href="https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinary-object-internal-methods-and-internal-slots">ECMAScript Language Specification - Ordinary Object Internal Methods and Internal Slots</a>，核验于 2026-08-10。</li><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html#sec-new-operator">ECMAScript Language Specification - The new Operator</a>，核验于 2026-08-10。</li><li><a href="https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#sec-class-definitions">ECMAScript Language Specification - Class Definitions</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain">MDN - Inheritance and the prototype chain</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new">MDN - new operator</a>，核验于 2026-08-10。</li><li><a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes">MDN - Classes</a>，核验于 2026-08-10。</li></ul>
</details>