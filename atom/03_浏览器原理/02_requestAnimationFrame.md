# requestAnimationFrame

## 一、【30字核心】

**requestAnimationFrame 是浏览器提供的动画帧同步 API，在每次屏幕刷新前执行回调，与硬件刷新率同步，是实现流畅动画的最佳方案。**

---

## 二、【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### requestAnimationFrame 的第一性原理 🎯

#### 1. 最基础的定义

**requestAnimationFrame = 在浏览器下一次重绘之前执行的回调函数**

仅此而已！没有更基础的了。

浏览器屏幕有固定的刷新率（通常 60Hz，即每秒 60 帧），requestAnimationFrame（简称 rAF）会在每一帧开始前调用你提供的回调函数。

```javascript
function animate() {
  // 更新动画状态
  element.style.left = x + 'px';
  x += 1;

  // 请求下一帧
  requestAnimationFrame(animate);
}

// 启动动画
requestAnimationFrame(animate);
```

#### 2. 为什么需要 requestAnimationFrame？

**核心问题：如何实现与屏幕刷新同步的流畅动画？**

人眼能察觉到的流畅动画需要至少 60fps（每秒 60 帧）。如果动画帧率低于 60fps，用户会感觉到卡顿。

传统方法的问题：
- ❌ **setInterval(fn, 16)**：不与屏幕刷新同步，可能在两次刷新之间执行多次或跳过刷新
- ❌ **setTimeout(fn, 16)**：同样的问题，且有 4ms 最小延迟
- ✅ **requestAnimationFrame**：由浏览器控制，保证在每次屏幕刷新前执行一次

#### 3. requestAnimationFrame 的三层价值

##### 价值1：与屏幕刷新同步 🖥️

rAF 的执行时机由浏览器控制，保证与硬件屏幕刷新率同步。

```javascript
let frameCount = 0;
let lastTime = performance.now();

function countFrames(currentTime) {
  frameCount++;

  // 每秒统计一次帧率
  if (currentTime - lastTime >= 1000) {
    console.log(`FPS: ${frameCount}`);
    frameCount = 0;
    lastTime = currentTime;
  }

  requestAnimationFrame(countFrames);
}

requestAnimationFrame(countFrames);
// 输出: FPS: 60（在 60Hz 显示器上）
// 输出: FPS: 120（在 120Hz 显示器上）
```

##### 价值2：自动节能优化 🔋

当页面不可见时（切换标签页、最小化），浏览器会自动暂停 rAF，节省 CPU 和电池。

```javascript
let animationCount = 0;

function animate() {
  animationCount++;
  console.log(`动画帧 ${animationCount}`);
  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);

// 切换到其他标签页：动画暂停
// 切换回来：动画继续
```

setTimeout/setInterval 不会暂停，继续消耗资源。

##### 价值3：高精度时间戳 ⏱️

rAF 的回调函数接收一个高精度时间戳参数，表示当前帧的开始时间。

```javascript
function animate(timestamp) {
  console.log(`当前帧时间戳: ${timestamp.toFixed(2)}ms`);

  // 基于时间戳计算动画进度
  const progress = (timestamp % 2000) / 2000; // 2秒循环
  element.style.opacity = Math.sin(progress * Math.PI * 2);

  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

#### 4. 从第一性原理推导 React 实现

**推理链：**
```
1. 浏览器屏幕有固定刷新率（60Hz = 16.6ms/帧）
   ↓
2. 每一帧包含：JavaScript 执行、样式计算、布局、绘制、合成
   ↓
3. JavaScript 执行时间过长会导致丢帧（超过 16.6ms）
   ↓
4. 需要将 JavaScript 任务限制在一定时间内（如 5ms）
   ↓
5. 剩余时间（11.6ms）留给浏览器渲染
   ↓
6. requestAnimationFrame 提供精确的帧开始时机
   ↓
7. React 使用 rAF 的时间戳判断是否应该让出线程
```

#### 5. 一句话总结第一性原理

**requestAnimationFrame 是浏览器的帧同步机制，通过在每次屏幕刷新前执行回调，确保动画与硬件刷新率同步，避免丢帧和不必要的资源消耗。**

---

## 三、【3个核心概念】

### 核心概念1：60fps 与 16.6ms 帧预算 ⏰

**屏幕刷新率为 60Hz 时，每帧时间为 16.6ms，这是 JavaScript 执行和浏览器渲染的总预算。**

```javascript
// 每帧的时间预算
const frameTime = 1000 / 60; // 16.6ms

function animate(timestamp) {
  const startTime = performance.now();

  // 执行动画逻辑
  updateAnimation();

  const elapsed = performance.now() - startTime;
  console.log(`本帧耗时: ${elapsed.toFixed(2)}ms / ${frameTime.toFixed(2)}ms`);

  if (elapsed > frameTime) {
    console.warn('⚠️ 超出帧预算，可能丢帧！');
  }

  requestAnimationFrame(animate);
}

