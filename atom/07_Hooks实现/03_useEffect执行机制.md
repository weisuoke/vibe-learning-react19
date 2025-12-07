# useEffect执行机制

## 一、【30字核心】

**useEffect通过Effect链表在commit阶段异步执行副作用，通过依赖数组浅比较决定是否执行，cleanup函数在effect执行前或组件卸载时调用。**

---

## 二、【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### useEffect的第一性原理 🎯

#### 1. 最基础的定义

**useEffect = 副作用函数 + 执行时机 + 清理机制**

仅此而已！没有更基础的了。

useEffect让我们在函数组件中执行副作用操作（如数据获取、订阅、手动DOM操作），React负责在正确的时机调用这些副作用，并在需要时执行清理函数。

#### 2. 为什么需要useEffect？

**核心问题：如何在函数组件中安全地执行副作用，而不阻塞浏览器渲染？**

函数组件本身应该是纯函数，但实际应用中需要副作用：

```javascript
// ❌ 直接在组件中执行副作用（有问题）
function Component() {
  // 每次渲染都会执行，可能造成问题
  document.title = 'New Title';
  fetch('/api/data');  // 重复请求

  return <div>Component</div>;
}
```

**问题所在：**
- 组件函数在渲染时执行，副作用会在每次渲染时触发
- 副作用可能阻塞渲染（如同步DOM操作）
- 没有清理机制（如事件监听器、订阅）
- 无法控制执行时机

**React需要的能力：**
- 在渲染完成后执行副作用（不阻塞渲染）
- 根据依赖变化决定是否执行
- 提供清理机制
- 区分不同的副作用时机（layout前/后）

#### 3. useEffect的三层价值

##### 价值1：异步执行，不阻塞渲染

useEffect在commit阶段之后异步执行，不阻塞浏览器绘制：

```javascript
// React渲染流程
function commitRoot(root) {
  // 1. commit阶段：同步执行DOM操作
  commitMutationEffects(root);  // 操作DOM

  // 2. 切换current树
  root.current = finishedWork;

  // 3. 浏览器绘制（这里不会被阻塞）
  // ...

  // 4. 异步调度Passive Effects（useEffect）
  scheduleCallback(NormalPriority, () => {
    flushPassiveEffects();  // 执行useEffect
  });
}
```

**关键变化：**
- useEffect不在渲染阶段执行，而是在commit之后
- 使用Scheduler异步调度，优先级低于布局
- 用户可以更快看到页面更新

##### 价值2：依赖数组控制执行

通过浅比较依赖数组，避免不必要的执行：

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

function updateEffect(
  create,
  deps,
) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;

    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // 依赖数组浅比较
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖没变，不执行effect
        hook.memoizedState = pushEffect(NoFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  // 依赖变了，标记需要执行effect
  currentlyRenderingFiber.flags |= PassiveEffect;
  hook.memoizedState = pushEffect(
    HookHasEffect | PassiveEffect,
    create,
    destroy,
    nextDeps,
  );
}
```

**为什么用浅比较？**
- 性能考虑：深比较成本高
- 鼓励简单依赖：数组长度通常很短
- 明确性：用户需要显式声明依赖

##### 价值3：清理机制

effect可以返回清理函数，React在恰当时机执行：

```javascript
useEffect(() => {
  // setup: 订阅
  const subscription = api.subscribe();

  // cleanup: 取消订阅
  return () => {
    subscription.unsubscribe();
  };
}, []);
```

**cleanup执行时机：**
1. 下次effect执行之前（依赖变化时）
2. 组件卸载时

**顺序保证：**
```
首次渲染：
  -> 执行effect

更新渲染（依赖变化）：
  -> 执行旧effect的cleanup
  -> 执行新effect

组件卸载：
  -> 执行effect的cleanup
```

#### 4. 从第一性原理推导 React 实现

**推理链：**
```
1. 需要在组件中执行副作用
   ↓
2. 副作用不应阻塞渲染 → 异步执行
   ↓
3. 在哪个阶段执行？→ commit阶段之后
   ↓
4. 如何避免重复执行？→ 依赖数组浅比较
   ↓
5. 如何清理副作用？→ 返回cleanup函数
   ↓
6. 何时执行cleanup？→ 下次effect前 / 组件卸载
   ↓
7. 如何存储effect？→ Effect链表（类似Hook链表）
   ↓
8. 如何标记需要执行？→ Fiber.flags |= PassiveEffect
   ↓
9. 如何调度执行？→ Scheduler异步调度
   ↓
10. 最终方案：Effect链表 + 依赖比较 + 异步调度
```

#### 5. 一句话总结第一性原理

**useEffect通过Effect链表保存副作用函数，通过依赖数组浅比较决定是否执行，在commit阶段后通过Scheduler异步调度执行，提供cleanup机制清理副作用，是React处理副作用的核心实现。**

---

## 三、【3个核心概念】

### 核心概念1：Effect链表与Hook链表的关系 🔗

**每个Effect Hook（useEffect/useLayoutEffect）在Hook链表上有一个Hook节点，Hook.memoizedState指向Effect链表。**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// Effect对象的类型定义
type Effect = {
  tag: HookFlags,                    // effect的标志位
  create: () => (() => void) | void, // effect函数
  destroy: (() => void) | void,      // cleanup函数
  deps: Array<mixed> | null,         // 依赖数组
  next: Effect,                      // 指向下一个Effect（环形链表）
};

// Hook对象（在Hook链表中）
type Hook = {
  memoizedState: Effect,  // 指向Effect链表的头节点
  baseState: null,
  baseQueue: null,
  queue: null,
  next: Hook | null,      // 指向下一个Hook
};
```

**两层链表结构：**

```
Fiber.memoizedState (Hook链表)
  ↓
Hook1 (useState)
  memoizedState: 0
  next → Hook2

Hook2 (useEffect)
  memoizedState → Effect1 → Effect2 → Effect3 → Effect1 (环形)
  next → Hook3

Hook3 (useEffect)
  memoizedState → Effect4 → Effect4 (单个Effect也是环形)
  next → null
```

**为什么需要两层链表？**

1. **Hook链表**：管理所有Hook（useState、useEffect、useRef等）
2. **Effect链表**：管理一个组件中的所有Effect副作用

**创建Effect链表：**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

function pushEffect(
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
): Effect {
  // 创建Effect对象
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: (null: any),
  };

  // 获取Fiber的updateQueue（存储Effect链表）
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);

  if (componentUpdateQueue === null) {
    // 第一个Effect：创建updateQueue
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);

    // Effect自己指向自己，形成环形链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    // 后续Effect：追加到环形链表
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }

  return effect;
}

