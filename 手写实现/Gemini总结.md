## 一、 原型与上下文机制

1. **手写 `bind` (或 `call`/`apply`)**
   - **考察核心**：闭包的运用、`this` 的强行绑定，以及**如何处理被 `new` 调用的优先级边界情况**。
   - **底层逻辑**：返回一个新函数，并在内部判断当前执行的 `this` 是否是被 `new` 实例化的（`this instanceof fBound`），以此决定是挂载给新对象还是预设的上下文。
2. **手写 `instanceof`**
   - **考察核心**：对 `[[Prototype]]` 隐式原型链查找机制的理解。
   - **底层逻辑**：一个 `while` 循环，不断向上读取左侧对象的 `__proto__`（或 `Object.getPrototypeOf`），直到它等于右侧构造函数的 `prototype`（返回 `true`），或查找到原型链尽头 `null`（返回 `false`）。

## 二、 异步并发控制

1. **手写 `Promise.all`**
   - **考察核心**：并发状态机、异步结果的时序映射。
   - **底层逻辑**：返回一个新 Promise。遍历传入的数组执行任务，用一个内部计数器记录完成数。**关键点在于按索引赋值**（而不是按完成先后 `push`），当计数器等于原数组长度时 `resolve`。任何一个 `reject` 则整体溃败。
2. **手写异步并发调度器 (Scheduler)**
   - **考察核心**：任务队列 (Queue) 管理、递归消费机制（大厂极高频压轴题）。
   - **底层逻辑**：维护一个等待队列数组和一个当前运行数字段。当运行数小于最大并发数时，将任务出队并执行；任务 `then/finally` 结束后，**主动触发下一次出队消费**，形成流水线。

## 三、 复杂数据处理

1. **手写深拷贝 (Deep Clone)**
   - **考察核心**：递归算法、边界数据类型处理（日期、正则），以及最关键的**循环引用**。
   - **底层逻辑**：递归遍历对象属性。必须引入一个 `WeakMap` 作为缓存字典，每次进入递归前先查缓存，如果该对象已经拷贝过，直接返回缓存引用，瞬间打破死循环。
2. **手写数组扁平化 (Array Flatten)**
   - **考察核心**：对多维嵌套数据结构的降维能力。
   - **底层逻辑**：基础解法是 `reduce` 配合递归。高端一点的解法是利用 `while(arr.some(Array.isArray))` 配合 `[].concat(...arr)` 进行逐层剥洋葱。

## 四、 进阶函数与设计模式

1. **手写发布订阅模式 (EventEmitter)**
   - **考察核心**：解耦组件通信的底层原理（Vue/React 状态机制的缩影）。
   - **底层逻辑**：在内部维护一个字典对象 `{ eventName: [fn1, fn2] }`。`on` 方法负责往数组里 `push` 回调，`emit` 负责遍历数组执行，`off` 负责用 `filter` 移除指定回调。
2. **手写函数柯里化 (Currying)**
   - **考察核心**：闭包的高级应用，延迟执行与参数收集。
   - **底层逻辑**：返回一个收集参数的闭包函数。内部判断：如果当前收集到的参数数量 `<` 原函数期望的参数数量（`fn.length`），则返回新函数继续收集；如果数量足够，则直接 `apply` 执行原函数。



# 实现

这些手写题的核心代码都是经过高度提炼的面试“黄金版本”，去掉了冗长多余的类型校验，只保留了最核心的算法与语言机制。

## 1. 手写 `bind`

**核心：** 闭包传参 + 识别 `new` 的优先级。

```javascript
Function.prototype.myBind = function(context, ...outArgs) {
  const fn = this; // 这里的 this 就是原函数
  
  if (typeof fn !== 'function') {
    throw new TypeError('调用 myBind 的必须是函数');
  }

  return function F(...innerArgs) {
    // 优先级判断：如果被 new 关键字调用，this 应该指向新创建的实例(即 F 的实例)
    if (this instanceof F) {
      return new fn(...outArgs, ...innerArgs);
    }
    // 否则作为普通函数调用，绑定到传入的 context
    return fn.apply(context, [...outArgs, ...innerArgs]);
  };
};
```

## 2. 手写 `instanceof`

**核心：** `while` 循环顺着 `__proto__` 向上爬，直到命中或触底。

```javascript
function myInstanceof(left, right) {
  // 基本数据类型直接返回 false
  if (left === null || (typeof left !== 'object' && typeof left !== 'function')) {
    return false;
  }
  
  let proto = Object.getPrototypeOf(left); // 获取隐式原型
  const prototype = right.prototype;       // 获取显式原型

  while (proto !== null) {
    if (proto === prototype) return true;  // 命中！
    proto = Object.getPrototypeOf(proto);  // 继续往上爬
  }
  
  return false; // 触底(null)未命中
}
```