function updateAnimation() {
  // 模拟耗时操作
  for (let i = 0; i < 100000; i++) {
    Math.sqrt(i);
  }
}

requestAnimationFrame(animate);
```

**详细解释：**

一帧 16.6ms 的分配：
- **JavaScript 执行**：5ms（推荐）
- **样式计算**：2ms
- **布局（Layout）**：2ms
- **绘制（Paint）**：2ms
- **合成（Composite）**：2ms
- **其他**：3.6ms

如果 JavaScript 执行超过 5ms，留给浏览器渲染的时间就不够，可能导致丢帧（掉到 30fps 甚至更低）。

**在 React 源码中的应用：**

React Scheduler 使用 5ms 作为时间切片的默认值，就是基于这个帧预算考虑。

```javascript
// React Scheduler 的时间切片逻辑（简化版）
const yieldInterval = 5; // 5ms

function workLoop(initialTime) {
  let currentTime = initialTime;

  while (workInProgress !== null) {
    currentTime = performance.now();

    if (currentTime - initialTime >= yieldInterval) {
      // 超过 5ms，让出线程
      break;
    }

    performUnitOfWork(workInProgress);
  }
}
```

---

### 核心概念2：时间戳（timestamp）参数 📅

**rAF 的回调函数接收一个高精度时间戳，表示当前帧开始的时间，所有 rAF 回调共享相同的时间戳。**

```javascript
let lastTimestamp = null;

function logTimestamp(timestamp) {
  if (lastTimestamp !== null) {
    const delta = timestamp - lastTimestamp;
    console.log(`距离上一帧: ${delta.toFixed(2)}ms`);
  }

  lastTimestamp = timestamp;
  requestAnimationFrame(logTimestamp);
}

requestAnimationFrame(logTimestamp);

// 输出（60fps）:
// 距离上一帧: 16.67ms
// 距离上一帧: 16.65ms
// 距离上一帧: 16.68ms
```

**详细解释：**

时间戳的特点：
1. **高精度**：精确到微秒级别
2. **相对值**：从页面加载开始计时（类似 performance.now()）
3. **同步性**：同一帧内的所有 rAF 回调收到相同的时间戳

```javascript
// 在同一帧内注册两个 rAF
requestAnimationFrame((t1) => {
  console.log('回调1:', t1);
});

requestAnimationFrame((t2) => {
  console.log('回调2:', t2);
});

// 输出:
// 回调1: 12345.678
// 回调2: 12345.678（相同！）
```

**在 React 源码中的应用：**

React 不直接使用 rAF 的时间戳，而是用 `performance.now()` 来计算时间切片。但理解时间戳对理解帧同步很重要。

---

### 核心概念3：回调队列与执行时机 🔄

**rAF 的回调在浏览器渲染管道的"渲染前"阶段执行，在微任务之后、Layout/Paint 之前。**

```javascript
console.log('1: 同步代码');

requestAnimationFrame(() => {
  console.log('4: rAF 回调');
});

Promise.resolve().then(() => {
  console.log('2: 微任务');
});

setTimeout(() => {
  console.log('3: setTimeout 宏任务');
}, 0);

console.log('1.5: 同步代码结束');

// 输出顺序:
// 1: 同步代码
// 1.5: 同步代码结束
// 2: 微任务
// 3: setTimeout 宏任务
// 4: rAF 回调（在下一帧开始时）
```

**详细解释：**

浏览器的事件循环与渲染时序：

```
1. 执行一个宏任务（如 script）
2. 执行所有微任务
3. 判断是否需要渲染（通常每 16.6ms 一次）
   └─ 如果需要：
      a. 执行 requestAnimationFrame 回调
      b. 重新计算样式（Recalculate Style）
      c. 布局（Layout）
      d. 绘制（Paint）
      e. 合成（Composite）
4. 执行 requestIdleCallback（如果有空闲时间）
5. 回到步骤 1
```

rAF 在"渲染前"执行的好处：
- 可以修改 DOM 和样式
- 这些修改会在本帧的渲染中生效
- 不会导致额外的重排重绘

```javascript
// rAF 中修改样式，本帧生效
requestAnimationFrame(() => {
  element.style.left = '100px';
  element.style.top = '200px';
  // 只会触发一次重排重绘
});

// setTimeout 中修改样式，可能分多次渲染
setTimeout(() => {
  element.style.left = '100px'; // 触发一次重排重绘
  element.style.top = '200px';  // 再触发一次重排重绘
}, 0);
```

**在 React 源码中的应用：**

React Concurrent Mode 需要在帧的不同阶段执行不同的工作：
- **协调（Reconciliation）**：在 JavaScript 时间片内
- **提交（Commit）**：在渲染前完成

---

## 四、【最小可用】

掌握以下内容，就能理解 React 源码中帧同步渲染的核心思想：

### 4.1 基础用法：实现简单动画

```javascript
let x = 0;

function animate() {
  // 更新状态
  x += 2;
  element.style.transform = `translateX(${x}px)`;

  // 继续下一帧
  if (x < 500) {
    requestAnimationFrame(animate);
  }
}