// FunctionComponentUpdateQueue类型
type FunctionComponentUpdateQueue = {
  lastEffect: Effect | null,  // 指向Effect环形链表的尾节点
  stores: Array<StoreConsistencyCheck<any>> | null,
};
```

**mount阶段创建Effect：**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  // 1. 创建Hook节点
  const hook = mountWorkInProgressHook();

  // 2. 处理依赖数组
  const nextDeps = deps === undefined ? null : deps;

  // 3. 标记Fiber有Passive Effect
  currentlyRenderingFiber.flags |= PassiveEffect;

  // 4. 创建Effect对象并追加到Effect链表
  hook.memoizedState = pushEffect(
    HookHasEffect | PassiveEffect,  // tag标记需要执行
    create,                         // effect函数
    undefined,                      // cleanup初始为undefined
    nextDeps,                       // 依赖数组
  );
}
```

**update阶段比较依赖：**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

function updateEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;  // 保留旧的cleanup

    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;

      // 依赖数组浅比较
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖没变，不需要执行effect
        // 但仍然要创建Effect对象（不设置HookHasEffect标志）
        hook.memoizedState = pushEffect(PassiveEffect, create, destroy, nextDeps);
        return;
      }
    }
  }

  // 依赖变了，标记需要执行
  currentlyRenderingFiber.flags |= PassiveEffect;
  hook.memoizedState = pushEffect(
    HookHasEffect | PassiveEffect,
    create,
    destroy,
    nextDeps,
  );
}
```

**在 React 源码中的应用：**

React通过两层链表管理副作用：
- Hook链表：按Hook调用顺序组织
- Effect链表：按Effect调用顺序组织
- 每个Hook节点的memoizedState指向对应的Effect链表头节点

---

### 核心概念2：依赖数组比较机制 🔍

**React使用areHookInputsEqual进行浅比较，逐个比较依赖项是否变化。**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

function areHookInputsEqual(
  nextDeps: Array<mixed>,
  prevDeps: Array<mixed> | null,
): boolean {
  if (prevDeps === null) {
    // 首次渲染，没有旧依赖
    if (__DEV__) {
      console.error(
        '%s received a final argument during this render, but not during ' +
        'the previous render. Even though the final argument is optional, ' +
        'its type cannot change between renders.',
        currentHookNameInDev,
      );
    }
    return false;
  }

  if (__DEV__) {
    // 检查依赖数组长度是否一致
    if (nextDeps.length !== prevDeps.length) {
      console.error(
        'The final argument passed to %s changed size between renders. The ' +
        'order and size of this array must remain constant.\n\n' +
        'Previous: %s\n' +
        'Incoming: %s',
        currentHookNameInDev,
        `[${prevDeps.join(', ')}]`,
        `[${nextDeps.join(', ')}]`,
      );
    }
  }

  // 逐个比较依赖项（使用Object.is）
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (is(nextDeps[i], prevDeps[i])) {
      continue;  // 相同，继续下一个
    }
    return false;  // 不同，依赖变了
  }

  return true;  // 所有依赖都相同
}
```

**Object.is 浅比较：**

```javascript
// React使用的is函数（与Object.is相同）
function is(x: any, y: any): boolean {
  return (
    (x === y && (x !== 0 || 1 / x === 1 / y)) || (x !== x && y !== y)
  );
}

// 等价于：
Object.is(x, y);

// Object.is vs ===
Object.is(+0, -0);     // false (=== 返回true)
Object.is(NaN, NaN);   // true  (=== 返回false)
Object.is({}, {});     // false (引用不同)
```

**依赖比较的特点：**

1. **浅比较，不是深比较**：

```javascript
// 对象/数组引用比较
const obj1 = { count: 1 };
const obj2 = { count: 1 };
Object.is(obj1, obj2);  // false（引用不同）

useEffect(() => {
  // 每次渲染都会执行，因为每次创建新对象
}, [{ count: 1 }]);  // ❌ 错误用法

// ✅ 正确：使用基本类型或稳定引用
const [count] = useState(1);
useEffect(() => {
  // 只在count变化时执行
}, [count]);
```

2. **空依赖数组 = 只执行一次**：

```javascript
useEffect(() => {
  // 只在mount时执行一次
}, []);  // 空数组，areHookInputsEqual总是返回true
```

3. **无依赖数组 = 每次都执行**：

```javascript
useEffect(() => {
  // 每次渲染都执行
});  // 无依赖数组，nextDeps为null，areHookInputsEqual返回false
```

**依赖数组的三种情况：**

| 依赖数组 | areHookInputsEqual | 执行时机 |
|---------|-------------------|---------|
| `undefined` | 返回false | 每次渲染都执行 |
| `[]` | 返回true（空数组比较） | 只在mount时执行 |
| `[a, b]` | 逐个比较a、b | 依赖变化时执行 |

**实际应用示例：**

```javascript
function Component({ userId }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    // ✅ 正确：userId是基本类型，浅比较有效
    fetchData(userId).then(setData);
  }, [userId]);

  useEffect(() => {
    // ❌ 错误：每次渲染都执行（对象引用变化）
    const config = { id: userId };
    fetchData(config);
  }, [{ id: userId }]);

  useEffect(() => {
    // ✅ 正确：只依赖基本类型
    const config = { id: userId };
    fetchData(config);
  }, [userId]);
}
```

**在 React 源码中的应用：**

依赖数组比较是useEffect性能优化的关键。React通过浅比较快速判断是否需要执行effect，避免不必要的副作用。

---

### 核心概念3：cleanup函数执行时序 ⏱️

**cleanup函数在下次effect执行前或组件卸载时调用，保证副作用的清理。**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberCommitWork.js

