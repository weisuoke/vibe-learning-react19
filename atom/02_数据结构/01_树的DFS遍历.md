# 树的DFS遍历

## 1. 【30字核心】

**DFS（深度优先遍历）是沿着树的深度方向优先访问节点的算法，通过递归或栈实现，是 React Fiber 树遍历的核心机制。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### 树的DFS遍历的第一性原理 🎯

#### 1. 最基础的定义

**DFS = 沿着一条路径走到底，然后回溯继续**

仅此而已！没有更基础的了。

#### 2. 为什么需要树的DFS遍历？

**核心问题：如何有序地访问树形结构的所有节点？**

树形结构天然具有层次关系，但计算机需要线性化的访问顺序。有两种思路：
- **广度优先（BFS）**：层层访问，先访问同层节点
- **深度优先（DFS）**：路径优先，先沿着一条路径走到底

DFS 的优势在于：
1. 使用递归天然契合树的结构
2. 内存占用小（只需保存当前路径）
3. 适合需要"先处理子树"的场景

#### 3. DFS 的三层价值

##### 价值1：递归结构天然匹配

树是递归定义的：一棵树 = 根节点 + 若干子树

DFS 也是递归定义的：访问一棵树 = 访问根节点 + 递归访问子树

这种天然匹配让代码简洁优雅。

**示例：**
```javascript
function dfs(node) {
  if (!node) return;

  // 处理当前节点
  process(node);

  // 递归处理子树
  node.children.forEach(child => dfs(child));
}
```

##### 价值2：内存占用优势

DFS 只需维护当前访问路径的栈空间，空间复杂度 O(h)（h 是树高）

相比 BFS 需要存储整层节点（空间复杂度 O(w)，w 是最大宽度），在高瘦的树上更高效。

##### 价值3：符合某些业务逻辑

很多实际场景需要"先处理子节点"：
- 计算文件夹大小（先统计子文件夹）
- 删除节点（先删除子节点）
- React Fiber 的 completeWork（先完成子 Fiber）

#### 4. 从第一性原理推导 React 实现

**推理链：**
```
1. React 需要渲染虚拟 DOM 树（Fiber 树）
   ↓
2. 渲染需要遍历整棵树，处理每个 Fiber 节点
   ↓
3. 处理顺序很重要：需要先创建子组件才能确定父组件最终状态
   ↓
4. DFS 后序遍历（Post-order）正好符合这个需求
   ↓
5. React 使用 beginWork（向下）+ completeWork（向上）实现 DFS
   ↓
6. 为了支持可中断渲染，使用迭代版本的 DFS（循环 + 指针移动）
   ↓
7. workLoopSync / workLoopConcurrent 就是 Fiber 树的 DFS 遍历
```

#### 5. 一句话总结第一性原理

**DFS 是沿着树的深度方向优先访问节点的算法，通过"深入-回溯"的递归模式，天然匹配树的递归结构，是 React Fiber 树遍历的基础。**

---

## 3. 【3个核心概念】

### 核心概念1：三种遍历顺序 🌳

**根据访问根节点的时机不同，DFS 分为前序、中序、后序三种遍历方式。**

```javascript
// 树节点定义
class TreeNode {
  constructor(val, left = null, right = null) {
    this.val = val;
    this.left = left;
    this.right = right;
  }
}

// 前序遍历：根 -> 左 -> 右
function preorder(node, result = []) {
  if (!node) return result;

  result.push(node.val);      // 先访问根
  preorder(node.left, result);  // 再访问左子树
  preorder(node.right, result); // 最后访问右子树

  return result;
}

// 中序遍历：左 -> 根 -> 右
function inorder(node, result = []) {
  if (!node) return result;

  inorder(node.left, result);   // 先访问左子树
  result.push(node.val);       // 再访问根
  inorder(node.right, result);  // 最后访问右子树

  return result;
}

// 后序遍历：左 -> 右 -> 根
function postorder(node, result = []) {
  if (!node) return result;

  postorder(node.left, result);  // 先访问左子树
  postorder(node.right, result); // 再访问右子树
  result.push(node.val);        // 最后访问根

  return result;
}
```

**详细解释：**

三种遍历顺序的区别在于**何时处理根节点**：
- **前序（Pre-order）**：先处理根，再处理子树（自上而下）
- **中序（In-order）**：先处理左子树，再处理根，最后处理右子树（对二叉搜索树可获得有序序列）
- **后序（Post-order）**：先处理子树，最后处理根（自下而上）

**在 React 源码中的应用：**

React Fiber 的遍历过程本质是**前序+后序的组合**：
- `beginWork`：前序遍历（向下处理节点，创建子 Fiber）
- `completeWork`：后序遍历（向上完成节点，收集副作用）

```javascript
// React 源码简化（packages/react-reconciler/src/ReactFiberWorkLoop.js）
function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;

  // 前序阶段：beginWork（处理当前 Fiber）
  let next = beginWork(current, unitOfWork, renderLanes);

  if (next === null) {
    // 没有子节点，进入后序阶段
    completeUnitOfWork(unitOfWork);
  } else {
    // 有子节点，继续向下遍历
    workInProgress = next;
  }
}

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;

  do {
    // 后序阶段：completeWork（完成当前 Fiber）
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    completeWork(current, completedWork, renderLanes);

    // 检查是否有兄弟节点
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // 有兄弟，继续处理兄弟
      workInProgress = siblingFiber;
      return;
    }

    // 没有兄弟，回到父节点继续完成
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);
}
```

---

### 核心概念2：递归 vs 迭代实现 🔄

**DFS 可以用递归或迭代（栈）实现，递归简洁但可能栈溢出，迭代可控但代码复杂。**

