# use() Hook

## 一、【30字核心】

**use() 是 React 19 的资源读取 API，可在条件语句和循环中调用，用于解包 Promise 和读取 Context，与 Suspense 和 ErrorBoundary 深度集成，实现优雅的异步数据处理和条件性 Context 访问。**

---

## 二、【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### use() 的第一性原理 🎯

#### 1. 最基础的定义

**use() = 读取资源（Promise / Context）的值**

仅此而已！没有更基础的了。

当你有一个 Promise 或 Context，use() 可以"解包"它，返回其中的值。Promise pending 时组件会 suspend（挂起），resolved 后返回值，rejected 后抛出错误。

```javascript
import { use, Suspense } from 'react';

const dataPromise = fetch('/api').then(r => r.json());

function MyComponent() {
  const data = use(dataPromise); // 解包 Promise
  return <div>{data.message}</div>;
}

function App() {
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <MyComponent />
    </Suspense>
  );
}
```

#### 2. 为什么需要 use()？

**核心问题：如何在渲染期间处理异步数据？**

在 React 中，组件渲染必须是同步的：
- 不能在组件中直接使用 `await`（函数组件不能是 async）
- 不能在渲染期间等待异步操作完成

传统解决方案：
```javascript
// ❌ 不能这样做
function Component() {
  const data = await fetch('/api'); // 错误：组件不能是 async
  return <div>{data}</div>;
}
```

use() 解决了这个问题：
```javascript
// ✅ 可以这样做
function Component() {
  const data = use(fetchPromise); // use() 解包 Promise
  return <div>{data}</div>;
}
```

#### 3. use() 的三层价值

##### 价值1：解包 Promise，简化异步处理 📦

use() 让你在组件中直接"使用"Promise 的值，无需手动管理loading/error 状态。

```javascript
import { use, Suspense } from 'react';

// 在组件外创建 Promise
const messagePromise = fetch('https://api.github.com/repos/facebook/react')
  .then(r => r.json())
  .then(data => data.description);

function Message() {
  // use() 解包 Promise
  const message = use(messagePromise);
  return <p>{message}</p>;
}

function App() {
  return (
    <Suspense fallback={<p>加载中...</p>}>
      <Message />
    </Suspense>
  );
}
```

**执行流程：**
1. `use(messagePromise)` 被调用
2. Promise 是 pending → 组件 suspend（挂起）
3. Suspense fallback 显示 "加载中..."
4. Promise resolved → 组件重新渲染
5. `use()` 返回 resolved 值
6. 显示最终内容

##### 价值2：可在条件中调用，突破 Hook 限制 🔓

传统 Hooks（如 useContext）必须在顶层调用，use() 可以在条件语句和循环中调用。

```javascript
import { use, createContext } from 'react';

const ThemeContext = createContext('light');

function Button({ showTheme }) {
  // ✅ use() 可以在条件中调用
  if (showTheme) {
    const theme = use(ThemeContext);
    return <button className={theme}>Themed</button>;
  }

  return <button>Default</button>;
}

// ❌ useContext 不能在条件中调用
function BadButton({ showTheme }) {
  if (showTheme) {
    const theme = useContext(ThemeContext); // ❌ 违反 Hook 规则
    return <button className={theme}>Themed</button>;
  }
  return <button>Default</button>;
}
```

**为什么 use() 可以突破限制？**
- use() 不是 Hook，而是一个特殊的 API
- 它不依赖于组件的渲染次数和调用顺序
- 每次调用都是独立的资源读取

##### 价值3：与 Suspense 和 ErrorBoundary 深度集成 🛡️

use() 自动和 Suspense/ErrorBoundary 协作，无需手动处理。

```javascript
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

async function fetchData() {
  const response = await fetch('/api/data');
  if (!response.ok) {
    throw new Error('获取失败');
  }
  return response.json();
}

const dataPromise = fetchData();

function DataDisplay() {
  const data = use(dataPromise);
  return <div>{data.content}</div>;
}

function App() {
  return (
    <ErrorBoundary fallback={<div>❌ 出错了</div>}>
      <Suspense fallback={<div>⌛ 加载中...</div>}>
        <DataDisplay />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**协作机制：**
- Promise pending → Suspense fallback
- Promise resolved → 正常显示
- Promise rejected → ErrorBoundary fallback

#### 4. 从第一性原理推导 React 实现

**推理链：**

```
1. 需求：在渲染期间使用异步数据
   ↓
2. 问题：组件不能是 async，不能用 await
   ↓
3. 解决思路：让 Promise 成为"可读的资源"
   ↓
4. 机制：Promise pending 时"抛出" Promise
   ↓
5. React 捕获抛出的 Promise → 触发 Suspense
   ↓
6. Promise resolved 后重新渲染组件
   ↓
7. use() 返回 resolved 值
   ↓
8. React 19 引入 use() API 简化这个模式
   ↓
9. 扩展到 Context：统一资源读取接口
   ↓
10. 允许条件调用：不依赖 Hook 规则
```

**use() 的底层原理（简化）：**

```javascript
function use(resource) {
  if (resource.$$typeof === REACT_CONTEXT_TYPE) {
    // 读取 Context
    return readContext(resource);
  }

  if (typeof resource.then === 'function') {
    // Promise
    const status = resource.status;

    if (status === 'fulfilled') {
      return resource.value;
    }

    if (status === 'rejected') {
      throw resource.reason;
    }

    if (status === 'pending') {
      throw resource; // 抛出 Promise，触发 Suspense
    }
  }

  throw new Error('Invalid resource type');
}
```

#### 5. 一句话总结第一性原理

**use() 是通过"抛出 Promise"触发 Suspense 挂起，Promise resolved 后重新渲染返回值，实现同步式异步数据读取，同时支持条件调用和 Context 读取，是 React 并发渲染和 Suspense 架构的高级应用。**

---

## 三、【3个核心概念】

### 核心概念1：Promise 解包（Promise Unwrapping）📦

**Promise 解包是指 use() 将 Promise 的 resolved 值提取出来，供组件直接使用。**

```javascript
import { use, Suspense } from 'react';

