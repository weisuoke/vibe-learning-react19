# Hooks链表

## 一、【30字核心】

**Hooks链表是React通过单链表结构保存多个Hook状态的机制，通过next指针串联，依赖严格的调用顺序，是Hooks实现的基础。**

---

## 二、【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Hooks链表的第一性原理 🎯

#### 1. 最基础的定义

**Hooks链表 = 单链表数据结构 + 保存多个Hook状态**

仅此而已！没有更基础的了。

每个Hook调用（如useState、useEffect）都会在Fiber节点上创建或读取一个Hook对象，这些Hook对象通过`next`指针连接成单链表。React通过遍历这个链表来管理组件的所有Hooks状态。

#### 2. 为什么需要Hooks链表？

**核心问题：如何在函数组件中保存多个状态，并在每次渲染时准确找回对应的状态？**

函数组件每次渲染都会重新执行，局部变量会被重置：

```javascript
function Counter() {
  // 每次渲染都会重新声明，无法保存状态
  let count = 0;

  return <button onClick={() => count++}>{count}</button>;
}
```

**问题所在：**
- 函数组件没有实例，无法像类组件那样用this保存状态
- 每次渲染都是全新的函数调用，局部变量会丢失
- 可能有多个Hook调用，需要区分不同的状态

**React需要的能力：**
- 在组件的多次渲染之间保持状态
- 区分和管理多个不同的Hook状态
- 保证每次渲染时Hook的顺序一致

#### 3. Hooks链表的三层价值

##### 价值1：状态持久化

通过Fiber节点保存Hooks链表，状态在渲染之间持久存在：

```javascript
// React源码简化示意
// packages/react-reconciler/src/ReactFiberHooks.js

// Fiber节点上保存的Hooks链表
const fiber = {
  memoizedState: {  // 第一个Hook
    memoizedState: 0,  // Hook的状态值
    next: {            // 第二个Hook
      memoizedState: false,
      next: {          // 第三个Hook
        memoizedState: [],
        next: null     // 链表结束
      }
    }
  }
};
```

**关键变化：**
- 状态存储在Fiber节点上，而非函数局部变量
- 多次渲染共享同一个Fiber节点
- 通过链表结构组织多个Hook状态

##### 价值2：顺序对应

通过链表顺序而非名称来匹配Hook和状态：

```javascript
function Component() {
  // 首次渲染：创建链表
  const [count, setCount] = useState(0);     // Hook #1
  const [visible, setVisible] = useState(false);  // Hook #2
  const [items, setItems] = useState([]);    // Hook #3

  // 更新渲染：遍历链表
  // useState调用顺序必须与首次渲染一致
  // 第1次调用 -> 读取Hook #1 -> count
  // 第2次调用 -> 读取Hook #2 -> visible
  // 第3次调用 -> 读取Hook #3 -> items
}
```

**为什么用顺序而非名称？**
- 简单高效：遍历链表O(n)，无需查找
- 避免命名冲突：不需要用户提供唯一标识
- 减少开销：不需要额外的映射表

##### 价值3：mount和update区分

同一个Hook在mount（首次渲染）和update（更新渲染）时有不同的行为：

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// mount阶段：创建新Hook
function mountState(initialState) {
  const hook = mountWorkInProgressHook();  // 创建新Hook节点
  hook.memoizedState = initialState;       // 初始化状态
  // ...
  return [hook.memoizedState, dispatch];
}

// update阶段：读取现有Hook
function updateState(initialState) {
  const hook = updateWorkInProgressHook(); // 读取现有Hook节点
  // 计算新状态...
  return [hook.memoizedState, dispatch];
}
```

#### 4. 从第一性原理推导 React 实现

**推理链：**
```
1. 函数组件需要在多次渲染间保持状态
   ↓
2. 状态不能存在函数局部变量中（会丢失）
   ↓
3. 需要外部存储 → 选择Fiber节点
   ↓
4. 一个组件可能有多个Hook → 需要数据结构组织
   ↓
5. Hook调用顺序固定 → 单链表最简单（顺序遍历）
   ↓
6. mount和update需要不同处理 → 用Dispatcher切换实现
   ↓
7. 最终方案：Fiber.memoizedState指向Hooks单链表
```

#### 5. 一句话总结第一性原理

**Hooks链表通过单链表结构在Fiber节点上保存多个Hook状态，利用严格的调用顺序实现状态的准确对应，是React Hooks状态管理的基础。**

---

## 三、【3个核心概念】

### 核心概念1：Hook对象结构 📦

**每个Hook调用都会创建一个Hook对象，包含状态值、更新队列和next指针。**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// Hook对象的TypeScript类型定义
type Hook = {
  memoizedState: any,      // Hook的状态值（useState的state、useEffect的effect等）
  baseState: any,          // 基础状态（用于计算最终状态）
  baseQueue: Update<any>,  // 基础更新队列
  queue: UpdateQueue<any>, // 待处理的更新队列
  next: Hook | null,       // 指向下一个Hook的指针
};
```

**字段详解：**

1. **memoizedState**：Hook的核心数据
   - `useState`: 保存state值
   - `useEffect`: 保存effect对象（包含create、deps等）
   - `useMemo`: 保存缓存的值
   - `useRef`: 保存ref对象

2. **queue**：更新队列
   - `useState/useReducer`: 保存setState触发的更新
   - 环形链表结构，支持批量更新

3. **next**：链表指针
   - 指向下一个Hook对象
   - 最后一个Hook的next为null

**简化示例：**

```javascript
// 简化的Hook对象示意
const hook1 = {
  memoizedState: 0,        // useState(0)
  queue: { pending: null },
  next: hook2              // 指向下一个Hook
};

const hook2 = {
  memoizedState: false,    // useState(false)
  queue: { pending: null },
  next: hook3
};

const hook3 = {
  memoizedState: { current: null },  // useRef(null)
  queue: null,
  next: null               // 链表结束
};
```

**在 React 源码中的应用：**

当你在组件中调用多个Hook时，React内部会创建对应的Hook链表：