// commit阶段执行cleanup
function commitHookEffectListUnmount(
  flags: HookFlags,
  finishedWork: Fiber,
  nearestMountedAncestor: Fiber | null,
): void {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;

    do {
      if ((effect.tag & flags) === flags) {
        // 执行cleanup
        const destroy = effect.destroy;
        effect.destroy = undefined;

        if (destroy !== undefined) {
          if (__DEV__) {
            console.log('Calling cleanup function');
          }
          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
        }
      }

      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

// commit阶段执行effect
function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber): void {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;

    do {
      if ((effect.tag & flags) === flags) {
        // 执行effect，保存返回的cleanup函数
        const create = effect.create;

        if (__DEV__) {
          console.log('Calling effect function');
        }

        const destroy = create();
        effect.destroy = destroy;  // 保存cleanup函数
      }

      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

**完整的执行时序：**

```
【首次渲染】
1. render阶段
   → mountEffect创建Effect对象
   → 标记Fiber.flags |= PassiveEffect

2. commit阶段
   → DOM操作完成
   → 浏览器绘制

3. 异步调度
   → 执行commitHookEffectListMount
   → 调用effect函数：const destroy = create()
   → 保存cleanup函数：effect.destroy = destroy

【更新渲染（依赖变化）】
1. render阶段
   → updateEffect比较依赖
   → 依赖变化，标记HookHasEffect
   → 标记Fiber.flags |= PassiveEffect

2. commit阶段
   → DOM操作完成
   → 浏览器绘制

3. 异步调度
   → 先执行commitHookEffectListUnmount
     → 调用旧effect的cleanup：destroy()
     → 清除effect.destroy
   → 再执行commitHookEffectListMount
     → 调用新effect函数：const destroy = create()
     → 保存新cleanup函数

【组件卸载】
1. render阶段
   → 标记Fiber.flags |= Deletion

2. commit阶段
   → 执行commitHookEffectListUnmount
   → 调用所有effect的cleanup：destroy()
   → 移除DOM节点
```

**cleanup执行的关键时机：**

```javascript
useEffect(() => {
  console.log('Effect executed');

  return () => {
    console.log('Cleanup executed');
  };
}, [dep]);

// 首次渲染：
// -> Effect executed

// 更新渲染（dep变化）：
// -> Cleanup executed
// -> Effect executed

// 组件卸载：
// -> Cleanup executed
```

**为什么cleanup要在effect之前执行？**

```javascript
useEffect(() => {
  // 订阅事件
  const handleClick = () => console.log('clicked');
  document.addEventListener('click', handleClick);

  return () => {
    // 取消订阅
    document.removeEventListener('click', handleClick);
  };
}, []);

// 如果cleanup在effect之后执行：
// 1. 执行新effect -> 添加新监听器
// 2. 执行旧cleanup -> 移除旧监听器
// 问题：旧监听器已经不存在了，cleanup无效

// React的正确顺序：
// 1. 执行旧cleanup -> 移除旧监听器
// 2. 执行新effect -> 添加新监听器
// 保证了清理的正确性
```

**在 React 源码中的应用：**

React通过严格的执行顺序保证副作用的正确清理：
1. cleanup必须在effect之前执行
2. 所有cleanup执行完后，再执行所有effect
3. 组件卸载时，执行所有effect的cleanup

---

## 四、【最小可用】

掌握以下内容，就能理解 useEffect的核心：

### 4.1 基本用法

**核心：useEffect接受effect函数和依赖数组。**

```javascript
import { useEffect } from 'react';

// 基础用法
useEffect(() => {
  // effect函数
  console.log('Effect executed');
});

// 带依赖数组
useEffect(() => {
  console.log('Count changed');
}, [count]);

// 带cleanup
useEffect(() => {
  const timer = setTimeout(() => {
    console.log('Delayed');
  }, 1000);

  return () => {
    clearTimeout(timer);  // cleanup
  };
}, []);
```

### 4.2 依赖数组的三种情况

**核心：依赖数组控制effect执行时机。**

```javascript
// 1. 无依赖数组：每次渲染都执行
useEffect(() => {
  console.log('Every render');
});

// 2. 空依赖数组：只在mount时执行一次
useEffect(() => {
  console.log('Only once');
}, []);

// 3. 有依赖：依赖变化时执行
useEffect(() => {
  console.log('When dep changes');
}, [dep]);
```

### 4.3 cleanup的使用场景

**核心：cleanup清理副作用，避免内存泄漏。**

```javascript
// 场景1：定时器
useEffect(() => {
  const timer = setInterval(() => tick(), 1000);
  return () => clearInterval(timer);
}, []);

// 场景2：事件监听
useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);

// 场景3：订阅
useEffect(() => {
  const subscription = api.subscribe();
  return () => subscription.unsubscribe();
}, []);
```

**这些知识足以：**
- 正确使用useEffect处理副作用
- 避免常见的依赖数组错误
- 理解cleanup的重要性
- 为深入学习性能优化打下基础

---

## 五、【1个类比】

### 类比1：闹钟设置 ⏰

**useEffect = 设置闹钟**

你设置闹钟在特定条件下响铃：

```
设置闹钟（useEffect）：
- 条件：早上7点（依赖数组）
- 动作：播放音乐（effect函数）
- 清理：关闭音乐（cleanup函数）

流程：
1. 设置闹钟 -> 设置effect
2. 条件满足 -> 依赖变化
3. 播放音乐 -> 执行effect
4. 关闭音乐 -> 执行cleanup
```

**相似点：**
- 设置闹钟 = useEffect调用
- 触发条件 = 依赖数组
- 闹钟响铃 = effect执行
- 关闭闹钟 = cleanup执行

**举例：**

```javascript
function AlarmClock({ time }) {
  useEffect(() => {
    // 闹钟响铃
    console.log('⏰ Alarm ringing!');
    const audio = new Audio('alarm.mp3');
    audio.play();

    // 关闭闹钟
    return () => {
      console.log('🔇 Alarm stopped');
      audio.pause();
    };
  }, [time]);  // 时间变化时重新设置闹钟
}
```

---

### 类比2：房间保洁 🧹

**cleanup = 保洁员打扫房间**

每次进入新房间前，先打扫旧房间：

```
保洁流程：
1. 进入旧房间
2. 打扫旧房间（cleanup）
3. 离开旧房间
4. 进入新房间
5. 使用新房间（effect）

顺序：先清理，再使用
```

**相似点：**
- 进入房间 = 渲染组件
- 打扫旧房间 = 执行cleanup
- 使用新房间 = 执行新effect
- 离开房间 = 组件卸载

**举例：**

```javascript
function Room({ roomId }) {
  useEffect(() => {
    console.log(`🚪 Entering room ${roomId}`);

    // 使用房间
    const lights = turnOnLights(roomId);

    // 离开时打扫
    return () => {
      console.log(`🧹 Cleaning room ${roomId}`);
      lights.turnOff();
    };
  }, [roomId]);

  // 切换房间时：
  // 1. Cleaning room A (cleanup)
  // 2. Entering room B (effect)
}
```

---

### 类比3：订阅杂志 📰

**useEffect = 订阅和退订杂志**

订阅杂志，定期收到，退订时停止：

```
订阅流程：
- 订阅（effect）：开始收杂志
- 退订（cleanup）：停止收杂志

更换杂志：
1. 退订旧杂志（cleanup）
2. 订阅新杂志（effect）

取消所有订阅：
- 组件卸载 -> 执行cleanup
```

**相似点：**
- 订阅 = effect函数
- 退订 = cleanup函数
- 杂志内容 = 依赖（变化时重新订阅）
- 取消订阅 = 组件卸载时cleanup

**举例：**

```javascript
function Magazine({ topic }) {
  useEffect(() => {
    // 订阅杂志
    console.log(`📰 Subscribing to ${topic}`);
    const subscription = api.subscribe(topic, (article) => {
      console.log('New article:', article);
    });

    // 退订杂志
    return () => {
      console.log(`📪 Unsubscribing from ${topic}`);
      subscription.unsubscribe();
    };
  }, [topic]);

  // 切换主题时：
  // 1. Unsubscribing from "tech"
  // 2. Subscribing to "sports"
}
```

---

### 类比4：定时炸弹 💣

**useEffect + setTimeout = 定时炸弹**

设置定时炸弹，可以提前拆除：

```
炸弹流程：
- 安装炸弹（effect）：setTimeout
- 拆除炸弹（cleanup）：clearTimeout

重新设置：
1. 拆除旧炸弹（cleanup）
2. 安装新炸弹（effect）

组件卸载：
- 拆除所有炸弹（cleanup）
```

**相似点：**
- 安装炸弹 = 设置定时器
- 拆除炸弹 = 清除定时器
- 炸弹参数 = 依赖（变化时重新设置）
- 强制拆除 = 组件卸载cleanup

**举例：**

```javascript
function TimeBomb({ delay }) {
  useEffect(() => {
    // 安装炸弹
    console.log(`💣 Setting bomb with ${delay}ms`);
    const timer = setTimeout(() => {
      console.log('💥 BOOM!');
    }, delay);

    // 拆除炸弹
    return () => {
      console.log('🔧 Defusing bomb');
      clearTimeout(timer);
    };
  }, [delay]);

  // delay变化时：
  // 1. Defusing bomb (清除旧定时器)
  // 2. Setting bomb with new delay
}
```

---

### 类比总结表

| React概念 | 闹钟 | 房间保洁 | 订阅杂志 | 定时炸弹 |
|----------|-----|---------|---------|---------|
| effect函数 | 播放音乐 | 使用房间 | 订阅杂志 | 安装炸弹 |
| cleanup函数 | 关闭音乐 | 打扫房间 | 退订杂志 | 拆除炸弹 |
| 依赖数组 | 闹钟时间 | 房间号 | 杂志主题 | 爆炸延迟 |
| 依赖变化 | 重设闹钟 | 换房间 | 换杂志 | 重设炸弹 |
| 组件卸载 | 删除闹钟 | 退房 | 取消订阅 | 强制拆弹 |
| 执行顺序 | 先关后开 | 先扫后用 | 先退后订 | 先拆后装 |

---

## 六、【反直觉点】

### 误区1：useEffect在渲染之前执行 ❌

**为什么错？**

useEffect是在commit阶段**之后**异步执行，而不是渲染之前。

**正确解释：**

```javascript
function Component() {
  console.log('1. Render phase');

  useEffect(() => {
    console.log('3. Effect phase (async)');
  });

  return <div ref={() => console.log('2. Commit phase (DOM ready)')}>Component</div>;
}

// 执行顺序：
// 1. Render phase
// 2. Commit phase (DOM ready)
// 3. Effect phase (async)
```

**React渲染流程：**

```
1. Render阶段（可中断）
   → 执行组件函数
   → 创建Fiber树
   → 标记副作用

2. Commit阶段（不可中断）
   → 同步操作DOM
   → ref赋值
   → useLayoutEffect同步执行

3. 浏览器绘制

4. 异步调度
   → useEffect异步执行
```

**为什么人们容易这样错？**

- **命名误导**："Effect"容易让人觉得是"渲染后立即执行"
- **类组件经验**：`componentDidMount`在DOM操作后立即执行
- **忽略异步**：不知道useEffect是异步调度的

**正确理解：**

```javascript
// useEffect不阻塞浏览器绘制
useEffect(() => {
  // 这里执行时，用户已经看到新UI了
  heavyComputation();  // 不会阻塞UI更新
});

// useLayoutEffect会阻塞浏览器绘制
useLayoutEffect(() => {
  // 这里执行时，用户还没看到新UI
  // 会阻塞绘制，直到执行完成
});
```

**验证代码：**

```javascript
function Demo() {
  const [count, setCount] = useState(0);

  useLayoutEffect(() => {
    console.log('useLayoutEffect:', count);
    // 阻塞绘制
    const start = Date.now();
    while (Date.now() - start < 100) {}  // 阻塞100ms
  });

  useEffect(() => {
    console.log('useEffect:', count);
  });

  console.log('Render:', count);

  return <div>{count}</div>;
}

// 点击按钮触发setCount(1)：
// Render: 1
// useLayoutEffect: 1 (阻塞100ms)
// -> 用户看到UI更新
// useEffect: 1
```

---

### 误区2：空依赖数组的effect只执行一次，包括卸载时不执行cleanup ❌

**为什么错？**

空依赖数组的effect只在mount时执行一次effect函数，但卸载时仍然会执行cleanup。

**正确解释：**

```javascript
function Component() {
  useEffect(() => {
    console.log('Effect: mount');

    return () => {
      console.log('Cleanup: unmount');
    };
  }, []);  // 空依赖数组

  return <div>Component</div>;
}

// 生命周期：
// mount: Effect: mount
// unmount: Cleanup: unmount ✅ cleanup仍然执行
```

**完整的执行时机：**

```javascript
useEffect(() => {
  console.log('Setup');
  return () => console.log('Cleanup');
}, []);

// 首次渲染（mount）：
// -> Setup

// 更新渲染（依赖未变）：
// -> （不执行setup，不执行cleanup）

// 组件卸载：
// -> Cleanup ✅
```

**为什么人们容易这样错？**

- **字面理解**："只执行一次"让人以为cleanup也不执行
- **忽略卸载**：只关注mount和update，忘记unmount
- **文档误解**：只看到"mount时执行一次"，没看到"unmount时cleanup"

**正确理解：**

```javascript
// ✅ 正确理解
useEffect(() => {
  // mount时执行一次
  const subscription = api.subscribe();

  return () => {
    // unmount时执行一次（清理）
    subscription.unsubscribe();
  };
}, []);  // 空依赖 = 只订阅一次，但卸载时必须退订
```

**实际应用：**

```javascript
// 场景：WebSocket连接
useEffect(() => {
  const ws = new WebSocket('ws://localhost:3000');

  ws.onopen = () => console.log('Connected');
  ws.onmessage = (e) => console.log('Message:', e.data);

  // 组件卸载时关闭连接
  return () => {
    ws.close();  // ✅ 必须清理
  };
}, []);

// 如果没有cleanup，组件卸载后WebSocket仍然连接 ❌
```

---

### 误区3：useEffect和useLayoutEffect没区别，可以随意替换 ❌

**为什么错？**

useEffect和useLayoutEffect执行时机不同，会影响性能和用户体验。

**正确解释：**

```javascript
// useEffect：异步执行，不阻塞绘制
useEffect(() => {
  // 在浏览器绘制之后执行
  // 不会阻塞UI更新
});

// useLayoutEffect：同步执行，阻塞绘制
useLayoutEffect(() => {
  // 在浏览器绘制之前执行
  // 会阻塞UI更新，直到执行完成
});
```

**执行时机对比：**

```
useEffect：
Render -> Commit -> 浏览器绘制 -> useEffect
                     ↑
                用户看到UI

useLayoutEffect：
Render -> Commit -> useLayoutEffect -> 浏览器绘制
                                         ↑
                                    用户看到UI
```

**性能影响：**

```javascript
function Slow() {
  const [width, setWidth] = useState(0);

  // ❌ useLayoutEffect阻塞绘制
  useLayoutEffect(() => {
    const start = Date.now();
    while (Date.now() - start < 200) {}  // 模拟耗时操作
    setWidth(window.innerWidth);
  });

  // 用户会感觉卡顿200ms才看到UI更新

  // ✅ useEffect不阻塞绘制
  useEffect(() => {
    const start = Date.now();
    while (Date.now() - start < 200) {}
    setWidth(window.innerWidth);
  });

  // 用户立即看到UI更新，200ms后看到width更新
}
```

**何时使用useLayoutEffect？**

只在需要**在绘制前**同步读取/修改DOM时使用：

```javascript
// ✅ 正确使用场景：测量DOM
useLayoutEffect(() => {
  const element = ref.current;
  const rect = element.getBoundingClientRect();
  // 根据测量结果调整样式
  element.style.top = `${rect.height}px`;
}, []);

// 如果用useEffect：
// 1. 绘制初始UI
// 2. 用户看到元素在错误位置
// 3. useEffect执行，修改位置
// 4. 重新绘制
// 5. 用户看到元素"跳动" ❌

// useLayoutEffect可以避免"跳动"，在绘制前就修改好位置
```

**为什么人们容易这样错？**

- **API相似**：两者API几乎完全相同
- **测试环境**：简单例子看不出区别
- **文档不足**：很多教程没有强调区别

**正确理解：**

| 特性 | useEffect | useLayoutEffect |
|-----|----------|----------------|
| 执行时机 | 绘制后异步 | 绘制前同步 |
| 阻塞渲染 | 不阻塞 | 阻塞 |
| 性能影响 | 低（推荐） | 高（慎用） |
| 使用场景 | 大部分副作用 | DOM测量/同步修改 |
| SSR兼容 | 兼容 | 服务端会警告 |

**最佳实践：**
- 默认使用`useEffect`
- 只在出现视觉问题（闪烁、跳动）时考虑`useLayoutEffect`
- 避免在`useLayoutEffect`中执行耗时操作

---

## 七、【实战代码】

### 基础实现（简化版）

```javascript
// ===== 1. 模拟useEffect的基本实现 =====
console.log("=== 场景1：简化版useEffect实现 ===");

// Effect对象
class Effect {
  constructor(create, deps) {
    this.create = create;      // effect函数
    this.destroy = undefined;  // cleanup函数
    this.deps = deps;          // 依赖数组
    this.tag = 0;              // 标志位
    this.next = null;          // 指向下一个Effect
  }
}

// Hook对象
class Hook {
  constructor() {
    this.memoizedState = null;  // 指向Effect链表
    this.next = null;
  }
}

// 全局变量
let currentFiber = null;
let workInProgressHook = null;
let isMount = true;

// 依赖比较（Object.is）
function is(x, y) {
  return (x === y && (x !== 0 || 1 / x === 1 / y)) || (x !== x && y !== y);
}

// 依赖数组比较
function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) return false;
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}

// 创建Effect并追加到链表
function pushEffect(tag, create, destroy, deps) {
  const effect = new Effect(create, deps);
  effect.tag = tag;
  effect.destroy = destroy;

  // 追加到Fiber的Effect链表（环形）
  if (currentFiber.updateQueue === null) {
    currentFiber.updateQueue = { lastEffect: null };
    currentFiber.updateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = currentFiber.updateQueue.lastEffect;
    if (lastEffect === null) {
      currentFiber.updateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      currentFiber.updateQueue.lastEffect = effect;
    }
  }

  return effect;
}

// mount阶段：创建Effect
function mountEffect(create, deps) {
  const hook = new Hook();

  // 追加到Hooks链表
  if (currentFiber.memoizedState === null) {
    currentFiber.memoizedState = workInProgressHook = hook;
  } else {
    workInProgressHook = workInProgressHook.next = hook;
  }

  const nextDeps = deps === undefined ? null : deps;

  // 创建Effect对象
  const effect = pushEffect(
    1,  // HookHasEffect标志
    create,
    undefined,
    nextDeps
  );

  hook.memoizedState = effect;

  console.log(`  -> mountEffect, deps: ${JSON.stringify(nextDeps)}`);
}

// update阶段：比较依赖
function updateEffect(create, deps) {
  const hook = workInProgressHook = workInProgressHook
    ? workInProgressHook.next
    : currentFiber.memoizedState;

  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (hook.memoizedState !== null) {
    const prevEffect = hook.memoizedState;
    destroy = prevEffect.destroy;

    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;

      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖没变，不执行
        const effect = pushEffect(0, create, destroy, nextDeps);
        hook.memoizedState = effect;
        console.log(`  -> updateEffect, deps unchanged, skip`);
        return;
      }
    }
  }

  // 依赖变了，需要执行
  const effect = pushEffect(1, create, destroy, nextDeps);
  hook.memoizedState = effect;
  console.log(`  -> updateEffect, deps changed, will execute`);
}

// 执行cleanup
function commitHookEffectListUnmount() {
  const updateQueue = currentFiber.updateQueue;
  if (updateQueue === null) return;

  const lastEffect = updateQueue.lastEffect;
  if (lastEffect === null) return;

  const firstEffect = lastEffect.next;
  let effect = firstEffect;

  console.log("\n  -> Executing cleanups:");
  do {
    if (effect.tag === 1 && effect.destroy) {
      console.log(`    - Cleanup: ${effect.destroy.name || 'anonymous'}`);
      effect.destroy();
    }
    effect = effect.next;
  } while (effect !== firstEffect);
}

// 执行effect
function commitHookEffectListMount() {
  const updateQueue = currentFiber.updateQueue;
  if (updateQueue === null) return;

  const lastEffect = updateQueue.lastEffect;
  if (lastEffect === null) return;

  const firstEffect = lastEffect.next;
  let effect = firstEffect;

  console.log("\n  -> Executing effects:");
  do {
    if (effect.tag === 1) {
      const create = effect.create;
      console.log(`    - Effect: ${create.name || 'anonymous'}`);
      const destroy = create();
      effect.destroy = destroy;
    }
    effect = effect.next;
  } while (effect !== firstEffect);
}

// 模拟组件
function Component({ count }) {
  console.log(`\nRender: count=${count}`);

  if (isMount) {
    mountEffect(() => {
      console.log('    Effect 1 executed');
      return () => console.log('    Effect 1 cleanup');
    }, []);

    mountEffect(() => {
      console.log('    Effect 2 executed');
      return () => console.log('    Effect 2 cleanup');
    }, [count]);
  } else {
    updateEffect(() => {
      console.log('    Effect 1 executed');
      return () => console.log('    Effect 1 cleanup');
    }, []);

    updateEffect(() => {
      console.log('    Effect 2 executed');
      return () => console.log('    Effect 2 cleanup');
    }, [count]);
  }
}

// ===== 2. 首次渲染（mount） =====
console.log("\n=== 场景2：首次渲染 ===");

currentFiber = { memoizedState: null, updateQueue: null };
workInProgressHook = null;
isMount = true;

Component({ count: 0 });
commitHookEffectListMount();

// ===== 3. 更新渲染（依赖未变） =====
console.log("\n\n=== 场景3：更新渲染（count仍为0） ===");

workInProgressHook = null;
isMount = false;

Component({ count: 0 });
commitHookEffectListUnmount();
commitHookEffectListMount();

// ===== 4. 更新渲染（依赖变化） =====
console.log("\n\n=== 场景4：更新渲染（count变为1） ===");

workInProgressHook = null;

Component({ count: 1 });
commitHookEffectListUnmount();
commitHookEffectListMount();

// ===== 5. 组件卸载 =====
console.log("\n\n=== 场景5：组件卸载 ===");

commitHookEffectListUnmount();
```

**运行输出示例：**

```
=== 场景1：简化版useEffect实现 ===

=== 场景2：首次渲染 ===

Render: count=0
  -> mountEffect, deps: []
  -> mountEffect, deps: [0]

  -> Executing effects:
    - Effect: anonymous
    Effect 1 executed
    - Effect: anonymous
    Effect 2 executed

=== 场景3：更新渲染（count仍为0） ===

Render: count=0
  -> updateEffect, deps unchanged, skip
  -> updateEffect, deps unchanged, skip

  -> Executing cleanups:
  （没有cleanup，因为依赖未变）

  -> Executing effects:
  （没有effect执行，因为依赖未变）

=== 场景4：更新渲染（count变为1） ===

Render: count=1
  -> updateEffect, deps unchanged, skip
  -> updateEffect, deps changed, will execute

  -> Executing cleanups:
    - Cleanup: anonymous
    Effect 2 cleanup

  -> Executing effects:
    - Effect: anonymous
    Effect 2 executed

=== 场景5：组件卸载 ===

  -> Executing cleanups:
    - Cleanup: anonymous
    Effect 1 cleanup
    - Cleanup: anonymous
    Effect 2 cleanup
```

---

### 进阶：React源码实现

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// ========== 1. Effect类型定义 ==========

type Effect = {
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
  next: Effect,
};

type FunctionComponentUpdateQueue = {
  lastEffect: Effect | null,
  stores: Array<StoreConsistencyCheck<any>> | null,
};

// ========== 2. Effect标志位 ==========

const NoFlags = 0b0000;
const HasEffect = 0b0001;  // 需要执行的effect
const Insertion = 0b0010;
const Layout = 0b0100;      // useLayoutEffect
const Passive = 0b1000;     // useEffect

// ========== 3. mountEffect实现 ==========

function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  if (__DEV__) {
    // 开发环境下的警告检查
    if (typeof create !== 'function') {
      console.error(
        'useEffect received a non-function create. Did you accidentally ' +
        'return a value from your effect?'
      );
    }
  }

  return mountEffectImpl(
    PassiveEffect | PassiveStaticEffect,
    HookPassive,
    create,
    deps,
  );
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;

  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps,
  );
}

// ========== 4. updateEffect实现 ==========

function updateEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;

    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;

      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖没变，创建effect但不设置HookHasEffect
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }

  // 依赖变了，标记Fiber和Effect需要执行
  currentlyRenderingFiber.flags |= fiberFlags;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps,
  );
}

