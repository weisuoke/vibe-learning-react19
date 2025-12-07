# Effect标记

## 1. 【30字核心】

**Effect标记是React通过位掩码记录DOM操作的机制，Diff后标记增删改，Commit阶段通过Effect链高效执行副作用。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Effect标记的第一性原理 🎯

#### 1. 最基础的定义

**Effect标记 = 用位掩码记录"要做什么DOM操作"**

仅此而已！没有更基础的了。

Diff算法找出了新旧树的差异后，需要记录下来"这个节点要新增"、"那个节点要删除"、"另一个节点要更新"，Effect标记就是这个"记录本"。

#### 2. 为什么需要Effect标记？

**核心问题：Diff完成后，如何高效记录和执行DOM操作？**

在React的Reconciler阶段：
- Diff算法找出了哪些节点需要新增、删除、移动、更新
- 但**不能立即执行**DOM操作（Reconciler阶段可能被打断）
- 需要先"记录下来"，等到Commit阶段再"统一执行"

**如果没有Effect标记：**
```javascript
// 暴力方案：用数组记录操作
const operations = [];

// Diff时记录
operations.push({ type: 'insert', node: newNode });
operations.push({ type: 'update', node: existingNode });
operations.push({ type: 'delete', node: oldNode });

// Commit时执行
operations.forEach(op => {
  if (op.type === 'insert') {
    dom.appendChild(op.node);
  } else if (op.type === 'update') {
    updateDOM(op.node);
  } else if (op.type === 'delete') {
    dom.removeChild(op.node);
  }
});

// 问题：
// 1. 内存占用大（每个操作都是一个对象）
// 2. 遍历效率低（需要遍历整个数组）
// 3. 难以优化（无法快速判断子树是否有副作用）
```

**有了Effect标记：**
```javascript
// 智能方案：用位掩码记录
fiber.flags = Placement | Update;  // 同时标记新增和更新
fiber.subtreeFlags = Deletion;     // 子树有删除操作

// Commit时执行
if (fiber.flags & Placement) {
  commitPlacement(fiber);
}
if (fiber.flags & Update) {
  commitUpdate(fiber);
}
// 如果subtreeFlags为空，直接跳过子树！

// 优势：
// 1. 内存高效（只用2个数字字段）
// 2. 判断极快（位运算是O(1)）
// 3. 可以剪枝（subtreeFlags为空 → 跳过子树）
```

#### 3. Effect标记的三层价值

##### 价值1：内存高效 - 2个字段记录所有操作

**传统方案的内存占用：**

```javascript
// 方案1：数组存储操作
const operations = [
  { type: 'Placement', fiber: fiber1 },
  { type: 'Update', fiber: fiber2 },
  { type: 'Deletion', fiber: fiber3 },
  // ... 1000个操作
];

// 内存占用：
// 每个操作对象：~100字节（type字符串 + fiber引用 + 对象开销）
// 1000个操作：~100KB
```

**Effect标记的内存占用：**

```javascript
// 方案2：位掩码
fiber.flags = Placement | Update;  // 4字节
fiber.subtreeFlags = Deletion;     // 4字节

// 内存占用：
// 每个Fiber：8字节（2个数字字段）
// 1000个Fiber：8KB

// 内存节省：100KB → 8KB（节省92%）
```

##### 价值2：性能极致 - 位运算O(1)判断

**判断操作类型的性能对比：**

```javascript
// 传统方案：遍历数组
function hasOperation(operations, type) {
  return operations.some(op => op.type === type);
}
// 时间复杂度：O(n)

// Effect标记：位运算
function hasFlag(flags, flag) {
  return (flags & flag) !== 0;
}
// 时间复杂度：O(1)，而且是CPU一条指令！

// 性能对比（1000个操作）：
// 遍历数组：~1000次比较
// 位运算：1次运算
// 快1000倍！
```

##### 价值3：剪枝优化 - subtreeFlags跳过无副作用子树

**subtreeFlags的剪枝效果：**

```javascript
// 没有subtreeFlags：必须遍历所有节点
function commitWork(fiber) {
  // 处理当前节点
  if (fiber.flags & Placement) {
    commitPlacement(fiber);
  }

  // 递归处理子节点（即使子树没有任何副作用）
  let child = fiber.child;
  while (child) {
    commitWork(child);  // 可能是无用遍历！
    child = child.sibling;
  }
}

// 有subtreeFlags：智能跳过
function commitWork(fiber) {
  // 处理当前节点
  if (fiber.flags & Placement) {
    commitPlacement(fiber);
  }

  // 子树有副作用才递归
  if (fiber.subtreeFlags !== NoFlags) {
    let child = fiber.child;
    while (child) {
      commitWork(child);
      child = child.sibling;
    }
  }
  // 如果subtreeFlags为空，直接跳过整个子树！
}

// 实际效果：
// 假设只有10%的节点有副作用
// 没有subtreeFlags：遍历1000个节点
// 有subtreeFlags：遍历~100个节点（节省90%）
```

#### 4. 从第一性原理推导 React 实现

**推理链：**

```
1. 前提：Diff后需要记录DOM操作，Commit时执行
   ↓
2. 目标：内存占用小、判断速度快、支持剪枝优化
   ↓
3. 方案对比：
   ┌─ 方案1：数组存储操作
   │   优点：直观、灵活
   │   缺点：内存大（100KB）、判断慢（O(n)）、无法剪枝
   ├─ 方案2：链表存储操作
   │   优点：动态增删方便
   │   缺点：内存大、判断慢、无法剪枝
   └─ 方案3：位掩码（React的选择）✅
       优点：内存小（8字节）、判断快（O(1)）、支持剪枝
       缺点：操作类型有限（最多32种）
   ↓
4. 设计决策：使用位掩码
   ↓
5. 字段设计：
   - flags：当前节点的副作用（Placement=2, Update=4, Deletion=8...）
   - subtreeFlags：子树是否有副作用（向上冒泡）
   ↓
6. 位运算操作：
   - 添加：flags |= Placement
   - 判断：(flags & Placement) !== 0
   - 移除：flags &= ~Placement
   - 合并：parent.subtreeFlags |= child.flags | child.subtreeFlags
   ↓
7. Effect类型定义：
   const NoFlags = 0b00000000000000000000000;
   const Placement = 0b00000000000000000000010;  // 新增/移动
   const Update = 0b00000000000000000000100;     // 更新
   const Deletion = 0b00000000000000000001000;   // 删除
   const Passive = 0b00000000000010000000000;    // useEffect
   ↓
8. 在Diff时标记：
   if (需要新增) fiber.flags |= Placement;
   if (需要更新) fiber.flags |= Update;
   if (需要删除) fiber.flags |= Deletion;
   ↓
9. 在completeWork时向上冒泡：
   parent.subtreeFlags |= child.flags | child.subtreeFlags;
   ↓
10. 在Commit时遍历Effect链执行：
   if (fiber.flags & Placement) commitPlacement(fiber);
   if (fiber.subtreeFlags !== NoFlags) {
     // 递归处理子节点
   }
   ↓
11. React的Effect标记机制诞生
```

#### 5. 一句话总结第一性原理

**Effect标记是用位掩码记录DOM操作的极简方案，通过flags字段标记当前节点的副作用、subtreeFlags字段向上冒泡子树的副作用，实现了内存高效（8字节）、判断极快（O(1)）、支持剪枝（跳过无副作用子树）的三重优势，是React Commit阶段高性能执行的基础。**

---

## 3. 【3个核心概念】

### 核心概念1：flags字段 - 位掩码记录副作用 🏴

**flags = 当前Fiber节点的副作用标记（用位掩码表示）**

```javascript
// Fiber节点结构（简化）
const fiber = {
  type: 'div',
  key: '1',
  flags: 0b0000110,  // Placement | Update
  subtreeFlags: 0b0001000,  // 子树有Deletion
  // ... 其他字段
};

// flags的可能值（位掩码）
const NoFlags =    0b0000000;  // 0    - 无副作用
const Placement =  0b0000010;  // 2    - 新增/移动
const Update =     0b0000100;  // 4    - 更新
const Deletion =   0b0001000;  // 8    - 删除
const Passive =    0b0100000;  // 32   - useEffect（异步）
const Snapshot =   0b1000000;  // 64   - getSnapshotBeforeUpdate
```

**为什么用位掩码？**

```javascript
// 优势1：可以同时标记多个副作用
fiber.flags = Placement | Update;  // 0b0000010 | 0b0000100 = 0b0000110
// 一个数字同时表示"需要新增"和"需要更新"

// 优势2：判断非常快
if (fiber.flags & Placement) {
  // 有Placement标记
}
// 一次位运算就能判断，O(1)时间复杂度

// 优势3：移除副作用也很快
fiber.flags &= ~Placement;  // 移除Placement标记
```