```javascript
function Component() {
  const [count, setCount] = useState(0);      // 创建hook1
  const [visible, setVisible] = useState(false);  // 创建hook2
  const ref = useRef(null);                   // 创建hook3

  // Fiber.memoizedState -> hook1 -> hook2 -> hook3 -> null
}
```

---

### 核心概念2：currentHook 和 workInProgressHook 指针 🎯

**React使用两个全局指针追踪链表遍历状态，确保mount和update时正确访问Hook。**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// 全局变量
let currentHook: Hook | null = null;              // 当前Fiber的当前Hook
let workInProgressHook: Hook | null = null;       // 工作中Fiber的当前Hook

// 当前正在渲染的Fiber节点
let currentlyRenderingFiber: Fiber | null = null;
```

**两个指针的作用：**

1. **currentHook**: 指向current树（当前屏幕显示）的Hook链表
   - 保存上一次渲染的状态
   - update阶段从这里读取旧值

2. **workInProgressHook**: 指向workInProgress树（正在构建）的Hook链表
   - 保存本次渲染的新状态
   - mount阶段在这里创建新Hook
   - update阶段在这里计算新值

**遍历机制：**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// mount阶段：创建新Hook
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 第一个Hook：挂载到Fiber.memoizedState
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 后续Hook：追加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}

// update阶段：复用现有Hook
function updateWorkInProgressHook(): Hook {
  let nextCurrentHook: Hook | null;

  if (currentHook === null) {
    // 第一个Hook：从Fiber读取
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current.memoizedState;
  } else {
    // 后续Hook：沿着链表移动
    nextCurrentHook = currentHook.next;
  }

  currentHook = nextCurrentHook;

  // 创建新的Hook对象（复用旧Hook的数据）
  const newHook: Hook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };

  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
  } else {
    workInProgressHook = workInProgressHook.next = newHook;
  }

  return workInProgressHook;
}
```

**可视化理解：**

```
mount阶段（首次渲染）：
┌─────────────────┐
│ workInProgressHook = null
│ 调用 useState()
│ ↓ mountWorkInProgressHook()
│ 创建 hook1
│ workInProgressHook = hook1
│
│ 调用 useState()
│ ↓ mountWorkInProgressHook()
│ 创建 hook2, 追加到 hook1.next
│ workInProgressHook = hook2
└─────────────────┘

update阶段（更新渲染）：
┌─────────────────┐
│ currentHook = null
│ workInProgressHook = null
│
│ 调用 useState()
│ ↓ updateWorkInProgressHook()
│ currentHook = oldFiber.memoizedState (hook1')
│ 基于 currentHook 创建新的 hook1
│ workInProgressHook = hook1
│
│ 调用 useState()
│ ↓ updateWorkInProgressHook()
│ currentHook = currentHook.next (hook2')
│ 基于 currentHook 创建新的 hook2
│ workInProgressHook = hook2
└─────────────────┘
```

**在 React 源码中的应用：**

每次组件渲染时，React会重置这两个指针，然后按Hook调用顺序遍历：

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

export function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  props: any,
  // ...
): any {
  currentlyRenderingFiber = workInProgress;

  // 重置Hooks链表
  workInProgress.memoizedState = null;

  // 重置指针
  currentHook = null;
  workInProgressHook = null;

  // 根据mount/update切换Dispatcher
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount      // mount阶段
      : HooksDispatcherOnUpdate;    // update阶段

  // 执行组件函数，触发Hook调用
  let children = Component(props);

  // 渲染完成后清理
  currentlyRenderingFiber = null;
  currentHook = null;
  workInProgressHook = null;

  return children;
}
```

---

### 核心概念3：Dispatcher切换机制 🔄

**React通过切换不同的Dispatcher对象来实现mount和update阶段Hook的不同行为。**

```javascript
// React源码：packages/react/src/ReactCurrentDispatcher.js

// 全局Dispatcher对象
const ReactCurrentDispatcher = {
  current: null,  // 当前使用的Dispatcher
};
```

**三种Dispatcher：**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// 1. mount阶段的Dispatcher
const HooksDispatcherOnMount: Dispatcher = {
  useState: mountState,
  useEffect: mountEffect,
  useRef: mountRef,
  useMemo: mountMemo,
  // ...
};

// 2. update阶段的Dispatcher
const HooksDispatcherOnUpdate: Dispatcher = {
  useState: updateState,
  useEffect: updateEffect,
  useRef: updateRef,
  useMemo: updateMemo,
  // ...
};

// 3. 错误情况的Dispatcher（在非渲染阶段调用Hook）
const ContextOnlyDispatcher: Dispatcher = {
  useState: throwInvalidHookError,
  useEffect: throwInvalidHookError,
  // ...
};
```

**切换时机：**

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

export function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  props: any,
): any {
  currentlyRenderingFiber = workInProgress;

  // 判断是mount还是update
  const isMount = current === null || current.memoizedState === null;

  // 切换Dispatcher
  ReactCurrentDispatcher.current = isMount
    ? HooksDispatcherOnMount    // 首次渲染
    : HooksDispatcherOnUpdate;  // 更新渲染

  // 执行组件函数
  let children = Component(props);

  // 渲染完成后，切换到错误Dispatcher（防止在非渲染阶段调用Hook）
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  return children;
}
```

**用户调用useState时发生了什么：**

```javascript
// React源码：packages/react/src/ReactHooks.js

export function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 获取当前Dispatcher
  const dispatcher = resolveDispatcher();

  // 调用Dispatcher的useState方法
  // mount阶段 -> mountState
  // update阶段 -> updateState
  return dispatcher.useState(initialState);
}