// ===== 1. 创建 Promise（组件外）=====
const fetchUser = async (id) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

const userPromise = fetchUser(123);

// ===== 2. 解包 Promise =====
function User() {
  // use() 解包 Promise，返回 { name, email }
  const user = use(userPromise);

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// ===== 3. Suspense 包裹 =====
function App() {
  return (
    <Suspense fallback={<div>加载用户信息...</div>}>
      <User />
    </Suspense>
  );
}
```

**详细解释：**

1. **执行流程**：
   ```
   第1次渲染：
     → use(userPromise) 被调用
     → Promise 是 pending
     → use() 抛出 Promise
     → React 捕获，触发 Suspense
     → 显示 fallback："加载用户信息..."

   Promise resolved 后：
     → React 重新渲染 User 组件
     → use(userPromise) 再次被调用
     → Promise 已 fulfilled
     → use() 返回 { name: 'Alice', email: 'alice@example.com' }
     → 显示用户信息
   ```

2. **Promise 必须在组件外创建**：
   ```javascript
   // ❌ 错误：每次渲染都创建新的 Promise
   function BadComponent() {
     const data = use(fetch('/api')); // 无限循环
   }

   // ✅ 正确：Promise 在组件外创建
   const dataPromise = fetch('/api');
   function GoodComponent() {
     const data = use(dataPromise);
   }
   ```

3. **Promise 缓存**：
   ```javascript
   // 使用 React Cache API（React 19）
   import { cache } from 'react';

   const fetchUser = cache(async (id) => {
     const response = await fetch(`/api/users/${id}`);
     return response.json();
   });

   // 多次调用 fetchUser(123) 只会执行1次
   ```

**在 React 源码/开发中的应用：**

Server Component → Client Component 传递 Promise：

```javascript
// app/page.jsx (Server Component)
async function getData() {
  const response = await fetch('https://api.example.com/data');
  return response.json();
}

export default function Page() {
  const dataPromise = getData();

  return <ClientComponent dataPromise={dataPromise} />;
}

// components/ClientComponent.jsx (Client Component)
'use client';
import { use } from 'react';

export default function ClientComponent({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data.message}</div>;
}
```

---

### 核心概念2：条件调用（Conditional Calls）🔀

**use() 可以在 if、for、while 等控制流中调用，打破了传统 Hook 的限制。**

```javascript
import { use, createContext } from 'react';

const UserContext = createContext(null);
const ThemeContext = createContext('light');

// ===== 示例1：条件读取 Context =====
function ProfileButton({ showUser, showTheme }) {
  // ✅ 根据 props 决定是否读取 Context
  const user = showUser ? use(UserContext) : null;
  const theme = showTheme ? use(ThemeContext) : 'default';

  return (
    <button className={theme}>
      {user ? user.name : 'Guest'}
    </button>
  );
}

// ===== 示例2：循环读取 =====
function MultiContextReader({ contexts }) {
  const values = [];

  // ✅ 在循环中调用 use()
  for (const context of contexts) {
    values.push(use(context));
  }

  return <div>{values.join(', ')}</div>;
}

// ===== 示例3：switch 语句 =====
function DynamicData({ dataType }) {
  let data;

  switch (dataType) {
    case 'user':
      data = use(userPromise);
      break;
    case 'posts':
      data = use(postsPromise);
      break;
    default:
      data = null;
  }

  return <div>{JSON.stringify(data)}</div>;
}
```

**为什么 use() 可以在条件中调用？**

1. **不依赖调用顺序**：
   ```javascript
   // ❌ useContext 必须在顶层（依赖调用顺序）
   function Bad({ showTheme }) {
     if (showTheme) {
       const theme = useContext(ThemeContext); // ❌ 违反规则
     }
   }

   // ✅ use() 不依赖顺序
   function Good({ showTheme }) {
     if (showTheme) {
       const theme = use(ThemeContext); // ✅ 允许
     }
   }
   ```

2. **每次都是独立读取**：
   - use() 不维护内部状态
   - 不依赖 Fiber 链表
   - 每次调用都是新的资源读取

3. **React 内部实现差异**：
   ```javascript
   // useContext（Hooks）：
   // - 依赖 currentlyRenderingFiber
   // - 依赖 Hook 链表
   // - 必须按顺序调用

   // use()（Resource API）：
   // - 直接读取资源
   // - 不依赖 Hook 链表
   // - 可以任意调用
   ```

**重要注意事项：**

⚠️ use() 仍然只能在组件或 Hook 中调用，不能在普通函数中：

```javascript
// ❌ 错误：不能在普通函数中
function helper() {
  const data = use(promise); // ❌ 错误
}

// ✅ 正确：在组件中
function MyComponent() {
  const data = use(promise); // ✅ 正确
}

// ✅ 正确：在自定义 Hook 中
function useData() {
  const data = use(promise); // ✅ 正确
  return data;
}
```

**在 React 源码/开发中的应用：**

根据用户角色条件性读取配置：

```javascript
const AdminConfigContext = createContext(null);
const UserConfigContext = createContext(null);

function Dashboard({ user }) {
  const config = user.role === 'admin'
    ? use(AdminConfigContext)
    : use(UserConfigContext);

  return <div>{config.dashboardTitle}</div>;
}
```

---

### 核心概念3：Suspense 集成（Suspense Integration）⏸️

**use() 与 Suspense 深度集成，Promise pending 时自动触发 Suspense 边界显示 fallback。**

```javascript
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

// ===== 模拟 API 请求 =====
function fetchWithDelay(url, delay) {
  return new Promise((resolve, reject) => {
    setTimeout(async () => {
      try {
        const response = await fetch(url);
        if (!response.ok) throw new Error('请求失败');
        const data = await response.json();
        resolve(data);
      } catch (error) {
        reject(error);
      }
    }, delay);
  });
}

const slowDataPromise = fetchWithDelay('/api/slow', 3000);
const fastDataPromise = fetchWithDelay('/api/fast', 1000);

// ===== 慢组件 =====
function SlowData() {
  const data = use(slowDataPromise); // 3秒
  return <div>慢数据: {data.message}</div>;
}

// ===== 快组件 =====
function FastData() {
  const data = use(fastDataPromise); // 1秒
  return <div>快数据: {data.message}</div>;
}

// ===== 并发 Suspense =====
function App() {
  return (
    <div>
      <h1>Suspense 演示</h1>

      {/* 快数据：1秒后显示 */}
      <Suspense fallback={<div>⌛ 加载快数据...</div>}>
        <FastData />
      </Suspense>

      {/* 慢数据：3秒后显示 */}
      <Suspense fallback={<div>⌛ 加载慢数据...</div>}>
        <SlowData />
      </Suspense>
    </div>
  );
}
```

**执行时间线：**

```
时间  | 快数据 Suspense | 慢数据 Suspense
------|----------------|----------------
0s    | "加载快数据..."  | "加载慢数据..."
1s    | "快数据: xxx"   | "加载慢数据..."
3s    | "快数据: xxx"   | "慢数据: yyy"
```

**Suspense 的工作机制：**

1. **组件挂起（Suspend）**：
   ```javascript
   // use() 内部实现（简化）
   function use(promise) {
     if (promise.status === 'pending') {
       throw promise; // 抛出 Promise，触发 Suspense
     }
     return promise.value;
   }
   ```

2. **React 捕获 Promise**：
   - React 的 Suspense 边界捕获抛出的 Promise
   - 显示 fallback UI
   - 等待 Promise resolved

3. **Promise resolved 后**：
   - React 重新渲染组件
   - use() 返回 resolved 值
   - 显示实际内容

**嵌套 Suspense：**

```javascript
function App() {
  return (
    <Suspense fallback={<div>外层加载中...</div>}>
      <Header />

      <Suspense fallback={<div>内容加载中...</div>}>
        <Content />

        <Suspense fallback={<div>评论加载中...</div>}>
          <Comments />
        </Suspense>
      </Suspense>

      <Footer />
    </Suspense>
  );
}
```

**加载顺序：**
1. Header 和 Footer 正常渲染（无异步）
2. Content use() Promise → 显示"内容加载中..."
3. Content loaded → 显示 Content + "评论加载中..."
4. Comments loaded → 全部显示

**ErrorBoundary 集成：**

```javascript
<ErrorBoundary fallback={<div>❌ 出错了</div>}>
  <Suspense fallback={<div>⌛ 加载中...</div>}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>

function DataComponent() {
  const data = use(dataPromise);
  // Promise rejected → ErrorBoundary 捕获
  return <div>{data}</div>;
}
```

**在 React 源码/开发中的应用：**

Next.js App Router 的流式渲染：

```javascript
// app/page.jsx
import { Suspense } from 'react';

async function getData() {
  await new Promise(r => setTimeout(r, 2000));
  return { message: 'Hello' };
}

export default function Page() {
  const dataPromise = getData();

  return (
    <Suspense fallback={<div>Loading...</div>}>
      <ClientComponent dataPromise={dataPromise} />
    </Suspense>
  );
}
```

---

## 四、【最小可用】

掌握以下内容，就能理解 React use() Hook 的核心：

### 4.1 基础用法：解包 Promise

```javascript
import { use, Suspense } from 'react';

// 1. 在组件外创建 Promise
const dataPromise = fetch('/api/data').then(r => r.json());

// 2. 使用 use() 解包
function DataDisplay() {
  const data = use(dataPromise);
  return <div>{data.message}</div>;
}

// 3. 必须用 Suspense 包裹
function App() {
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <DataDisplay />
    </Suspense>
  );
}
```

**核心要点：**
- Promise 必须在组件外创建（避免无限循环）
- use() 解包 Promise，返回 resolved 值
- 必须用 Suspense 包裹（否则会报错）

---

### 4.2 条件读取 Context

```javascript
import { use, createContext } from 'react';