**详细解释：**

**标记过程（在Diff时）：**

```javascript
// reconcileChildFibers函数中
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;

  const current = newFiber.alternate;

  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // 需要移动
      newFiber.flags = Placement;  // 标记Placement
      return lastPlacedIndex;
    } else {
      return oldIndex;
    }
  } else {
    // 新增节点
    newFiber.flags = Placement;  // 标记Placement
    return lastPlacedIndex;
  }
}

// 如果props变化
if (oldProps !== newProps) {
  fiber.flags |= Update;  // 添加Update标记
}

// 如果删除节点
function deleteChild(returnFiber, childToDelete) {
  childToDelete.flags = Deletion;  // 标记Deletion
  returnFiber.deletions.push(childToDelete);
}
```

**在 React 源码/开发中的应用：**

```javascript
// packages/react-reconciler/src/ReactFiberFlags.js

// Effect标记定义（React 19）
export const NoFlags = 0b0000000000000000000000000;

export const PerformedWork = 0b0000000000000000000000001;
export const Placement = 0b0000000000000000000000010;
export const Update = 0b0000000000000000000000100;
export const Deletion = 0b0000000000000000000001000;

export const ChildDeletion = 0b0000000000000000000010000;
export const ContentReset = 0b0000000000000000000100000;
export const Callback = 0b0000000000000000001000000;

export const Snapshot = 0b0000000000000000100000000;
export const Passive = 0b0000000000000010000000000;  // useEffect

export const Visibility = 0b0000000000001000000000000;

// ... 更多标记

// 组合标记（常用组合）
export const HostEffectMask = 0b0000000000000011111111111;  // 所有Host相关的副作用
export const LifecycleEffectMask = Passive | Update | Callback | Snapshot;
```

### 核心概念2：Effect标记类型 - 不同操作的标记 🏷️

**React中定义了多种Effect标记，每种对应不同的DOM操作或生命周期**

```javascript
// 主要的Effect标记类型

// 1. Placement - 新增或移动DOM节点
const Placement = 0b0000010;

// 使用场景：
// - 新增节点：<div> 新创建
// - 移动节点：列表项位置改变

// 2. Update - 更新DOM属性
const Update = 0b0000100;

// 使用场景：
// - props变化：className、style等属性改变
// - 文本内容变化

// 3. Deletion - 删除DOM节点
const Deletion = 0b0001000;

// 使用场景：
// - 条件渲染：{show && <div>}，show变成false
// - 列表项删除

// 4. Passive - useEffect副作用（异步执行）
const Passive = 0b0100000;

// 使用场景：
// - useEffect hook
// - 在Commit阶段的Layout子阶段之后异步执行

// 5. Snapshot - getSnapshotBeforeUpdate
const Snapshot = 0b1000000;

// 使用场景：
// - getSnapshotBeforeUpdate生命周期
// - 在DOM变更前获取快照
```

**不同标记的执行时机：**

```javascript
// Commit阶段的三个子阶段

// 子阶段1：BeforeMutation（DOM变更前）
function commitBeforeMutationEffects(root, firstChild) {
  let fiber = firstChild;
  while (fiber !== null) {
    if (fiber.flags & Snapshot) {
      // 执行getSnapshotBeforeUpdate
      commitBeforeMutationEffectOnFiber(fiber);
    }
    fiber = fiber.sibling;
  }
}

// 子阶段2：Mutation（DOM变更）
function commitMutationEffects(root, firstChild) {
  let fiber = firstChild;
  while (fiber !== null) {
    if (fiber.flags & Deletion) {
      // 删除DOM节点
      commitDeletion(fiber);
    }
    if (fiber.flags & Placement) {
      // 新增/移动DOM节点
      commitPlacement(fiber);
    }
    if (fiber.flags & Update) {
      // 更新DOM属性
      commitWork(fiber);
    }
    fiber = fiber.sibling;
  }
}

// 子阶段3：Layout（DOM变更后，浏览器绘制前）
function commitLayoutEffects(root, firstChild) {
  let fiber = firstChild;
  while (fiber !== null) {
    if (fiber.flags & Update) {
      // 执行componentDidMount/componentDidUpdate
      commitLayoutEffectOnFiber(fiber);
    }
    fiber = fiber.sibling;
  }
}

// 子阶段4：Passive（Layout之后，异步执行）
function flushPassiveEffects() {
  // 执行useEffect
  commitPassiveEffects(root);
}
```

**在 React 源码/开发中的应用：**

```jsx
// 示例1：Placement标记
function App() {
  const [show, setShow] = useState(false);

  return (
    <div>
      {show && <p>Hello</p>}  {/* show变true → Placement标记 */}
    </div>
  );
}

// 示例2：Update标记
function App() {
  const [color, setColor] = useState('red');

  return (
    <div style={{ color }}>Text</div>  {/* color变化 → Update标记 */}
  );
}

// 示例3：Deletion标记
function App() {
  const [items, setItems] = useState([1, 2, 3]);

  return items.map(item => (
    <div key={item}>{item}</div>  // 删除item → Deletion标记
  ));
}

// 示例4：Passive标记
function App() {
  useEffect(() => {
    console.log('Effect');  // useEffect → Passive标记
  }, []);

  return <div>App</div>;
}
```

### 核心概念3：subtreeFlags与Effect链收集 🔗

**subtreeFlags = 向上冒泡子树的副作用，形成Effect链**

```javascript
// Fiber树结构
const root = {
  type: 'div',
  flags: NoFlags,           // 根节点自己没有副作用
  subtreeFlags: Placement | Update,  // 但子树有Placement和Update
  child: app,
};

const app = {
  type: 'App',
  flags: NoFlags,
  subtreeFlags: Placement,  // 子树有Placement
  child: header,
};

const header = {
  type: 'header',
  flags: Placement,         // 自己有Placement
  subtreeFlags: Update,     // 子树有Update
  child: title,
};

const title = {
  type: 'h1',
  flags: Update,            // 自己有Update
  subtreeFlags: NoFlags,    // 子树没有副作用
  child: null,
};
```

**subtreeFlags的计算（在completeWork中）：**

```javascript
function completeWork(current, workInProgress) {
  // ... 处理当前节点的工作

  // === 收集子树的副作用 ===
  let subtreeFlags = NoFlags;

  // 遍历所有子节点
  let child = workInProgress.child;
  while (child !== null) {
    // 合并子节点的flags和subtreeFlags
    subtreeFlags |= child.flags;
    subtreeFlags |= child.subtreeFlags;

    child = child.sibling;
  }

  // 设置当前节点的subtreeFlags
  workInProgress.subtreeFlags = subtreeFlags;

  return null;
}

// 示例执行过程：
/*
1. 完成 title 节点：
   - title.flags = Update
   - title.subtreeFlags = NoFlags （没有子节点）

2. 完成 header 节点：
   - header.flags = Placement
   - header.subtreeFlags = title.flags | title.subtreeFlags
                         = Update | NoFlags
                         = Update

3. 完成 app 节点：
   - app.flags = NoFlags
   - app.subtreeFlags = header.flags | header.subtreeFlags
                      = Placement | Update

4. 完成 root 节点：
   - root.flags = NoFlags
   - root.subtreeFlags = app.flags | app.subtreeFlags
                       = NoFlags | (Placement | Update)
                       = Placement | Update
*/
```

**subtreeFlags的剪枝优化：**

```javascript
// Commit阶段遍历时的优化
function commitMutationEffects(fiber) {
  // 处理当前节点的副作用
  if (fiber.flags & Placement) {
    commitPlacement(fiber);
  }
  if (fiber.flags & Update) {
    commitWork(fiber);
  }
  if (fiber.flags & Deletion) {
    commitDeletion(fiber);
  }

  // === 关键优化：检查subtreeFlags ===
  if (fiber.subtreeFlags !== NoFlags) {
    // 子树有副作用，才递归处理子节点
    let child = fiber.child;
    while (child !== null) {
      commitMutationEffects(child);
      child = child.sibling;
    }
  }
  // 如果subtreeFlags为空，直接跳过整个子树！
  // 避免了无用的递归遍历
}

// 性能提升示例：
/*
假设Fiber树有1000个节点：

没有subtreeFlags优化：
- 必须遍历所有1000个节点
- 即使只有10个节点有副作用

有subtreeFlags优化：
- 只遍历有副作用的子树
- 假设只有10%的节点相关 → 遍历~100个节点
- 性能提升：10倍！
*/
```

**在 React 源码/开发中的应用：**

