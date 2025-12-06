# MessageChannel

## 一、【30字核心】

**MessageChannel 是浏览器提供的双向通信通道，基于事件驱动的宏任务机制，是 React Scheduler 实现时间切片调度的核心。**

---

## 二、【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### MessageChannel 的第一性原理 🎯

#### 1. 最基础的定义

**MessageChannel = 两个相互连接的端口（port1 和 port2）**

仅此而已！没有更基础的了。

当你创建一个 MessageChannel 实例时，它会自动生成两个端口，这两个端口可以互相发送消息。一个端口发送的消息，另一个端口可以接收到。

```javascript
const channel = new MessageChannel();
// channel.port1 和 channel.port2 自动创建
// port1 发送 → port2 接收
// port2 发送 → port1 接收
```

#### 2. 为什么需要 MessageChannel？

**核心问题：如何在 JavaScript 中实现异步任务调度，且不阻塞浏览器渲染？**

JavaScript 是单线程的，如果我们执行一个长任务，浏览器会卡顿。理想的方案是：
- 将长任务拆分成小块
- 每执行一小块后，让出线程给浏览器
- 浏览器完成渲染后，再继续执行下一小块

但如何"让出线程"？我们需要一个异步机制：
- ❌ **微任务（Promise、MutationObserver）**：会在当前宏任务结束后立即执行，效果和同步一样，无法让出线程
- ❌ **setTimeout(fn, 0)**：有 4ms 的最小延迟（浏览器规范），且优先级低
- ✅ **MessageChannel**：没有最小延迟，优先级合适，是完美的宏任务方案

#### 3. MessageChannel 的三层价值

##### 价值1：实现真正的异步调度 🚀

MessageChannel 创建的是宏任务，会在当前宏任务和微任务都执行完后，在下一个事件循环中执行。

```javascript
console.log('1: 同步代码');

const channel = new MessageChannel();
channel.port1.onmessage = () => {
  console.log('3: MessageChannel 宏任务');
};

Promise.resolve().then(() => {
  console.log('2: Promise 微任务');
});

channel.port2.postMessage(null);

console.log('1.5: 同步代码结束');

// 输出顺序：
// 1: 同步代码
// 1.5: 同步代码结束
// 2: Promise 微任务
// 3: MessageChannel 宏任务
```

##### 价值2：零延迟的宏任务 ⚡

setTimeout 有 4ms 的最小延迟限制，而 MessageChannel 没有这个限制。

```javascript
const start = performance.now();

// setTimeout 方式
setTimeout(() => {
  console.log(`setTimeout 延迟: ${performance.now() - start}ms`);
  // 通常是 4-5ms
}, 0);

// MessageChannel 方式
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  console.log(`MessageChannel 延迟: ${performance.now() - start}ms`);
  // 通常是 0-1ms
};
channel.port2.postMessage(null);
```

##### 价值3：React 时间切片的基础 🎮

React 的 Scheduler 使用 MessageChannel 来实现任务调度。每个任务执行 5ms 后，通过 MessageChannel 调度下一个任务，给浏览器留出时间渲染。

```javascript
// React Scheduler 的简化逻辑
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = () => {
  // 执行调度任务
  performWorkUntilDeadline();
};

function scheduleCallback(callback) {
  // 将任务放入队列
  taskQueue.push(callback);

  // 通过 MessageChannel 触发执行
  port.postMessage(null);
}
```

#### 4. 从第一性原理推导 React 实现

**推理链：**
```
1. JavaScript 是单线程，长任务会阻塞浏览器渲染
   ↓
2. 需要将长任务拆分成小块，执行一块后让出线程
   ↓
3. 让出线程需要异步机制（宏任务）
   ↓
4. 微任务（Promise）无法让出线程（会在当前宏任务后立即执行）
   ↓
5. setTimeout 有 4ms 延迟，且在浏览器节流后可能延迟更久
   ↓
6. MessageChannel 是宏任务，零延迟，优先级合适
   ↓
7. React Scheduler 选择 MessageChannel 实现时间切片调度
```

#### 5. 一句话总结第一性原理

**MessageChannel 是双端口通信机制，通过宏任务实现零延迟的异步调度，解决了 React 需要频繁让出线程而 setTimeout 延迟过高的问题。**

---

## 三、【3个核心概念】

### 核心概念1：port1 和 port2 双端口 🔗

**MessageChannel 创建两个相互连接的端口，它们可以互相发送和接收消息。**