```javascript
// 递归实现（简洁但有栈溢出风险）
function dfsRecursive(node) {
  if (!node) return;

  console.log(node.val);           // 处理当前节点

  if (node.left) dfsRecursive(node.left);   // 递归左子树
  if (node.right) dfsRecursive(node.right); // 递归右子树
}

// 迭代实现（使用显式栈，可控）
function dfsIterative(root) {
  if (!root) return;

  const stack = [root];  // 手动维护栈

  while (stack.length > 0) {
    const node = stack.pop();  // 弹出栈顶节点
    console.log(node.val);     // 处理节点

    // 注意：先压右子树，再压左子树（因为栈是后进先出）
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }
}
```

**详细解释：**

**递归实现：**
- 优点：代码简洁，逻辑清晰
- 缺点：深度过大时可能栈溢出（JS 调用栈限制约 10000-50000）

**迭代实现：**
- 优点：完全可控，可以随时中断/恢复
- 缺点：需要手动管理栈，代码稍复杂

**在 React 源码中的应用：**

React Fiber 使用**迭代版本的 DFS**，原因有二：
1. **可中断渲染**：需要随时暂停/恢复遍历过程
2. **避免栈溢出**：组件树可能非常深

```javascript
// React 源码简化（workLoop 就是迭代版 DFS）
function workLoopSync() {
  // workInProgress 是当前正在处理的 Fiber（相当于栈顶）
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function workLoopConcurrent() {
  // 可中断版本：检查是否需要让出控制权
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

---

### 核心概念3：Fiber 的三指针遍历 🔗

**Fiber 树通过 child/sibling/return 三指针实现树的链表化，支持迭代版 DFS 遍历。**

```javascript
// Fiber 节点结构（简化）
class FiberNode {
  constructor(tag, key) {
    this.tag = tag;
    this.key = key;

    // 三指针构成树结构
    this.return = null;   // 指向父 Fiber
    this.child = null;    // 指向第一个子 Fiber
    this.sibling = null;  // 指向下一个兄弟 Fiber
  }
}

// Fiber 树的 DFS 遍历
function traverseFiber(fiber) {
  let node = fiber;

  while (true) {
    // 处理当前节点（beginWork）
    console.log('beginWork:', node.tag);

    // 1. 如果有子节点，向下遍历
    if (node.child) {
      node = node.child;
      continue;
    }

    // 2. 没有子节点，完成当前节点（completeWork）
    console.log('completeWork:', node.tag);

    // 3. 回到起点，遍历结束
    if (node === fiber) {
      return;
    }

    // 4. 如果有兄弟节点，处理兄弟
    while (!node.sibling) {
      // 没有兄弟，向上回溯
      if (!node.return || node.return === fiber) {
        return;
      }

      node = node.return;
      console.log('completeWork:', node.tag);
    }

    // 5. 处理兄弟节点
    node = node.sibling;
  }
}
```

**详细解释：**

传统树节点通常只有 `children` 数组，要迭代遍历需要额外的栈。

Fiber 节点使用三指针将树"链表化"：
- `child`：第一个子节点
- `sibling`：下一个兄弟节点
- `return`：父节点

这样可以通过**指针移动**实现遍历，无需额外栈：

遍历策略（DFS 顺序）：
1. 有子节点 → 向下走（child）
2. 无子节点 → 完成当前节点
3. 有兄弟节点 → 处理兄弟（sibling）
4. 无兄弟节点 → 向上回溯（return）

**在 React 源码中的应用：**

这正是 `performUnitOfWork` 和 `completeUnitOfWork` 的实现原理：

```javascript
// React 源码（packages/react-reconciler/src/ReactFiberWorkLoop.js）
function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;
  let next = beginWork(current, unitOfWork, renderLanes);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // 没有子节点，进入完成阶段
    completeUnitOfWork(unitOfWork);
  } else {
    // 有子节点，继续向下
    workInProgress = next;
  }
}

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;

  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // 完成当前 Fiber
    let next = completeWork(current, completedWork, renderLanes);

    if (next !== null) {
      workInProgress = next;
      return;
    }

    // 尝试处理兄弟节点
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      workInProgress = siblingFiber;
      return;
    }

    // 没有兄弟，向上回溯
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);

  // 完成了根节点，工作结束
  workInProgressRootExitStatus = RootCompleted;
}
```

---

## 4. 【最小可用】

掌握以下内容，就能理解 React 源码核心：

### 4.1 DFS 的递归实现模板

```javascript
function dfs(node) {
  // 终止条件
  if (!node) return;

  // 前序位置：进入节点时处理
  console.log('进入节点:', node.val);

  // 递归处理子节点
  dfs(node.left);
  dfs(node.right);

  // 后序位置：离开节点时处理
  console.log('离开节点:', node.val);
}
```

**应用：** 理解 React 的 beginWork（前序）和 completeWork（后序）

### 4.2 DFS 的迭代实现模板

```javascript
function dfsIterative(root) {
  if (!root) return;

  const stack = [root];

  while (stack.length > 0) {
    const node = stack.pop();

    // 处理节点
    console.log(node.val);

    // 先压右子树，再压左子树（保证左子树先被处理）
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }
}
```

**应用：** 理解 React workLoop 的循环结构

### 4.3 Fiber 三指针遍历

```javascript
// 向下：child
if (node.child) {
  node = node.child;
}
// 横向：sibling
else if (node.sibling) {
  node = node.sibling;
}
// 向上：return
else {
  node = node.return;
}
```

**应用：** 理解 React Fiber 树的遍历逻辑

### 4.4 前序 vs 后序的选择

- **前序**：需要先处理父节点再处理子节点（如创建 DOM）
- **后序**：需要先处理子节点再处理父节点（如计算树高、删除节点）

**应用：** 理解为什么 React 需要 beginWork + completeWork 两个阶段

### 4.5 递归深度限制

JavaScript 调用栈深度有限（约 10000-50000），超过会栈溢出。

**解决方案：**
1. 使用迭代代替递归（React 的选择）
2. 限制树的深度
3. 使用尾递归优化（部分引擎支持）

**应用：** 理解为什么 React 必须使用迭代版 DFS

---

**这些知识足以：**
- ✅ 理解 React Fiber 树的遍历机制
- ✅ 看懂 workLoop、performUnitOfWork、completeUnitOfWork 源码
- ✅ 明白为什么 React 使用迭代而非递归
- ✅ 为后续学习 Reconciler 协调过程打基础

---

## 5. 【1个类比】

### 类比1：DFS = 走迷宫策略 🚶‍♂️

**在迷宫中，DFS 的策略是"一条路走到黑，走不通就回头"。**

**举例：**

想象你在一个迷宫中找出口：
- 遇到岔路口，选择一条路一直走
- 走到死胡同，退回到上一个岔路口
- 选择另一条路继续
- 重复直到找到出口或走遍所有路径

这正是 DFS 的核心思想！

```javascript
// 走迷宫 = DFS 遍历树
function exploreMaze(position) {
  if (position.isExit) {
    console.log('找到出口！');
    return true;
  }

  if (position.isWall || position.visited) {
    return false;  // 走不通，回退
  }

  // 标记已访问
  position.visited = true;
  console.log('探索位置:', position.x, position.y);

  // 尝试四个方向（上下左右）
  const directions = [
    { x: 0, y: 1 },   // 上
    { x: 0, y: -1 },  // 下
    { x: -1, y: 0 },  // 左
    { x: 1, y: 0 }    // 右
  ];

  for (const dir of directions) {
    const nextPos = {
      x: position.x + dir.x,
      y: position.y + dir.y
    };

    if (exploreMaze(nextPos)) {
      return true;  // 找到出口
    }
  }

  return false;  // 四个方向都不通，回退
}
```

**对应关系：**
- 岔路口 = 树节点
- 选择一条路 = 递归访问子节点
- 走到死胡同 = 到达叶子节点
- 回退 = 回溯到父节点

---

### 类比2：DFS = 文件夹深度搜索 📁

**在文件系统中查找文件，DFS 会先深入子文件夹，再搜索同级文件夹。**

**举例：**

```
项目根目录/
├── src/
│   ├── components/
│   │   ├── Button.js  ← 第2个访问
│   │   └── Input.js   ← 第3个访问
│   └── utils/         ← 第4个访问
│       └── helper.js  ← 第5个访问
└── public/            ← 第6个访问
    └── index.html     ← 第7个访问