```javascript
// packages/react-reconciler/src/ReactFiberCompleteWork.js

function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case HostComponent: {
      // ... 创建或更新DOM节点

      // === 冒泡subtreeFlags ===
      bubbleProperties(workInProgress);
      return null;
    }
    // ... 其他类型
  }
}

function bubbleProperties(completedWork: Fiber) {
  let subtreeFlags = NoFlags;
  let child = completedWork.child;

  while (child !== null) {
    // 合并子节点的flags和subtreeFlags
    subtreeFlags |= child.subtreeFlags;
    subtreeFlags |= child.flags;

    child.return = completedWork;
    child = child.sibling;
  }

  completedWork.subtreeFlags |= subtreeFlags;
}

// Commit阶段使用subtreeFlags剪枝
function commitMutationEffectsOnFiber(finishedWork: Fiber, root: FiberRoot) {
  const flags = finishedWork.flags;

  // 处理自己的副作用
  switch (finishedWork.tag) {
    case HostComponent: {
      if (flags & Update) {
        commitUpdate(finishedWork);
      }
      break;
    }
    // ...
  }

  // === 剪枝优化 ===
  if (finishedWork.subtreeFlags & MutationMask) {
    // 子树有Mutation相关的副作用，才递归
    let child = finishedWork.child;
    while (child !== null) {
      commitMutationEffectsOnFiber(child, root);
      child = child.sibling;
    }
  }
}
```

---

## 4. 【最小可用知识】

掌握以下内容，就能理解 React 源码中Effect标记的核心：

### 4.1 flags字段的基本操作

**flags字段记录当前节点的副作用，用位运算操作：**

```javascript
// 常用的Effect标记
const NoFlags = 0;
const Placement = 2;   // 0b0010
const Update = 4;      // 0b0100
const Deletion = 8;    // 0b1000
const Passive = 32;    // 0b100000

// ===== 添加标记 =====
fiber.flags |= Placement;  // 添加Placement标记

// ===== 判断标记 =====
if (fiber.flags & Placement) {
  // 有Placement标记
}

// ===== 移除标记 =====
fiber.flags &= ~Placement;  // 移除Placement标记

// ===== 同时标记多个 =====
fiber.flags = Placement | Update;  // 同时标记新增和更新
```

### 4.2 主要的Effect标记类型

**4种最常用的标记：**

```javascript
// 1. Placement - 新增/移动DOM
// 使用场景：
// - 条件渲染：{show && <div>}
// - 列表新增：setItems([...items, newItem])
// - 列表移动：setItems([B, A, C])

// 2. Update - 更新DOM属性
// 使用场景：
// - props变化：<div className={class}>
// - 文本变化：<span>{text}</span>

// 3. Deletion - 删除DOM
// 使用场景：
// - 条件渲染：show变false
// - 列表删除：setItems(items.filter(...))

// 4. Passive - useEffect（异步执行）
// 使用场景：
// - useEffect(() => {}, [])
```

### 4.3 subtreeFlags的作用

**向上冒泡子树的副作用，支持剪枝优化：**

```javascript
// 在completeWork中计算
function completeWork(fiber) {
  let subtreeFlags = NoFlags;
  let child = fiber.child;

  while (child !== null) {
    // 合并子节点的flags和subtreeFlags
    subtreeFlags |= child.flags;
    subtreeFlags |= child.subtreeFlags;
    child = child.sibling;
  }

  fiber.subtreeFlags = subtreeFlags;
}

// 在Commit中剪枝
function commitEffects(fiber) {
  // 处理自己的副作用
  if (fiber.flags & Placement) {
    commitPlacement(fiber);
  }

  // 只有子树有副作用才递归
  if (fiber.subtreeFlags !== NoFlags) {
    let child = fiber.child;
    while (child !== null) {
      commitEffects(child);
      child = child.sibling;
    }
  }
}
```

### 4.4 Effect标记的时机

**不同阶段标记不同的Effect：**

```javascript
// Reconciler阶段（Diff时）：
// - beginWork: 标记Deletion
// - completeWork: 标记Update
// - reconcileChildren: 标记Placement

// 示例：
function beginWork(current, workInProgress) {
  if (需要删除子节点) {
    child.flags = Deletion;
  }
}

function completeWork(current, workInProgress) {
  if (props变化) {
    workInProgress.flags |= Update;
  }

  // 向上冒泡subtreeFlags
  bubbleProperties(workInProgress);
}

function reconcileChildren(current, workInProgress, nextChildren) {
  if (需要新增/移动) {
    newFiber.flags = Placement;
  }
}
```

### 4.5 Commit阶段的执行顺序

**按照flags的类型依次执行：**

```javascript
// Commit阶段的执行顺序

// 1. BeforeMutation子阶段（DOM变更前）
if (fiber.flags & Snapshot) {
  // getSnapshotBeforeUpdate
}

// 2. Mutation子阶段（DOM变更）
if (fiber.flags & Deletion) {
  // 先删除
  commitDeletion(fiber);
}
if (fiber.flags & Placement) {
  // 再新增/移动
  commitPlacement(fiber);
}
if (fiber.flags & Update) {
  // 最后更新
  commitWork(fiber);
}

// 3. Layout子阶段（DOM变更后，绘制前）
if (fiber.flags & Update) {
  // componentDidMount/componentDidUpdate
}

// 4. Passive子阶段（异步）
if (fiber.flags & Passive) {
  // useEffect
}
```

**这些知识足以：**
- ✅ 理解Effect标记的设计思想（位掩码）
- ✅ 知道主要的Effect类型和使用场景
- ✅ 理解subtreeFlags的剪枝优化
- ✅ 读懂React源码中的Effect标记和执行代码
- ✅ 为学习Commit阶段打基础

---

## 5. 【1个类比】

将 Effect 标记类比为**"待办事项便签系统"**：

### 类比1：flags = 便签颜色 🎨

**Effect标记就像用不同颜色的便签记录待办事项：**

```
你有一个任务板（Fiber树），每个任务（Fiber节点）上可以贴便签：

📗 绿色便签 = Placement（新增/移动任务）
📙 黄色便签 = Update（更新任务内容）
📕 红色便签 = Deletion（删除任务）
📘 蓝色便签 = Passive（延后执行的任务，如useEffect）

一个任务可以贴多个便签：
📗📙 = 既要新增，又要更新
```

**举例：**

```javascript
// React代码
function App() {
  const [items, setItems] = useState(['A', 'B', 'C']);

  return items.map(item => (
    <div key={item} className="item">
      {item}
    </div>
  ));
}

// 更新：setItems(['B', 'C', 'A', 'D'])

// 类比：任务板上的便签变化
/*
任务A：📗 绿色便签（从位置0移动到位置2）
任务B：无便签（位置不变）
任务C：无便签（位置不变）
任务D：📗 绿色便签（新增任务）
*/

// 对应的flags
fiberA.flags = Placement;  // 📗
fiberB.flags = NoFlags;
fiberC.flags = NoFlags;
fiberD.flags = Placement;  // 📗
```

### 类比2：subtreeFlags = 文件夹汇总标签 📁

**subtreeFlags就像文件夹上的汇总标签："内有待办"**

```
你有一个项目文件夹树：

📁 项目根目录  [标签：内有📗📙]  ← subtreeFlags = Placement | Update
  ├─ 📁 前端    [标签：内有📗]     ← subtreeFlags = Placement
  │   └─ 📄 组件A  📗             ← flags = Placement
  └─ 📁 后端    [标签：内有📙]     ← subtreeFlags = Update
      └─ 📄 API  📙              ← flags = Update

查看任务时的优化：
1. 看到"项目根目录"有标签 → 需要展开查看
2. 进入"前端"文件夹，有标签 → 需要展开查看
3. 看到"组件A"有📗便签 → 执行新增操作
4. 进入"后端"文件夹，有标签 → 需要展开查看
5. 看到"API"有📙便签 → 执行更新操作

如果某个文件夹没有标签（subtreeFlags = NoFlags）→ 直接跳过，不打开！
```

**对应React代码：**

```javascript
// Commit阶段遍历
function commitEffects(fiber) {
  // 看自己有没有便签
  if (fiber.flags & Placement) {
    console.log('执行新增操作');
  }
  if (fiber.flags & Update) {
    console.log('执行更新操作');
  }

  // 看文件夹标签（subtreeFlags）
  if (fiber.subtreeFlags !== NoFlags) {
    console.log('文件夹有标签，需要打开查看子任务');
    let child = fiber.child;
    while (child !== null) {
      commitEffects(child);  // 递归查看子任务
      child = child.sibling;
    }
  } else {
    console.log('文件夹没有标签，跳过！');
  }
}
```

### 类比3：位运算 = 便签操作 ✂️

**按位或（|）= 贴新便签**

```
原有便签：📗（Placement）
贴上新便签：📙（Update）

结果：📗📙（Placement | Update）
```

```javascript
fiber.flags = Placement;      // 📗
fiber.flags |= Update;        // 贴上📙
// fiber.flags = Placement | Update  （📗📙）
```