```javascript
// 创建消息通道
const channel = new MessageChannel();

// port1 接收消息
channel.port1.onmessage = (event) => {
  console.log('port1 收到消息:', event.data);
};

// port2 发送消息
channel.port2.postMessage('Hello from port2!');

// 输出: port1 收到消息: Hello from port2!
```

**详细解释：**

MessageChannel 的两个端口是完全对等的：
- port1 可以发送消息给 port2，也可以接收来自 port2 的消息
- port2 可以发送消息给 port1，也可以接收来自 port1 的消息
- 两个端口就像对讲机的两端，双向通信

**在 React 源码中的应用：**

React Scheduler 只用了单向通信（port2 发送，port1 接收），因为它只需要"触发下一个任务执行"，不需要真正传递数据。

```javascript
// React Scheduler 源码简化
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = performWorkUntilDeadline;

// 调度任务
function schedulePerformWorkUntilDeadline() {
  port.postMessage(null); // 只是触发，不传数据
}
```

---

### 核心概念2：宏任务特性 📦

**MessageChannel 的消息处理是宏任务，在事件循环的宏任务阶段执行，晚于微任务。**

```javascript
console.log('1: 同步开始');

// 宏任务：MessageChannel
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  console.log('4: MessageChannel 宏任务');
};
channel.port2.postMessage(null);

// 微任务：Promise
Promise.resolve().then(() => {
  console.log('3: Promise 微任务');
});

// 宏任务：setTimeout
setTimeout(() => {
  console.log('5: setTimeout 宏任务');
}, 0);

console.log('2: 同步结束');

// 输出顺序：
// 1: 同步开始
// 2: 同步结束
// 3: Promise 微任务
// 4: MessageChannel 宏任务
// 5: setTimeout 宏任务
```

**详细解释：**

JavaScript 事件循环的执行顺序：
1. 执行同步代码
2. 执行所有微任务（Promise、MutationObserver）
3. 执行一个宏任务（MessageChannel、setTimeout、setInterval）
4. 重复 2-3

MessageChannel 作为宏任务的优势：
- 晚于微任务执行，给了 React 处理状态更新的机会
- 早于 setTimeout 执行（没有 4ms 延迟）
- 优先级适中，不会饿死其他任务

**在 React 源码中的应用：**

React 在状态更新后，会先将更新放入队列（同步操作），然后通过 MessageChannel 触发调度（宏任务），这样可以批量处理多个更新。

```javascript
// 状态更新（同步）
setState(newValue);

// 批量更新标记（微任务）
Promise.resolve().then(() => {
  // 合并所有状态更新
});

// 触发渲染（宏任务）
scheduleCallback(() => {
  // 执行实际渲染
});
```

---

### 核心概念3：postMessage 与 onmessage 🔔

**postMessage 用于发送消息，onmessage 用于接收消息，消息通过 event.data 传递。**

```javascript
const channel = new MessageChannel();

// 设置消息接收器
channel.port1.onmessage = (event) => {
  console.log('接收到的数据类型:', typeof event.data);
  console.log('接收到的数据:', event.data);
};

// 发送不同类型的数据
channel.port2.postMessage('字符串');
channel.port2.postMessage(42);
channel.port2.postMessage({ name: 'React', version: 19 });
channel.port2.postMessage([1, 2, 3]);

// 输出:
// 接收到的数据类型: string
// 接收到的数据: 字符串
// 接收到的数据类型: number
// 接收到的数据: 42
// 接收到的数据类型: object
// 接收到的数据: { name: 'React', version: 19 }
// 接收到的数据类型: object
// 接收到的数据: [1, 2, 3]
```

**详细解释：**

postMessage 的数据传递机制：
- 数据会被**结构化克隆**（structured clone），而不是传递引用
- 支持大多数 JavaScript 类型（字符串、数字、对象、数组等）
- 不支持函数、Symbol、DOM 节点等

```javascript
const channel = new MessageChannel();

const obj = { count: 0 };

channel.port1.onmessage = (event) => {
  event.data.count = 100;
  console.log('接收端修改:', event.data.count); // 100
};

channel.port2.postMessage(obj);

console.log('发送端原对象:', obj.count); // 0（未被修改）
```

**在 React 源码中的应用：**

React Scheduler 实际上不传递数据，只用 `postMessage(null)` 来触发回调执行。

```javascript
// React Scheduler 只关心"触发执行"，不关心数据传递
port.postMessage(null);

// 接收端
channel.port1.onmessage = () => {
  // 不使用 event.data
  performWorkUntilDeadline();
};
```

---

## 四、【最小可用】

掌握以下内容，就能理解 React 源码中 MessageChannel 的核心应用：