function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;

  if (dispatcher === null) {
    // 在非渲染阶段调用Hook，抛出错误
    throw new Error(
      'Invalid hook call. Hooks can only be called inside of the body of a function component.'
    );
  }

  return dispatcher;
}
```

**完整流程示例：**

```javascript
function Counter() {
  // 用户代码
  const [count, setCount] = useState(0);

  // React内部流程：
  // 1. useState() 调用
  //    ↓
  // 2. resolveDispatcher() 获取当前Dispatcher
  //    ↓
  // 3. dispatcher.useState(0)
  //    ↓
  // 4. mount阶段 -> mountState(0)
  //    或 update阶段 -> updateState(0)
  //    ↓
  // 5. mountWorkInProgressHook() 或 updateWorkInProgressHook()
  //    ↓
  // 6. 返回 [state, dispatch]

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**在 React 源码中的应用：**

Dispatcher机制让同一个Hook API（如useState）在不同阶段有不同实现，既保证了API的一致性，又实现了内部逻辑的灵活切换。这是React Hooks设计的精妙之处。

---

## 四、【最小可用】

掌握以下内容，就能理解 React Hooks链表的核心：

### 4.1 链表遍历规则

**核心：按调用顺序遍历，每次调用移动指针到下一个Hook。**

```javascript
// 简化实现
let currentHook = null;

function nextHook() {
  if (currentHook === null) {
    // 第一个Hook：从Fiber读取
    currentHook = fiber.memoizedState;
  } else {
    // 后续Hook：移动到下一个
    currentHook = currentHook.next;
  }
  return currentHook;
}

// 使用
function Component() {
  const hook1 = nextHook();  // 读取第1个Hook
  const hook2 = nextHook();  // 读取第2个Hook
  const hook3 = nextHook();  // 读取第3个Hook
}
```

### 4.2 mount vs update 区分

**核心：mount创建新Hook，update复用现有Hook。**

```javascript
// 简化实现
function useState(initialState) {
  if (isMount) {
    // mount：创建新Hook
    const hook = { memoizedState: initialState, next: null };
    appendHook(hook);
    return [hook.memoizedState, dispatch];
  } else {
    // update：读取现有Hook
    const hook = nextHook();
    return [hook.memoizedState, dispatch];
  }
}
```

### 4.3 为什么Hook不能放在条件语句中

**核心：条件语句会改变Hook调用顺序，导致状态错乱。**

```javascript
function Component({ showExtra }) {
  const [count, setCount] = useState(0);

  // ❌ 错误：条件调用
  if (showExtra) {
    const [extra, setExtra] = useState(0);  // 有时调用，有时不调用
  }

  const [name, setName] = useState('');

  // 问题：
  // 首次渲染 (showExtra=true):  hook1(count) -> hook2(extra) -> hook3(name)
  // 更新渲染 (showExtra=false): hook1(count) -> hook2(name) ❌ 错位！
  //                              useState期望读取hook2(extra)，实际读到hook2(name)
}
```

**正确做法：**

```javascript
function Component({ showExtra }) {
  const [count, setCount] = useState(0);
  const [extra, setExtra] = useState(0);     // ✅ 始终调用
  const [name, setName] = useState('');

  // 用条件渲染控制UI，而非Hook调用
  return (
    <>
      <div>{count}</div>
      {showExtra && <div>{extra}</div>}
      <div>{name}</div>
    </>
  );
}
```

**这些知识足以：**
- 理解Hooks为什么需要链表结构
- 明白Hooks规则背后的原理
- 避免最常见的Hooks使用错误
- 为深入学习useState、useEffect等打下基础

---

## 五、【1个类比】

### 类比1：火车车厢 🚂

**Hooks链表 = 火车车厢的单向连接**

想象一列火车，每节车厢通过挂钩单向连接：

```
火车头(Fiber) -> 车厢1 -> 车厢2 -> 车厢3 -> null
              (Hook1)  (Hook2)  (Hook3)
```

**相似点：**
- 火车头 = Fiber节点（memoizedState指向第一节车厢）
- 每节车厢 = 一个Hook对象
- 挂钩 = next指针
- 只能从头到尾单向遍历

**举例：**

```javascript
function Train() {
  // 挂上第1节车厢
  const [passengers1, setPassengers1] = useState(100);  // 车厢1：载客100人

  // 挂上第2节车厢
  const [passengers2, setPassengers2] = useState(80);   // 车厢2：载客80人

  // 挂上第3节车厢
  const [cargo, setCargo] = useState('goods');          // 车厢3：运货物

  // 火车结构：
  // Fiber -> Hook1(100) -> Hook2(80) -> Hook3('goods') -> null
}
```

**为什么不能跳过车厢？**

```javascript
function Train({ hasMiddleCarriage }) {
  const [passengers1] = useState(100);

  // ❌ 错误：有时挂车厢，有时不挂
  if (hasMiddleCarriage) {
    const [passengers2] = useState(80);
  }

  const [cargo] = useState('goods');

  // 首次渲染（hasMiddleCarriage=true）：
  // 车厢1 -> 车厢2 -> 车厢3

  // 更新渲染（hasMiddleCarriage=false）：
  // 车厢1 -> 车厢3
  // 但React还在找"车厢2"的位置读取货物，实际读到了车厢3 ❌
}
```

---

### 类比2：待办清单 📝

**Hooks链表 = 按顺序执行的待办清单**

每天早上你有个固定的待办清单，必须按顺序完成：

```
待办清单：
1. ☐ 刷牙洗脸
2. ☐ 吃早餐
3. ☐ 检查邮件
```

**相似点：**
- 清单项 = Hook调用
- 完成顺序 = Hook调用顺序
- 每个清单项有自己的状态（完成/未完成） = Hook有自己的状态值

**举例：**

```javascript
function Morning() {
  // 第1项：刷牙洗脸（完成状态）
  const [brushed, setBrushed] = useState(false);

  // 第2项：吃早餐（食物选择）
  const [breakfast, setBreakfast] = useState('面包');

  // 第3项：检查邮件（邮件数量）
  const [emails, setEmails] = useState(0);

  // 每次"执行清单"（渲染）时，按相同顺序完成
}
```

**为什么不能跳过清单项？**

```javascript
function Morning({ isWeekend }) {
  const [brushed, setBrushed] = useState(false);

  // ❌ 错误：周末跳过检查邮件
  if (!isWeekend) {
    const [emails, setEmails] = useState(0);
  }

  const [breakfast, setBreakfast] = useState('面包');

  // 工作日：刷牙 -> 邮件 -> 早餐
  // 周末：  刷牙 -> 早餐
  // React期望第2项是"邮件"，实际读到了"早餐" ❌
}
```

---

### 类比3：点名册 📋

**Hooks链表 = 老师的点名册**

老师每天按点名册顺序点名：

```
点名册：
1. 张三 - 在（状态）
2. 李四 - 在
3. 王五 - 请假
```

**相似点：**
- 点名册 = Hooks链表
- 学生顺序 = Hook调用顺序
- 出勤状态 = Hook的状态值
- 按序点名 = 按序遍历链表

**举例：**

```javascript
function Classroom() {
  // 点第1个学生
  const [student1, setStudent1] = useState('在');

  // 点第2个学生
  const [student2, setStudent2] = useState('在');

  // 点第3个学生
  const [student3, setStudent3] = useState('请假');

  // 每次点名（渲染）必须按相同顺序
}
```

**为什么不能跳着点名？**

```javascript
function Classroom({ skipAbsent }) {
  const [student1] = useState('在');

  // ❌ 错误：跳过请假的学生
  const [student2] = useState('请假');
  if (!skipAbsent && student2 === '在') {
    const [student3] = useState('在');
  }

  // 有时点3个，有时点2个，顺序混乱 ❌
}
```

---

### 类比总结表

| React概念 | 火车类比 | 待办清单类比 | 点名册类比 |
|----------|---------|------------|-----------|
| Fiber节点 | 火车头 | 清单标题 | 点名册封面 |
| Hook对象 | 车厢 | 清单项 | 学生名字 |
| next指针 | 挂钩 | 序号 | 序号 |
| memoizedState | 车厢载荷 | 完成状态 | 出勤状态 |
| 遍历链表 | 从头到尾检查车厢 | 按序完成清单 | 按序点名 |
| 顺序错误 | 车厢脱钩 | 漏做清单项 | 漏点学生 |

---

## 六、【反直觉点】

### 误区1：Hooks可以放在条件语句中 ❌

**为什么错？**

Hooks链表依赖调用顺序来匹配状态。条件语句会改变调用顺序，导致状态读取错位。

**正确解释：**

```javascript
// ❌ 错误示例
function Component({ condition }) {
  const [a, setA] = useState(1);

  if (condition) {
    const [b, setB] = useState(2);  // 有时调用，有时不调用
  }

  const [c, setC] = useState(3);
}

// 首次渲染（condition=true）：
// hook1(a=1) -> hook2(b=2) -> hook3(c=3)

// 更新渲染（condition=false）：
// hook1(a=1) -> hook2(c=3) ❌ 错位！
// useState期望hook2是b，实际读到了c
```

**为什么人们容易这样错？**

- **直觉认知**：条件语句是编程的基本结构，很自然会用
- **类组件经验**：在类组件中可以条件性地使用state（`if (condition) this.setState(...)`）
- **误解本质**：以为Hook就是普通函数调用，不知道背后有链表遍历

**正确理解：**

```javascript
// ✅ 正确：Hook始终调用，条件逻辑放在后面
function Component({ condition }) {
  const [a, setA] = useState(1);
  const [b, setB] = useState(2);  // 始终调用
  const [c, setC] = useState(3);

  // 用条件控制逻辑，而非Hook调用
  if (condition) {
    // 使用b
  }
}

// 或者：拆分成两个组件
function Component({ condition }) {
  const [a, setA] = useState(1);
  const [c, setC] = useState(3);

  return (
    <>
      {condition && <ExtraComponent />}
    </>
  );
}

function ExtraComponent() {
  const [b, setB] = useState(2);  // 独立的Hooks链表
}
```

---

### 误区2：每次渲染都会重新创建Hooks链表 ❌

**为什么错？**

只有mount阶段（首次渲染）才创建新链表，update阶段（后续渲染）是复用现有链表，只更新状态值。

**正确解释：**

```javascript
// React源码逻辑

// mount阶段：创建新链表
function mountState(initialState) {
  const hook = mountWorkInProgressHook();  // 创建新Hook对象
  hook.memoizedState = initialState;
  // ...
  return [hook.memoizedState, dispatch];
}

// update阶段：复用现有链表
function updateState() {
  const hook = updateWorkInProgressHook();  // 读取现有Hook对象
  // 计算新状态，但Hook对象是复用的
  // ...
  return [hook.memoizedState, dispatch];
}
```

**可视化：**

```
首次渲染（mount）：
  创建 hook1 -> hook2 -> hook3
  Fiber.memoizedState = hook1

第2次渲染（update）：
  复用 hook1 -> hook2 -> hook3
  只更新 memoizedState 的值

第3次渲染（update）：
  继续复用同一个链表
  只更新 memoizedState 的值
```

**为什么人们容易这样错？**

- **函数组件特性**：每次渲染都重新执行函数，局部变量都是新的
- **错误推理**：既然函数重新执行，那Hook也是"重新调用"，就以为链表也重新创建
- **忽略外部存储**：没意识到Hook状态存在Fiber上，而非函数内部

**正确理解：**

```javascript
function Counter() {
  // 虽然函数每次重新执行，但useState不是"重新创建状态"
  // 而是"从Fiber的Hooks链表中读取状态"
  const [count, setCount] = useState(0);

  // 等价于：
  // const hook = nextHookFromFiber();  // 读取现有Hook
  // const count = hook.memoizedState;
  // const setCount = hook.dispatch;
}
```

---

### 误区3：Hooks只是普通的函数调用，没有魔法 ❌

**为什么错？**

Hooks看起来像普通函数，但实际上依赖全局状态（currentlyRenderingFiber、Dispatcher）和副作用（修改Fiber链表）。

**正确解释：**

```javascript
// 用户以为的useState
function useState(initialState) {
  let state = initialState;  // 局部变量
  const setState = (newState) => { state = newState; };
  return [state, setState];
}

// 实际的useState（简化）
function useState(initialState) {
  // 1. 访问全局变量
  const fiber = currentlyRenderingFiber;
  const dispatcher = ReactCurrentDispatcher.current;

  // 2. 修改Fiber的链表结构
  const hook = fiber.memoizedState
    ? updateWorkInProgressHook()  // 读取现有
    : mountWorkInProgressHook();  // 创建新的

  // 3. 返回状态和dispatch函数
  return [hook.memoizedState, hook.dispatch];
}
```

**真实的复杂性：**

1. **全局状态依赖**：
   - `currentlyRenderingFiber`: 知道当前渲染哪个组件
   - `ReactCurrentDispatcher`: 知道当前是mount还是update
   - `currentHook`, `workInProgressHook`: 追踪链表遍历位置

2. **副作用**：
   - 修改Fiber.memoizedState
   - 创建或复用Hook对象
   - 修改链表指针

3. **Dispatcher切换**：
   - mount: `mountState`
   - update: `updateState`
   - 非渲染阶段: 抛出错误

**为什么人们容易这样错？**

- **API简洁**：`useState`看起来就是普通函数调用
- **隐藏复杂性**：React刻意隐藏了内部实现，让用户用起来简单
- **闭包误导**：以为useState返回的setState利用闭包保存状态

**正确理解：**

```javascript
// ✅ Hooks是有"魔法"的
function Component() {
  const [count, setCount] = useState(0);

  // 背后发生了：
  // 1. resolveDispatcher() -> 获取当前Dispatcher
  // 2. dispatcher.useState(0) -> 调用mountState或updateState
  // 3. mountWorkInProgressHook() 或 updateWorkInProgressHook()
  // 4. 读取/修改 Fiber.memoizedState 链表
  // 5. 返回 [state, dispatch]

  // 这些都是"魔法"，依赖React的内部状态和调度系统
}

// ❌ 不能在React外部使用
function regularFunction() {
  const [count, setCount] = useState(0);  // 报错！
  // Error: Invalid hook call. Hooks can only be called inside
  // of the body of a function component.
}
```

**关键点**：Hooks的"简洁"是精心设计的结果，背后有复杂的链表管理、Dispatcher切换、Fiber协调等机制。理解这些"魔法"才能真正掌握Hooks。

---

## 七、【实战代码】

### 基础实现（简化版）

```javascript
// ===== 1. 模拟Hooks链表的基本结构 =====
console.log("=== 场景1：创建Hooks链表 ===");

// Hook对象结构
class Hook {
  constructor(initialState) {
    this.memoizedState = initialState;  // 保存状态值
    this.next = null;                    // 指向下一个Hook
  }
}

// 模拟Fiber节点
class FiberNode {
  constructor() {
    this.memoizedState = null;  // 指向第一个Hook
    this.alternate = null;       // 指向另一棵树的对应节点
  }
}

// 全局变量
let currentlyRenderingFiber = null;
let workInProgressHook = null;
let currentHook = null;
let isMount = true;  // 是否是首次渲染

// 创建Fiber节点
const fiber = new FiberNode();
console.log("创建Fiber节点:", fiber);

// ===== 2. mount阶段：创建Hooks链表 =====
console.log("\n=== 场景2：mount阶段 - 创建Hooks链表 ===");

// mount阶段创建新Hook
function mountWorkInProgressHook() {
  const hook = new Hook(null);

  if (workInProgressHook === null) {
    // 第一个Hook：挂载到Fiber
    fiber.memoizedState = workInProgressHook = hook;
    console.log("创建第一个Hook，挂载到Fiber.memoizedState");
  } else {
    // 后续Hook：追加到链表末尾
    workInProgressHook.next = hook;
    workInProgressHook = hook;
    console.log("创建新Hook，追加到链表末尾");
  }

  return workInProgressHook;
}

// 简化的useState（mount阶段）
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = initialState;
  console.log(`  -> Hook.memoizedState = ${initialState}`);
  return [hook.memoizedState];
}

// 模拟组件首次渲染
function Component_Mount() {
  currentlyRenderingFiber = fiber;
  workInProgressHook = null;
  isMount = true;

  console.log("\n执行组件函数（mount）：");
  const [count] = mountState(0);
  const [visible] = mountState(false);
  const [name] = mountState('React');

  console.log("\n最终Hooks链表：");
  let hook = fiber.memoizedState;
  let index = 1;
  while (hook) {
    console.log(`  Hook${index}: memoizedState = ${hook.memoizedState}`);
    hook = hook.next;
    index++;
  }
}

Component_Mount();

// ===== 3. update阶段：复用Hooks链表 =====
console.log("\n\n=== 场景3：update阶段 - 复用Hooks链表 ===");

// 模拟创建新的Fiber（双缓冲）
const newFiber = new FiberNode();
newFiber.alternate = fiber;  // 指向旧Fiber

// update阶段复用现有Hook
function updateWorkInProgressHook() {
  let nextCurrentHook;

  if (currentHook === null) {
    // 第一个Hook：从旧Fiber读取
    nextCurrentHook = currentlyRenderingFiber.alternate.memoizedState;
    console.log("读取第一个Hook from old Fiber");
  } else {
    // 后续Hook：沿着链表移动
    nextCurrentHook = currentHook.next;
    console.log("读取下一个Hook from链表");
  }

  currentHook = nextCurrentHook;

  // 创建新Hook（复用旧Hook的数据）
  const newHook = new Hook(currentHook.memoizedState);

  if (workInProgressHook === null) {
    newFiber.memoizedState = workInProgressHook = newHook;
  } else {
    workInProgressHook.next = newHook;
    workInProgressHook = newHook;
  }

  return workInProgressHook;
}

// 简化的useState（update阶段）
function updateState() {
  const hook = updateWorkInProgressHook();
  console.log(`  -> 读取Hook.memoizedState = ${hook.memoizedState}`);
  return [hook.memoizedState];
}

// 模拟组件更新渲染
function Component_Update() {
  currentlyRenderingFiber = newFiber;
  currentHook = null;
  workInProgressHook = null;
  isMount = false;

  console.log("\n执行组件函数（update）：");
  const [count] = updateState();
  const [visible] = updateState();
  const [name] = updateState();

  console.log("\n新Fiber的Hooks链表：");
  let hook = newFiber.memoizedState;
  let index = 1;
  while (hook) {
    console.log(`  Hook${index}: memoizedState = ${hook.memoizedState}`);
    hook = hook.next;
    index++;
  }
}

Component_Update();

// ===== 4. 错误示例：条件调用Hook =====
console.log("\n\n=== 场景4：错误示例 - 条件调用Hook ===");

function Component_Error(condition) {
  currentlyRenderingFiber = fiber;
  currentHook = null;
  workInProgressHook = null;

  console.log(`\n执行组件（condition=${condition}）：`);

  try {
    const [a] = isMount ? mountState(1) : updateState();
    console.log(`Hook1: ${a}`);

    if (condition) {
      const [b] = isMount ? mountState(2) : updateState();
      console.log(`Hook2: ${b}`);
    }

    const [c] = isMount ? mountState(3) : updateState();
    console.log(`Hook3: ${c}`);

  } catch (e) {
    console.error("错误：", e.message);
  }
}

// 首次渲染 (condition=true)
const errorFiber1 = new FiberNode();
fiber = errorFiber1;
isMount = true;
Component_Error(true);

console.log("\nHooks链表：hook1 -> hook2 -> hook3");

// 更新渲染 (condition=false)
const errorFiber2 = new FiberNode();
errorFiber2.alternate = errorFiber1;
fiber = errorFiber2;
currentlyRenderingFiber = errorFiber2;
isMount = false;
console.log("\n更新渲染 (condition=false)：");
console.log("期望读取：hook1 -> hook3");
console.log("实际读取：hook1 -> hook2（错位！）");
```

**运行输出示例：**

```
=== 场景1：创建Hooks链表 ===
创建Fiber节点: FiberNode { memoizedState: null, alternate: null }

=== 场景2：mount阶段 - 创建Hooks链表 ===

执行组件函数（mount）：
创建第一个Hook，挂载到Fiber.memoizedState
  -> Hook.memoizedState = 0
创建新Hook，追加到链表末尾
  -> Hook.memoizedState = false
创建新Hook，追加到链表末尾
  -> Hook.memoizedState = React

最终Hooks链表：
  Hook1: memoizedState = 0
  Hook2: memoizedState = false
  Hook3: memoizedState = React

=== 场景3：update阶段 - 复用Hooks链表 ===

执行组件函数（update）：
读取第一个Hook from old Fiber
  -> 读取Hook.memoizedState = 0
读取下一个Hook from链表
  -> 读取Hook.memoizedState = false
读取下一个Hook from链表
  -> 读取Hook.memoizedState = React

新Fiber的Hooks链表：
  Hook1: memoizedState = 0
  Hook2: memoizedState = false
  Hook3: memoizedState = React

=== 场景4：错误示例 - 条件调用Hook ===

执行组件（condition=true）：
Hook1: 1
Hook2: 2
Hook3: 3

Hooks链表：hook1 -> hook2 -> hook3

更新渲染 (condition=false)：
期望读取：hook1 -> hook3
实际读取：hook1 -> hook2（错位！）
```

---

### 进阶：React源码实现

```javascript
// React源码：packages/react-reconciler/src/ReactFiberHooks.js

// ========== 1. 核心类型定义 ==========

type Hook = {
  memoizedState: any,
  baseState: any,
  baseQueue: Update<any, any> | null,
  queue: any,
  next: Hook | null,
};

type Effect = {
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
  next: Effect,
};

// ========== 2. 全局变量 ==========

// 当前正在渲染的Fiber节点
let currentlyRenderingFiber: Fiber = (null: any);

// current树的Hook指针
let currentHook: Hook | null = null;

// workInProgress树的Hook指针
let workInProgressHook: Hook | null = null;

// ========== 3. renderWithHooks：渲染入口 ==========

export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,           // 当前屏幕上的Fiber
  workInProgress: Fiber,           // 正在构建的新Fiber
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  // 设置全局变量
  renderLanes = nextRenderLanes;
  currentlyRenderingFiber = workInProgress;

  // 重置Hooks链表
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  // 根据mount/update切换Dispatcher
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount    // mount阶段
      : HooksDispatcherOnUpdate;  // update阶段

  // 执行组件函数，触发Hook调用
  let children = Component(props, secondArg);

  // 如果有re-render（如在渲染期间调用了setState）
  if (didScheduleRenderPhaseUpdateDuringThisPass) {
    let numberOfReRenders: number = 0;
    do {
      didScheduleRenderPhaseUpdateDuringThisPass = false;

      // 切换到re-render的Dispatcher
      ReactCurrentDispatcher.current = HooksDispatcherOnRerender;

      // 重新渲染
      children = Component(props, secondArg);

      numberOfReRenders += 1;

      // 防止无限循环
      if (numberOfReRenders > RE_RENDER_LIMIT) {
        throw new Error('Too many re-renders...');
      }
    } while (didScheduleRenderPhaseUpdateDuringThisPass);
  }

  // 渲染完成后清理全局状态
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  // 清理指针
  currentHook = null;
  workInProgressHook = null;
  currentlyRenderingFiber = (null: any);

  return children;
}

// ========== 4. mountWorkInProgressHook：mount阶段创建Hook ==========

function mountWorkInProgressHook(): Hook {
  // 创建新Hook对象
  const hook: Hook = {
    memoizedState: null,   // 保存状态值
    baseState: null,       // 基础状态
    baseQueue: null,       // 基础更新队列
    queue: null,           // 待处理更新队列
    next: null,            // 指向下一个Hook
  };

  if (workInProgressHook === null) {
    // 第一个Hook：挂载到Fiber.memoizedState
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 后续Hook：追加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }

  return workInProgressHook;
}

// ========== 5. updateWorkInProgressHook：update阶段复用Hook ==========

function updateWorkInProgressHook(): Hook {
  // 1. 移动currentHook指针
  let nextCurrentHook: Hook | null;
  if (currentHook === null) {
    // 第一个Hook：从current Fiber读取
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    // 后续Hook：沿着链表移动
    nextCurrentHook = currentHook.next;
  }

  // 2. 移动workInProgressHook指针
  let nextWorkInProgressHook: Hook | null;
  if (workInProgressHook === null) {
    nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    nextWorkInProgressHook = workInProgressHook.next;
  }

  if (nextWorkInProgressHook !== null) {
    // 已经有workInProgress Hook（re-render情况）
    workInProgressHook = nextWorkInProgressHook;
    nextWorkInProgressHook = workInProgressHook.next;

    currentHook = nextCurrentHook;
  } else {
    // 没有workInProgress Hook，需要基于current Hook创建

    if (nextCurrentHook === null) {
      throw new Error('Rendered more hooks than during the previous render.');
    }

    currentHook = nextCurrentHook;

    // 创建新Hook，复用current Hook的数据
    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,
      baseState: currentHook.baseState,
      baseQueue: currentHook.baseQueue,
      queue: currentHook.queue,
      next: null,
    };

    if (workInProgressHook === null) {
      // 第一个Hook
      currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
    } else {
      // 后续Hook
      workInProgressHook = workInProgressHook.next = newHook;
    }
  }

  return workInProgressHook;
}

// ========== 6. Dispatcher切换 ==========

// mount阶段的Dispatcher
const HooksDispatcherOnMount: Dispatcher = {
  readContext,
  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  // ...
};

// update阶段的Dispatcher
const HooksDispatcherOnUpdate: Dispatcher = {
  readContext,
  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  // ...
};

// 非渲染阶段的Dispatcher（抛出错误）
const ContextOnlyDispatcher: Dispatcher = {
  readContext,
  useCallback: throwInvalidHookError,
  useContext: throwInvalidHookError,
  useEffect: throwInvalidHookError,
  // ...
};

function throwInvalidHookError() {
  throw new Error(
    'Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for one of the following reasons:\n' +
    '1. You might have mismatching versions of React and the renderer (such as React DOM)\n' +
    '2. You might be breaking the Rules of Hooks\n' +
    '3. You might have more than one copy of React in the same app\n' +
    'See https://reactjs.org/link/invalid-hook-call for tips about how to debug and fix this problem.'
  );
}
```

**代码解读要点：**

1. **renderWithHooks**：每次渲染组件时的入口
   - 重置Hooks链表和指针
   - 切换Dispatcher（mount/update）
   - 执行组件函数
   - 处理re-render
   - 清理全局状态

2. **mountWorkInProgressHook**：mount阶段创建Hook
   - 创建新Hook对象
   - 挂载到Fiber或追加到链表

3. **updateWorkInProgressHook**：update阶段复用Hook
   - 从current树读取旧Hook
   - 创建新Hook，复用数据
   - 移动两个指针（currentHook和workInProgressHook）

4. **Dispatcher机制**：同一个Hook API，不同阶段不同实现
   - mount: 创建新Hook
   - update: 读取并更新Hook
   - 非渲染阶段: 抛出错误

---

## 八、【面试必问】

### 问题："为什么Hooks不能放在条件语句、循环或嵌套函数中？"

**普通回答（❌ 不出彩）：**

"因为React规定了Hooks的使用规则，必须在顶层调用。"

**出彩回答（✅ 推荐）：**

> **Hooks不能放在条件语句中，是因为React通过链表和调用顺序来管理Hooks状态，有三个核心原因：**
>
> 1. **数据结构层面**：React使用单链表保存所有Hook状态，每个Hook对象通过`next`指针连接。React不会给Hook分配ID或名称，完全依赖调用顺序来匹配状态。
>
> 2. **匹配机制层面**：首次渲染（mount）时按顺序创建链表，后续渲染（update）时按相同顺序遍历链表读取状态。如果调用顺序改变（比如条件语句跳过某个Hook），链表遍历的位置会错位，导致读取到错误的状态。
>
> 3. **设计权衡层面**：React选择"顺序匹配"而非"名称匹配"，是为了性能和简洁性。顺序匹配只需遍历链表O(n)，而名称匹配需要维护额外的映射表。这个设计牺牲了灵活性（不能条件调用），换来了性能和简洁的API。
>
> **与Vue Composition API的对比**：Vue 3的`ref`、`computed`等可以放在条件语句中，因为Vue使用响应式对象，每个响应式数据都有独立的依赖追踪，不依赖调用顺序。React选择了不同的路径。
>
> **在实际工作中的应用**：
> - 需要条件性逻辑时，将条件放在Hook内部或返回值的使用处
> - 需要条件性组件时，拆分成独立组件，每个组件有自己的Hooks链表
> - 使用ESLint插件`eslint-plugin-react-hooks`在开发阶段自动检测违规

**为什么这个回答出彩？**

1. ✅ 从数据结构、匹配机制、设计权衡三个层面深入解释
2. ✅ 对比了其他框架（Vue），展示更广的视野
3. ✅ 给出了实际的解决方案和工具
4. ✅ 展示了对React内部实现的深入理解

---

## 九、【化骨绵掌】

### 卡片1：直觉理解 - Hooks链表是什么 🎯

**一句话：** Hooks链表是React用单链表保存多个Hook状态的机制。

**举例：**

```javascript
function Component() {
  const [a] = useState(1);   // Hook1
  const [b] = useState(2);   // Hook2
  const [c] = useState(3);   // Hook3

  // Fiber.memoizedState -> Hook1 -> Hook2 -> Hook3 -> null
}
```

**应用：** 每次调用Hook（useState、useEffect等），React就在链表上创建或读取一个节点，所有Hook状态都串在这条链上。

---

### 卡片2：为什么需要链表 📐

**一句话：** 函数组件没有实例，需要外部存储保存多个状态。

**举例：**

```javascript
// ❌ 函数局部变量无法保存状态
function Counter() {
  let count = 0;  // 每次渲染都重新声明
  return <button onClick={() => count++}>{count}</button>;
}

// ✅ Hooks链表保存在Fiber上
function Counter() {
  const [count, setCount] = useState(0);  // 状态存在Fiber链表中
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**应用：** Fiber节点的`memoizedState`指向Hooks链表，状态在渲染之间持久存在。

---

### 卡片3：Hook对象的结构 🔍

**一句话：** 每个Hook对象包含状态值、更新队列和next指针。

**举例：**

```javascript
const hook = {
  memoizedState: 0,       // 状态值
  queue: { pending: null }, // 更新队列
  next: nextHook          // 指向下一个Hook
};
```

**应用：** `useState`的状态存在`memoizedState`，`useEffect`的effect对象也存在`memoizedState`，不同Hook复用相同的数据结构。

---

### 卡片4：顺序匹配原理 🔢

**一句话：** React通过调用顺序而非名称匹配Hook和状态。

**举例：**

```javascript
// 首次渲染：创建链表 hook1 -> hook2 -> hook3
// 更新渲染：遍历链表 hook1 -> hook2 -> hook3
// 第1次调用useState -> 读hook1
// 第2次调用useState -> 读hook2
// 第3次调用useState -> 读hook3
```

**应用：** 这就是为什么Hook调用顺序必须固定，任何改变顺序的操作（条件语句、循环）都会导致错位。

---

### 卡片5：mount vs update ⚙️

**一句话：** mount阶段创建新链表，update阶段复用现有链表。

**举例：**

```javascript
// mount: mountWorkInProgressHook() -> 创建新Hook对象
const hook = { memoizedState: null, next: null };

// update: updateWorkInProgressHook() -> 读取现有Hook
const hook = currentHook;  // 复用旧Hook的数据
```

**应用：** 只有首次渲染才创建链表，后续渲染都是复用，这是性能优化的关键。

---

### 卡片6：currentHook 和 workInProgressHook 🎮

**一句话：** 两个全局指针追踪链表遍历位置。

**举例：**

```javascript
// currentHook: 指向current树的Hook（旧状态）
// workInProgressHook: 指向workInProgress树的Hook（新状态）

// 每次调用Hook都会移动这两个指针
currentHook = currentHook ? currentHook.next : fiber.memoizedState;
```

**应用：** update阶段需要对比新旧状态，两个指针分别追踪新旧两棵树的Hooks链表。

---

### 卡片7：Dispatcher切换机制 🔄

**一句话：** React通过切换Dispatcher实现mount和update的不同行为。

**举例：**

```javascript
// mount阶段
ReactCurrentDispatcher.current = {
  useState: mountState,  // 创建新Hook
};

// update阶段
ReactCurrentDispatcher.current = {
  useState: updateState,  // 读取现有Hook
};
```

**应用：** 用户调用`useState()`时，实际调用的是`dispatcher.useState()`，React在渲染前切换Dispatcher。

---

### 卡片8：为什么不能条件调用 ❌

**一句话：** 条件语句会改变Hook调用顺序，导致链表遍历错位。

**举例：**

```javascript
// 首次渲染(condition=true):  hook1 -> hook2 -> hook3
// 更新渲染(condition=false): hook1 -> hook3
// React期望hook2，实际读到hook3 ❌
```

**应用：** 条件逻辑应该放在Hook内部或返回值使用处，而不是Hook调用处。

---

### 卡片9：正确的条件使用 ✅

**一句话：** Hook始终调用，条件逻辑放在后面。

**举例：**

```javascript
// ✅ 正确
const [state, setState] = useState(initialValue);
if (condition) {
  // 使用state
}

// ✅ 或者拆分组件
{condition && <ComponentWithHooks />}
```

**应用：** 需要条件性状态时，考虑拆分组件，每个组件有独立的Hooks链表。

---

### 卡片10：总结与延伸 🚀

**一句话：** Hooks链表是React Hooks实现的基础，后续所有Hook都基于此机制。

**关键要点：**
- 单链表结构，通过next指针连接
- 依赖严格的调用顺序
- mount创建，update复用
- Dispatcher切换实现不同行为

**延伸学习：**
- `useState/useReducer`: Hook.queue更新队列
- `useEffect`: Effect链表，依赖数组比较
- `useRef`: 简单的memoizedState存储
- `useMemo/useCallback`: 缓存值和依赖数组

**在React源码中**：`packages/react-reconciler/src/ReactFiberHooks.js` 是Hooks实现的核心文件，建议深入阅读。

---

## 十、【一句话总结】

**Hooks链表是React通过单链表结构在Fiber节点上保存多个Hook状态的机制，依赖严格的调用顺序实现状态匹配，通过Dispatcher切换区分mount和update行为，是React Hooks状态管理的核心基础。**

---

## 附录

### 学习检查清单

- [ ] 理解Hook对象的结构（memoizedState, queue, next）
- [ ] 掌握currentHook和workInProgressHook的作用
- [ ] 理解mount和update阶段的区别
- [ ] 知道Dispatcher切换机制
- [ ] 明白为什么Hook不能条件调用
- [ ] 能够解释链表遍历原理
- [ ] 了解renderWithHooks的完整流程
- [ ] 能够阅读ReactFiberHooks.js源码

### 下一步学习建议

1. **useState/useReducer实现** - 学习状态Hook如何利用Hooks链表保存和更新状态
2. **useEffect执行机制** - 学习副作用Hook如何构建Effect链表和执行时序
3. **Fiber双缓冲机制** - 理解current和workInProgress树的关系
4. **调度和优先级** - 学习React如何调度Hooks的更新

### 快速参考

**Hooks链表关键API：**

```javascript
// mount阶段
mountWorkInProgressHook()  // 创建新Hook
mountState()               // 创建状态Hook
mountEffect()              // 创建副作用Hook

// update阶段
updateWorkInProgressHook() // 读取现有Hook
updateState()              // 更新状态Hook
updateEffect()             // 更新副作用Hook

// 渲染入口
renderWithHooks()          // 组件渲染时调用
```

**Hooks规则：**

1. 只在函数组件顶层调用Hook
2. 不在条件语句、循环、嵌套函数中调用Hook
3. 只在React函数组件或自定义Hook中调用Hook

**调试技巧：**

```javascript
// 查看Fiber的Hooks链表
function Component() {
  const fiber = ReactInternals.currentDispatcherRef.current;
  console.log(fiber.memoizedState);  // 第一个Hook

  let hook = fiber.memoizedState;
  while (hook) {
    console.log('Hook:', hook);
    hook = hook.next;
  }
}
```

### 参考资源

- [React官方文档 - Hooks规则](https://react.dev/reference/rules/rules-of-hooks)
- [React源码 - ReactFiberHooks.js](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)
- [React Hooks RFC](https://github.com/reactjs/rfcs/blob/main/text/0068-react-hooks.md)