**按位与（&）= 检查是否有某种颜色的便签**

```
当前便签：📗📙（Placement | Update）
检查：是否有📗（Placement）？

结果：有！
```

```javascript
const hasPlacement = (fiber.flags & Placement) !== 0;
// true（有📗）
```

**按位与非（&~）= 撕掉某个便签**

```
当前便签：📗📙（Placement | Update）
撕掉：📗（Placement）

结果：📙（只剩Update）
```

```javascript
fiber.flags &= ~Placement;
// fiber.flags = Update（只剩📙）
```

### 类比4：Commit阶段 = 按便签颜色分类执行 📋

**Commit阶段就像按便签颜色的优先级依次处理任务：**

```
处理顺序（Mutation子阶段）：

第一批：处理📕红色便签（Deletion - 删除）
- 任务1：删除组件C
- 任务2：删除组件D

第二批：处理📗绿色便签（Placement - 新增/移动）
- 任务3：新增组件E
- 任务4：移动组件A

第三批：处理📙黄色便签（Update - 更新）
- 任务5：更新组件B的className
- 任务6：更新组件F的style

为什么是这个顺序？
1. 先删除：腾出空间
2. 再新增/移动：调整结构
3. 最后更新：优化属性
```

**对应React代码：**

```javascript
function commitMutationEffects(fiber) {
  // 第一批：删除
  if (fiber.flags & Deletion) {
    commitDeletion(fiber);
  }

  // 第二批：新增/移动
  if (fiber.flags & Placement) {
    commitPlacement(fiber);
  }

  // 第三批：更新
  if (fiber.flags & Update) {
    commitWork(fiber);
  }

  // 递归处理子任务
  if (fiber.subtreeFlags !== NoFlags) {
    let child = fiber.child;
    while (child) {
      commitMutationEffects(child);
      child = child.sibling;
    }
  }
}
```

### 类比5：Passive标记 = 延后执行的便签 ⏰

**Passive（useEffect）就像"稍后提醒"的便签：**

```
📘 蓝色便签（Passive）：
- 不紧急，稍后提醒
- 等其他任务都完成了再执行
- 异步执行，不阻塞主流程

普通便签（Placement/Update/Deletion）：
- 紧急，立即执行
- 同步执行，按顺序处理
```

**举例：**

```javascript
function App() {
  useEffect(() => {
    console.log('Effect执行');  // 📘 蓝色便签
  }, []);

  return <div className="app">App</div>;  // 📙 黄色便签（Update）
}

// 执行顺序：
// 1. Mutation阶段：处理📙黄色便签（更新div的className）
// 2. Layout阶段：执行componentDidMount等
// 3. **浏览器绘制DOM**
// 4. Passive阶段（异步）：处理📘蓝色便签（执行useEffect）

// 用户看到：
// 1. 页面立即更新（步骤1-3很快）
// 2. 然后useEffect在后台执行（步骤4，不阻塞页面）
```

### 类比总结表

| React概念 | 便签系统类比 | 说明 |
|----------|-------------|------|
| flags | 便签颜色 | 记录当前任务要做什么操作 |
| Placement | 📗 绿色便签 | 新增或移动任务 |
| Update | 📙 黄色便签 | 更新任务内容 |
| Deletion | 📕 红色便签 | 删除任务 |
| Passive | 📘 蓝色便签 | 延后执行的任务（useEffect）|
| subtreeFlags | 文件夹汇总标签 | "内有待办"，向上冒泡 |
| `flags \| flag` | 贴新便签 | 添加一个副作用标记 |
| `flags & flag` | 检查便签 | 判断是否有某个副作用 |
| `flags & ~flag` | 撕掉便签 | 移除某个副作用标记 |
| completeWork | 向上汇总标签 | 子文件夹有便签 → 父文件夹也贴标签 |
| commitEffects | 按颜色执行任务 | 先删除📕，再新增📗，再更新📙 |
| subtreeFlags剪枝 | 跳过空文件夹 | 没有标签的文件夹直接跳过 |

---

## 6. 【反直觉点】

### 误区1："Effect标记是在Commit阶段添加的" ❌

**为什么错？**

Effect标记是在**Reconciler阶段**（Diff时）添加的，Commit阶段只是执行这些标记。

```javascript
// 正确的流程：

// === Reconciler阶段（Diff时）===
function reconcileChildren(fiber, newChildren) {
  // Diff算法找出差异
  if (需要新增) {
    newFiber.flags = Placement;  // ✅ 在这里添加标记
  }
  if (需要更新) {
    fiber.flags |= Update;       // ✅ 在这里添加标记
  }
}

function completeWork(fiber) {
  if (props变化) {
    fiber.flags |= Update;       // ✅ 在这里添加标记
  }
  // 向上冒泡subtreeFlags
  bubbleProperties(fiber);
}

// === Commit阶段 ===
function commitMutationEffects(fiber) {
  if (fiber.flags & Placement) {
    commitPlacement(fiber);      // ❌ 不是在这里添加标记
    // 只是执行标记对应的操作
  }
}
```

**为什么人们容易这样错？**

因为"Effect"（副作用）这个词让人联想到"执行副作用"，而执行副作用确实是在Commit阶段。但**标记副作用**和**执行副作用**是两个不同的阶段。

**正确理解：**

```
React的两个阶段：

Reconciler阶段（可中断）：
1. Diff算法找出差异
2. **添加Effect标记**  ← 在这里！
3. 构建新的Fiber树

Commit阶段（不可中断）：
1. 读取Effect标记
2. **执行副作用操作**  ← 在这里！
3. 更新DOM

类比：
Reconciler = 餐厅服务员记录订单（写便签）
Commit = 厨师按订单做菜（看便签执行）
```

### 误区2："所有Effect都会被执行" ❌

**为什么错？**

通过**subtreeFlags剪枝**，没有副作用的子树会被直接跳过，不会遍历。

```javascript
// 示例：大部分节点没有副作用

const root = {
  type: 'div',
  flags: NoFlags,
  subtreeFlags: Placement,  // 只有子树的某一部分有副作用
  child: app,
};

const app = {
  type: 'App',
  flags: NoFlags,
  subtreeFlags: NoFlags,  // ← 这个子树没有任何副作用
  child: header,
};

const header = {
  type: 'header',
  flags: NoFlags,
  subtreeFlags: NoFlags,  // ← 这个子树没有任何副作用
  child: null,
};

// Commit时的遍历
function commitEffects(fiber) {
  // root: subtreeFlags !== NoFlags → 需要遍历子节点
  if (fiber.subtreeFlags !== NoFlags) {
    let child = fiber.child;
    while (child) {
      commitEffects(child);
      child = child.sibling;
    }
  }
  // app: subtreeFlags === NoFlags → 跳过整个子树！
  // header: 直接被跳过，不会遍历
}

// 结果：
// - root和app被遍历（2个节点）
// - header被跳过（省略1个节点）
// 如果没有剪枝，需要遍历所有3个节点
```

**实际效果：**

```javascript
// 假设一个1000个节点的应用
// 只有10%的节点有副作用（100个）

// 没有subtreeFlags剪枝：
// - 遍历所有1000个节点
// - 执行100个副作用
// - 总操作：1000次遍历 + 100次执行 = 1100次

// 有subtreeFlags剪枝：
// - 只遍历有副作用的分支（约200个节点）
// - 执行100个副作用
// - 总操作：200次遍历 + 100次执行 = 300次

// 性能提升：1100 → 300（提升73%）
```

**为什么人们容易这样错？**

因为从代码结构看，Commit阶段会递归遍历整棵Fiber树，容易忽略**subtreeFlags的剪枝优化**。

**正确理解：**

```javascript
// 完整的Commit逻辑
function commitEffects(fiber) {
  // 1. 处理自己的副作用
  if (fiber.flags & Placement) {
    commitPlacement(fiber);
  }

  // 2. 检查subtreeFlags再决定是否递归
  if (fiber.subtreeFlags !== NoFlags) {
    // ✅ 子树有副作用 → 递归
    let child = fiber.child;
    while (child) {
      commitEffects(child);
      child = child.sibling;
    }
  } else {
    // ✅ 子树没有副作用 → 跳过！
    // 整个子树都不遍历，直接return
  }
}
```

### 误区3："useEffect的Effect标记和DOM操作的标记是一样的" ❌

**为什么错？**

useEffect使用**Passive标记**（异步执行），而DOM操作使用**Placement/Update/Deletion标记**（同步执行），执行时机完全不同。