### 4.1 基础用法：创建和使用 MessageChannel

```javascript
// 1. 创建通道
const channel = new MessageChannel();

// 2. 设置接收器
channel.port1.onmessage = (event) => {
  console.log('收到消息:', event.data);
};

// 3. 发送消息
channel.port2.postMessage('Hello!');
```

### 4.2 宏任务调度：实现异步执行

```javascript
// 将任务推迟到下一个宏任务
function nextTick(callback) {
  const channel = new MessageChannel();
  channel.port1.onmessage = callback;
  channel.port2.postMessage(null);
}

// 使用
console.log('1: 同步代码');
nextTick(() => {
  console.log('3: 宏任务');
});
console.log('2: 同步代码');

// 输出: 1 → 2 → 3
```

### 4.3 React Scheduler 模式：任务队列调度

```javascript
// 任务队列
const taskQueue = [];

// 创建调度器
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  if (taskQueue.length > 0) {
    const task = taskQueue.shift();
    task();
  }
};

// 调度任务
function scheduleTask(task) {
  taskQueue.push(task);
  channel.port2.postMessage(null);
}

// 使用
scheduleTask(() => console.log('任务1'));
scheduleTask(() => console.log('任务2'));
scheduleTask(() => console.log('任务3'));

// 输出: 任务1 → 任务2 → 任务3 (分别在不同的宏任务中执行)
```

**这些知识足以：**
- 理解 MessageChannel 的基本工作原理
- 理解为什么 React 选择它而不是 setTimeout
- 为后续学习 React Scheduler 源码打基础

---

## 五、【1个类比】

### 类比1：MessageChannel = 对讲机系统 📻

**MessageChannel 就像两个对讲机，port1 和 port2 是两个对讲机终端，按下 port2 的发送键（postMessage），port1 就会响起（onmessage）。**

**举例：**

想象你在一个工地上，有两个对讲机：
- **port1**：工头手里的对讲机（接收指令）
- **port2**：调度员手里的对讲机（发送指令）

调度员按下发送键："开始浇筑混凝土"（postMessage）
工头听到后执行任务（onmessage 触发）

```javascript
const channel = new MessageChannel();

// 工头设置接收器
channel.port1.onmessage = (event) => {
  console.log(`工头收到指令: ${event.data}`);
  // 执行任务...
};

// 调度员发送指令
channel.port2.postMessage('开始浇筑混凝土');

// 输出: 工头收到指令: 开始浇筑混凝土
```

---

### 类比2：宏任务 = 快递配送 📦

**MessageChannel（宏任务）、Promise（微任务）、同步代码就像快递配送的三种方式。**

| 配送方式 | 对应机制 | 特点 |
|---------|---------|------|
| 你立即拿在手里 | 同步代码 | 立即执行，阻塞后续操作 |
| 快递员放在你家门口 | 微任务（Promise） | 当前事情做完立即取，不用等配送员 |
| 快递放在快递柜 | 宏任务（MessageChannel） | 需要等一个配送周期，但比送货上门快 |
| 送货上门（签收） | setTimeout | 需要等配送员上门，最慢 |

```javascript
console.log('我拿到手里的东西'); // 立即拿到

Promise.resolve().then(() => {
  console.log('我去门口取快递'); // 做完手头的事就去取
});

const channel = new MessageChannel();
channel.port1.onmessage = () => {
  console.log('我去快递柜取快递'); // 下一个配送周期
};
channel.port2.postMessage(null);

setTimeout(() => {
  console.log('快递员送货上门'); // 等配送员上门（4ms+）
}, 0);
```

---

### 类比3：React Scheduler = 游戏引擎的帧循环 🎮

**React 的时间切片调度就像游戏引擎每帧的更新循环。**

游戏引擎的每一帧（16.6ms）：
1. **处理输入**（用户点击、键盘）→ React 的事件处理
2. **更新游戏状态**（计算位置、碰撞）→ React 的协调（Reconciliation）
3. **渲染画面**（绘制到屏幕）→ React 的提交（Commit）
4. **等待下一帧**（通过 MessageChannel）→ React 的调度

```javascript
// 游戏引擎的帧循环（简化版）
const channel = new MessageChannel();
let frameCount = 0;

channel.port1.onmessage = () => {
  const startTime = performance.now();

  // 1. 更新游戏状态（类似 React Reconciliation）
  updateGameState();

  // 2. 渲染画面（类似 React Commit）
  render();

  const elapsed = performance.now() - startTime;
  console.log(`第 ${++frameCount} 帧耗时: ${elapsed.toFixed(2)}ms`);

  // 3. 调度下一帧
  channel.port2.postMessage(null);
};

// 启动游戏循环
channel.port2.postMessage(null);

function updateGameState() {
  // 更新游戏逻辑（5ms 预算）
}

function render() {
  // 渲染画面（11.6ms 预算）
}
```