requestAnimationFrame(animate);
```

### 4.2 基于时间戳的动画：避免帧率影响

```javascript
let startTime = null;

function animate(timestamp) {
  if (!startTime) startTime = timestamp;

  // 计算经过的时间
  const elapsed = timestamp - startTime;

  // 基于时间的动画（2秒移动 500px）
  const progress = Math.min(elapsed / 2000, 1);
  const x = progress * 500;

  element.style.transform = `translateX(${x}px)`;

  if (progress < 1) {
    requestAnimationFrame(animate);
  }
}

requestAnimationFrame(animate);
```

### 4.3 帧预算监控：检测性能问题

```javascript
function animate(timestamp) {
  const startTime = performance.now();

  // 执行动画逻辑
  updateAnimation();

  const frameTime = performance.now() - startTime;

  if (frameTime > 16.6) {
    console.warn(`⚠️ 帧超时: ${frameTime.toFixed(2)}ms`);
  }

  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

**这些知识足以：**
- 实现流畅的动画效果
- 理解 React 为什么需要时间切片
- 为后续学习 React Scheduler 的帧管理打基础

---

## 五、【1个类比】

### 类比1：requestAnimationFrame = 电影放映机 🎬

**requestAnimationFrame 就像电影放映机，每秒固定播放 24 帧胶片，浏览器的 60fps 就像每秒播放 60 帧画面。**

**举例：**

电影放映过程：
1. **放映员准备胶片**（JavaScript 执行）
2. **放映机投影画面**（浏览器渲染）
3. **等待下一帧时间**（16.6ms 间隔）
4. **重复**

如果放映员准备胶片太慢（JavaScript 执行超过 5ms），观众会感觉卡顿（丢帧）。

```javascript
// 电影放映机模拟
let frame = 0;
const totalFrames = 240; // 4秒电影（60fps × 4）

function playMovie(timestamp) {
  // 准备当前帧（JavaScript 工作）
  prepareFrame(frame);

  // 投影画面（浏览器渲染会自动完成）
  console.log(`播放第 ${frame} 帧`);

  frame++;

  if (frame < totalFrames) {
    requestAnimationFrame(playMovie);
  } else {
    console.log('电影结束');
  }
}

function prepareFrame(frameNumber) {
  // 计算当前帧的画面内容
  // 类似 React 计算组件状态
}

requestAnimationFrame(playMovie);
```

---

### 类比2：帧预算 = 地铁发车时刻表 🚇

**16.6ms 的帧时间就像地铁每隔固定时间发车，JavaScript 必须在发车前完成工作，否则乘客（渲染任务）就赶不上这趟车。**

**举例：**

地铁时刻表（60fps）：
- **00:00** → 发车（第 1 帧渲染）
- **00:16.6ms** → 发车（第 2 帧渲染）
- **00:33.2ms** → 发车（第 3 帧渲染）
- **00:49.8ms** → 发车（第 4 帧渲染）

如果 JavaScript 执行时间超过 16.6ms，就会错过下一趟发车（丢帧），用户看到的画面是上一帧的（卡顿）。

```javascript
// 地铁发车模拟
function animate(timestamp) {
  const startTime = performance.now();

  // 乘客上车（JavaScript 任务）
  boardPassengers();

  const elapsedTime = performance.now() - startTime;

  if (elapsedTime > 16.6) {
    console.log('❌ 错过发车，乘客需要等下一趟（丢帧）');
  } else {
    console.log(`✅ 准时发车，剩余 ${(16.6 - elapsedTime).toFixed(2)}ms`);
  }

  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

---

### 类比3：游戏引擎的帧循环 🎮

**requestAnimationFrame 就像游戏引擎的主循环，每帧执行：更新游戏状态 → 渲染画面 → 等待下一帧。**

**举例：**

游戏引擎的每一帧：

```javascript
// 游戏状态
const player = { x: 0, y: 0, speed: 5 };
const enemies = [];

// 游戏主循环
function gameLoop(timestamp) {
  // 1. 处理输入
  handleInput();

  // 2. 更新游戏状态
  updateGameState();

  // 3. 渲染画面
  render();

  // 4. 请求下一帧
  requestAnimationFrame(gameLoop);
}

function handleInput() {
  // 检测键盘输入
  if (keys['ArrowRight']) {
    player.x += player.speed;
  }
}

function updateGameState() {
  // 更新敌人位置、碰撞检测等
  enemies.forEach(enemy => {
    enemy.x += enemy.speed;
  });
}

function render() {
  // 清空画布
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 绘制玩家
  ctx.fillRect(player.x, player.y, 50, 50);

  // 绘制敌人
  enemies.forEach(enemy => {
    ctx.fillRect(enemy.x, enemy.y, 30, 30);
  });
}

// 启动游戏循环
requestAnimationFrame(gameLoop);
```

React 的协调过程类似"更新游戏状态"，提交过程类似"渲染画面"。

---

### 类比4：setTimeout vs rAF = 闹钟 vs 鸡叫 ⏰🐓

**setTimeout 像闹钟（固定延迟），rAF 像公鸡打鸣（与自然节律同步）。**

| 对比项 | setTimeout（闹钟） | rAF（公鸡打鸣） |
|-------|------------------|----------------|
| 触发时机 | 固定延迟（如 16ms） | 跟随日出（屏幕刷新） |
| 精确度 | 可能延迟（系统繁忙） | 与自然同步（硬件刷新率） |
| 节能 | 一直响（即使睡觉） | 天亮才叫（页面可见才执行） |
| 适用场景 | 定时任务 | 动画渲染 |

```javascript
// 闹钟方式（setTimeout）- 固定间隔，不管是否需要
let count1 = 0;
function alarmClock() {
  count1++;
  console.log(`闹钟响了第 ${count1} 次`);
  setTimeout(alarmClock, 16); // 固定 16ms
}
setTimeout(alarmClock, 16);

// 公鸡方式（rAF）- 跟随自然节律
let count2 = 0;
function rooster(timestamp) {
  count2++;
  console.log(`公鸡打鸣第 ${count2} 次，天亮时间: ${timestamp}`);
  requestAnimationFrame(rooster); // 跟随屏幕刷新
}
requestAnimationFrame(rooster);
```

---

### 类比总结表

| React/浏览器概念 | 生活场景 | 关键相似点 |
|-----------------|---------|----------|
| 60fps 帧循环 | 电影放映（24fps） | 固定时间间隔播放画面 |
| 16.6ms 帧预算 | 地铁发车时刻表 | 必须在规定时间内完成工作 |
| 游戏引擎循环 | 游戏主循环 | 更新状态 → 渲染 → 下一帧 |
| rAF vs setTimeout | 公鸡打鸣 vs 闹钟 | 自然同步 vs 固定延迟 |

---

## 六、【反直觉点】

### 误区1：rAF 在每个微任务后执行 ❌

**为什么错？**
- rAF 不是在每个微任务后执行
- rAF 在**浏览器决定需要渲染时**才执行
- 通常每 16.6ms（60fps）执行一次

**为什么人们容易这样错？**

因为 rAF 是异步 API，人们容易将它与 Promise（微任务）混淆，误以为它也是"尽快执行"的机制。实际上，rAF 的执行完全由浏览器的渲染时序控制。

**正确理解：**

```javascript
console.log('1: 同步代码');

Promise.resolve().then(() => {
  console.log('2: 微任务（立即执行）');
});

requestAnimationFrame(() => {
  console.log('3: rAF（等到下一帧）');
});

console.log('1.5: 同步代码结束');

// 输出顺序:
// 1: 同步代码
// 1.5: 同步代码结束
// 2: 微任务（立即执行）
// ... 等待约 16ms ...
// 3: rAF（等到下一帧）
```

浏览器渲染时序：
```
宏任务 → 所有微任务 → 【判断是否需要渲染】
                        ↓
                   如果需要渲染:
                   → rAF 回调
                   → 样式计算
                   → 布局
                   → 绘制
```

---

### 误区2：setTimeout(fn, 16) 等同于 rAF ❌

**为什么错？**
- setTimeout(fn, 16) 无法与屏幕刷新同步
- 可能在两次刷新之间执行多次，或跳过某次刷新
- rAF 保证每次刷新前执行一次，不多不少

**为什么人们容易这样错？**

因为 60fps 对应 16.6ms 间隔，直觉上认为 `setTimeout(fn, 16)` 就能达到 60fps 的效果。但 setTimeout 和屏幕刷新是两个独立的时钟，不会自动同步。

**正确理解：**

```javascript
// setTimeout 方式 - 可能与刷新不同步
let frameCount1 = 0;
function animateWithTimeout() {
  frameCount1++;
  element.style.left = frameCount1 + 'px';
  setTimeout(animateWithTimeout, 16);
}
setTimeout(animateWithTimeout, 16);

// rAF 方式 - 完美同步
let frameCount2 = 0;
function animateWithRAF() {
  frameCount2++;
  element.style.left = frameCount2 + 'px';
  requestAnimationFrame(animateWithRAF);
}
requestAnimationFrame(animateWithRAF);

// setTimeout 问题示例：
// 屏幕刷新：  0ms   16.6ms   33.2ms   49.8ms
// setTimeout: 0ms   16ms     32ms     48ms
//                   ↑ 偏移    ↑ 偏移    ↑ 偏移
// 结果：画面抖动、不流畅
```

时间轴对比：
```
屏幕刷新:     |-------|-------|-------|  (16.6ms 间隔)
setTimeout:   |------|------|------|    (16ms 间隔)
              不同步！会逐渐产生偏移

屏幕刷新:     |-------|-------|-------|
rAF:          |-------|-------|-------|
              完美同步！
```

---

### 误区3：rAF 保证每次间隔 16.6ms ❌

**为什么错？**
- rAF 的间隔取决于**硬件刷新率**，不是固定 16.6ms
- 60Hz 显示器：16.6ms
- 120Hz 显示器：8.3ms
- 144Hz 显示器：6.9ms

**为什么人们容易这样错？**

大多数教程和文档使用 60fps 作为示例，导致人们认为 rAF 就是"每 16.6ms 执行一次"。实际上，rAF 的执行频率由硬件决定。

**正确理解：**

```javascript
let lastTime = null;
let intervals = [];

function measureInterval(timestamp) {
  if (lastTime !== null) {
    const interval = timestamp - lastTime;
    intervals.push(interval);

    if (intervals.length >= 60) {
      // 统计 60 帧的平均间隔
      const avgInterval = intervals.reduce((a, b) => a + b) / intervals.length;
      const fps = 1000 / avgInterval;
      console.log(`平均间隔: ${avgInterval.toFixed(2)}ms, FPS: ${fps.toFixed(2)}`);
      intervals = [];
    }
  }

  lastTime = timestamp;
  requestAnimationFrame(measureInterval);
}

requestAnimationFrame(measureInterval);

// 60Hz 显示器输出: 平均间隔: 16.67ms, FPS: 60.00
// 120Hz 显示器输出: 平均间隔: 8.33ms, FPS: 120.00
// 144Hz 显示器输出: 平均间隔: 6.94ms, FPS: 144.00
```

**正确的动画实现**（兼容不同刷新率）：

```javascript
// ❌ 错误：假设固定间隔
function wrongAnimate() {
  x += 2; // 每帧移动 2px（在 120Hz 显示器上会移动得更快！）
  element.style.left = x + 'px';
  requestAnimationFrame(wrongAnimate);
}

// ✅ 正确：基于时间戳
let startTime = null;
function correctAnimate(timestamp) {
  if (!startTime) startTime = timestamp;

  const elapsed = timestamp - startTime;
  const x = (elapsed / 1000) * 100; // 每秒移动 100px（不受刷新率影响）

  element.style.left = x + 'px';
  requestAnimationFrame(correctAnimate);
}
```

---

## 七、【实战代码】

### 基础实现（简化版）

```javascript
// ===== 1. 场景1：基础动画 =====
console.log("=== 场景1：简单的移动动画 ===");

const box = document.createElement('div');
box.style.cssText = 'width:50px;height:50px;background:blue;position:absolute;';
document.body.appendChild(box);

let x = 0;

function animate() {
  x += 2;
  box.style.left = x + 'px';

  if (x < 500) {
    requestAnimationFrame(animate);
  }
}

requestAnimationFrame(animate);

// ===== 2. 场景2：基于时间戳的动画（推荐） =====
console.log("\n=== 场景2：基于时间戳的动画 ===");

const box2 = document.createElement('div');
box2.style.cssText = 'width:50px;height:50px;background:red;position:absolute;top:60px;';
document.body.appendChild(box2);

let startTime = null;
const duration = 2000; // 2秒动画
const distance = 500; // 移动 500px

function animateWithTime(timestamp) {
  if (!startTime) startTime = timestamp;

  const elapsed = timestamp - startTime;
  const progress = Math.min(elapsed / duration, 1); // 0 到 1

  // 缓动函数（easeInOutQuad）
  const easeProgress = progress < 0.5
    ? 2 * progress * progress
    : 1 - Math.pow(-2 * progress + 2, 2) / 2;

  const currentX = easeProgress * distance;
  box2.style.left = currentX + 'px';

  if (progress < 1) {
    requestAnimationFrame(animateWithTime);
  } else {
    console.log('动画完成！');
  }
}

requestAnimationFrame(animateWithTime);

// ===== 3. 场景3：帧率监控 =====
console.log("\n=== 场景3：FPS 监控 ===");

let frameCount = 0;
let lastFpsTime = performance.now();
let fps = 0;

function monitorFPS(timestamp) {
  frameCount++;

  const currentTime = performance.now();
  const deltaTime = currentTime - lastFpsTime;

  if (deltaTime >= 1000) {
    fps = Math.round((frameCount * 1000) / deltaTime);
    console.log(`当前 FPS: ${fps}`);

    frameCount = 0;
    lastFpsTime = currentTime;
  }

  requestAnimationFrame(monitorFPS);
}

requestAnimationFrame(monitorFPS);

// 输出（每秒一次）:
// 当前 FPS: 60

// ===== 4. 场景4：帧预算监控 =====
console.log("\n=== 场景4：帧预算监控 ===");

function heavyWork() {
  // 模拟耗时操作
  const start = performance.now();
  while (performance.now() - start < 10) {
    // 故意超出 16.6ms 预算
  }
}

function monitorFrameBudget(timestamp) {
  const frameStart = performance.now();

  // 执行工作
  heavyWork();

  const frameTime = performance.now() - frameStart;
  const budget = 16.6; // 60fps 的帧预算

  if (frameTime > budget) {
    console.warn(`⚠️ 帧超时: ${frameTime.toFixed(2)}ms (预算: ${budget}ms)`);
  } else {
    console.log(`✅ 帧正常: ${frameTime.toFixed(2)}ms`);
  }

  requestAnimationFrame(monitorFrameBudget);
}

requestAnimationFrame(monitorFrameBudget);

// 输出:
// ⚠️ 帧超时: 10.23ms (预算: 16.6ms)

// ===== 5. 场景5：多对象动画（模拟 React 批量更新） =====
console.log("\n=== 场景5：批量动画 ===");

const particles = [];
for (let i = 0; i < 100; i++) {
  const particle = {
    x: Math.random() * window.innerWidth,
    y: Math.random() * window.innerHeight,
    vx: (Math.random() - 0.5) * 5,
    vy: (Math.random() - 0.5) * 5,
    element: document.createElement('div')
  };

  particle.element.style.cssText = `
    width: 5px;
    height: 5px;
    background: orange;
    position: absolute;
    border-radius: 50%;
  `;

  document.body.appendChild(particle.element);
  particles.push(particle);
}

function animateParticles() {
  const frameStart = performance.now();

  // 批量更新所有粒子（类似 React 批量更新组件）
  particles.forEach(particle => {
    particle.x += particle.vx;
    particle.y += particle.vy;

    // 边界检测
    if (particle.x < 0 || particle.x > window.innerWidth) particle.vx *= -1;
    if (particle.y < 0 || particle.y > window.innerHeight) particle.vy *= -1;

    // 更新 DOM（类似 React 提交阶段）
    particle.element.style.left = particle.x + 'px';
    particle.element.style.top = particle.y + 'px';
  });

  const frameTime = performance.now() - frameStart;
  console.log(`更新 ${particles.length} 个粒子耗时: ${frameTime.toFixed(2)}ms`);

  requestAnimationFrame(animateParticles);
}

requestAnimationFrame(animateParticles);

// ===== 6. 场景6：暂停和恢复 =====
console.log("\n=== 场景6：可控制的动画 ===");

let animationId = null;
let isPaused = false;
let pausedTime = null;

function controlledAnimate(timestamp) {
  if (isPaused) {
    pausedTime = timestamp;
    return;
  }

  // 动画逻辑...
  console.log('动画进行中...');

  animationId = requestAnimationFrame(controlledAnimate);
}

// 启动动画
animationId = requestAnimationFrame(controlledAnimate);

// 暂停动画
function pause() {
  isPaused = true;
  console.log('动画已暂停');
}

// 恢复动画
function resume() {
  isPaused = false;
  animationId = requestAnimationFrame(controlledAnimate);
  console.log('动画已恢复');
}

// 停止动画
function stop() {
  cancelAnimationFrame(animationId);
  console.log('动画已停止');
}

// 使用:
// pause();   // 暂停
// resume();  // 恢复
// stop();    // 停止
```

**运行输出示例：**

```
=== 场景1：简单的移动动画 ===
（蓝色方块从左向右移动）

=== 场景2：基于时间戳的动画 ===
（红色方块平滑移动，带缓动效果）
动画完成！

=== 场景3：FPS 监控 ===
当前 FPS: 60
当前 FPS: 60
当前 FPS: 60

=== 场景4：帧预算监控 ===
⚠️ 帧超时: 10.23ms (预算: 16.6ms)
⚠️ 帧超时: 10.18ms (预算: 16.6ms)

=== 场景5：批量动画 ===
更新 100 个粒子耗时: 3.45ms
更新 100 个粒子耗时: 3.52ms
（100个橙色粒子在屏幕上弹跳）

=== 场景6：可控制的动画 ===
动画进行中...
动画进行中...
动画已暂停
动画已恢复
动画进行中...
```

---

### 进阶：React 时间切片思想

```javascript
// React 时间切片的核心思想（简化版）
// 不直接使用 rAF，但理解帧预算

class TimeSlicingRenderer {
  constructor() {
    this.taskQueue = [];
    this.isRendering = false;
    this.frameDeadline = 5; // 每帧预算 5ms
  }

  // 添加渲染任务
  scheduleRender(component) {
    this.taskQueue.push(component);

    if (!this.isRendering) {
      this.startRender();
    }
  }

  // 开始渲染
  startRender() {
    this.isRendering = true;
    requestAnimationFrame((timestamp) => {
      this.performWork(timestamp);
    });
  }

  // 执行工作
  performWork(frameStartTime) {
    let taskStartTime = performance.now();

    // 在帧预算内尽可能多地处理任务
    while (
      this.taskQueue.length > 0 &&
      performance.now() - taskStartTime < this.frameDeadline
    ) {
      const component = this.taskQueue.shift();
      this.renderComponent(component);
    }

    // 如果还有任务，调度到下一帧
    if (this.taskQueue.length > 0) {
      console.log(`剩余 ${this.taskQueue.length} 个任务，调度到下一帧`);
      requestAnimationFrame((timestamp) => {
        this.performWork(timestamp);
      });
    } else {
      this.isRendering = false;
      console.log('所有任务完成');
    }
  }

  // 渲染单个组件（模拟）
  renderComponent(component) {
    console.log(`渲染组件: ${component}`);

    // 模拟渲染耗时
    const start = performance.now();
    while (performance.now() - start < 1) {}
  }
}

// 使用
const renderer = new TimeSlicingRenderer();

// 添加 20 个组件渲染任务
for (let i = 1; i <= 20; i++) {
  renderer.scheduleRender(`Component${i}`);
}

// 输出:
// 渲染组件: Component1
// 渲染组件: Component2
// ...
// 渲染组件: Component5
// 剩余 15 个任务，调度到下一帧
// 渲染组件: Component6
// ...
```

**React 实际使用的方式**：

React 不直接使用 rAF 进行调度，而是用 MessageChannel（见 MessageChannel 文档）。但 rAF 的时间戳概念对理解帧预算很重要。

```javascript
// React Scheduler 的帧预算检测（简化版）
function shouldYieldToHost() {
  const currentTime = performance.now();
  return currentTime >= deadline;
}

// 工作循环
function workLoop(initialTime) {
  let currentTime = initialTime;

  while (workInProgress !== null && !shouldYieldToHost()) {
    performUnitOfWork(workInProgress);
    currentTime = performance.now();
  }

  // 如果还有工作，调度到下一个宏任务（MessageChannel）
  if (workInProgress !== null) {
    return true; // 还有更多工作
  }

  return false; // 工作完成
}
```

---

## 八、【面试必问】

### 问题："requestAnimationFrame 和 setTimeout 的区别是什么？为什么动画要用 rAF？"

**普通回答（❌ 不出彩）：**

"requestAnimationFrame 性能更好，因为它在每一帧执行。setTimeout 有延迟。"

**出彩回答（✅ 推荐）：**

> **requestAnimationFrame 和 setTimeout 的本质区别在于执行时机和同步机制：**
>
> 1. **执行时机**：
>    - **setTimeout**：在指定延迟后执行，属于宏任务队列，执行时机不确定（受其他任务影响）
>    - **rAF**：在浏览器下一次重绘之前执行，由浏览器的渲染时序控制
>
> 2. **屏幕刷新同步**：
>    - **setTimeout(fn, 16)**：16ms 是固定延迟，与屏幕刷新率（60Hz = 16.6ms）是两个独立的时钟，无法同步
>    - **rAF**：完全与硬件刷新率同步，60Hz 显示器每 16.6ms 执行，120Hz 显示器每 8.3ms 执行
>
> 3. **性能优化**：
>    - **setTimeout**：页面不可见时仍继续执行，浪费 CPU 和电池
>    - **rAF**：页面不可见时自动暂停，节省资源
>
> 4. **帧率稳定性**：
>    - **setTimeout**：可能在两次刷新之间执行多次（浪费），或跳过某次刷新（丢帧）
>    - **rAF**：保证每次刷新前执行一次，不多不少，帧率稳定
>
> **为什么动画要用 rAF？**
>
> 假设用 setTimeout(fn, 16) 实现动画：
> ```
> 屏幕刷新:    0ms   16.6ms   33.2ms   49.8ms   66.4ms
> setTimeout:  0ms   16ms     32ms     48ms     64ms
>                    ↓偏移     ↓偏移     ↓偏移     ↓偏移
> ```
> 偏移会累积，导致画面抖动。而 rAF 完美同步屏幕刷新。
>
> **在实际工作中的应用**：
>
> - **动画库**（如 GSAP、Anime.js）底层都用 rAF
> - **React Concurrent Mode** 使用帧预算（5ms）概念，确保在每帧内完成尽可能多的工作
> - **性能监控**：用 rAF 实现 FPS 监控
>
> **一个容易被忽略的点**：rAF 的回调接收高精度时间戳参数，所有同一帧的回调收到相同的时间戳，这对实现时间同步的动画非常重要。

**为什么这个回答出彩？**

1. ✅ 从多个维度对比（执行时机、同步机制、性能、稳定性）
2. ✅ 用时间轴图示化说明问题
3. ✅ 结合实际应用（动画库、React）
4. ✅ 提到了容易被忽略的细节（时间戳参数）

---

## 九、【化骨绵掌】

### 卡片1：直觉理解 🎯

**一句话：** requestAnimationFrame 在浏览器每次重绘前执行回调函数。

**举例：**

```javascript
function animate() {
  element.style.left = x + 'px';
  x += 2;
  requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

**应用：** 实现与屏幕刷新同步的流畅动画。

---

### 卡片2：60fps 帧率 📐

**一句话：** 60Hz 显示器每秒刷新 60 次，每帧时间为 16.6ms。

**计算：**

```
1 秒 = 1000ms
60 帧 = 1000ms ÷ 60 = 16.6ms/帧
```

**应用：** JavaScript 必须在 16.6ms 内完成工作，否则丢帧。

---

### 卡片3：帧预算分配 💰

**一句话：** 每帧 16.6ms 的时间预算，JavaScript 推荐使用 5ms，剩余给浏览器渲染。

**分配：**

```
JavaScript: 5ms
样式计算: 2ms
布局: 2ms
绘制: 2ms
合成: 2ms
其他: 3.6ms
总计: 16.6ms
```

**应用：** React 时间切片的 5ms 默认值来源。

---

### 卡片4：时间戳参数 ⏱️

**一句话：** rAF 回调接收高精度时间戳，表示当前帧开始时间。

**举例：**

```javascript
let lastTime = null;
function animate(timestamp) {
  if (lastTime) {
    console.log(`距上一帧: ${timestamp - lastTime}ms`);
  }
  lastTime = timestamp;
  requestAnimationFrame(animate);
}
```

**应用：** 实现基于时间的动画，不受帧率影响。

---

### 卡片5：基于时间的动画 🎨

**一句话：** 动画进度应基于时间戳，而非帧数，以适应不同刷新率。

**对比：**

```javascript
// ❌ 基于帧数（受刷新率影响）
x += 2;

// ✅ 基于时间（不受刷新率影响）
const elapsed = timestamp - startTime;
x = (elapsed / 1000) * 100; // 每秒 100px
```

**应用：** 确保动画在不同显示器上速度一致。

---

### 卡片6：自动节能 🔋

**一句话：** 页面不可见时（切换标签页），rAF 自动暂停执行。

**对比：**

| API | 页面不可见时 |
|-----|------------|
| rAF | 暂停执行 ✅ |
| setTimeout | 继续执行 ❌ |

**应用：** 节省 CPU 和电池，提升用户体验。

---

### 卡片7：执行时序 🔄

**一句话：** rAF 在浏览器渲染管道的"渲染前"阶段执行。

**时序：**

```
宏任务 → 微任务 → [需要渲染？]
                    ↓ 是
                 rAF 回调
                    ↓
                 样式计算
                    ↓
                   布局
                    ↓
                   绘制
```

**应用：** rAF 中修改 DOM 会在本帧渲染中生效。

---

### 卡片8：取消动画 🛑

**一句话：** 使用 cancelAnimationFrame 取消已调度的动画帧。

**举例：**

```javascript
const id = requestAnimationFrame(animate);

// 取消
cancelAnimationFrame(id);
```

**应用：** 组件卸载时清理动画，避免内存泄漏。

---

### 卡片9：FPS 监控 📊

**一句话：** 用 rAF 实现帧率监控，检测性能问题。

**实现：**

```javascript
let frames = 0;
let lastTime = performance.now();

function monitor() {
  frames++;
  const now = performance.now();

  if (now - lastTime >= 1000) {
    console.log(`FPS: ${frames}`);
    frames = 0;
    lastTime = now;
  }

  requestAnimationFrame(monitor);
}
```

**应用：** 性能分析和优化。

---

### 卡片10：总结与延伸 🎓

**总结：**

requestAnimationFrame 是浏览器提供的帧同步 API，通过在每次屏幕刷新前执行回调，确保动画流畅、节能，是现代 Web 动画的标准方案。

**核心优势：**
1. 与硬件刷新率同步
2. 自动节能优化
3. 高精度时间戳
4. 稳定的帧率

**延伸学习：**
- 浏览器渲染管道（Rendering Pipeline）
- React 时间切片（Time Slicing）
- Web Animations API
- Performance API

---

## 十、【一句话总结】

**requestAnimationFrame 是浏览器的帧同步回调机制，在每次屏幕刷新前执行，与硬件刷新率完美同步，自动节能优化，提供高精度时间戳，是实现流畅动画和帧预算管理的最佳方案，也是 React 性能优化和时间切片思想的基础。**

---

## 附录

### 学习检查清单

- [ ] 理解 rAF 的执行时机（渲染前）
- [ ] 掌握 60fps 和 16.6ms 帧预算的概念
- [ ] 理解为什么 rAF 比 setTimeout 更适合动画
- [ ] 能够实现基于时间戳的动画
- [ ] 理解帧预算分配（5ms JavaScript + 11.6ms 渲染）
- [ ] 掌握 rAF 的自动节能特性
- [ ] 能够实现 FPS 监控
- [ ] 理解 React 时间切片与帧预算的关系

### 下一步学习建议

1. **深入浏览器渲染管道**：了解样式计算、布局、绘制、合成的完整流程
2. **学习 Performance API**：掌握性能监控和分析工具
3. **研究 React Scheduler**：理解 React 如何利用帧预算进行时间切片
4. **实践 Web Animations API**：学习声明式动画 API

### 参考资源

- [MDN - requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame)
- [Paul Irish - requestAnimationFrame for Smart Animating](https://www.paulirish.com/2011/requestanimationframe-for-smart-animating/)
- [Google Developers - Rendering Performance](https://developers.google.com/web/fundamentals/performance/rendering)
- [Jake Archibald - Rendering Performance](https://jakearchibald.com/2013/animated-line-drawing-svg/)
- [React - Time Slicing](https://github.com/facebook/react/issues/11171)