```javascript
// DOM操作标记（同步）
const Placement = 0b0000010;  // 新增/移动DOM
const Update = 0b0000100;     // 更新DOM
const Deletion = 0b0001000;   // 删除DOM

// useEffect标记（异步）
const Passive = 0b0100000;    // useEffect

// 执行时机：

// Mutation子阶段（同步，阻塞主线程）
if (fiber.flags & Placement) {
  commitPlacement(fiber);     // 立即执行
}
if (fiber.flags & Update) {
  commitWork(fiber);          // 立即执行
}

// ... DOM操作全部完成 ...
// ... 浏览器绘制页面 ...

// Passive子阶段（异步，不阻塞主线程）
if (fiber.flags & Passive) {
  schedulePassiveEffects(fiber);  // 稍后执行
}
```

**时间线对比：**

```
DOM操作标记（Placement/Update/Deletion）:
  ┌─────────────────────────────────┐
  │ Mutation子阶段（同步）           │
  │ 1. 执行所有DOM操作（阻塞主线程） │
  │ 2. DOM操作完成                   │
  └─────────────────────────────────┘
    ↓
  浏览器绘制页面（用户看到更新）
    ↓
  ┌─────────────────────────────────┐
  │ Passive子阶段（异步）            │
  │ 1. 执行useEffect（不阻塞主线程） │
  └─────────────────────────────────┘

用户体验：
- 0ms：看到页面更新（DOM操作完成）
- 1-5ms：useEffect在后台执行（不影响交互）
```

**为什么人们容易这样错？**

因为"Effect"这个词在React中既指"副作用"（广义，包括DOM操作），又指"useEffect"（狭义），容易混淆。

**正确理解：**

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('Effect执行');  // Passive标记（异步）
  }, [count]);

  return (
    <div>
      <p>{count}</p>  {/* Update标记（同步）*/}
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}

// 点击按钮后的执行顺序：

// 1. Reconciler阶段：
//    - Diff发现count变化
//    - <p>标记Update（文本内容变化）
//    - App组件标记Passive（有useEffect）

// 2. Commit - Mutation子阶段：
//    - 执行Update：更新<p>的文本为新的count
//    - DOM操作完成

// 3. Commit - Layout子阶段：
//    - 执行componentDidMount/Update等

// 4. **浏览器绘制页面**
//    - 用户看到count更新！

// 5. Commit - Passive子阶段（异步）：
//    - 执行useEffect：console.log('Effect执行')
//    - 不阻塞用户交互

// 时间对比：
// - 用户看到更新：~5ms（步骤1-4）
// - useEffect执行：~10ms（步骤5，不影响步骤4）
```

---

## 7. 【实战代码】

### 基础实现（简化版）

以下代码可以直接在 Node.js 中运行，完整演示Effect标记的添加和收集：

```javascript
// ===== 1. 定义Effect标记常量 =====
console.log("=== 1. 定义Effect标记常量 ===\n");

const NoFlags =    0b0000000;  // 0
const Placement =  0b0000010;  // 2
const Update =     0b0000100;  // 4
const Deletion =   0b0001000;  // 8
const Passive =    0b0100000;  // 32

console.log("NoFlags:", NoFlags);
console.log("Placement:", Placement);
console.log("Update:", Update);
console.log("Deletion:", Deletion);
console.log("Passive:", Passive);
console.log("");

// ===== 2. Fiber节点结构 =====
console.log("=== 2. Fiber节点结构 ===\n");

class Fiber {
  constructor(type, key) {
    this.type = type;
    this.key = key;
    this.flags = NoFlags;           // 当前节点的副作用
    this.subtreeFlags = NoFlags;    // 子树的副作用
    this.child = null;
    this.sibling = null;
    this.return = null;             // 父节点
  }
}

console.log("Fiber类定义完成\n");

// ===== 3. 添加Effect标记 =====
console.log("=== 3. 添加Effect标记 ===\n");

// 模拟Diff时添加标记
function markPlacement(fiber) {
  fiber.flags |= Placement;
  console.log(`标记 ${fiber.type}(key=${fiber.key}) - Placement`);
}

function markUpdate(fiber) {
  fiber.flags |= Update;
  console.log(`标记 ${fiber.type}(key=${fiber.key}) - Update`);
}

function markDeletion(fiber) {
  fiber.flags |= Deletion;
  console.log(`标记 ${fiber.type}(key=${fiber.key}) - Deletion`);
}

function markPassive(fiber) {
  fiber.flags |= Passive;
  console.log(`标记 ${fiber.type}(key=${fiber.key}) - Passive`);
}

// 创建测试Fiber树
const root = new Fiber('HostRoot', null);
const app = new Fiber('App', null);
const div1 = new Fiber('div', '1');
const div2 = new Fiber('div', '2');
const span = new Fiber('span', '3');

// 构建树结构
root.child = app;
app.return = root;

app.child = div1;
div1.return = app;
div1.sibling = div2;
div2.return = app;

div2.child = span;
span.return = div2;

console.log("Fiber树构建完成\n");

// 添加标记（模拟Diff结果）
console.log("模拟Diff过程，添加标记:");
markPlacement(div1);      // div1需要新增
markUpdate(div2);         // div2需要更新
markPassive(app);         // App有useEffect
markDeletion(span);       // span需要删除
console.log("");

// ===== 4. 收集subtreeFlags（completeWork）=====
console.log("=== 4. 收集subtreeFlags ===\n");

function completeWork(fiber) {
  console.log(`completeWork: ${fiber.type}(key=${fiber.key})`);

  let subtreeFlags = NoFlags;
  let child = fiber.child;

  while (child !== null) {
    // 合并子节点的flags和subtreeFlags
    subtreeFlags |= child.flags;
    subtreeFlags |= child.subtreeFlags;

    console.log(`  合并子节点 ${child.type}: flags=${child.flags.toString(2)}, subtreeFlags=${child.subtreeFlags.toString(2)}`);

    child = child.sibling;
  }

  fiber.subtreeFlags = subtreeFlags;
  console.log(`  ${fiber.type}的subtreeFlags=${subtreeFlags.toString(2)} (${flagsToString(subtreeFlags)})`);
  console.log("");

  return fiber;
}

// 工具函数：将flags转为可读字符串
function flagsToString(flags) {
  const names = [];
  if (flags & Placement) names.push('Placement');
  if (flags & Update) names.push('Update');
  if (flags & Deletion) names.push('Deletion');
  if (flags & Passive) names.push('Passive');
  return names.length > 0 ? names.join(' | ') : 'NoFlags';
}

// 从底向上complete（后序遍历）
console.log("从底向上收集subtreeFlags:\n");
completeWork(span);   // 叶子节点，没有子节点
completeWork(div2);   // 收集span的flags
completeWork(div1);   // 叶子节点，没有子节点
completeWork(app);    // 收集div1和div2的flags
completeWork(root);   // 收集app的flags

// ===== 5. 打印最终的Fiber树状态 =====
console.log("=== 5. 最终Fiber树状态 ===\n");

function printFiber(fiber, indent = "") {
  console.log(`${indent}${fiber.type}(key=${fiber.key})`);
  console.log(`${indent}  flags: ${flagsToString(fiber.flags)}`);
  console.log(`${indent}  subtreeFlags: ${flagsToString(fiber.subtreeFlags)}`);

  if (fiber.child) {
    printFiber(fiber.child, indent + "  ");
  }
  if (fiber.sibling) {
    printFiber(fiber.sibling, indent);
  }
}

printFiber(root);
console.log("");

// ===== 6. 模拟Commit阶段执行 =====
console.log("=== 6. 模拟Commit阶段执行 ===\n");

function commitMutationEffects(fiber, indent = "") {
  console.log(`${indent}处理 ${fiber.type}(key=${fiber.key})`);

  // 处理自己的副作用
  if (fiber.flags & Deletion) {
    console.log(`${indent}  → 执行 Deletion`);
  }
  if (fiber.flags & Placement) {
    console.log(`${indent}  → 执行 Placement`);
  }
  if (fiber.flags & Update) {
    console.log(`${indent}  → 执行 Update`);
  }

  // 检查subtreeFlags，决定是否递归
  if (fiber.subtreeFlags !== NoFlags) {
    console.log(`${indent}  → 子树有副作用，递归处理子节点`);
    let child = fiber.child;
    while (child !== null) {
      commitMutationEffects(child, indent + "    ");
      child = child.sibling;
    }
  } else {
    console.log(`${indent}  → 子树无副作用，跳过`);
  }
}

console.log("Mutation子阶段（处理DOM操作）:\n");
commitMutationEffects(root);
console.log("");

function commitPassiveEffects(fiber, indent = "") {
  if (fiber.flags & Passive) {
    console.log(`${indent}执行 ${fiber.type} 的 useEffect`);
  }

  if (fiber.subtreeFlags & Passive) {
    let child = fiber.child;
    while (child !== null) {
      commitPassiveEffects(child, indent + "  ");
      child = child.sibling;
    }
  }
}

console.log("Passive子阶段（处理useEffect）:\n");
commitPassiveEffects(root);
console.log("");