const ThemeContext = createContext('light');

function Button({ showTheme }) {
  // ✅ 可以在条件中调用
  if (showTheme) {
    const theme = use(ThemeContext);
    return <button className={theme}>Themed</button>;
  }

  return <button>Default</button>;
}
```

**核心要点：**
- use() 可以在 if、for、switch 中调用
- 不能在普通函数中调用（只能在组件/Hook）
- 比 useContext 更灵活

---

### 4.3 ErrorBoundary 错误处理

```javascript
import { ErrorBoundary } from 'react-error-boundary';

async function fetchData() {
  const response = await fetch('/api');
  if (!response.ok) throw new Error('失败');
  return response.json();
}

const dataPromise = fetchData();

function App() {
  return (
    <ErrorBoundary fallback={<div>❌ 出错了</div>}>
      <Suspense fallback={<div>⌛ 加载中...</div>}>
        <DataDisplay dataPromise={dataPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}

function DataDisplay({ dataPromise }) {
  const data = use(dataPromise);
  // Promise rejected → ErrorBoundary 捕获
  return <div>{data}</div>;
}
```

**核心要点：**
- Promise rejected → ErrorBoundary fallback
- 不需要 try-catch（use() 不能在 try-catch 中）
- ErrorBoundary 必须在 Suspense 外层

---

**这些知识足以：**
- 使用 use() 读取异步数据
- 条件性读取 Context
- 理解 Suspense 和 ErrorBoundary 的协作
- 为学习 Server Component 打基础

---

## 五、【1个类比】

### 类比1：自动取款机 🏧

**use() = 自动取款机操作**

想象你去自动取款机取钱：

**传统方式（useState + useEffect）：**
```
1. 插卡（发起请求）
2. 你站着等（loading 状态）
3. 机器处理中（Promise pending）
4. 吐钱（数据到达）
5. 你拿走钱（更新 state）

问题：你必须站着等，不能离开，很无聊
```

**use() 方式（Suspense）：**
```
1. 插卡（发起请求）
2. 机器说"请等待，你可以坐下休息"（Suspense fallback）
3. 你去旁边坐着（组件挂起）
4. 机器好了，叫你（Promise resolved）
5. 你回来拿钱（组件重新渲染）

优点：你不用一直站着等，可以做其他事
```

```javascript
// 传统方式：你必须站着等
function Traditional() {
  const [money, setMoney] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    withdrawMoney().then(amount => {
      setMoney(amount);
      setLoading(false);
    });
  }, []);

  if (loading) return <div>站着等...</div>;
  return <div>拿到 {money} 元</div>;
}