在 React 中，MessageChannel 扮演"下一帧"的触发器：
- 执行一小段渲染任务（5ms）
- 让出线程给浏览器渲染（11.6ms）
- 触发下一个任务（通过 MessageChannel）

---

### 类比4：时间切片 = 做饭时穿插洗碗 🍳

**React 的时间切片就像你在做饭时，每做一会儿菜就去洗一个碗，避免做完所有菜后碗堆成山。**

不用时间切片（传统方式）：
```
做菜1（10分钟）→ 做菜2（10分钟）→ 做菜3（10分钟）→ 洗碗（30分钟）
总共70分钟，前30分钟厨房很乱
```

用时间切片（React 方式）：
```
做菜1（10分钟）→ 洗碗（5分钟）
→ 做菜2（10分钟）→ 洗碗（5分钟）
→ 做菜3（10分钟）→ 洗碗（5分钟）
总共45分钟，厨房一直保持整洁
```

```javascript
// 不用时间切片：一次性渲染 1000 个组件
function renderAllAtOnce() {
  for (let i = 0; i < 1000; i++) {
    renderComponent(i);
  }
  // 页面卡顿！
}

// 用时间切片：每次渲染 10 个，分批执行
const channel = new MessageChannel();
let index = 0;

channel.port1.onmessage = () => {
  const startTime = performance.now();

  // 执行 5ms 的任务
  while (index < 1000 && performance.now() - startTime < 5) {
    renderComponent(index++);
  }

  if (index < 1000) {
    // 还有任务，调度下一批
    channel.port2.postMessage(null);
  }
};

// 启动
channel.port2.postMessage(null);
```

---

### 类比总结表

| React 概念 | 生活场景 | 关键相似点 |
|-----------|---------|----------|
| MessageChannel 双端口 | 对讲机的两个终端 | 双向通信，互相发送消息 |
| 宏任务 vs 微任务 | 快递柜 vs 门口快递 | 执行时机不同，延迟不同 |
| 时间切片调度 | 游戏引擎帧循环 | 每帧执行一部分，保持流畅 |
| 分批渲染 | 做饭穿插洗碗 | 避免长任务阻塞，保持响应 |

---

## 六、【反直觉点】

### 误区1：MessageChannel 是微任务 ❌

**为什么错？**
- MessageChannel 创建的是**宏任务**（Macro Task），不是微任务（Micro Task）
- 宏任务在微任务之后执行
- 这是 MessageChannel 的核心优势之一

**为什么人们容易这样错？**

因为 MessageChannel 常用于异步通信，而我们最熟悉的异步机制是 Promise（微任务）。人们容易将"异步"等同于"微任务"，忽略了宏任务和微任务的区别。

**正确理解：**

```javascript
console.log('1: 同步代码');

// 微任务
Promise.resolve().then(() => {
  console.log('2: Promise 微任务');
});

// 宏任务
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  console.log('3: MessageChannel 宏任务');
};
channel.port2.postMessage(null);

// 输出顺序：
// 1: 同步代码
// 2: Promise 微任务
// 3: MessageChannel 宏任务

// 关键：微任务先于宏任务执行！
```

---

### 误区2：MessageChannel 比 setTimeout 慢 ❌

**为什么错？**
- MessageChannel **比 setTimeout 快**
- setTimeout 有 4ms 的最小延迟（浏览器规范）
- MessageChannel 没有最小延迟，几乎立即执行

**为什么人们容易这样错？**

setTimeout 是最常用的异步 API，人们习惯了它的执行速度。看到一个新的 API（MessageChannel），会潜意识认为"肯定没有 setTimeout 快"。

**正确理解：**

```javascript
function testDelay(name, executor) {
  const start = performance.now();
  executor(() => {
    const delay = performance.now() - start;
    console.log(`${name} 延迟: ${delay.toFixed(2)}ms`);
  });
}

// 测试 setTimeout
testDelay('setTimeout', (callback) => {
  setTimeout(callback, 0);
});
// 输出: setTimeout 延迟: 4.20ms （至少 4ms）

// 测试 MessageChannel
testDelay('MessageChannel', (callback) => {
  const channel = new MessageChannel();
  channel.port1.onmessage = callback;
  channel.port2.postMessage(null);
});
// 输出: MessageChannel 延迟: 0.50ms （几乎立即）
```