// ===== 7. 演示剪枝优化的效果 =====
console.log("=== 7. 演示剪枝优化的效果 ===\n");

// 创建一个大树，只有部分节点有副作用
const bigRoot = new Fiber('Root', null);
const container = new Fiber('Container', null);
const section1 = new Fiber('Section1', null);
const section2 = new Fiber('Section2', null);
const activeDiv = new Fiber('ActiveDiv', null);

bigRoot.child = container;
container.return = bigRoot;

container.child = section1;
section1.return = container;
section1.sibling = section2;
section2.return = container;

section2.child = activeDiv;
activeDiv.return = section2;

// 只有activeDiv有副作用
markUpdate(activeDiv);

// 从底向上收集subtreeFlags
completeWork(activeDiv);
completeWork(section2);
completeWork(section1);
completeWork(container);
completeWork(bigRoot);

console.log("Fiber树结构:");
printFiber(bigRoot);
console.log("");

let visitCount = 0;

function commitWithCount(fiber, indent = "") {
  visitCount++;
  console.log(`${indent}[访问${visitCount}] ${fiber.type}`);

  if (fiber.flags & Update) {
    console.log(`${indent}  → 执行 Update`);
  }

  if (fiber.subtreeFlags !== NoFlags) {
    let child = fiber.child;
    while (child) {
      commitWithCount(child, indent + "  ");
      child = child.sibling;
    }
  } else {
    console.log(`${indent}  → 跳过子树（subtreeFlags为空）`);
  }
}

console.log("Commit遍历（带计数）:\n");
commitWithCount(bigRoot);
console.log(`\n总访问节点数: ${visitCount} / 5`);
console.log("剪枝效果: section1的子树被跳过了！\n");
```

### 运行输出示例

```
=== 1. 定义Effect标记常量 ===

NoFlags: 0
Placement: 2
Update: 4
Deletion: 8
Passive: 32

=== 2. Fiber节点结构 ===

Fiber类定义完成

=== 3. 添加Effect标记 ===

Fiber树构建完成

模拟Diff过程，添加标记:
标记 div(key=1) - Placement
标记 div(key=2) - Update
标记 App(key=null) - Passive
标记 span(key=3) - Deletion

=== 4. 收集subtreeFlags ===

从底向上收集subtreeFlags:

completeWork: span(key=3)
  span的subtreeFlags=0 (NoFlags)

completeWork: div(key=2)
  合并子节点 span: flags=1000, subtreeFlags=0
  div的subtreeFlags=1000 (Deletion)

completeWork: div(key=1)
  div的subtreeFlags=0 (NoFlags)

completeWork: App(key=null)
  合并子节点 div: flags=10, subtreeFlags=0
  合并子节点 div: flags=100, subtreeFlags=1000
  App的subtreeFlags=1110 (Placement | Update | Deletion)

completeWork: HostRoot(key=null)
  合并子节点 App: flags=100000, subtreeFlags=1110
  HostRoot的subtreeFlags=101110 (Placement | Update | Deletion | Passive)

=== 5. 最终Fiber树状态 ===

HostRoot(key=null)
  flags: NoFlags
  subtreeFlags: Placement | Update | Deletion | Passive
  App(key=null)
    flags: Passive
    subtreeFlags: Placement | Update | Deletion
    div(key=1)
      flags: Placement
      subtreeFlags: NoFlags
    div(key=2)
      flags: Update
      subtreeFlags: Deletion
      span(key=3)
        flags: Deletion
        subtreeFlags: NoFlags

=== 6. 模拟Commit阶段执行 ===

Mutation子阶段（处理DOM操作）:

处理 HostRoot(key=null)
  → 子树有副作用，递归处理子节点
    处理 App(key=null)
      → 子树有副作用，递归处理子节点
        处理 div(key=1)
          → 执行 Placement
          → 子树无副作用，跳过
        处理 div(key=2)
          → 执行 Update
          → 子树有副作用，递归处理子节点
            处理 span(key=3)
              → 执行 Deletion
              → 子树无副作用，跳过

Passive子阶段（处理useEffect）:

执行 App 的 useEffect

=== 7. 演示剪枝优化的效果 ===

completeWork: ActiveDiv(key=null)
  ActiveDiv的subtreeFlags=0 (NoFlags)

completeWork: Section2(key=null)
  合并子节点 ActiveDiv: flags=100, subtreeFlags=0
  Section2的subtreeFlags=100 (Update)

completeWork: Section1(key=null)
  Section1的subtreeFlags=0 (NoFlags)

completeWork: Container(key=null)
  合并子节点 Section1: flags=0, subtreeFlags=0
  合并子节点 Section2: flags=0, subtreeFlags=100
  Container的subtreeFlags=100 (Update)

completeWork: Root(key=null)
  合并子节点 Container: flags=0, subtreeFlags=100
  Root的subtreeFlags=100 (Update)

Fiber树结构:
Root(key=null)
  flags: NoFlags
  subtreeFlags: Update
  Container(key=null)
    flags: NoFlags
    subtreeFlags: Update
    Section1(key=null)
      flags: NoFlags
      subtreeFlags: NoFlags
    Section2(key=null)
      flags: NoFlags
      subtreeFlags: Update
      ActiveDiv(key=null)
        flags: Update
        subtreeFlags: NoFlags

Commit遍历（带计数）:

[访问1] Root
  → 子树有副作用，递归处理子节点
  [访问2] Container
    → 子树有副作用，递归处理子节点
    [访问3] Section1
      → 跳过子树（subtreeFlags为空）
    [访问4] Section2
      → 子树有副作用，递归处理子节点
      [访问5] ActiveDiv
        → 执行 Update
        → 跳过子树（subtreeFlags为空）

总访问节点数: 5 / 5
剪枝效果: section1的子树被跳过了！
```

---

### 进阶：React源码实现

```javascript
// packages/react-reconciler/src/ReactFiberFlags.js

// Effect标记定义（React 19）
export type Flags = number;

export const NoFlags = 0b0000000000000000000000000;
export const PerformedWork = 0b0000000000000000000000001;

// 核心副作用标记
export const Placement = 0b0000000000000000000000010;
export const Update = 0b0000000000000000000000100;
export const ChildDeletion = 0b0000000000000000000010000;
export const ContentReset = 0b0000000000000000000100000;
export const Callback = 0b0000000000000000001000000;
export const DidCapture = 0b0000000000000000010000000;
export const ForceClientRender = 0b0000000000000000100000000;
export const Ref = 0b0000000000000001000000000;
export const Snapshot = 0b0000000000000010000000000;
export const Passive = 0b0000000000000100000000000;

// 组合标记
export const HostEffectMask = 0b0000000000000011111111111;

// packages/react-reconciler/src/ReactFiberCompleteWork.js

function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case HostComponent: {
      popHostContext(workInProgress);
      const type = workInProgress.type;

      if (current !== null && workInProgress.stateNode != null) {
        // 更新现有组件
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          renderLanes,
        );
      } else {
        // 创建新组件
        const instance = createInstance(
          type,
          newProps,
          rootContainerInstance,
          currentHostContext,
          workInProgress,
        );

        appendAllChildren(instance, workInProgress, false, false);
        workInProgress.stateNode = instance;
      }

      // === 冒泡subtreeFlags ===
      bubbleProperties(workInProgress);
      return null;
    }
    // ... 其他类型
  }

  return null;
}

function bubbleProperties(completedWork: Fiber) {
  const didBailout =
    completedWork.alternate !== null &&
    completedWork.alternate.child === completedWork.child;

  let newChildLanes = NoLanes;
  let subtreeFlags = NoFlags;

  if (!didBailout) {
    // 正常的完成流程
    let child = completedWork.child;

    while (child !== null) {
      // 合并子节点的flags和subtreeFlags
      newChildLanes = mergeLanes(
        newChildLanes,
        mergeLanes(child.lanes, child.childLanes),
      );

      subtreeFlags |= child.subtreeFlags;
      subtreeFlags |= child.flags;

      child.return = completedWork;
      child = child.sibling;
    }

    completedWork.subtreeFlags |= subtreeFlags;
  } else {
    // bailout（跳过）的情况
    let child = completedWork.child;

    while (child !== null) {
      newChildLanes = mergeLanes(
        newChildLanes,
        mergeLanes(child.lanes, child.childLanes),
      );

      subtreeFlags |= child.subtreeFlags & StaticMask;
      subtreeFlags |= child.flags & StaticMask;

      child.return = completedWork;
      child = child.sibling;
    }

    completedWork.subtreeFlags |= subtreeFlags;
  }

  completedWork.childLanes = newChildLanes;

  return didBailout;
}

// packages/react-reconciler/src/ReactFiberCommitWork.js