// use() 方式：可以坐下休息
function WithUse() {
  const money = use(moneyPromise); // 机器处理中，你去休息
  return <div>拿到 {money} 元</div>;
}

// Suspense = 休息区
<Suspense fallback={<div>坐着等...</div>}>
  <WithUse />
</Suspense>
```

**对应关系：**
- 插卡 = 创建 Promise
- 机器处理 = Promise pending
- 休息区 = Suspense fallback
- 叫号 = Promise resolved
- 拿钱 = use() 返回值

---

### 类比2：图书馆预约 📚

**use() = 图书馆预约借书**

**场景：** 你想借一本书，但书被别人借走了

**传统方式：**
```
1. 去图书馆（组件渲染）
2. 查询书（发起请求）
3. 书被借走了（Promise pending）
4. 你站在柜台等（loading）
5. 书归还了（Promise resolved）
6. 管理员给你（更新 state）

问题：你必须站在柜台等，不能离开
```

**use() 方式：**
```
1. 去图书馆（组件渲染）
2. 登记预约（use(bookPromise)）
3. 书被借走了（Promise pending）
4. 回家等通知（组件挂起，Suspense fallback）
5. 收到短信"书到了"（Promise resolved）
6. 回图书馆拿书（组件重新渲染）

优点：不用在图书馆等，回家做其他事
```

```javascript
// 传统方式：站在柜台等
function TraditionalLibrary() {
  const [book, setBook] = useState(null);
  const [waiting, setWaiting] = useState(true);

  useEffect(() => {
    reserveBook().then(book => {
      setBook(book);
      setWaiting(false);
    });
  }, []);

  if (waiting) return <div>站在柜台等...</div>;
  return <div>拿到书: {book.title}</div>;
}

// use() 方式：回家等通知
function WithUseLibrary() {
  const book = use(bookPromise); // 登记后回家
  return <div>拿到书: {book.title}</div>;
}

<Suspense fallback={<div>回家等通知...</div>}>
  <WithUseLibrary />
</Suspense>
```

---

### 类比3：餐厅点餐 🍔

**use() = 餐厅点餐系统**

**传统方式（站着等）：**
```
1. 点餐（发起请求）
2. 站在柜台前等（loading）
3. 厨房做饭（Promise pending）
4. 做好了叫号（Promise resolved）
5. 拿走食物（更新 state）

问题：你必须站着等，堵住后面的人
```

**use() 方式（拿号牌）：**
```
1. 点餐（发起请求）
2. 拿号牌（use(orderPromise)）
3. 找座位坐下（组件挂起，Suspense fallback）
4. 听到叫号（Promise resolved）
5. 去柜台拿食物（组件重新渲染）

优点：不堵柜台，可以坐下玩手机
```

```javascript
// 传统方式
function TraditionalOrder() {
  const [food, setFood] = useState(null);
  const [waiting, setWaiting] = useState(true);

  useEffect(() => {
    placeOrder().then(food => {
      setFood(food);
      setWaiting(false);
    });
  }, []);

  if (waiting) return <div>站着等...</div>;
  return <div>食物: {food.name}</div>;
}

// use() 方式
function WithUseOrder() {
  const food = use(orderPromise); // 拿号牌，坐下等
  return <div>食物: {food.name}</div>;
}