这就是为什么 React 选择 MessageChannel 而不是 setTimeout：
- 更快的响应速度
- 更精确的时间控制

---

### 误区3：MessageChannel 只能在 Web Worker 中使用 ❌

**为什么错？**
- MessageChannel 可以在**主线程**和 **Web Worker** 中使用
- React Scheduler 就在主线程中使用 MessageChannel
- Worker 之间的通信是 MessageChannel 的另一个用途，但不是唯一用途

**为什么人们容易这样错？**

MessageChannel 的官方文档和教程经常强调它在 Worker 通信中的应用，导致人们误以为它"只能"或"主要"用于 Worker。实际上，在主线程中实现异步调度是它的另一个重要用途。

**正确理解：**

```javascript
// ===== 用途1：主线程中的异步调度（React Scheduler） =====
const channel = new MessageChannel();
channel.port1.onmessage = () => {
  console.log('主线程中的异步任务');
};
channel.port2.postMessage(null);

// ===== 用途2：Worker 通信 =====
// 主线程
const worker = new Worker('worker.js');
const channel2 = new MessageChannel();

// 将 port2 传递给 Worker
worker.postMessage({ port: channel2.port2 }, [channel2.port2]);

// 主线程通过 port1 通信
channel2.port1.onmessage = (event) => {
  console.log('从 Worker 收到:', event.data);
};
channel2.port1.postMessage('Hello Worker!');

// worker.js
self.onmessage = (event) => {
  const port = event.data.port;

  port.onmessage = (e) => {
    console.log('Worker 收到:', e.data);
    port.postMessage('Hello Main Thread!');
  };
};
```

React Scheduler 完全不涉及 Worker，纯粹在主线程中使用 MessageChannel 进行任务调度。

---

## 七、【实战代码】

### 基础实现（简化版）

```javascript
// ===== 1. 场景1：基础使用 =====
console.log("=== 场景1：MessageChannel 基础使用 ===");

const channel = new MessageChannel();

// 设置 port1 的消息接收器
channel.port1.onmessage = (event) => {
  console.log(`port1 收到消息: ${event.data}`);
};

// port2 发送消息
channel.port2.postMessage('Hello from port2!');

// 输出:
// port1 收到消息: Hello from port2!

// ===== 2. 场景2：双向通信 =====
console.log("\n=== 场景2：双向通信 ===");

const channel2 = new MessageChannel();

// port1 接收并回复
channel2.port1.onmessage = (event) => {
  console.log(`port1 收到: ${event.data}`);
  channel2.port1.postMessage('port1 收到，回复你！');
};

// port2 接收并发起
channel2.port2.onmessage = (event) => {
  console.log(`port2 收到: ${event.data}`);
};

// port2 发起通信
channel2.port2.postMessage('port2 发起通信');

// 输出:
// port1 收到: port2 发起通信
// port2 收到: port1 收到，回复你！

// ===== 3. 场景3：宏任务执行顺序 =====
console.log("\n=== 场景3：宏任务执行顺序 ===");

console.log('1: 同步开始');

const channel3 = new MessageChannel();
channel3.port1.onmessage = () => {
  console.log('4: MessageChannel 宏任务');
};
channel3.port2.postMessage(null);

Promise.resolve().then(() => {
  console.log('3: Promise 微任务');
});

setTimeout(() => {
  console.log('5: setTimeout 宏任务');
}, 0);

console.log('2: 同步结束');

// 输出顺序:
// 1: 同步开始
// 2: 同步结束
// 3: Promise 微任务
// 4: MessageChannel 宏任务
// 5: setTimeout 宏任务

// ===== 4. 场景4：任务队列调度（模拟 React Scheduler） =====
console.log("\n=== 场景4：任务队列调度 ===");

class SimpleScheduler {
  constructor() {
    this.taskQueue = [];
    this.isScheduled = false;

    // 创建 MessageChannel
    this.channel = new MessageChannel();
    this.channel.port1.onmessage = () => {
      this.flushWork();
    };
  }

  // 调度任务
  scheduleTask(task) {
    this.taskQueue.push(task);

    if (!this.isScheduled) {
      this.isScheduled = true;
      // 触发宏任务
      this.channel.port2.postMessage(null);
    }
  }

  // 执行任务
  flushWork() {
    this.isScheduled = false;

    while (this.taskQueue.length > 0) {
      const task = this.taskQueue.shift();
      task();
    }
  }
}

// 使用调度器
const scheduler = new SimpleScheduler();

scheduler.scheduleTask(() => console.log('任务1 执行'));
scheduler.scheduleTask(() => console.log('任务2 执行'));
scheduler.scheduleTask(() => console.log('任务3 执行'));

console.log('所有任务已调度');

// 输出:
// 所有任务已调度
// 任务1 执行
// 任务2 执行
// 任务3 执行

// ===== 5. 场景5：时间切片（每 5ms 执行一批任务） =====
console.log("\n=== 场景5：时间切片 ===");

class TimeSlicingScheduler {
  constructor() {
    this.taskQueue = [];
    this.channel = new MessageChannel();
    this.channel.port1.onmessage = () => {
      this.workLoop();
    };
  }

  scheduleTask(task) {
    this.taskQueue.push(task);
    this.channel.port2.postMessage(null);
  }

  workLoop() {
    const startTime = performance.now();
    const timeSlice = 5; // 5ms 时间片

    // 在时间片内执行尽可能多的任务
    while (
      this.taskQueue.length > 0 &&
      performance.now() - startTime < timeSlice
    ) {
      const task = this.taskQueue.shift();
      task();
    }

    // 如果还有任务，调度下一个时间片
    if (this.taskQueue.length > 0) {
      console.log(`时间片结束，剩余 ${this.taskQueue.length} 个任务`);
      this.channel.port2.postMessage(null);
    }
  }
}

// 使用时间切片调度器
const tsScheduler = new TimeSlicingScheduler();

// 添加 20 个任务
for (let i = 1; i <= 20; i++) {
  tsScheduler.scheduleTask(() => {
    console.log(`执行任务 ${i}`);
    // 模拟耗时操作
    const start = performance.now();
    while (performance.now() - start < 1) {}
  });
}

// 输出:
// 执行任务 1
// 执行任务 2
// ...
// 执行任务 5
// 时间片结束，剩余 15 个任务
// 执行任务 6
// ...
```