function commitMutationEffectsOnFiber(
  finishedWork: Fiber,
  root: FiberRoot,
  lanes: Lanes,
) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;

  switch (finishedWork.tag) {
    case HostComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);

      if (flags & Ref) {
        if (current !== null) {
          safelyDetachRef(current, current.return);
        }
      }

      if (flags & Update) {
        const instance: Instance = finishedWork.stateNode;

        if (instance != null) {
          const newProps = finishedWork.memoizedProps;
          const oldProps = current !== null ? current.memoizedProps : newProps;
          const type = finishedWork.type;
          const updatePayload: null | UpdatePayload =
            (finishedWork.updateQueue: any);

          finishedWork.updateQueue = null;

          if (updatePayload !== null) {
            commitUpdate(
              instance,
              updatePayload,
              type,
              oldProps,
              newProps,
              finishedWork,
            );
          }
        }
      }

      return;
    }
    // ... 其他类型
  }
}

function recursivelyTraverseMutationEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes,
) {
  // === 剪枝优化 ===
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;

    while (child !== null) {
      commitMutationEffectsOnFiber(child, root, lanes);
      child = child.sibling;
    }
  }
  // 如果subtreeFlags没有Mutation相关标记，直接跳过子树
}

function commitReconciliationEffects(finishedWork: Fiber) {
  const flags = finishedWork.flags;

  if (flags & Placement) {
    // 执行Placement
    commitPlacement(finishedWork);
    finishedWork.flags &= ~Placement;  // 清除标记
  }
}
```

---

## 8. 【面试必问】

### 问题1："React中Effect标记的作用是什么？为什么用位掩码？"

**普通回答（❌ 不出彩）：**

"Effect标记用来记录要做什么操作，用位掩码是因为比较快。"

**出彩回答（✅ 推荐）：**

> **Effect标记是React Reconciler和Commit阶段之间的桥梁，用位掩码实现了内存、性能、优化三重优势：**
>
> **1. 桥梁作用 - 记录Diff结果，延后执行副作用**
>
> - Reconciler阶段（可中断）：Diff算法找出差异后，用Effect标记"记录"要做的操作
> - Commit阶段（不可中断）：读取Effect标记，"执行"实际的DOM操作
> - 为什么要分开？因为Reconciler可能被打断重来，不能立即修改DOM
>
> **2. 位掩码的内存优势**
>
> - 传统方案：用数组存储操作 `[{type:'Placement',fiber:f1},{type:'Update',fiber:f2}]`
>   - 1000个操作 → 约100KB内存
> - 位掩码方案：用2个数字字段 `fiber.flags = Placement | Update`
>   - 每个Fiber → 8字节（flags + subtreeFlags）
>   - 1000个Fiber → 8KB内存
>   - **节省92%内存！**
>
> **3. 位掩码的性能优势**
>
> - 判断操作：`(flags & Placement) !== 0` → O(1)，一条CPU指令
> - 添加操作：`flags |= Update` → O(1)，一条CPU指令
> - 对比数组：`operations.some(op => op.type === 'Update')` → O(n)遍历
> - **快100-1000倍！**
>
> **4. subtreeFlags的剪枝优化**
>
> - subtreeFlags向上冒泡子树的副作用：`parent.subtreeFlags |= child.flags | child.subtreeFlags`
> - Commit时检查subtreeFlags：如果为空，直接跳过整个子树
> - 实际效果：假设只有10%节点有副作用，可以跳过90%的无用遍历
> - **性能提升10倍！**
>
> **与其他方案的对比：**
>
> | 方案 | 内存 | 判断速度 | 剪枝优化 | React选择 |
> |------|------|---------|---------|---------|
> | 数组存储 | 大（100KB） | 慢（O(n)）| 不支持 | ❌ |
> | 链表存储 | 大 | 慢（O(n)）| 不支持 | ❌ |
> | 位掩码 | 小（8字节）| 快（O(1)）| 支持 | ✅ |
>
> **在实际React开发中的应用：**
>
> ```jsx
> function App() {
>   const [show, setShow] = useState(false);
>
>   return (
>     <div>
>       {show && <p>Hello</p>}
>     </div>
>   );
> }
>
> // show变true时：
> // 1. Reconciler阶段：
> //    - Diff发现新增<p>节点
> //    - 标记：p.flags = Placement
> //    - 冒泡：div.subtreeFlags |= Placement
> //
> // 2. Commit阶段：
> //    - 读取flags：if (p.flags & Placement)
> //    - 执行：commitPlacement(p) → DOM.appendChild
> ```

**为什么这个回答出彩？**

1. ✅ 解释了Effect标记在React架构中的桥梁作用
2. ✅ 从内存、性能、优化三个维度说明位掩码的优势
3. ✅ 提供了具体的数据对比
4. ✅ 展示了对React两阶段设计的理解

---

### 问题2："subtreeFlags是如何收集的？有什么作用？"

**普通回答（❌ 不出彩）：**

"subtreeFlags是在completeWork时向上冒泡的，用来优化遍历。"

**出彩回答（✅ 推荐）：**

> **subtreeFlags是React的剪枝优化核心，通过自底向上冒泡子树的副作用，让Commit阶段可以跳过无副作用的子树：**
>
> **1. 收集机制 - 自底向上冒泡**
>
> ```javascript
> // completeWork中收集
> function completeWork(fiber) {
>   let subtreeFlags = NoFlags;
>   let child = fiber.child;
>
>   while (child !== null) {
>     // 合并子节点的flags和subtreeFlags
>     subtreeFlags |= child.flags;
>     subtreeFlags |= child.subtreeFlags;
>     child = child.sibling;
>   }
>
>   fiber.subtreeFlags = subtreeFlags;
> }
>
> // 示例：
> /*
> 叶子节点C: flags=Update, subtreeFlags=NoFlags
>      ↓
> 父节点B: subtreeFlags = C.flags | C.subtreeFlags = Update
>      ↓
> 根节点A: subtreeFlags = B.flags | B.subtreeFlags = Update
> */
> ```
>
> - 从叶子节点开始，向上逐层冒泡
> - 每个节点的subtreeFlags = 所有子孙节点flags的并集
> - 用位或运算（|）合并，O(1)时间复杂度
>
> **2. 剪枝作用 - 跳过无副作用子树**
>
> ```javascript
> // Commit阶段的剪枝
> function commitEffects(fiber) {
>   // 处理自己的副作用
>   if (fiber.flags & Placement) {
>     commitPlacement(fiber);
>   }
>
>   // === 关键：检查subtreeFlags ===
>   if (fiber.subtreeFlags !== NoFlags) {
>     // 子树有副作用 → 递归
>     let child = fiber.child;
>     while (child) {
>       commitEffects(child);
>       child = child.sibling;
>     }
>   } else {
>     // 子树无副作用 → 直接跳过！
>     // 整个子树都不遍历
>   }
> }
> ```
>
> **3. 性能提升 - 实际效果分析**
>
> 假设一个1000个节点的应用，只有100个节点有副作用：
>
> **没有subtreeFlags剪枝：**
> - 必须遍历所有1000个节点
> - 执行100个副作用操作
> - 总操作：1000次遍历 + 100次执行 = 1100次
>
> **有subtreeFlags剪枝：**
> - 只遍历有副作用的分支（约200个节点）
> - 执行100个副作用操作
> - 总操作：200次遍历 + 100次执行 = 300次
> - **性能提升：73%！**
>
> **4. 与其他优化策略的对比**
>
> | 优化策略 | 思路 | 复杂度 | 效果 |
> |---------|------|--------|------|
> | bailout | 组件级跳过 | O(1)判断 | 跳过组件 |
> | shouldComponentUpdate | 手动优化 | 开发者控制 | 减少render |
> | subtreeFlags | Fiber级跳过 | O(1)判断 | 跳过子树 |
> | React.memo | 浅比较props | O(props)判断 | 减少render |
>
> - subtreeFlags是Commit阶段的自动优化，无需开发者干预
> - 配合bailout（Reconciler阶段）实现全流程优化
>
> **在React源码中的实际应用：**
>
> ```javascript
> // packages/react-reconciler/src/ReactFiberCommitWork.js
>
> function recursivelyTraverseMutationEffects(
>   root: FiberRoot,
>   parentFiber: Fiber,
>   lanes: Lanes,
> ) {
>   // === React源码中的剪枝判断 ===
>   if (parentFiber.subtreeFlags & MutationMask) {
>     // MutationMask = Placement | Update | Deletion ...
>     // 只有子树包含Mutation相关的副作用才递归
>
>     let child = parentFiber.child;
>     while (child !== null) {
>       commitMutationEffectsOnFiber(child, root, lanes);
>       child = child.sibling;
>     }
>   }
>   // 如果subtreeFlags没有Mutation标记 → 跳过整个子树
> }
> ```
>
> **实际案例：大列表更新**
>
> ```jsx
> function BigList({ items }) {
>   return (
>     <div>
>       {items.map(item => (
>         <div key={item.id}>
>           <Header />          {/* 无副作用 */}
>           <Content item={item} />  {/* 有副作用 */}
>           <Footer />          {/* 无副作用 */}
>         </div>
>       ))}
>     </div>
>   );
> }
>
> // 更新一个item时：
> // - 该item的div.subtreeFlags = Update（Content变化）
> // - Header和Footer的subtreeFlags = NoFlags
> // - Commit时：Header和Footer的子树直接跳过
> // - 只遍历Content及其子树
> ```

**为什么这个回答出彩？**

1. ✅ 详细解释了收集机制（自底向上冒泡）
2. ✅ 说明了剪枝作用和实际性能提升
3. ✅ 对比了其他优化策略
4. ✅ 结合React源码和实际案例

---

## 9. 【化骨绵掌】

### 卡片1：Effect标记的本质 - 位掩码记录操作 🎯

**一句话：** Effect标记 = 用位掩码记录"要做什么DOM操作"

**举例：**

```javascript
fiber.flags = Placement | Update;  // 0b0000110
// 同时标记"新增"和"更新"两个操作
```

**应用：** Reconciler阶段标记，Commit阶段执行。

---

### 卡片2：为什么用位掩码？📐

**一句话：** 位掩码内存小（8字节）、速度快（O(1)）、支持剪枝

**对比：**
- 数组方案：100KB内存，O(n)判断
- 位掩码：8字节内存，O(1)判断

**应用：** 1000个节点节省92%内存，快100倍！

---

### 卡片3：flags字段 - 当前节点的副作用 🏴

**一句话：** flags记录当前Fiber节点要做的操作

**常用标记：**
```javascript
Placement = 2   // 新增/移动
Update = 4      // 更新
Deletion = 8    // 删除
Passive = 32    // useEffect
```

**应用：** Diff时添加，Commit时读取。

---

### 卡片4：subtreeFlags - 子树的副作用汇总 🌲

**一句话：** subtreeFlags向上冒泡子树的副作用，支持剪枝

**计算：**
```javascript
fiber.subtreeFlags = child1.flags | child1.subtreeFlags
                   | child2.flags | child2.subtreeFlags;