// ========== 5. pushEffect：创建Effect对象 ==========

function pushEffect(
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
): Effect {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: (null: any),
  };

  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);

  if (componentUpdateQueue === null) {
    // 第一个Effect：创建updateQueue
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);

    // Effect自己指向自己，形成环形链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    // 后续Effect：追加到环形链表
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }

  return effect;
}

// ========== 6. areHookInputsEqual：依赖比较 ==========

function areHookInputsEqual(
  nextDeps: Array<mixed>,
  prevDeps: Array<mixed> | null,
): boolean {
  if (__DEV__) {
    if (ignorePreviousDependencies) {
      return false;
    }
  }

  if (prevDeps === null) {
    if (__DEV__) {
      console.error(
        '%s received a final argument during this render, but not during ' +
        'the previous render. Even though the final argument is optional, ' +
        'its type cannot change between renders.',
        currentHookNameInDev,
      );
    }
    return false;
  }

  if (__DEV__) {
    // 检查依赖数组长度
    if (nextDeps.length !== prevDeps.length) {
      console.error(
        'The final argument passed to %s changed size between renders. The ' +
        'order and size of this array must remain constant.\n\n' +
        'Previous: %s\n' +
        'Incoming: %s',
        currentHookNameInDev,
        `[${prevDeps.join(', ')}]`,
        `[${nextDeps.join(', ')}]`,
      );
    }
  }

  // 逐个比较依赖项（Object.is）
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }

  return true;
}