**运行输出示例：**

```
=== 场景1：MessageChannel 基础使用 ===
port1 收到消息: Hello from port2!

=== 场景2：双向通信 ===
port1 收到: port2 发起通信
port2 收到: port1 收到，回复你！

=== 场景3：宏任务执行顺序 ===
1: 同步开始
2: 同步结束
3: Promise 微任务
4: MessageChannel 宏任务
5: setTimeout 宏任务

=== 场景4：任务队列调度 ===
所有任务已调度
任务1 执行
任务2 执行
任务3 执行

=== 场景5：时间切片 ===
执行任务 1
执行任务 2
执行任务 3
执行任务 4
执行任务 5
时间片结束，剩余 15 个任务
执行任务 6
...
```

---

### 进阶：React 源码实现

```javascript
// React Scheduler 源码片段（packages/scheduler/src/forks/Scheduler.js）
// 以下是简化版，展示核心逻辑

let isMessageLoopRunning = false;
let scheduledHostCallback = null;
let taskQueue = [];
let currentTask = null;

const channel = new MessageChannel();
const port = channel.port2;

// 消息处理函数
channel.port1.onmessage = performWorkUntilDeadline;

function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = performance.now();

    // 设置截止时间（5ms 后）
    const deadline = currentTime + 5;

    const hasTimeRemaining = true;

    try {
      // 执行调度的回调
      const hasMoreWork = scheduledHostCallback(
        hasTimeRemaining,
        currentTime
      );

      if (!hasMoreWork) {
        // 没有更多工作了
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        // 还有工作，调度下一个宏任务
        port.postMessage(null);
      }
    } catch (error) {
      // 出错了，但仍然调度下一个任务
      port.postMessage(null);
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  }
}

// 请求主机回调（开始调度）
function requestHostCallback(callback) {
  scheduledHostCallback = callback;

  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    port.postMessage(null);
  }
}

// 取消主机回调
function cancelHostCallback() {
  scheduledHostCallback = null;
}

// 使用示例
function workLoop(hasTimeRemaining, currentTime) {
  // 执行任务队列中的任务
  currentTask = taskQueue[0];

  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime && !hasTimeRemaining) {
      // 时间不够了，退出
      break;
    }

    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;

      // 执行任务
      const continuationCallback = callback(didUserCallbackTimeout);

      if (typeof continuationCallback === 'function') {
        // 任务还没完成，继续
        currentTask.callback = continuationCallback;
      } else {
        // 任务完成，移除
        if (currentTask === taskQueue[0]) {
          taskQueue.shift();
        }
      }

      currentTask = taskQueue[0];
    } else {
      taskQueue.shift();
      currentTask = taskQueue[0];
    }
  }

  // 返回是否还有更多工作
  return currentTask !== null;
}

// 开始调度
requestHostCallback(workLoop);
```