```

**DFS 搜索顺序：**
1. 项目根目录/
2. 进入 src/
3. 进入 components/
4. 访问 Button.js（叶子节点）
5. 访问 Input.js（叶子节点）
6. 回到 src/，进入 utils/
7. 访问 helper.js（叶子节点）
8. 回到 根目录/，进入 public/
9. 访问 index.html（叶子节点）

```javascript
// 文件系统遍历 = DFS
function traverseDirectory(dir, depth = 0) {
  const indent = '  '.repeat(depth);
  console.log(`${indent}📁 ${dir.name}`);

  // 先处理所有子文件夹（深度优先）
  for (const subDir of dir.subdirectories) {
    traverseDirectory(subDir, depth + 1);
  }

  // 再处理文件
  for (const file of dir.files) {
    console.log(`${indent}  📄 ${file.name}`);
  }
}
```

---

### 类比3：DFS = 家谱树查找 👨‍👩‍👧‍👦

**在家谱中查找某人，DFS 会先查完一个家庭分支的所有后代，再查另一个分支。**

**举例：**

```
爷爷
├── 父亲
│   ├── 我
│   │   ├── 儿子
│   │   └── 女儿
│   └── 弟弟
└── 叔叔
    └── 堂弟
```

**DFS 访问顺序：**
1. 爷爷
2. 父亲
3. 我
4. 儿子（叶子）
5. 女儿（叶子）
6. 回到"我"，再回到"父亲"
7. 弟弟（叶子）
8. 回到"爷爷"
9. 叔叔
10. 堂弟（叶子）

```javascript
// 家谱遍历 = DFS
function traverseFamily(person) {
  console.log('访问:', person.name);

  // 递归访问所有子女
  for (const child of person.children) {
    traverseFamily(child);
  }

  console.log('结束访问:', person.name);
}
```

**对应到 React Fiber：**
- 家谱成员 = Fiber 节点
- 子女关系 = child 指针
- 兄弟关系 = sibling 指针
- 父母关系 = return 指针

---

### 类比总结表

| DFS 概念 | 走迷宫 | 文件系统 | 家谱树 | React Fiber |
|---------|--------|----------|--------|-------------|
| 节点 | 岔路口 | 文件夹 | 家庭成员 | Fiber 节点 |
| 边（关系） | 通道 | 包含关系 | 血缘关系 | child/sibling/return |
| 访问节点 | 探索位置 | 列出文件夹内容 | 记录成员信息 | beginWork |
| 离开节点 | 退回岔路口 | 返回上级文件夹 | 完成分支统计 | completeWork |
| 递归深入 | 选择一条路走到底 | 进入子文件夹 | 查看子女 | 处理 child |
| 回溯 | 走不通就回头 | 返回上级 | 回到父母辈 | 通过 return 向上 |

---

## 6. 【反直觉点】

### 误区1：DFS 就是前序遍历 ❌

**为什么错？**

DFS 是一种遍历策略（深度优先），前序/中序/后序是访问顺序的细分。

**关键区别：**
- **DFS**：沿着深度方向优先访问（vs BFS 沿着广度方向）
- **前序/中序/后序**：在 DFS 基础上，根节点的访问时机不同

**为什么人们容易这样错？**

很多教程把前序遍历直接叫"DFS"，因为前序遍历是最常见的 DFS 实现。但实际上后序遍历、中序遍历也都是 DFS。

**正确理解：**

```javascript
// 都是 DFS，但访问顺序不同

// 前序 DFS：根 -> 左 -> 右
function preorderDFS(node) {
  if (!node) return;
  console.log(node.val);       // 先访问根
  preorderDFS(node.left);
  preorderDFS(node.right);
}

// 后序 DFS：左 -> 右 -> 根
function postorderDFS(node) {
  if (!node) return;
  postorderDFS(node.left);
  postorderDFS(node.right);
  console.log(node.val);       // 后访问根
}