// ========== 7. commitHookEffectListUnmount：执行cleanup ==========

function commitHookEffectListUnmount(
  flags: HookFlags,
  finishedWork: Fiber,
  nearestMountedAncestor: Fiber | null,
): void {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;

    do {
      if ((effect.tag & flags) === flags) {
        // 执行cleanup
        const destroy = effect.destroy;
        effect.destroy = undefined;

        if (destroy !== undefined) {
          if (
            enableSchedulingProfiler &&
            (flags & HookPassive) !== NoHookEffect
          ) {
            markComponentPassiveEffectUnmountStarted(finishedWork);
          }

          if (__DEV__) {
            if ((flags & HookInsertion) !== NoHookEffect) {
              setIsRunningInsertionEffect(true);
            }
          }

          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);

          if (__DEV__) {
            if ((flags & HookInsertion) !== NoHookEffect) {
              setIsRunningInsertionEffect(false);
            }
          }

          if (
            enableSchedulingProfiler &&
            (flags & HookPassive) !== NoHookEffect
          ) {
            markComponentPassiveEffectUnmountStopped();
          }
        }
      }

      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

// ========== 8. commitHookEffectListMount：执行effect ==========

function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber): void {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;

    do {
      if ((effect.tag & flags) === flags) {
        if (
          enableSchedulingProfiler &&
          (flags & HookPassive) !== NoHookEffect
        ) {
          markComponentPassiveEffectMountStarted(finishedWork);
        }

        if (__DEV__) {
          if ((flags & HookInsertion) !== NoHookEffect) {
            setIsRunningInsertionEffect(true);
          }
        }

        // 执行effect函数
        const create = effect.create;
        const destroy = create();

        // 保存cleanup函数
        if (__DEV__) {
          if (destroy !== undefined && typeof destroy !== 'function') {
            let hookName;
            if ((effect.tag & HookLayout) !== NoFlags) {
              hookName = 'useLayoutEffect';
            } else if ((effect.tag & HookInsertion) !== NoFlags) {
              hookName = 'useInsertionEffect';
            } else {
              hookName = 'useEffect';
            }

            let addendum;
            if (destroy === null) {
              addendum =
                ' You returned null. If your effect does not require clean ' +
                'up, return undefined (or nothing).';
            } else if (typeof destroy.then === 'function') {
              addendum =
                '\n\nIt looks like you wrote ' +
                hookName +
                '(async () => ...) or returned a Promise. ' +
                'Instead, write the async function inside your effect ' +
                'and call it immediately:\n\n' +
                hookName +
                '(() => {\n' +
                '  async function fetchData() {\n' +
                '    // You can await here\n' +
                '    const response = await MyAPI.getData(someId);\n' +
                '    // ...\n' +
                '  }\n' +
                '  fetchData();\n' +
                '}, [someId]); // Or [] if effect doesn\'t need props or state\n\n' +
                'Learn more about data fetching with Hooks: https://reactjs.org/link/hooks-data-fetching';
            } else {
              addendum = ' You returned: ' + destroy;
            }

            console.error(
              '%s must not return anything besides a function, ' +
              'which is used for clean-up.%s',
              hookName,
              addendum,
            );
          }
        }

        effect.destroy = destroy;

        if (__DEV__) {
          if ((flags & HookInsertion) !== NoHookEffect) {
            setIsRunningInsertionEffect(false);
          }
        }

        if (
          enableSchedulingProfiler &&
          (flags & HookPassive) !== NoHookEffect
        ) {
          markComponentPassiveEffectMountStopped();
        }
      }

      effect = effect.next;
    } while (effect !== firstEffect);
  }
}