<Suspense fallback={<div>号牌#42，请等待...</div>}>
  <WithUseOrder />
</Suspense>
```

---

### 类比4：包裹快递 📦

**use() = 快递签收通知**

**传统方式：**
```
1. 网购下单（发起请求）
2. 每10分钟刷新物流（轮询）
3. 一直盯着手机（loading）
4. 到了通知你（Promise resolved）

问题：一直盯着手机，浪费时间
```

**use() 方式：**
```
1. 网购下单（use(deliveryPromise)）
2. 设置"到货提醒"（Suspense）
3. 该干嘛干嘛（组件挂起）
4. 收到短信"包裹到了"（Promise resolved）
5. 去拿包裹（组件重新渲染）

优点：不用一直盯着，该做什么做什么
```

---

### 类比总结表

| React 概念 | 生活场景类比 | 相似点 |
|-----------|-------------|--------|
| Promise | 取钱操作 / 预约借书 / 点餐 | 异步操作 |
| use() | 号牌 / 预约登记 / 到货提醒 | 发起请求，等通知 |
| Suspense fallback | 休息区 / 回家等 / 找座位 | 等待期间做其他事 |
| Promise resolved | 叫号 / 短信通知 / 到货提醒 | 操作完成 |
| 组件重新渲染 | 拿钱 / 拿书 / 拿食物 | 获取结果 |
| ErrorBoundary | 故障提示 / 书丢了通知 | 错误处理 |

---

## 六、【反直觉点】

### 误区1：use() 是一个 Hook ❌

**为什么错？**

use() 不是 Hook，而是一个特殊的 API，它不遵守 Hook 规则。

```javascript
// ❌ 误解：use() 是 Hook，必须在顶层调用
function Component() {
  const data = use(promise);  // 必须在顶层？
}

// ✅ 正确：use() 可以在条件中调用
function Component({ show }) {
  if (show) {
    const data = use(promise);  // ✅ 允许
  }
}
```

**为什么人们容易这样错？**

因为 use() 的用法类似 useContext、useState 等 Hook：
- 都在组件中调用
- 都返回值
- 名字也像 Hook（use 开头）

但实际上：
- Hook 依赖调用顺序（使用 Fiber 链表）
- use() 不依赖顺序（直接读取资源）

**正确理解：**

```javascript
// Hook 规则（useContext、useState 等）
function Hooks() {
  // ✅ 必须在顶层
  const theme = useContext(ThemeContext);
  const [count, setCount] = useState(0);

  // ❌ 不能在条件中
  if (someCondition) {
    const user = useContext(UserContext); // ❌ 错误
  }
}

// use() 规则（不是 Hook）
function UseAPI({ show }) {
  // ✅ 可以在条件中
  if (show) {
    const data = use(dataPromise); // ✅ 允许
  }

  // ✅ 可以在循环中
  for (const promise of promises) {
    const value = use(promise); // ✅ 允许
  }
}
```

**但是：** use() 仍然只能在组件或自定义 Hook 中调用，不能在普通函数中：

```javascript
// ❌ 不能在普通函数中
function fetchData() {
  const data = use(promise); // ❌ 错误
}

// ✅ 可以在组件中
function Component() {
  const data = use(promise); // ✅ 正确
}

// ✅ 可以在自定义 Hook 中
function useData() {
  const data = use(promise); // ✅ 正确
  return data;
}
```

---

### 误区2：Promise 可以在组件内创建 ❌

**为什么错？**

如果在组件内创建 Promise，每次渲染都会创建新的 Promise，导致无限循环。

```javascript
// ❌ 错误：每次渲染都创建新 Promise
function BadComponent() {
  const dataPromise = fetch('/api'); // 每次渲染都 fetch
  const data = use(dataPromise);     // 触发 Suspense → 重新渲染 → 再次 fetch → 无限循环
  return <div>{data}</div>;
}

// ✅ 正确：Promise 在组件外创建
const dataPromise = fetch('/api'); // 只创建1次

function GoodComponent() {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

**为什么人们容易这样错？**

因为传统的数据获取是在 useEffect 中，每次渲染时判断：

```javascript
// 传统方式（useEffect）
function Traditional() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch('/api').then(r => setData(r));  // 只执行1次（依赖数组 []）
  }, []);

  return <div>{data}</div>;
}
```

use() 不同，它会在每次渲染时调用，所以 Promise 必须稳定。

**正确理解：**

```javascript
// ===== 方案1：组件外创建（最简单）=====
const dataPromise = fetch('/api');

function Component() {
  const data = use(dataPromise);
  return <div>{data}</div>;
}

// ===== 方案2：使用 React Cache API =====
import { cache } from 'react';

const fetchData = cache(async (id) => {
  const response = await fetch(`/api/${id}`);
  return response.json();
});

function Component({ id }) {
  const data = use(fetchData(id)); // cache 确保相同 id 只 fetch 1次
  return <div>{data}</div>;
}

// ===== 方案3：Server Component 传递 Promise =====
// app/page.jsx (Server Component)
async function getData() {
  return fetch('/api').then(r => r.json());
}

export default function Page() {
  const dataPromise = getData();
  return <ClientComponent dataPromise={dataPromise} />;
}

// components/ClientComponent.jsx
'use client';
function ClientComponent({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

---

### 误区3：use() 可以在 try-catch 中调用 ❌

**为什么错？**

use() 不能在 try-catch 中调用，错误处理必须使用 ErrorBoundary。

```javascript
// ❌ 错误：use() 不能在 try-catch 中
function BadComponent() {
  let data;
  try {
    data = use(dataPromise); // ❌ 不允许
  } catch (error) {
    return <div>错误: {error.message}</div>;
  }
  return <div>{data}</div>;
}

// ✅ 正确：使用 ErrorBoundary
import { ErrorBoundary } from 'react-error-boundary';

function GoodComponent() {
  const data = use(dataPromise); // ✅ 允许
  return <div>{data}</div>;
}

function App() {
  return (
    <ErrorBoundary fallback={<div>❌ 出错了</div>}>
      <Suspense fallback={<div>⌛ 加载中...</div>}>
        <GoodComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**为什么人们容易这样错？**

因为传统的异步代码用 try-catch 处理错误：

```javascript
// 传统 async/await（可以用 try-catch）
async function fetchData() {
  try {
    const response = await fetch('/api');
    return response.json();
  } catch (error) {
    console.error('错误:', error);
  }
}
```

但 use() 的错误处理机制不同：
- Promise rejected → use() 抛出错误
- ErrorBoundary 捕获错误
- 显示 fallback UI

**正确理解：**

```javascript
// use() 的错误处理流程
async function fetchData() {
  const response = await fetch('/api');
  if (!response.ok) {
    throw new Error('请求失败'); // 抛出错误
  }
  return response.json();
}

const dataPromise = fetchData();

function Component() {
  const data = use(dataPromise);
  // Promise rejected → ErrorBoundary 捕获
  return <div>{data}</div>;
}

<ErrorBoundary
  fallback={({ error }) => <div>❌ {error.message}</div>}
>
  <Suspense fallback={<div>⌛ 加载中...</div>}>
    <Component />
  </Suspense>
</ErrorBoundary>
```

**为什么不能用 try-catch？**

因为 use() 会"抛出" Promise 或错误，中断组件渲染：

```javascript
function use(promise) {
  if (promise.status === 'pending') {
    throw promise; // 抛出 Promise，Suspense 捕获
  }
  if (promise.status === 'rejected') {
    throw promise.reason; // 抛出错误，ErrorBoundary 捕获
  }
  return promise.value;
}
```

如果用 try-catch，会捕获抛出的 Promise，破坏 Suspense 机制。

---

## 七、【实战代码】

（由于篇幅限制，这里提供关键的实战代码场景，完整版在文档中）

### 实战场景1：基础 Promise 解包

```javascript
import { use, Suspense } from 'react';

const fetchMessage = async () => {
  await new Promise(r => setTimeout(r, 2000));
  return { text: 'Hello from use() Hook!' };
};

const messagePromise = fetchMessage();

function Message() {
  const message = use(messagePromise);
  return <p>{message.text}</p>;
}

function App() {
  return (
    <Suspense fallback={<p>⌛ 加载中...</p>}>
      <Message />
    </Suspense>
  );
}
```

### 实战场景2：条件读取 Context

```javascript
import { use, createContext } from 'react';

const ThemeContext = createContext('light');
const UserContext = createContext(null);

function ConditionalContextButton({ showTheme, showUser }) {
  const theme = showTheme ? use(ThemeContext) : 'default';
  const user = showUser ? use(UserContext) : null;

  return (
    <button className={theme}>
      {user ? user.name : 'Guest'}
    </button>
  );
}
```

### 实战场景3：Server Component → Client Component

```javascript
// app/page.jsx (Server Component)
async function getData() {
  const response = await fetch('https://api.github.com/repos/facebook/react');
  return response.json();
}

export default function Page() {
  const dataPromise = getData();
  return <ClientComponent dataPromise={dataPromise} />;
}

// components/ClientComponent.jsx
'use client';
import { use, Suspense } from 'react';

export default function ClientComponent({ dataPromise }) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <DataDisplay dataPromise={dataPromise} />
    </Suspense>
  );
}

function DataDisplay({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data.description}</div>;
}
```

---

## 八、【面试必问】

### 问题1："use() 和 useContext 有什么区别？为什么 use() 可以在条件中调用？"

**普通回答（❌ 不出彩）：**

"use() 可以在 if 语句中调用，useContext 不行。"

**出彩回答（✅ 推荐）：**

> **use() 和 useContext 有三个核心区别：**
>
> 1. **调用位置限制不同**：
>    - `useContext`：必须在组件顶层调用（Hook 规则）
>    - `use()`：可以在条件、循环、switch 中调用
>
>    示例：
>    ```javascript
>    // ❌ useContext 违反规则
>    function Button({ show }) {
>      if (show) {
>        const theme = useContext(ThemeContext); // ❌ 错误
>      }
>    }
>
>    // ✅ use() 允许
>    function Button({ show }) {
>      if (show) {
>        const theme = use(ThemeContext); // ✅ 正确
>      }
>    }
>    ```
>
> 2. **底层实现机制不同**：
>    - `useContext`：依赖 Fiber Hook 链表，必须按顺序调用
>    - `use()`：直接读取资源，不依赖调用顺序
>
>    ```javascript
>    // useContext 实现（简化）
>    function useContext(context) {
>      const hook = mountWorkInProgressHook(); // 依赖 Hook 链表
>      return readContext(context);
>    }
>
>    // use() 实现（简化）
>    function use(resource) {
>      if (resource.$$typeof === REACT_CONTEXT_TYPE) {
>        return readContext(resource); // 直接读取，不依赖链表
>      }
>    }
>    ```
>
> 3. **功能范围不同**：
>    - `useContext`：只能读取 Context
>    - `use()`：可以读取 Context 和 Promise
>
>    ```javascript
>    // useContext：只读 Context
>    const theme = useContext(ThemeContext);
>
>    // use()：读 Context
>    const theme = use(ThemeContext);
>
>    // use()：读 Promise
>    const data = use(dataPromise);
>    ```
>
> **为什么 use() 可以在条件中调用？**
>
> 因为 use() 不是 Hook，而是一个**资源读取 API**：
> - 不维护内部状态（无需 Hook 链表）
> - 每次调用都是独立的资源读取
> - 不依赖组件渲染次数和调用顺序
>
> **在实际工作中的应用：**
>
> - 根据用户角色条件读取配置
> - 根据 feature flag 决定是否读取数据
> - 动态选择 Context 来源

**为什么这个回答出彩？**

1. ✅ **三个维度**：调用位置、底层实现、功能范围
2. ✅ **代码对比**：清晰展示差异
3. ✅ **原理深入**：提到 Fiber Hook 链表
4. ✅ **实际应用**：给出具体使用场景

---

### 问题2："use() 为什么不能在 try-catch 中调用？"

**普通回答（❌ 不出彩）：**

"因为 use() 会抛出 Promise，try-catch 会捕获它，导致 Suspense 不工作。"

**出彩回答（✅ 推荐）：**

> **use() 不能在 try-catch 中调用的原因涉及 Suspense 的工作机制：**
>
> 1. **Suspense 的核心原理**：
>    Suspense 通过捕获抛出的 Promise 来工作：
>    ```javascript
>    // use() 内部实现（简化）
>    function use(promise) {
>      if (promise.status === 'pending') {
>        throw promise; // ⬅️ 关键：抛出 Promise
>      }
>      return promise.value;
>    }
>
>    // React 的 Suspense 边界捕获这个 Promise
>    try {
>      return <Component />; // 渲染组件
>    } catch (thrown) {
>      if (isPromise(thrown)) {
>        // 是 Promise → 显示 fallback
>        return fallback;
>      }
>      throw thrown;
>    }
>    ```
>
> 2. **try-catch 会破坏这个机制**：
>    ```javascript
>    // ❌ 错误：try-catch 捕获了 Promise
>    function BadComponent() {
>      try {
>        const data = use(dataPromise);
>        // use() 抛出 Promise
>      } catch (error) {
>        // 这里捕获了 Promise（不是错误）
>        // Suspense 收不到 Promise，无法工作
>        return <div>错误</div>;
>      }
>    }
>    ```
>
> 3. **正确的错误处理方式**：
>    使用 ErrorBoundary：
>    ```javascript
>    <ErrorBoundary fallback={<div>❌ 出错</div>}>
>      <Suspense fallback={<div>⌛ 加载中</div>}>
>        <Component />
>      </Suspense>
>    </ErrorBoundary>
>
>    function Component() {
>      const data = use(dataPromise);
>      // Promise pending → 抛出 Promise → Suspense 捕获
>      // Promise rejected → 抛出 Error → ErrorBoundary 捕获
>      return <div>{data}</div>;
>    }
>    ```
>
> **完整的错误处理流程：**
>
> ```
> use(dataPromise) 被调用
>   ↓
> Promise pending？
>   → 是 → throw promise → Suspense 捕获 → 显示 fallback
>   ↓
> Promise rejected？
>   → 是 → throw error → ErrorBoundary 捕获 → 显示错误 UI
>   ↓
> Promise fulfilled
>   → 返回 promise.value → 正常渲染
> ```
>
> **与 async/await 的对比：**
>
> ```javascript
> // async/await：可以用 try-catch
> async function fetchData() {
>   try {
>     const response = await fetch('/api');
>     return response.json();
>   } catch (error) {
>     // ✅ 这里可以捕获
>   }
> }
>
> // use()：不能用 try-catch
> function Component() {
>   const data = use(dataPromise);
>   // ❌ 不能包在 try-catch 中
> }
> ```
>
> **在实际工作中的应用：**
>
> - 用 ErrorBoundary 统一处理组件树的错误
> - 区分 loading 状态（Suspense）和 error 状态（ErrorBoundary）
> - 提供优雅的降级体验

**为什么这个回答出彩？**

1. ✅ **原理深入**：解释了 Suspense 的工作机制
2. ✅ **代码演示**：展示了错误和正确的做法
3. ✅ **流程图**：清晰的错误处理流程
4. ✅ **对比 async/await**：帮助理解差异

---

## 九、【化骨绵掌】

（10个2分钟知识卡片）

### 卡片1：直觉理解 - use() 是什么？ 🎯

**一句话：** use() 是读取资源（Promise/Context）的 API，Promise pending 时组件挂起，resolved 后返回值。

**举例：**

```javascript
const dataPromise = fetch('/api').then(r => r.json());

function Component() {
  const data = use(dataPromise); // 解包 Promise
  return <div>{data.message}</div>;
}

<Suspense fallback={<div>Loading...</div>}>
  <Component />
</Suspense>
```

**应用：** 简化异步数据处理，无需手动管理 loading 状态

---

### 卡片2：不是 Hook 🚫

**一句话：** use() 不是 Hook，不遵守 Hook 规则，可以在条件、循环中调用。

**对比：**

```javascript
// ❌ Hook 规则
if (show) {
  const theme = useContext(ThemeContext); // 违反规则
}

// ✅ use() 规则
if (show) {
  const theme = use(ThemeContext); // 允许
}
```

**应用：** 更灵活的 Context 读取

---

### 卡片3：Suspense 集成 ⏸️

**一句话：** use() 与 Suspense 深度集成，Promise pending 时触发 Suspense fallback。

**流程：**

```
use(promise)
  → pending → throw promise
  → Suspense 捕获 → 显示 fallback
  → resolved → 重新渲染 → 返回值
```

**应用：** 无需手动管理 loading，Suspense 自动显示加载状态

---

### 卡片4：ErrorBoundary 错误处理 ⚠️

**一句话：** Promise rejected 时，use() 抛出错误，ErrorBoundary 捕获并显示错误 UI。

**代码：**

```javascript
<ErrorBoundary fallback={<div>❌ 错误</div>}>
  <Suspense fallback={<div>⌛ 加载</div>}>
    <Component />
  </Suspense>
</ErrorBoundary>
```

**应用：** 统一处理异步错误，提供优雅的降级体验

---

### 卡片5：Promise 必须在组件外创建 📌

**一句话：** Promise 必须在组件外或通过 props 传递，否则会无限循环。

**示例：**

```javascript
// ❌ 错误：每次渲染都创建
function Bad() {
  const data = use(fetch('/api')); // 无限循环
}

// ✅ 正确：组件外创建
const dataPromise = fetch('/api');
function Good() {
  const data = use(dataPromise);
}
```

**应用：** 使用 React Cache API 或 Server Component 传递 Promise

---

### 卡片6：条件读取 Context 🔀

**一句话：** use() 可以根据条件选择读取不同的 Context。

**代码：**

```javascript
function Button({ role }) {
  const config = role === 'admin'
    ? use(AdminConfigContext)
    : use(UserConfigContext);

  return <button>{config.title}</button>;
}
```

**应用：** 根据用户角色、feature flag 等动态选择配置

---

### 卡片7：Server Component 传递 Promise 🌐

**一句话：** Server Component 可以创建 Promise 并传递给 Client Component。

**示例：**

```javascript
// Server Component
export default function Page() {
  const dataPromise = fetch('/api').then(r => r.json());
  return <ClientComponent dataPromise={dataPromise} />;
}

// Client Component
'use client';
function ClientComponent({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

**应用：** Next.js App Router 的流式渲染

---

### 卡片8：不能在 try-catch 中 ⛔

**一句话：** use() 不能在 try-catch 中，会破坏 Suspense 机制。

**原因：**

use() 抛出 Promise → Suspense 捕获
try-catch 捕获 Promise → Suspense 收不到 → 不工作

**应用：** 必须使用 ErrorBoundary 处理错误

---

### 卡片9：与 async/await 的区别 🔄

**一句话：** async/await 暂停函数执行，use() 挂起组件渲染。

**对比：**

```javascript
// async/await：函数级别
async function fetch() {
  const data = await api(); // 函数暂停
  return data;
}

// use()：组件级别
function Component() {
  const data = use(promise); // 组件挂起
  return <div>{data}</div>;
}
```

**应用：** use() 专为 React 组件设计

---

### 卡片10：最佳实践 ✅

**一句话：** Promise 在组件外创建，用 Suspense 包裹，用 ErrorBoundary 处理错误。

**完整模板：**

```javascript
// 1. 组件外创建 Promise
const dataPromise = fetch('/api').then(r => r.json());

// 2. Suspense + ErrorBoundary
<ErrorBoundary fallback={<div>❌ 错误</div>}>
  <Suspense fallback={<div>⌛ 加载</div>}>
    <Component />
  </Suspense>
</ErrorBoundary>

// 3. use() 解包
function Component() {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

**应用：** 标准的 use() 使用模式

---

## 十、【一句话总结】

**use() 是 React 19 的资源读取 API，通过抛出 pending 的 Promise 触发 Suspense 挂起，Promise resolved 后重新渲染并返回值，支持在条件语句和循环中调用突破传统 Hook 限制，同时读取 Promise 和 Context 提供统一接口，与 Suspense 和 ErrorBoundary 深度集成实现声明式异步数据处理，特别适用于 Server Component 向 Client Component 传递 Promise 实现流式渲染，是 React 并发架构和 Suspense 机制的高级应用。**

---

## 附录

### 学习检查清单

#### 基础理解
- [ ] 理解 use() 的定义：读取资源（Promise/Context）
- [ ] 知道 use() 不是 Hook，可以条件调用
- [ ] 理解 Promise pending 时组件挂起
- [ ] 掌握 Suspense 和 ErrorBoundary 的协作

#### 实战应用
- [ ] 会使用 use() 解包 Promise
- [ ] 会条件读取 Context
- [ ] 知道 Promise 必须在组件外创建
- [ ] 会使用 ErrorBoundary 处理错误

#### 进阶理解
- [ ] 理解 use() 抛出 Promise 的机制
- [ ] 了解 Server Component 传递 Promise
- [ ] 知道为什么不能在 try-catch 中使用
- [ ] 理解与 async/await 的区别

#### 面试准备
- [ ] 能够对比 use() vs useContext
- [ ] 能够解释为什么不能在 try-catch 中
- [ ] 能够说明 Suspense 的工作原理
- [ ] 能够举例 Server/Client Component 协作

---

### 下一步学习建议

**基础路径（适合初学者）：**
1. 复习 Suspense 的基本用法
2. 学习 ErrorBoundary 的使用
3. 了解 Promise 的状态（pending/fulfilled/rejected）
4. 学习 React Cache API

**进阶路径（适合深入学习）：**
1. 研究 Suspense 的底层实现（抛出 Promise）
2. 学习 Server Component 和 Client Component
3. 阅读 React 源码：`ReactFiberHooks.js` 中的 use 实现
4. 理解流式渲染（Streaming SSR）

**实战建议：**
1. 使用 use() 重构数据获取逻辑
2. 在 Next.js App Router 中使用 Server Component
3. 实现条件性 Context 读取
4. 配合 Suspense 和 ErrorBoundary 提升用户体验

---

### 参考资源

#### 官方文档
- [use API Reference](https://react.dev/reference/react/use)
- [React v19 Release Notes](https://react.dev/blog/2024/12/05/react-19)
- [Suspense Reference](https://react.dev/reference/react/Suspense)
- [Server Components](https://react.dev/reference/rsc/server-components)

#### 社区文章
- [React 19 New Hooks - FreeCodeCamp](https://www.freecodecamp.org/news/react-19-new-hooks-explained-with-examples/)
- [use() Hook Guide - DEV Community](https://dev.to/hasunnilupul/react-19s-use-hook-a-practical-guide-with-examples-beb)
- [React Suspense Deep Dive](https://dev.to/a1guy/react-19-suspense-deep-dive-data-fetching-streaming-and-error-handling-like-a-pro-3k74)
- [Replacing throw with use() - Peter Kellner](https://peterkellner.net/2024/12/31/replacing-legacy-throw-in-react-19-with-suspense-and-use/)

#### 相关知识点（本项目其他文档）
- `atom/08_React19新特性/01_自动批处理增强.md` - 理解批处理机制
- `atom/08_React19新特性/02_Actions与useTransition.md` - 理解 Transition 和 Actions
- `atom/03_Fiber架构/01_Fiber节点结构.md` - 理解 Fiber 架构
- `atom/07_Hooks实现/01_Hooks链表.md` - 理解 Hook 的实现原理

---

**版本：** v1.0
**创建日期：** 2025-12-07
**适用 React 版本：** React 19.0+
**作者：** Claude Code
**项目：** React19 源码学习