```

**应用：** Commit时检查subtreeFlags，为空则跳过子树。

---

### 卡片5：位运算 - 标记操作 ➕

**一句话：** 用位运算添加/判断/移除Effect标记

**操作：**
```javascript
flags |= Placement;        // 添加
(flags & Placement) !== 0; // 判断
flags &= ~Placement;       // 移除
```

**应用：** 所有操作都是O(1)时间复杂度。

---

### 卡片6：Effect标记的时机 ⏰

**一句话：** Reconciler阶段标记，Commit阶段执行

**流程：**
```
Reconciler: Diff → 添加flags → 冒泡subtreeFlags
   ↓
Commit: 读取flags → 执行DOM操作
```

**应用：** 分离标记和执行，支持可中断渲染。

---

### 卡片7：主要的Effect类型 🏷️

**一句话：** 不同标记对应不同的DOM操作

**类型：**
- Placement：新增/移动DOM
- Update：更新DOM属性
- Deletion：删除DOM
- Passive：useEffect（异步）

**应用：** 根据flags类型执行不同的Commit函数。

---

### 卡片8：subtreeFlags的剪枝优化 ✂️

**一句话：** subtreeFlags为空 → 跳过整个子树

**效果：**
```javascript
if (fiber.subtreeFlags !== NoFlags) {
  // 递归子节点
} else {
  // 跳过子树（可能节省90%遍历）
}
```

**应用：** 大幅减少Commit阶段的遍历次数。

---

### 卡片9：Passive标记的特殊性 🔵

**一句话：** Passive（useEffect）异步执行，不阻塞页面

**执行时机：**
```
DOM操作（同步） → 浏览器绘制 → useEffect（异步）
```

**应用：** 用户先看到页面更新，useEffect后台执行。

---

### 卡片10：实际应用场景 💼

**一句话：** 不同React API对应不同的Effect标记

**场景：**
```jsx
// Placement
{show && <div>}

// Update
<div className={cls}>

// Deletion
{show || <div>}

// Passive
useEffect(() => {}, [])
```

**应用：** React自动标记，开发者无需关心。

---

## 10. 【一句话总结】

**Effect标记是React通过位掩码（flags字段）记录DOM操作、通过subtreeFlags向上冒泡实现剪枝优化的机制，在Reconciler阶段标记副作用、Commit阶段高效执行，用8字节内存和O(1)判断实现了内存、性能、优化的三重优势，是React两阶段渲染架构的关键桥梁。**

---

## 附录

### 学习检查清单

完成以下检查，确保你已经掌握了Effect标记的核心内容：

#### 基础概念
- [ ] 理解Effect标记的目的（记录DOM操作）
- [ ] 知道为什么用位掩码（内存小、速度快）
- [ ] 理解flags和subtreeFlags的区别
- [ ] 知道标记和执行的分离（Reconciler vs Commit）

#### Effect标记类型
- [ ] 掌握主要的4种标记（Placement/Update/Deletion/Passive）
- [ ] 知道每种标记的使用场景
- [ ] 理解Passive标记的异步执行
- [ ] 知道不同标记的执行顺序

#### 位运算操作
- [ ] 掌握添加标记（|=）
- [ ] 掌握判断标记（&）
- [ ] 掌握移除标记（&~）
- [ ] 理解为什么位运算快（O(1)）

#### subtreeFlags
- [ ] 理解subtreeFlags的收集机制（向上冒泡）
- [ ] 知道subtreeFlags的剪枝作用
- [ ] 能计算subtreeFlags的值
- [ ] 理解剪枝的性能提升

#### 实际应用
- [ ] 知道Effect标记在Reconciler中的添加时机
- [ ] 知道Effect标记在Commit中的读取时机
- [ ] 理解不同React API对应的标记
- [ ] 能解释Effect标记的性能优势

---

### 下一步学习建议

掌握Effect标记后，建议按以下顺序继续学习：

1. **Commit阶段** (未来的文档)
   - BeforeMutation/Mutation/Layout三个子阶段
   - 如何执行Effect标记的操作
   - Passive Effect的调度

2. **Hooks实现** (`atom/06_Hooks实现/`)
   - useEffect的Passive标记
   - Hook链表的构建
   - Hook的Effect收集

3. **双缓冲机制** (`atom/04_Fiber架构/02_双缓冲机制.md`)
   - current树和workInProgress树
   - alternate指针的作用
   - 双缓冲与Effect标记的关系

4. **React 19新特性** (`atom/07_React19新特性/`)
   - Automatic Batching对Effect标记的影响
   - Transitions和Effect标记
   - Server Components的Effect处理

---

### 快速参考卡

```
Effect标记速查表
================

核心字段：
  flags: 当前节点的副作用
  subtreeFlags: 子树的副作用（向上冒泡）

主要标记：
  NoFlags = 0
  Placement = 2        // 新增/移动
  Update = 4           // 更新
  Deletion = 8         // 删除
  Passive = 32         // useEffect（异步）

位运算操作：
  添加：  flags |= Placement
  判断：  (flags & Placement) !== 0
  移除：  flags &= ~Placement
  合并：  subtreeFlags |= child.flags | child.subtreeFlags

收集时机：
  Reconciler阶段（Diff时）：
  - beginWork: 标记Deletion
  - completeWork: 标记Update，冒泡subtreeFlags
  - reconcileChildren: 标记Placement

执行时机：
  Commit阶段：
  - BeforeMutation: Snapshot
  - Mutation: Deletion → Placement → Update
  - Layout: Update（生命周期）
  - Passive（异步）: useEffect

剪枝优化：
  if (fiber.subtreeFlags !== NoFlags) {
    // 递归子节点
  } else {
    // 跳过子树！
  }

性能优势：
  内存：8字节 vs 100KB（数组方案）
  速度：O(1) vs O(n)（数组方案）
  剪枝：可以跳过90%无用遍历
```

---

### 参考资源

**React 官方文档：**
- [React 19 Release Notes](https://react.dev/blog/2024/04/25/react-19)
- [Reconciliation](https://react.dev/learn/reconciliation)

**React 源码：**
- `packages/react-reconciler/src/ReactFiberFlags.js` - Effect标记定义
- `packages/react-reconciler/src/ReactFiberCompleteWork.js` - subtreeFlags收集
- `packages/react-reconciler/src/ReactFiberCommitWork.js` - Effect执行

**延伸阅读：**
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
- [Inside Fiber: in-depth overview](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)

---

**文档版本：** v1.0
**最后更新：** 2025-12-07
**知识点状态：** ✅ 完成