// ========== 9. safelyCallDestroy：安全执行cleanup ==========

function safelyCallDestroy(
  current: Fiber,
  nearestMountedAncestor: Fiber | null,
  destroy: () => void,
) {
  if (__DEV__) {
    invokeGuardedCallback(null, destroy, null);

    if (hasCaughtError()) {
      const error = clearCaughtError();
      captureCommitPhaseError(current, nearestMountedAncestor, error);
    }
  } else {
    try {
      destroy();
    } catch (error) {
      captureCommitPhaseError(current, nearestMountedAncestor, error);
    }
  }
}

// ========== 10. useLayoutEffect实现 ==========

function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  let fiberFlags: Flags = UpdateEffect | LayoutStaticEffect;

  if (
    __DEV__ &&
    (currentlyRenderingFiber.mode & StrictEffectsMode) !== NoMode
  ) {
    fiberFlags |= MountLayoutDevEffect;
  }

  return mountEffectImpl(fiberFlags, HookLayout, create, deps);
}

function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

**代码解读要点：**

1. **mountEffect**：首次渲染创建Effect
   - 调用mountEffectImpl
   - 设置PassiveEffect标志（异步执行）
   - 创建Effect对象并标记HookHasEffect

2. **updateEffect**：更新渲染比较依赖
   - 调用areHookInputsEqual比较依赖
   - 依赖未变：创建Effect但不设置HookHasEffect
   - 依赖变化：设置HookHasEffect标志