**React 源码关键点：**

1. **MessageChannel 的创建**：在模块加载时创建，全局唯一
2. **port.postMessage(null)**：只是触发，不传递数据
3. **performWorkUntilDeadline**：每个宏任务的入口
4. **5ms 时间片**：每次最多执行 5ms，然后让出线程
5. **hasMoreWork**：如果还有任务，继续调度下一个宏任务

---

## 八、【面试必问】

### 问题："React 为什么选择 MessageChannel 而不是 setTimeout 实现任务调度？"

**普通回答（❌ 不出彩）：**

"因为 MessageChannel 更快，setTimeout 有延迟。"

**出彩回答（✅ 推荐）：**

> **React 选择 MessageChannel 主要基于三个原因：**
>
> 1. **执行时机**：MessageChannel 创建的是宏任务，会在当前宏任务和所有微任务执行完后立即执行，而 setTimeout 有 4ms 的最小延迟（HTML5 规范规定）。对于需要频繁调度的 Scheduler 来说，4ms 的延迟累积起来会严重影响性能。
>
> 2. **浏览器节流**：setTimeout 在后台标签页或嵌套调用超过 5 次后会被浏览器节流，延迟可能达到 1000ms。而 MessageChannel 不受这些限制，调度更加可靠。
>
> 3. **优先级控制**：MessageChannel 的执行时机在微任务之后、其他宏任务之前，优先级适中。这让 React 可以在处理完状态更新（微任务）后，立即开始渲染工作（宏任务），而不会被其他宏任务（如用户点击事件）打断。
>
> **为什么不用微任务（Promise）？**
>
> 微任务会在当前宏任务结束后立即执行所有排队的微任务，无法实现"让出线程"的效果。如果用 Promise，React 会在一个宏任务中执行完所有任务，页面仍然会卡顿。
>
> **在实际工作中的应用**：
>
> React 的时间切片（Time Slicing）依赖 MessageChannel 实现可中断渲染。每个时间片（5ms）执行一部分组件渲染，然后通过 MessageChannel 调度下一个时间片，给浏览器留出时间更新 UI。这就是 Concurrent Mode 能保持页面流畅的原因。

**为什么这个回答出彩？**

1. ✅ 从多个维度解释（执行时机、浏览器限制、优先级）
2. ✅ 对比了其他方案（setTimeout、Promise）
3. ✅ 结合 React 源码设计（时间切片、Concurrent Mode）
4. ✅ 展示了对浏览器事件循环的深刻理解

---

## 九、【化骨绵掌】

### 卡片1：直觉理解 🎯

**一句话：** MessageChannel 是一个有两个端口的消息通道，一端发送，另一端接收。

**举例：**

```javascript
const channel = new MessageChannel();
channel.port1.onmessage = (e) => console.log(e.data);
channel.port2.postMessage('Hello!');
// 输出: Hello!
```

**应用：** React Scheduler 用 port2 触发任务，port1 接收并执行。

---

### 卡片2：宏任务定位 📐

**一句话：** MessageChannel 的消息处理是宏任务，晚于微任务执行。

**举例：**

```javascript
Promise.resolve().then(() => console.log('微任务'));
const ch = new MessageChannel();
ch.port1.onmessage = () => console.log('宏任务');
ch.port2.postMessage(null);
// 输出: 微任务 → 宏任务
```

**应用：** 确保状态更新（微任务）完成后再开始渲染（宏任务）。

---

### 卡片3：零延迟优势 ⚡

**一句话：** MessageChannel 没有 setTimeout 的 4ms 最小延迟。

**举例：**

```javascript
// setTimeout 至少 4ms
setTimeout(() => console.log('4ms+'), 0);

// MessageChannel 几乎立即
const ch = new MessageChannel();
ch.port1.onmessage = () => console.log('~0ms');
ch.port2.postMessage(null);
```

**应用：** 高频调度场景下，累积延迟会严重影响性能。

---

### 卡片4：双向通信 🔄

**一句话：** port1 和 port2 可以互相发送消息，实现双向通信。

**举例：**

```javascript
const ch = new MessageChannel();
ch.port1.onmessage = (e) => {
  console.log('port1:', e.data);
  ch.port1.postMessage('回复');
};
ch.port2.onmessage = (e) => console.log('port2:', e.data);
ch.port2.postMessage('你好');
```

**应用：** 主线程与 Worker 之间的双向通信。

---

### 卡片5：结构化克隆 📋