## 3. 手写 `Promise.all`

**核心：** 结果按原数组索引赋值，计数器等于长度时放行，任何一个报错则全盘崩溃。

```javascript
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('参数必须是数组'));
    }

    const result = [];
    let count = 0;
    if (promises.length === 0) return resolve(result);

    promises.forEach((p, index) => {
      // 包装 Promise.resolve 以防数组里混入非 Promise 的基本类型
      Promise.resolve(p).then(res => {
        result[index] = res; // 必须按索引存，保证输出顺序与输入一致
        count++;
        if (count === promises.length) resolve(result);
      }).catch(err => {
        reject(err); // 一旦有一个失败，直接 reject
      });
    });
  });
}
```

## 4. 手写异步并发调度器 (Scheduler)

**核心：** 队列存任务，运行数做卡点。任务 `finally` 结束后主动拉取下一个。

```javascript
class Scheduler {
  constructor(maxLimit = 2) {
    this.maxLimit = maxLimit;
    this.activeCount = 0; // 当前正在运行的任务数
    this.queue = [];      // 等待队列
  }

  add(task) {
    return new Promise(resolve => {
      // 将任务和它的 resolve 绑定推入队列
      this.queue.push(() => task().then(resolve));
      this.run(); // 尝试触发
    });
  }

  run() {
    // 只要没达到上限，且队列里有活，就一直出队消费
    if (this.activeCount < this.maxLimit && this.queue.length > 0) {
      this.activeCount++;
      const task = this.queue.shift(); 
      
      task().finally(() => {
        this.activeCount--; // 任务完成，腾出空位
        this.run();         // 核心操作：主动拉取下一个任务！
      });
    }
  }
}
```

## 5. 手写深拷贝 (带循环引用处理)

**核心：** 递归 + `WeakMap` 作为查重字典。

```javascript
function deepClone(obj, map = new WeakMap()) {
  // 处理 null 或基本类型，以及函数
  if (obj === null || typeof obj !== 'object') return obj;
  
  // 处理特殊对象 (根据实际需求可补充 Error, Map, Set 等)
  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj);

  // 核心：处理循环引用！如果字典里查到已经拷贝过这个对象，直接返回
  if (map.has(obj)) return map.get(obj);

  // 初始化新对象或数组
  const cloneObj = Array.isArray(obj) ? [] : {};
  map.set(obj, cloneObj); // 记住自己，防止递归死循环

  for (let key in obj) {
    // 只拷贝对象自身的属性，不拷贝原型链上的
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      cloneObj[key] = deepClone(obj[key], map); // 递归
    }
  }
  
  return cloneObj;
}
```

## 6. 手写数组扁平化 (Array Flatten)

**核心：** `reduce` 配合递归。

```javascript
function flatten(arr) {
  return arr.reduce((acc, current) => {
    // 如果当前元素还是数组，就递归剥开；如果是值，就拼进去
    return acc.concat(Array.isArray(current) ? flatten(current) : current);
  }, []);
}

/* 
// 备用极其精简的 while 剥洋葱解法：
function flatten(arr) {
  while (arr.some(Array.isArray)) {
    arr = [].concat(...arr);
  }
  return arr;
}
*/
```

## 7. 手写发布订阅模式 (EventEmitter)

**核心：** 维护一个以事件名为 key，回调数组为 value 的字典。

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
  }

  on(eventName, callback) {
    if (!this.events[eventName]) {
      this.events[eventName] = [];
    }
    this.events[eventName].push(callback);
  }

  emit(eventName, ...args) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      callbacks.forEach(cb => cb.apply(this, args));
    }
  }

  off(eventName, callback) {
    const callbacks = this.events[eventName];
    if (callbacks) {
      // 过滤掉匹配的 callback
      this.events[eventName] = callbacks.filter(cb => cb !== callback);
    }
  }
}
```

## 8. 手写函数柯里化 (Currying)

**核心：** 比较“已收集参数的数量”与“原函数期望的形参数量（`fn.length`）”。

```javascript
function curry(fn) {
  // 返回一个闭包函数，负责收集参数
  return function curried(...args) {
    // 如果收到的参数够了，直接原路执行
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    } else {
      // 如果没够，返回一个新的函数继续收集
      return function(...moreArgs) {
        // 把之前存的 args 和新传进来的 moreArgs 拼起来，递归检查
        return curried.apply(this, args.concat(moreArgs));
      };
    }
  };
}

/*
// 测试案例：
function sum(a, b, c) { return a + b + c; }
const curriedSum = curry(sum);
console.log(curriedSum(1)(2)(3)); // 输出 6
*/
```