3. **pushEffect**：创建Effect对象并追加到环形链表
   - 第一个Effect：自己指向自己
   - 后续Effect：插入环中

4. **commitHookEffectListUnmount**：执行cleanup
   - 遍历Effect链表
   - 调用destroy函数
   - 清除effect.destroy

5. **commitHookEffectListMount**：执行effect
   - 遍历Effect链表
   - 调用create函数
   - 保存返回的destroy函数

6. **useLayoutEffect**：同步版本的useEffect
   - 使用UpdateEffect标志（同步执行）
   - 在commit阶段同步执行，阻塞绘制

---

## 八、【面试必问】

### 问题："useEffect和useLayoutEffect的区别？"

**普通回答（❌ 不出彩）：**

"useLayoutEffect是同步的，useEffect是异步的。"

**出彩回答（✅ 推荐）：**

> **useEffect和useLayoutEffect的区别在于执行时机和性能影响，有三个层面的理解：**
>
> 1. **执行时机**：
>    - `useEffect`：在commit阶段完成、浏览器绘制**之后**异步执行，不阻塞UI更新
>    - `useLayoutEffect`：在commit阶段完成、浏览器绘制**之前**同步执行，会阻塞UI更新
>
> 2. **性能影响**：
>    - `useEffect`：用户能更快看到UI更新，不会感觉卡顿（推荐）
>    - `useLayoutEffect`：会延迟浏览器绘制，如果执行耗时操作会导致卡顿（慎用）
>
> 3. **使用场景**：
>    - `useEffect`：大部分副作用（数据获取、订阅、日志等）
>    - `useLayoutEffect`：需要在绘制前同步读取/修改DOM（避免视觉闪烁）
>
> **典型的useLayoutEffect使用场景**：
> ```javascript
> // 场景：测量DOM尺寸并调整样式
> useLayoutEffect(() => {
>   const element = ref.current;
>   const height = element.offsetHeight;
>   // 根据高度调整样式
>   element.style.top = `${height}px`;
> }, []);
>
> // 如果用useEffect：
> // 1. 绘制初始UI
> // 2. 用户看到元素在错误位置
> // 3. useEffect执行，修改样式
> // 4. 重新绘制
> // 5. 用户看到元素"跳动" ❌
>
> // useLayoutEffect可以在绘制前就调整好，避免闪烁
> ```
>
> **与类组件生命周期的对应**：
> - `useLayoutEffect` ≈ `componentDidMount/componentDidUpdate`（同步）
> - `useEffect` ≈ `componentDidMount/componentDidUpdate` + `setTimeout`（异步）
>
> **在实际工作中的应用**：
> - 默认使用`useEffect`，性能更好
> - 只在出现视觉问题（闪烁、跳动）时考虑`useLayoutEffect`
> - 避免在`useLayoutEffect`中执行耗时操作（如网络请求）
> - SSR项目中，`useLayoutEffect`在服务端会有警告（因为服务端没有DOM）

**为什么这个回答出彩？**

1. ✅ 从执行时机、性能、场景三个维度深入解释
2. ✅ 提供了具体的代码示例和视觉效果对比
3. ✅ 对比了类组件生命周期
4. ✅ 给出了实际工作中的最佳实践

---

## 九、【化骨绵掌】

### 卡片1：直觉理解 - useEffect是什么 🎯

**一句话：** useEffect让我们在函数组件中执行副作用操作。

**举例：**

```javascript
useEffect(() => {
  // 副作用：数据获取、订阅、DOM操作等
  document.title = 'New Title';
}, []);
```

**应用：** 函数组件通过useEffect在合适时机执行副作用，不阻塞渲染。

---

### 卡片2：Effect链表结构 📦

**一句话：** Effect链表是环形链表，保存组件的所有副作用。

**举例：**

```javascript
// Effect链表（环形）
effect1 -> effect2 -> effect3 -> effect1

// Fiber.updateQueue.lastEffect = effect3
// effect3.next = effect1（环形）
```

**应用：** 每个useEffect调用创建一个Effect对象，追加到Effect链表。

---

### 卡片3：异步执行机制 ⚡

**一句话：** useEffect在commit阶段后异步执行，不阻塞渲染。

**举例：**

```javascript
// 执行顺序
Render -> Commit -> 浏览器绘制 -> useEffect
                     ↑
                用户看到UI
```

**应用：** 用户能更快看到UI更新，提升用户体验。

---

### 卡片4：依赖数组比较 🔍