**一句话：** postMessage 传递的数据会被结构化克隆，而非传递引用。

**举例：**

```javascript
const obj = { count: 0 };
const ch = new MessageChannel();
ch.port1.onmessage = (e) => {
  e.data.count = 100;
};
ch.port2.postMessage(obj);
console.log(obj.count); // 0（未被修改）
```

**应用：** 保证数据隔离，避免意外修改。

---

### 卡片6：对比 setTimeout 🆚

**一句话：** MessageChannel 比 setTimeout 更快、更可靠，不受浏览器节流限制。

**对比表：**

| 特性 | MessageChannel | setTimeout |
|-----|---------------|-----------|
| 最小延迟 | 0ms | 4ms |
| 后台标签页 | 正常执行 | 可能延迟到 1000ms |
| 嵌套调用 | 无限制 | 超过 5 次会节流 |
| 优先级 | 适中 | 较低 |

**应用：** React Scheduler 选择 MessageChannel 的核心原因。

---

### 卡片7：时间切片原理 ⏱️

**一句话：** 每执行 5ms 任务，通过 MessageChannel 调度下一个任务，实现可中断渲染。

**举例：**

```javascript
const ch = new MessageChannel();
let tasks = [1, 2, 3, 4, 5];

ch.port1.onmessage = () => {
  const start = performance.now();
  while (tasks.length && performance.now() - start < 5) {
    console.log(tasks.shift());
  }
  if (tasks.length) ch.port2.postMessage(null);
};

ch.port2.postMessage(null);
```

**应用：** React Concurrent Mode 的核心机制。

---

### 卡片8：任务队列 📝

**一句话：** MessageChannel 配合任务队列实现异步调度。

**举例：**

```javascript
const queue = [];
const ch = new MessageChannel();

ch.port1.onmessage = () => {
  while (queue.length) {
    queue.shift()();
  }
};

function schedule(task) {
  queue.push(task);
  ch.port2.postMessage(null);
}

schedule(() => console.log('任务1'));
schedule(() => console.log('任务2'));
```

**应用：** React Scheduler 的任务队列管理。

---

### 卡片9：Worker 通信 👷

**一句话：** MessageChannel 可用于主线程与 Worker 的通信。

**举例：**

```javascript
// 主线程
const worker = new Worker('worker.js');
const ch = new MessageChannel();

worker.postMessage({ port: ch.port2 }, [ch.port2]);

ch.port1.onmessage = (e) => console.log('收到:', e.data);

// worker.js
self.onmessage = (e) => {
  const port = e.data.port;
  port.postMessage('Hello from Worker!');
};
```

**应用：** 复杂的多线程通信场景。

---

### 卡片10：总结与延伸 🎓

**总结：**

MessageChannel 是浏览器提供的双向通信机制，基于宏任务实现零延迟异步调度，是 React Scheduler 时间切片的核心实现。

**核心优势：**
1. 宏任务特性（晚于微任务）
2. 零延迟（无 4ms 限制）
3. 不受浏览器节流影响
4. 优先级适中

**延伸学习：**
- React Scheduler 源码分析
- 浏览器事件循环机制
- Concurrent Mode 原理
- 时间切片调度算法

---

## 十、【一句话总结】

**MessageChannel 是浏览器的双端口通信机制，通过宏任务实现零延迟异步调度，解决了 setTimeout 延迟高和浏览器节流的问题，是 React Scheduler 实现时间切片和可中断渲染的核心基础。**

---

## 附录

### 学习检查清单

- [ ] 理解 MessageChannel 的双端口机制
- [ ] 掌握宏任务和微任务的区别
- [ ] 理解为什么 MessageChannel 比 setTimeout 更适合调度
- [ ] 能够实现简单的任务队列调度器
- [ ] 理解 React Scheduler 如何使用 MessageChannel
- [ ] 理解时间切片的实现原理
- [ ] 能够解释 MessageChannel 在 React 中的价值

### 下一步学习建议

1. **深入 React Scheduler**：阅读 React Scheduler 源码，理解完整的调度逻辑
2. **学习事件循环**：深入理解浏览器事件循环的宏任务和微任务机制
3. **学习 requestIdleCallback**：了解另一个调度 API 及其与 MessageChannel 的区别
4. **实践时间切片**：实现一个简单的时间切片渲染系统

### 参考资源

- [HTML Standard - MessageChannel](https://html.spec.whatwg.org/multipage/web-messaging.html#message-channels)
- [React Scheduler 源码](https://github.com/facebook/react/tree/main/packages/scheduler)
- [MDN - MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)
- [Jake Archibald - Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