// 两者都是"深度优先"，只是访问根节点的时机不同
```

**React 源码示例：**

React Fiber 的遍历是**前序+后序的组合 DFS**：

```javascript
function performUnitOfWork(unitOfWork) {
  // 前序阶段：先处理当前节点（beginWork）
  let next = beginWork(current, unitOfWork, renderLanes);

  if (next === null) {
    // 后序阶段：再完成节点（completeWork）
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

---

### 误区2：递归就一定比迭代慢 ❌

**为什么错？**

递归和迭代的时间复杂度相同，性能差异主要来自：
1. 函数调用开销（递归多）
2. 栈空间使用（递归用系统栈，迭代用手动栈）
3. 编译器优化（尾递归优化等）

**关键区别：**

| 维度 | 递归 | 迭代 |
|-----|------|------|
| 时间复杂度 | O(n) | O(n) |
| 空间复杂度 | O(h)（系统栈） | O(h)（手动栈） |
| 函数调用开销 | 多 | 少 |
| 代码可读性 | 高 | 低 |
| 可控性 | 低（难以中断） | 高（可随时暂停） |

**为什么人们容易这样错？**

递归确实有函数调用开销，但在现代 JS 引擎中，这个开销通常可以忽略（除非递归深度极大）。

真正的问题是**可控性**：递归难以中断和恢复。

**正确理解：**

```javascript
// 递归版本（简洁，但难以中断）
function sumRecursive(node) {
  if (!node) return 0;
  return node.val + sumRecursive(node.left) + sumRecursive(node.right);
}

// 迭代版本（可控，可随时暂停）
function sumIterative(root) {
  if (!root) return 0;

  let sum = 0;
  const stack = [root];

  while (stack.length > 0) {
    const node = stack.pop();
    sum += node.val;

    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);

    // 可以在这里检查是否需要暂停
    // if (shouldPause()) break;
  }

  return sum;
}
```

**React 源码示例：**

React 选择迭代不是因为性能，而是因为**需要可中断渲染**：

```javascript
function workLoopConcurrent() {
  // 迭代版 DFS，可以随时中断
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
  // 如果 shouldYield() 返回 true，循环暂停，可以稍后恢复
}
```

如果用递归，就无法在渲染中途让出控制权给浏览器。

---

### 误区3：Fiber 的 child/sibling/return 只是换了个名字的 children 数组 ❌

**为什么错？**

`children` 数组和 `child/sibling/return` 三指针虽然都能表示树结构，但**遍历方式完全不同**：
- **children 数组**：需要额外栈来记录遍历位置
- **三指针链表**：通过指针移动即可遍历，无需额外栈

**关键区别：**

```javascript
// 传统树节点（children 数组）
class TreeNode {
  constructor(val) {
    this.val = val;
    this.children = [];  // 数组存储所有子节点
  }
}

// Fiber 节点（三指针链表）
class FiberNode {
  constructor(val) {
    this.val = val;
    this.child = null;    // 只指向第一个子节点
    this.sibling = null;  // 指向下一个兄弟节点
    this.return = null;   // 指向父节点
  }
}
```

**为什么人们容易这样错？**

从数据结构角度看，两者确实都能表示树。但从**遍历角度**看，三指针设计有独特优势：

1. **遍历位置隐式保存**：当前节点的指针就是遍历位置，无需额外栈
2. **双向遍历**：既可以向下（child），也可以向上（return）
3. **中断友好**：保存当前 Fiber 指针即可恢复遍历

**正确理解：**

```javascript
// children 数组遍历：需要额外栈
function traverseWithArray(root) {
  const stack = [{ node: root, childIndex: 0 }];  // 需要记录遍历到第几个子节点

  while (stack.length > 0) {
    const { node, childIndex } = stack.pop();

    if (childIndex === 0) {
      console.log('进入:', node.val);
    }

    if (childIndex < node.children.length) {
      // 还有子节点未处理，压回栈并更新索引
      stack.push({ node, childIndex: childIndex + 1 });
      stack.push({ node: node.children[childIndex], childIndex: 0 });
    } else {
      console.log('离开:', node.val);
    }
  }
}

// Fiber 三指针遍历：无需额外栈
function traverseFiber(fiber) {
  let node = fiber;

  while (true) {
    console.log('进入:', node.val);

    // 简单的指针移动，无需记录索引
    if (node.child) {
      node = node.child;
      continue;
    }

    console.log('离开:', node.val);

    if (node === fiber) return;

    while (!node.sibling) {
      if (!node.return || node.return === fiber) return;
      node = node.return;
      console.log('离开:', node.val);
    }

    node = node.sibling;
  }
}
```

**React 源码示例：**

Fiber 的三指针设计让 React 能够**仅通过保存 workInProgress 指针**就实现中断和恢复：

```javascript
// 只需保存一个指针，就能恢复遍历
let workInProgress = null;

function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
  // 暂停后，workInProgress 指针保存了当前位置
  // 下次继续执行时，从 workInProgress 继续即可
}
```

如果用 children 数组，就需要保存每层的遍历索引，复杂度高得多。

---

## 7. 【实战代码】

### 基础实现（简化版）

```javascript
// ===== 1. 树节点定义 =====
console.log("=== 1. 树节点定义 ===");

class TreeNode {
  constructor(val, left = null, right = null) {
    this.val = val;
    this.left = left;
    this.right = right;
  }
}

// 构建示例树：
//       1
//      / \
//     2   3
//    / \
//   4   5

const tree = new TreeNode(
  1,
  new TreeNode(2, new TreeNode(4), new TreeNode(5)),
  new TreeNode(3)
);

console.log("树结构已创建");

// ===== 2. 三种 DFS 遍历 =====
console.log("\n=== 2. 三种 DFS 遍历 ===");

// 前序遍历：根 -> 左 -> 右
function preorder(node, result = []) {
  if (!node) return result;

  result.push(node.val);         // 访问根
  preorder(node.left, result);   // 递归左子树
  preorder(node.right, result);  // 递归右子树

  return result;
}

// 中序遍历：左 -> 根 -> 右
function inorder(node, result = []) {
  if (!node) return result;

  inorder(node.left, result);    // 递归左子树
  result.push(node.val);         // 访问根
  inorder(node.right, result);   // 递归右子树

  return result;
}

// 后序遍历：左 -> 右 -> 根
function postorder(node, result = []) {
  if (!node) return result;

  postorder(node.left, result);  // 递归左子树
  postorder(node.right, result); // 递归右子树
  result.push(node.val);         // 访问根

  return result;
}

console.log("前序遍历:", preorder(tree));  // [1, 2, 4, 5, 3]
console.log("中序遍历:", inorder(tree));   // [4, 2, 5, 1, 3]
console.log("后序遍历:", postorder(tree)); // [4, 5, 2, 3, 1]

// ===== 3. 递归 vs 迭代实现 =====
console.log("\n=== 3. 递归 vs 迭代实现 ===");

// 递归版前序遍历
function preorderRecursive(node, result = []) {
  if (!node) return result;

  result.push(node.val);
  preorderRecursive(node.left, result);
  preorderRecursive(node.right, result);

  return result;
}

// 迭代版前序遍历（使用栈）
function preorderIterative(root) {
  if (!root) return [];

  const result = [];
  const stack = [root];

  while (stack.length > 0) {
    const node = stack.pop();
    result.push(node.val);

    // 先压右子树，再压左子树（保证左子树先处理）
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }

  return result;
}

console.log("递归前序:", preorderRecursive(tree));  // [1, 2, 4, 5, 3]
console.log("迭代前序:", preorderIterative(tree));  // [1, 2, 4, 5, 3]

// ===== 4. Fiber 三指针结构 =====
console.log("\n=== 4. Fiber 三指针结构 ===");

class FiberNode {
  constructor(tag) {
    this.tag = tag;
    this.child = null;    // 第一个子节点
    this.sibling = null;  // 下一个兄弟节点
    this.return = null;   // 父节点
  }
}

// 构建 Fiber 树（同样的结构）：
//       1
//      /
//     2 - 3
//    /
//   4 - 5

const fiber1 = new FiberNode('div');
const fiber2 = new FiberNode('span');
const fiber3 = new FiberNode('p');
const fiber4 = new FiberNode('button');
const fiber5 = new FiberNode('input');

// 建立 child 关系
fiber1.child = fiber2;
fiber2.child = fiber4;

// 建立 sibling 关系
fiber2.sibling = fiber3;
fiber4.sibling = fiber5;

// 建立 return 关系
fiber2.return = fiber1;
fiber3.return = fiber1;
fiber4.return = fiber2;
fiber5.return = fiber2;

console.log("Fiber 树结构已创建");

// Fiber 树的 DFS 遍历（模拟 React workLoop）
function traverseFiber(fiber) {
  let node = fiber;
  const beginWorkOrder = [];
  const completeWorkOrder = [];

  while (true) {
    // beginWork：前序处理
    beginWorkOrder.push(node.tag);

    // 1. 有子节点，向下遍历
    if (node.child) {
      node = node.child;
      continue;
    }

    // 2. 没有子节点，完成当前节点
    completeWorkOrder.push(node.tag);

    // 3. 回到起点，遍历结束
    if (node === fiber) {
      break;
    }

    // 4. 处理兄弟节点或向上回溯
    while (!node.sibling) {
      if (!node.return || node.return === fiber) {
        return { beginWorkOrder, completeWorkOrder };
      }

      node = node.return;
      completeWorkOrder.push(node.tag);
    }

    node = node.sibling;
  }

  return { beginWorkOrder, completeWorkOrder };
}

const { beginWorkOrder, completeWorkOrder } = traverseFiber(fiber1);
console.log("beginWork 顺序 (前序):", beginWorkOrder);
// ['div', 'span', 'button', 'input', 'p']
console.log("completeWork 顺序 (后序):", completeWorkOrder);
// ['button', 'input', 'span', 'p', 'div']

// ===== 5. 实战场景：计算树的最大深度 =====
console.log("\n=== 5. 实战场景：计算树的最大深度 ===");

// 递归实现（后序遍历思想）
function maxDepthRecursive(node) {
  if (!node) return 0;

  // 先计算左右子树深度
  const leftDepth = maxDepthRecursive(node.left);
  const rightDepth = maxDepthRecursive(node.right);

  // 当前节点深度 = max(左子树深度, 右子树深度) + 1
  return Math.max(leftDepth, rightDepth) + 1;
}

// 迭代实现（使用栈）
function maxDepthIterative(root) {
  if (!root) return 0;

  const stack = [{ node: root, depth: 1 }];
  let maxDepth = 0;

  while (stack.length > 0) {
    const { node, depth } = stack.pop();
    maxDepth = Math.max(maxDepth, depth);

    if (node.right) stack.push({ node: node.right, depth: depth + 1 });
    if (node.left) stack.push({ node: node.left, depth: depth + 1 });
  }

  return maxDepth;
}

console.log("递归计算深度:", maxDepthRecursive(tree));  // 3
console.log("迭代计算深度:", maxDepthIterative(tree));  // 3
```

### 运行输出示例

```
=== 1. 树节点定义 ===
树结构已创建

=== 2. 三种 DFS 遍历 ===
前序遍历: [ 1, 2, 4, 5, 3 ]
中序遍历: [ 4, 2, 5, 1, 3 ]
后序遍历: [ 4, 5, 2, 3, 1 ]

=== 3. 递归 vs 迭代实现 ===
递归前序: [ 1, 2, 4, 5, 3 ]
迭代前序: [ 1, 2, 4, 5, 3 ]

=== 4. Fiber 三指针结构 ===
Fiber 树结构已创建
beginWork 顺序 (前序): [ 'div', 'span', 'button', 'input', 'p' ]
completeWork 顺序 (后序): [ 'button', 'input', 'span', 'p', 'div' ]

=== 5. 实战场景：计算树的最大深度 ===
递归计算深度: 3
迭代计算深度: 3
```

---

### 进阶：React 源码实现

```javascript
// React 源码片段（packages/react-reconciler/src/ReactFiberWorkLoop.js）

/**
 * workLoopSync：同步模式的工作循环（不可中断）
 */
function workLoopSync() {
  // workInProgress 是当前正在处理的 Fiber 节点
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

/**
 * workLoopConcurrent：并发模式的工作循环（可中断）
 */
function workLoopConcurrent() {
  // shouldYield() 检查是否需要让出控制权
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

/**
 * performUnitOfWork：处理单个 Fiber 节点
 * 这是 DFS 遍历的核心：beginWork（向下）+ completeWork（向上）
 */
function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;

  let next;

  // beginWork：前序阶段，处理当前 Fiber
  // 返回第一个子 Fiber，如果没有子 Fiber 则返回 null
  next = beginWork(current, unitOfWork, renderLanes);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // 没有子 Fiber，进入完成阶段（后序阶段）
    completeUnitOfWork(unitOfWork);
  } else {
    // 有子 Fiber，继续向下遍历
    workInProgress = next;
  }
}

/**
 * completeUnitOfWork：完成当前 Fiber 及其兄弟节点
 * 实现回溯逻辑：完成当前节点 -> 处理兄弟 -> 回到父节点
 */
function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;

  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    let next;

    // completeWork：后序阶段，完成当前 Fiber
    next = completeWork(current, completedWork, renderLanes);

    if (next !== null) {
      // 某些情况下 completeWork 可能产生新的工作
      workInProgress = next;
      return;
    }

    // 收集副作用（Effect）到父 Fiber
    if (returnFiber !== null) {
      // 将当前 Fiber 的副作用链接到父 Fiber
      if (returnFiber.firstEffect === null) {
        returnFiber.firstEffect = completedWork.firstEffect;
      }
      if (completedWork.lastEffect !== null) {
        if (returnFiber.lastEffect !== null) {
          returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
        }
        returnFiber.lastEffect = completedWork.lastEffect;
      }

      // 如果当前 Fiber 有副作用，也添加到链表
      const flags = completedWork.flags;
      if (flags > PerformedWork) {
        if (returnFiber.lastEffect !== null) {
          returnFiber.lastEffect.nextEffect = completedWork;
        } else {
          returnFiber.firstEffect = completedWork;
        }
        returnFiber.lastEffect = completedWork;
      }
    }

    // 尝试处理兄弟节点（DFS 的横向移动）
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // 有兄弟节点，开始处理兄弟
      workInProgress = siblingFiber;
      return;
    }

    // 没有兄弟节点，向上回溯到父节点
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);

  // 完成了整棵树的遍历
  if (workInProgressRootExitStatus === RootIncomplete) {
    workInProgressRootExitStatus = RootCompleted;
  }
}

/**
 * beginWork：前序阶段的工作
 * 根据 Fiber 类型创建子 Fiber，返回第一个子 Fiber
 */
function beginWork(current, workInProgress, renderLanes) {
  // 根据 Fiber 的 tag 执行不同的处理逻辑
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        unresolvedProps,
        renderLanes,
      );
    }
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        unresolvedProps,
        renderLanes,
      );
    }
    case HostComponent: {
      return updateHostComponent(current, workInProgress, renderLanes);
    }
    // ... 其他类型
  }
}

/**
 * completeWork：后序阶段的工作
 * 创建或更新真实 DOM，收集副作用
 */
function completeWork(current, workInProgress, renderLanes) {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case HostComponent: {
      // 对于原生 DOM 节点
      const type = workInProgress.type;

      if (current !== null && workInProgress.stateNode != null) {
        // 更新阶段：标记需要更新的属性
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          renderLanes,
        );
      } else {
        // 挂载阶段：创建 DOM 节点
        const instance = createInstance(
          type,
          newProps,
          rootContainerInstance,
          currentHostContext,
          workInProgress,
        );

        // 将子 DOM 节点添加到当前 DOM 节点
        appendAllChildren(instance, workInProgress);

        workInProgress.stateNode = instance;

        // 设置 DOM 属性
        if (
          finalizeInitialChildren(
            instance,
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
          )
        ) {
          markUpdate(workInProgress);
        }
      }

      return null;
    }
    // ... 其他类型
  }
}
```

**关键要点：**

1. **workLoop 是 DFS 的迭代实现**：
   - `while (workInProgress !== null)` 循环遍历 Fiber 树
   - 通过移动 `workInProgress` 指针实现遍历

2. **前序+后序的组合 DFS**：
   - `beginWork`：前序阶段，创建子 Fiber
   - `completeWork`：后序阶段，创建 DOM、收集副作用

3. **三指针遍历逻辑**：
   - 有 child → 向下（`workInProgress = next`）
   - 无 child → 完成当前节点，检查 sibling
   - 有 sibling → 处理兄弟（`workInProgress = siblingFiber`）
   - 无 sibling → 向上回溯（`completedWork = returnFiber`）

4. **可中断设计**：
   - `shouldYield()` 检查是否需要暂停
   - 保存 `workInProgress` 指针即可恢复遍历

---

## 8. 【面试必问】

### 问题1："什么是 DFS？它和 BFS 有什么区别？"

**普通回答（❌ 不出彩）：**

"DFS 是深度优先搜索，先访问子节点再访问兄弟节点。BFS 是广度优先搜索，先访问同层节点。"

**出彩回答（✅ 推荐）：**

> **DFS 和 BFS 是两种不同的树遍历策略：**
>
> 1. **遍历顺序不同**：
>    - DFS（深度优先）：沿着一条路径走到底，再回溯处理其他路径
>    - BFS（广度优先）：逐层访问，先访问完同层节点再访问下一层
>
> 2. **数据结构不同**：
>    - DFS 使用**栈**（递归 = 隐式栈，迭代 = 显式栈）
>    - BFS 使用**队列**（先进先出）
>
> 3. **空间复杂度不同**：
>    - DFS：O(h)，h 是树高（只需保存当前路径）
>    - BFS：O(w)，w 是最大宽度（需保存整层节点）
>
> 4. **适用场景不同**：
>    - DFS：适合"路径相关"问题（如求最大深度、路径和）
>    - BFS：适合"层次相关"问题（如层序遍历、最短路径）
>
> **在 React 源码中的应用**：
>
> React Fiber 树使用 DFS 遍历（beginWork + completeWork），因为：
> - 组件渲染是"深度优先"的（先渲染子组件才能确定父组件）
> - DFS 的空间占用更小（树通常是高瘦的，而非矮胖的）
> - DFS 易于通过迭代实现，支持可中断渲染
>
> 具体实现：
> ```javascript
> // React workLoop 是迭代版 DFS
> function workLoopConcurrent() {
>   while (workInProgress !== null && !shouldYield()) {
>     performUnitOfWork(workInProgress);
>   }
> }
> ```

**为什么这个回答出彩？**
1. ✅ 多维度对比（顺序、数据结构、复杂度、场景）
2. ✅ 结合 React 源码实际应用
3. ✅ 展示对"为什么 React 选择 DFS"的深度思考

---

### 问题2："React Fiber 树是怎么遍历的？为什么不用递归？"

**普通回答（❌ 不出彩）：**

"React 用循环遍历 Fiber 树，因为递归可能栈溢出。"

**出彩回答（✅ 推荐）：**

> **React Fiber 树的遍历是迭代版 DFS，核心原因不是性能，而是可中断性：**
>
> 1. **遍历方式：迭代版 DFS**
>    - 不使用递归，而是通过 `while` 循环 + 指针移动
>    - `workInProgress` 指针记录当前遍历位置
>    - 通过 `child/sibling/return` 三指针实现遍历
>
> 2. **遍历逻辑：前序+后序组合**
>    - `beginWork`（前序）：向下处理 Fiber，创建子 Fiber
>    - `completeWork`（后序）：向上完成 Fiber，创建 DOM 并收集副作用
>    - 遍历顺序：child → sibling → return（深度优先）
>
> 3. **为什么不用递归？关键是可中断性**
>    - 递归无法中断：一旦开始递归，必须走完整个调用栈
>    - 迭代可中断：通过 `shouldYield()` 检查，随时暂停
>    - 恢复遍历：保存 `workInProgress` 指针，下次从这里继续
>
>    ```javascript
>    function workLoopConcurrent() {
>      // 可中断的工作循环
>      while (workInProgress !== null && !shouldYield()) {
>        performUnitOfWork(workInProgress);
>      }
>      // 暂停后，workInProgress 保存了当前位置
>    }
>    ```
>
> 4. **三指针设计的巧妙之处**
>    - 传统树节点用 `children` 数组，遍历需要额外栈记录索引
>    - Fiber 用 `child/sibling/return`，遍历只需移动指针
>    - 这让"保存遍历状态"变得极简：只需保存一个指针
>
> **实际价值：**
>
> 这种设计支持了 React 18 的并发渲染特性：
> - Time Slicing（时间切片）：将长任务拆分，避免阻塞主线程
> - Suspense：暂停组件渲染，等待数据加载
> - Transitions：区分紧急和非紧急更新
>
> 如果用递归，这些特性都无法实现，因为无法在递归过程中"暂停"。

**为什么这个回答出彩？**
1. ✅ 直击核心：可中断性比性能更重要
2. ✅ 解释了三指针设计的巧妙之处
3. ✅ 联系到 React 18 的并发特性，展示对新特性的理解
4. ✅ 包含具体代码示例，展示源码阅读能力

---

## 9. 【化骨绵掌】

### 卡片1：DFS 的直觉理解 🎯

**一句话：** DFS 就是"一条路走到黑，走不通再回头"的遍历策略。

**举例：**

想象你在图书馆找书：
- DFS 策略：进入第一排书架，一直走到最里面，没找到再退回来，换第二排
- BFS 策略：先看每排书架的入口，都看完再深入每一排

```javascript
function dfs(node) {
  if (!node) return;
  visit(node);           // 访问当前节点
  dfs(node.left);        // 一路向左
  dfs(node.right);       // 再向右
}
```

**应用：** React Fiber 树的遍历就是 DFS，先处理子组件，再处理兄弟组件。

---

### 卡片2：三种遍历顺序 📐

**一句话：** 前序、中序、后序的区别在于**何时访问根节点**。

**举例：**

```javascript
//       1
//      / \
//     2   3

// 前序（根-左-右）：1, 2, 3（先访问根）
// 中序（左-根-右）：2, 1, 3（中间访问根）
// 后序（左-右-根）：2, 3, 1（最后访问根）
```

**应用：** React 的 beginWork 是前序（向下处理），completeWork 是后序（向上完成）。

---

### 卡片3：递归的本质 🔄

**一句话：** 递归 = 递推（向下分解） + 回归（向上合并）。

**举例：**

```javascript
// 计算阶乘：5! = 5 × 4!
function factorial(n) {
  if (n === 1) return 1;      // 终止条件
  return n * factorial(n-1);  // 递推 + 回归
}

// 递推：5! -> 4! -> 3! -> 2! -> 1!
// 回归：1! -> 2 -> 6 -> 24 -> 120
```

**应用：** DFS 递归版就是利用系统栈实现"递推+回归"。

---

### 卡片4：递归 vs 迭代 ⚖️

**一句话：** 递归简洁但不可控，迭代复杂但可中断。

**举例：**

```javascript
// 递归：简洁，但无法暂停
function dfsRec(node) {
  if (!node) return;
  visit(node);
  dfsRec(node.left);
  dfsRec(node.right);
}

// 迭代：复杂，但可控
function dfsIter(root) {
  const stack = [root];
  while (stack.length > 0) {
    const node = stack.pop();
    visit(node);
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);

    // 可以在这里暂停
    if (shouldPause()) break;
  }
}
```

**应用：** React 必须用迭代实现 DFS，才能支持可中断渲染。

---

### 卡片5：栈在 DFS 中的作用 📚

**一句话：** 栈记录"待处理的节点"，后进先出保证深度优先。

**举例：**

```javascript
// 遍历 [1, 2, 3] 树
// 栈变化：
// [1]           -> 弹出1，压入3、2
// [3, 2]        -> 弹出2（深度优先！）
// [3]           -> 弹出3
// []            -> 完成
```

**应用：** React 的 `workInProgress` 就相当于"栈顶指针"。

---

### 卡片6：Fiber 三指针 vs 传统 children 数组 🔗

**一句话：** 三指针将树"链表化"，遍历时无需额外栈。

**举例：**

```javascript
// 传统树：需要栈记录遍历进度
class TreeNode {
  children = [];  // 不知道遍历到第几个子节点
}

// Fiber 树：指针移动即可
class FiberNode {
  child = null;    // 指向第一个子节点
  sibling = null;  // 指向下一个兄弟
  return = null;   // 指向父节点（可双向遍历！）
}
```

**应用：** Fiber 的三指针让"保存遍历状态"变得极简，只需保存一个指针。

---

### 卡片7：beginWork 和 completeWork 的分工 🏗️

**一句话：** beginWork 向下创建，completeWork 向上完成。

**举例：**

```javascript
function performUnitOfWork(fiber) {
  // beginWork：向下处理，返回子 Fiber
  let next = beginWork(fiber);

  if (next === null) {
    // 没有子 Fiber，开始向上完成
    completeUnitOfWork(fiber);
  } else {
    // 有子 Fiber，继续向下
    workInProgress = next;
  }
}
```

**应用：**
- beginWork：调用函数组件、创建子 Fiber
- completeWork：创建 DOM、收集副作用

---

### 卡片8：可中断遍历的实现 ⏸️

**一句话：** 通过 `shouldYield()` 检查时间，超时则暂停，保存指针后恢复。

**举例：**

```javascript
function workLoopConcurrent() {
  // 每次循环前检查是否超时
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
  // 暂停后，workInProgress 保存了断点
}

// 恢复渲染时，从 workInProgress 继续
function resumeRendering() {
  workLoopConcurrent();  // 继续之前的循环
}
```

**应用：** React 18 的并发渲染（Concurrent Rendering）核心机制。

---

### 卡片9：DFS 的空间复杂度 💾

**一句话：** DFS 的空间复杂度是 O(h)（树高），而非 O(n)（节点数）。

**举例：**

```javascript
// 假设树有 1000 个节点，但高度只有 10
// DFS 栈最多只需要 10 层（当前路径）
// 而不需要存储全部 1000 个节点
function dfs(node, depth = 0) {
  if (!node) return;

  // 当前栈深度 = depth
  dfs(node.left, depth + 1);
  dfs(node.right, depth + 1);

  // 回溯后栈深度恢复
}
```

**应用：** 为什么 React 选择 DFS 而非 BFS（BFS 需要 O(w) 空间，w 是最大宽度）。

---

### 卡片10：总结与延伸 🎓

**一句话：** DFS 是树遍历的基础算法，React Fiber 通过迭代版 DFS 实现了可中断渲染。

**核心要点：**

1. **DFS 三种顺序**：前序、中序、后序（区别在访问根的时机）
2. **两种实现**：递归（简洁）、迭代（可控）
3. **Fiber 三指针**：child/sibling/return 支持无栈遍历
4. **React 应用**：beginWork（前序）+ completeWork（后序）

**延伸学习：**
- BFS（广度优先搜索）
- 回溯算法（DFS 的变体）
- React Fiber 架构的完整渲染流程
- React 18 并发特性（Time Slicing、Suspense）

---

## 10. 【一句话总结】

**DFS（深度优先遍历）是沿着树的深度方向优先访问节点的算法，通过递归或栈实现前序/中序/后序三种遍历顺序，React Fiber 使用迭代版 DFS（child/sibling/return 三指针）实现可中断的树遍历，支持并发渲染。**

---

## 附录

### 学习检查清单

- [ ] 理解 DFS 的基本思想（深度优先 vs 广度优先）
- [ ] 掌握前序、中序、后序三种遍历顺序
- [ ] 能写出 DFS 的递归和迭代实现
- [ ] 理解 Fiber 三指针结构（child/sibling/return）
- [ ] 理解 React workLoop 的遍历逻辑
- [ ] 理解 beginWork 和 completeWork 的分工
- [ ] 知道为什么 React 使用迭代而非递归
- [ ] 理解可中断渲染的实现原理

### 下一步学习建议

1. **BFS（广度优先搜索）**：对比学习，理解两种遍历策略的差异
2. **双向链表操作**：深入理解 Fiber 节点的链表结构
3. **React Fiber 架构**：学习 Fiber 的完整设计和渲染流程
4. **React 调度器（Scheduler）**：理解时间切片和优先级调度

### 快速参考卡

| DFS 概念 | 关键点 | React 应用 |
|---------|--------|------------|
| 前序遍历 | 根 → 左 → 右 | beginWork（向下处理） |
| 中序遍历 | 左 → 根 → 右 | 二叉搜索树排序 |
| 后序遍历 | 左 → 右 → 根 | completeWork（向上完成） |
| 递归实现 | 简洁，系统栈 | 不适合 React（不可中断） |
| 迭代实现 | 可控，显式栈 | React workLoop |
| 空间复杂度 | O(h) | h 是树高 |
| Fiber 三指针 | child/sibling/return | 无栈遍历 |
| 可中断性 | shouldYield() | 并发渲染 |

---

**版本：** v1.0
**创建日期：** 2025-12-06
**适用于：** React 19 源码学习
**前置知识：** JavaScript 基础、树数据结构
**后续学习：** 双向链表操作、Fiber 架构