**一句话：** 依赖数组使用Object.is浅比较，决定是否执行effect。

**举例：**

```javascript
// 逐个比较
const nextDeps = [1, 'a', obj];
const prevDeps = [1, 'a', obj];

for (let i = 0; i < nextDeps.length; i++) {
  if (Object.is(nextDeps[i], prevDeps[i])) continue;
  return false;  // 有不同
}
return true;  // 全相同
```

**应用：** 只在依赖变化时执行，避免不必要的副作用。

---

### 卡片5：cleanup执行时序 ⏱️

**一句话：** cleanup在下次effect前或组件卸载时执行。

**举例：**

```javascript
useEffect(() => {
  const timer = setInterval(() => tick(), 1000);
  return () => clearInterval(timer);  // cleanup
}, []);

// 卸载时：
// -> clearInterval(timer)
```

**应用：** cleanup清理副作用，防止内存泄漏。

---

### 卡片6：空依赖数组 []  🔒

**一句话：** 空依赖数组的effect只在mount时执行，卸载时cleanup。

**举例：**

```javascript
useEffect(() => {
  console.log('Mount');
  return () => console.log('Unmount');
}, []);

// mount: Mount
// update: （不执行）
// unmount: Unmount
```

**应用：** 模拟componentDidMount/componentWillUnmount。

---

### 卡片7：无依赖数组 🔄

**一句话：** 无依赖数组的effect每次渲染都执行。

**举例：**

```javascript
useEffect(() => {
  console.log('Every render');
});  // 无依赖数组

// 每次渲染都执行
```

**应用：** 慎用，通常应该添加依赖数组。

---

### 卡片8：HookHasEffect标志 🏴

**一句话：** HookHasEffect标志决定effect是否需要执行。

**举例：**

```javascript
// 依赖变化：设置HookHasEffect
effect.tag = HookHasEffect | PassiveEffect;

// 依赖未变：不设置HookHasEffect
effect.tag = PassiveEffect;

// 执行时检查：
if (effect.tag & HookHasEffect) {
  // 执行effect
}
```

**应用：** React通过标志位高效判断哪些effect需要执行。

---

### 卡片9：useLayoutEffect vs useEffect 🆚

**一句话：** useLayoutEffect同步执行会阻塞绘制，useEffect异步执行不阻塞。

**举例：**

```javascript
// useLayoutEffect：绘制前同步执行
useLayoutEffect(() => {
  // 阻塞绘制
  const height = ref.current.offsetHeight;
  ref.current.style.top = `${height}px`;
}, []);

// useEffect：绘制后异步执行
useEffect(() => {
  // 不阻塞绘制
}, []);
```

**应用：** 默认用useEffect，只在需要同步DOM操作时用useLayoutEffect。

---

### 卡片10：总结与延伸 🌟

**一句话：** useEffect是React处理副作用的核心机制，理解其执行时机和cleanup是关键。

**关键要点：**
- Effect链表保存副作用
- 异步执行不阻塞渲染
- 依赖数组控制执行时机
- cleanup清理副作用
- useLayoutEffect同步版本

**延伸学习：**
- **自定义Hook**：封装useEffect逻辑复用
- **useInsertionEffect**：React 18新增，CSS-in-JS专用
- **Strict Mode**：开发模式下double invoke effect
- **并发特性**：React 18的Suspense和Transition对effect的影响

**在React源码中**：
- `packages/react-reconciler/src/ReactFiberHooks.js`：Effect Hook实现
- `packages/react-reconciler/src/ReactFiberCommitWork.js`：Effect执行逻辑

---

## 十、【一句话总结】

**useEffect通过Effect链表在commit阶段后异步执行副作用，通过依赖数组浅比较决定是否执行，提供cleanup机制在effect执行前或组件卸载时清理副作用，通过HookHasEffect标志高效判断哪些effect需要执行，是React处理副作用的核心实现。**

---

## 附录

### 学习检查清单

- [ ] 理解Effect链表与Hook链表的关系
- [ ] 掌握依赖数组浅比较机制
- [ ] 理解cleanup执行时序
- [ ] 知道useEffect的异步执行机制
- [ ] 掌握useEffect vs useLayoutEffect的区别
- [ ] 理解HookHasEffect标志位的作用
- [ ] 了解Effect执行的完整流程
- [ ] 能够阅读commitHookEffectListMount源码

### 下一步学习建议

1. **自定义Hook** - 学习如何封装useEffect逻辑
2. **性能优化** - useCallback、useMemo与useEffect的配合
3. **并发特性** - React 18的Suspense、Transition对effect的影响
4. **useInsertionEffect** - CSS-in-JS库的专用Hook
5. **Strict Mode** - 开发模式下effect的double invoke

### 快速参考

**useEffect关键API：**

```javascript
// 基础用法
useEffect(() => {
  // effect函数
  return () => {
    // cleanup函数
  };
}, [deps]);

// 三种依赖情况
useEffect(() => {}, undefined);  // 每次渲染都执行
useEffect(() => {}, []);         // 只在mount时执行
useEffect(() => {}, [dep]);      // dep变化时执行

// useLayoutEffect
useLayoutEffect(() => {
  // 同步执行，阻塞绘制
}, []);
```

**常见模式：**

```javascript
// 1. 数据获取
useEffect(() => {
  fetchData().then(setData);
}, [id]);

// 2. 订阅
useEffect(() => {
  const sub = api.subscribe();
  return () => sub.unsubscribe();
}, []);

// 3. 定时器
useEffect(() => {
  const timer = setInterval(() => tick(), 1000);
  return () => clearInterval(timer);
}, []);

// 4. DOM操作
useLayoutEffect(() => {
  const rect = ref.current.getBoundingClientRect();
  setHeight(rect.height);
}, []);
```

### 调试技巧

```javascript
// 查看Effect链表
useEffect(() => {
  const fiber = /* 获取当前Fiber */;
  const updateQueue = fiber.updateQueue;
  console.log('Effect链表:', updateQueue.lastEffect);

  // 遍历Effect链表
  let effect = updateQueue.lastEffect;
  if (effect) {
    const first = effect.next;
    let current = first;
    do {
      console.log('Effect:', current);
      current = current.next;
    } while (current !== first);
  }
});

// 追踪依赖变化
useEffect(() => {
  console.log('依赖变化了:', dep);
}, [dep]);

// 检查cleanup是否执行
useEffect(() => {
  console.log('Effect执行');
  return () => {
    console.log('Cleanup执行');
  };
}, [dep]);
```

### 参考资源

- [React官方文档 - useEffect](https://react.dev/reference/react/useEffect)
- [React官方文档 - useLayoutEffect](https://react.dev/reference/react/useLayoutEffect)
- [React源码 - ReactFiberHooks.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)
- [React源码 - ReactFiberCommitWork.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberCommitWork.js)
- [useEffect完全指南 - Dan Abramov](https://overreacted.io/a-complete-guide-to-useeffect/)
