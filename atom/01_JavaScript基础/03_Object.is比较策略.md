# Object.is 比较策略

## 一、【30字核心】

**Object.is 是严格相等的增强版，正确处理 NaN 和 +0/-0，是 React 依赖比较和浅比较优化的核心算法。**

---

## 二、【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Object.is 的第一性原理 🎯

#### 1. 最基础的定义

**Object.is = SameValue 算法的 JavaScript 实现**

仅此而已！没有更基础的了。

Object.is 判断两个值是否"完全相同"，这里的"相同"比 `===` 更严格、更符合直觉：
- `NaN` 等于 `NaN`（`===` 认为不等）
- `+0` 不等于 `-0`（`===` 认为相等）

#### 2. 为什么需要 Object.is？

**核心问题：=== 对某些值的处理不符合预期**

JavaScript 的 `===` 严格相等运算符有两个反直觉的行为：

```javascript
// 问题1：NaN 不等于自己
console.log(NaN === NaN); // false（违反直觉）

// 问题2：+0 等于 -0
console.log(+0 === -0);   // true（在某些场景下不合理）
```

这导致实际开发中的问题：

```javascript
// 检查数组是否包含 NaN
const arr = [1, NaN, 3];
console.log(arr.indexOf(NaN));  // -1（找不到！）
console.log(arr.includes(NaN)); // true（使用 SameValueZero 算法）

// 依赖追踪中的 NaN
const deps1 = [1, NaN];
const deps2 = [1, NaN];
// 如果用 ===，会认为 NaN !== NaN，导致误判
```

#### 3. Object.is 的三层价值

##### 价值1：语义正确的相等判断 ✅

```javascript
// === 的反直觉行为
console.log(NaN === NaN);     // false
console.log(+0 === -0);       // true

// Object.is 的直觉行为
console.log(Object.is(NaN, NaN));   // true（NaN 就是 NaN）
console.log(Object.is(+0, -0));     // false（+0 和 -0 不同）

// 实际应用：数学计算
function divide(a, b) {
  const result = a / b;

  if (Object.is(result, -0)) {
    console.log('负零：从负数方向趋近零');
  } else if (Object.is(result, +0)) {
    console.log('正零：从正数方向趋近零');
  }
}

divide(1, Infinity);   // 正零
divide(-1, Infinity);  // 负零
```

##### 价值2：依赖追踪的基础 🔍

```javascript
// React useEffect 依赖比较
function Component({ userId }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    // 如果 userId 是 NaN，用 === 会导致无限循环
    // 因为 [NaN] !== [NaN]（每次都不相等）

    // React 使用 Object.is 比较依赖
    fetchData(userId).then(setData);
  }, [userId]); // Object.is(prevUserId, userId)

  return <div>{data}</div>;
}
```

##### 价值3：性能优化的判断依据 ⚡

```javascript
// React.memo 的浅比较
const MemoComponent = React.memo(function MyComponent({ value }) {
  console.log('渲染');
  return <div>{value}</div>;
});

function App() {
  const [count, setCount] = useState(0);

  // React 用 Object.is 比较 props
  // 如果 value 没变，MemoComponent 不会重新渲染
  return <MemoComponent value={count} />;
}
```

#### 4. 从第一性原理推导 React 实现

**推理链：**

```
1. React 需要判断状态/props 是否变化
   ↓
2. 变化了才重新渲染（性能优化）
   ↓
3. 需要一个可靠的相等性判断算法
   ↓
4. === 对 NaN 和 ±0 处理不当
   ↓
5. ES6 提供了 Object.is（SameValue 算法）
   ↓
6. Object.is 语义正确、性能好
   ↓
7. React 使用 Object.is 作为基础比较函数
   ↓
8. Hook 依赖比较、props 浅比较都基于它
```

**为什么 React 选择 Object.is？**

- **标准化**：ES6 标准，语义明确
- **正确性**：处理 NaN 和 ±0 符合预期
- **性能**：与 === 性能相同（引擎优化）
- **一致性**：在所有场景下行为一致

#### 5. 一句话总结第一性原理

**Object.is 实现了 ECMAScript 的 SameValue 算法，提供了比 === 更语义化的相等性判断，解决了 NaN 自比较和零值符号区分的问题，是 JavaScript 中最严格的相等性判断。**

---

## 三、【3个核心概念】

### 核心概念1：SameValue 算法 📏

**SameValue 是 ECMAScript 规范定义的相等性语义，Object.is 是其在 JavaScript 中的实现。**

```javascript
// SameValue 算法的判断规则

// 1. 类型不同 → false
console.log(Object.is(1, '1'));     // false
console.log(Object.is(null, undefined)); // false

// 2. 都是 NaN → true（这是与 === 的关键区别）
console.log(Object.is(NaN, NaN));   // true
console.log(NaN === NaN);           // false

// 3. 都是 +0 或都是 -0 → true
// 一个是 +0，一个是 -0 → false（这也是与 === 的区别）
console.log(Object.is(+0, +0));     // true
console.log(Object.is(-0, -0));     // true
console.log(Object.is(+0, -0));     // false
console.log(+0 === -0);             // true

// 4. 其他情况同 ===
console.log(Object.is(1, 1));       // true
console.log(Object.is('a', 'a'));   // true
console.log(Object.is(true, true)); // true
const obj = {};
console.log(Object.is(obj, obj));   // true（同一引用）
```

**SameValue 的形式化定义：**

```javascript
// ECMAScript 规范的伪代码
function SameValue(x, y) {
  // 1. 类型不同
  if (Type(x) !== Type(y)) {
    return false;
  }

  // 2. 都是 Number 类型
  if (Type(x) === Number) {
    // 2.1 都是 NaN
    if (isNaN(x) && isNaN(y)) {
      return true;
    }
    // 2.2 一个是 +0，一个是 -0
    if (x === 0 && y === 0) {
      return 1 / x === 1 / y; // +0: Infinity, -0: -Infinity
    }
    // 2.3 其他数字
    return x === y;
  }

  // 3. 其他类型使用 ===
  return x === y;
}
```

**在 React 源码/开发中的应用：**

React 的 `objectIs` polyfill：

```javascript
// packages/shared/objectIs.js
const objectIs =
  typeof Object.is === 'function'
    ? Object.is
    : function(x, y) {
        // SameValue 算法
        if (x === y) {
          // +0 vs -0
          return x !== 0 || 1 / x === 1 / y;
        } else {
          // NaN vs NaN
          return x !== x && y !== y;
        }
      };

export default objectIs;
```

---

### 核心概念2：NaN 特殊处理 🔢

**NaN（Not-a-Number）是 JavaScript 中唯一不等于自身的值，Object.is 纠正了这一反直觉行为。**

```javascript
// NaN 的产生方式
console.log(0 / 0);           // NaN
console.log(Math.sqrt(-1));   // NaN
console.log(parseInt('abc')); // NaN
console.log(Number('xyz'));   // NaN

// === 无法判断 NaN
console.log(NaN === NaN);     // false

// 传统的 NaN 检测方法
console.log(isNaN(NaN));      // true
console.log(Number.isNaN(NaN)); // true
console.log(NaN !== NaN);     // true（利用不等于自身的特性）

// Object.is 的直觉方式
console.log(Object.is(NaN, NaN)); // true
```

**为什么 === 认为 NaN 不等于 NaN？**

这是 IEEE 754 浮点数标准的规定，目的是让 NaN 代表"未知值"：

```javascript
// NaN 代表"未知值"
const unknown1 = 0 / 0; // 未知
const unknown2 = 0 / 0; // 未知

// 两个未知值无法判断是否相等
console.log(unknown1 === unknown2); // false（逻辑上合理）
```

**Object.is 的选择：**

Object.is 认为"NaN 在值层面上是相同的"，更适合实际编程需要。

**在 React 源码/开发中的应用：**

```javascript
// useEffect 依赖比较
function Component() {
  const [value, setValue] = useState(NaN);

  useEffect(() => {
    console.log('Effect 执行');
  }, [value]); // Object.is(prevValue, value)

  // 点击按钮设置 NaN
  function handleClick() {
    setValue(NaN);
  }

  // 如果用 ===：NaN !== NaN，会无限循环
  // 使用 Object.is：NaN === NaN，不会重新执行
  return <button onClick={handleClick}>Set NaN</button>;
}
```

**NaN 比较的三种方法：**

| 方法 | 语法 | 结果 | 适用场景 |
|------|------|------|---------|
| `===` | `NaN === NaN` | `false` | 不适合判断 NaN |
| `isNaN()` | `isNaN(value)` | `true` | 会将非数字转换为 NaN |
| `Number.isNaN()` | `Number.isNaN(value)` | `true` | 严格判断是否为 NaN |
| `Object.is()` | `Object.is(NaN, NaN)` | `true` | 值相等性判断 |

---

### 核心概念3：零值区分（+0 vs -0） ➕➖

**JavaScript 中存在 +0 和 -0，它们在 === 下相等，但 Object.is 可以区分。**

```javascript
// +0 和 -0 的产生
const positiveZero = 0;
const negativeZero = -0;

console.log(positiveZero);    // 0
console.log(negativeZero);    // 0（显示相同）

// === 认为相等
console.log(+0 === -0);       // true

// Object.is 认为不等
console.log(Object.is(+0, -0)); // false
```

**如何区分 +0 和 -0？**

```javascript
// 方法1：除法
console.log(1 / +0);  // Infinity
console.log(1 / -0);  // -Infinity

// 方法2：Object.is
console.log(Object.is(value, +0)); // true 表示是 +0
console.log(Object.is(value, -0)); // true 表示是 -0

// 方法3：检查符号
function isNegativeZero(value) {
  return value === 0 && 1 / value < 0;
}

console.log(isNegativeZero(-0));  // true
console.log(isNegativeZero(+0));  // false
```

**±0 在实际中的应用：**

```javascript
// 1. 数学运算的方向性
function getDirection(velocity) {
  if (Object.is(velocity, +0)) {
    return '从正方向趋近零';
  } else if (Object.is(velocity, -0)) {
    return '从负方向趋近零';
  } else if (velocity > 0) {
    return '正方向';
  } else {
    return '负方向';
  }
}

console.log(getDirection(+0)); // 从正方向趋近零
console.log(getDirection(-0)); // 从负方向趋近零

// 2. 图形学中的符号
const angle = -0; // 表示逆时针方向的极小角度

// 3. 金融计算中的舍入方向
const roundedLoss = -0; // 亏损舍入到零
```

**在 React 源码/开发中的应用：**

虽然 React 日常开发中很少遇到 ±0 的区分，但使用 Object.is 确保了行为的标准化和一致性：

```javascript
// React 的 Hook 依赖比较
function areHookInputsEqual(nextDeps, prevDeps) {
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (objectIs(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}

// 即使极少遇到 ±0，使用标准算法也能保证正确性
```

---

## 四、【最小可用】

掌握以下内容，就能理解 React 源码核心：

### 4.1 Object.is vs === 的差异

```javascript
// 只有两个差异点

// 差异1：NaN
console.log(NaN === NaN);           // false
console.log(Object.is(NaN, NaN));   // true

// 差异2：±0
console.log(+0 === -0);             // true
console.log(Object.is(+0, -0));     // false

// 其他所有情况都相同
console.log(1 === 1);               // true
console.log(Object.is(1, 1));       // true

console.log('a' === 'a');           // true
console.log(Object.is('a', 'a'));   // true

const obj = {};
console.log(obj === obj);           // true
console.log(Object.is(obj, obj));   // true

console.log({} === {});             // false（不同引用）
console.log(Object.is({}, {}));     // false（不同引用）
```

### 4.2 React 为何使用 Object.is

```javascript
// React 的选择：Object.is

// 1. 依赖比较（useEffect、useMemo、useCallback）
useEffect(() => {
  console.log('执行');
}, [dep1, dep2]); // 用 Object.is 逐个比较

// 2. props 浅比较（React.memo）
const Memo = React.memo(Component); // 用 Object.is 比较 props

// 3. 状态比较（useState、useReducer）
setState(newValue); // 用 Object.is 判断是否需要更新

// 为什么不用 ===？
// ❌ 如果 dep 是 NaN，=== 会导致无限循环
// ✅ Object.is 正确处理 NaN
```

### 4.3 依赖数组比较原理

```javascript
// React Hook 依赖比较的实现
function areHookInputsEqual(nextDeps, prevDeps) {
  // 没有依赖数组，总是执行
  if (prevDeps === null) {
    return false;
  }

  // 逐个比较依赖项
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    // 使用 Object.is
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue; // 相等，检查下一项
    }
    return false; // 不相等，需要执行
  }

  return true; // 所有依赖都相等
}

// 使用示例
const prevDeps = [1, 'a', NaN];
const nextDeps = [1, 'a', NaN];

console.log(areHookInputsEqual(nextDeps, prevDeps)); // true
// 如果用 ===，NaN !== NaN，会返回 false
```

### 4.4 浅比较的实现

```javascript
// React 的浅比较实现（简化版）
function shallowEqual(objA, objB) {
  // 1. 先用 Object.is 快速判断
  if (Object.is(objA, objB)) {
    return true;
  }

  // 2. 排除非对象
  if (
    typeof objA !== 'object' ||
    objA === null ||
    typeof objB !== 'object' ||
    objB === null
  ) {
    return false;
  }

  // 3. 比较键的数量
  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) {
    return false;
  }

  // 4. 逐个比较键值（浅比较，只比较一层）
  for (let i = 0; i < keysA.length; i++) {
    const key = keysA[i];

    if (
      !Object.prototype.hasOwnProperty.call(objB, key) ||
      !Object.is(objA[key], objB[key]) // 使用 Object.is
    ) {
      return false;
    }
  }

  return true;
}

// 使用示例
const obj1 = { a: 1, b: NaN };
const obj2 = { a: 1, b: NaN };

console.log(shallowEqual(obj1, obj2)); // true
// 如果用 ===，NaN !== NaN，会返回 false
```

**这些知识足以：**
- 理解 React Hook 的依赖比较逻辑
- 正确使用 useEffect/useMemo/useCallback 的依赖数组
- 理解 React.memo 的浅比较行为
- 避免因 NaN 导致的无限循环

---

## 五、【1个类比】

### 类比1：Object.is = 精密天平 ⚖️

**解释相似性：**

比较相等性就像称重：

1. **普通秤（===）**
   - 大部分情况都准确
   - 但对特殊物品有误差
   - NaN：称不出重量，每次都显示"错误"
   - ±0：无法区分正负，都显示"0.00kg"

2. **精密天平（Object.is）**
   - 在普通秤的基础上升级
   - NaN：能识别"这是无法测量的物品"
   - ±0：能区分"从上面放下"（+0）和"从下面托起"（-0）

**举例：**

```javascript
// 普通秤（===）
const scale = (a, b) => a === b;

scale(NaN, NaN);   // false（秤：两次都测不出来，所以不相等？）
scale(+0, -0);     // true（秤：都是0kg，相等）

// 精密天平（Object.is）
const precisionScale = (a, b) => Object.is(a, b);

precisionScale(NaN, NaN);   // true（天平：都是无法测量，本质相同）
precisionScale(+0, -0);     // false（天平：一个向上，一个向下，不同）
```

---

### 类比2：React 依赖比较 = 监控摄像头 📹

**解释相似性：**

React 检查依赖是否变化就像监控摄像头对比画面：

1. **每次渲染拍一张照片**
   ```javascript
   useEffect(() => {
     // 副作用
   }, [dep1, dep2]); // 拍照：记录 dep1 和 dep2
   ```

2. **下次渲染时对比照片**
   ```javascript
   // 对比：Object.is(旧dep1, 新dep1) && Object.is(旧dep2, 新dep2)
   ```

3. **发现变化就执行副作用**
   ```javascript
   if (dependenciesChanged) {
     执行副作用();
   }
   ```

**举例：**

```javascript
// 监控系统
function SecurityCamera() {
  const [motion, setMotion] = useState(0);

  useEffect(() => {
    // Object.is 对比前后两张照片
    console.log('检测到变化，触发警报');
  }, [motion]); // 每次 motion 变化就重新检测

  // 如果用 ===，motion 是 NaN 时会误报
  // Object.is(NaN, NaN) === true，不会误报
}
```

---

### 类比3：浅比较 = 检查快递包装 📦

**解释相似性：**

浅比较就像检查快递是否变了：

1. **只检查外包装**
   - 箱子大小
   - 箱子上的标签
   - 箱子里物品的引用（不打开看）

2. **不检查内部内容**
   - 不打开盒子看里面
   - 只要盒子是同一个就认为没变

**举例：**

```javascript
// 快递检查系统
const package1 = {
  size: 'large',
  label: 'fragile',
  items: [{ name: 'vase' }] // 物品数组
};

const package2 = {
  size: 'large',
  label: 'fragile',
  items: [{ name: 'vase' }] // 新数组，内容相同
};

// 浅比较
shallowEqual(package1, package2); // false
// 原因：items 是不同的数组引用（不同的盒子）

// 深比较（需要打开盒子检查）
deepEqual(package1, package2); // true
// 原因：items 内容相同
```

---

### 类比总结表

| React 概念 | 生活场景类比 | 相似点 |
|-----------|------------|--------|
| `===` | 普通秤 | 大部分情况准确，特殊情况有误差 |
| `Object.is` | 精密天平 | 更准确，能处理特殊情况 |
| 依赖比较 | 监控摄像头 | 对比前后状态，检测变化 |
| 浅比较 | 检查快递包装 | 只看外层，不看内部 |
| NaN | 无法测量的物品 | 本质相同但 === 认为不同 |
| ±0 | 从上/从下放置 | 方向不同但 === 认为相同 |

---

## 六、【反直觉点】

### 误区1：Object.is 比 === 更慢 ❌

**为什么错？**

Object.is 和 === 的性能几乎相同，现代 JavaScript 引擎对两者都有高度优化。

**为什么人们容易这样错？**

因为 Object.is 看起来是"函数调用"，而 === 是"运算符"，直觉上认为运算符更快。

**正确理解：**

```javascript
// 性能测试
function testPerformance() {
  const iterations = 10000000;

  // 测试 ===
  console.time('===');
  for (let i = 0; i < iterations; i++) {
    let result = i === i;
  }
  console.timeEnd('===');

  // 测试 Object.is
  console.time('Object.is');
  for (let i = 0; i < iterations; i++) {
    let result = Object.is(i, i);
  }
  console.timeEnd('Object.is');
}

testPerformance();
// 结果：两者耗时几乎相同（差异在 ±5% 以内）
```

**引擎优化：**

```javascript
// V8 引擎会将 Object.is 内联（inline）优化
// 编译后的机器码几乎与 === 相同

// Object.is 源码（V8 简化版）
function ObjectIs(x, y) {
  if (x === y) {
    // 处理 +0 vs -0
    return x !== 0 || 1 / x === 1 / y;
  }
  // 处理 NaN
  return x !== x && y !== y;
}

// 引擎会将这些逻辑直接编译成机器码
// 不会有函数调用的开销
```

**React 的选择：**

React 优先考虑**正确性**而非微小的性能差异。Object.is 的语义更清晰，值得使用。

---

### 误区2：React 使用深比较 ❌

**为什么错？**

React 在大多数情况下使用**浅比较**（基于 Object.is），而不是深比较。

**为什么人们容易这样错？**

因为听说"React 会自动检测变化"，误以为 React 会深入对象内部检查每个属性。

**正确理解：**

```javascript
// React.memo 只做浅比较
const Component = React.memo(function({ obj }) {
  console.log('渲染');
  return <div>{obj.name}</div>;
});

function App() {
  const [count, setCount] = useState(0);

  const obj = { name: 'Alice', age: 25 };

  return (
    <>
      <Component obj={obj} />
      <button onClick={() => setCount(count + 1)}>
        Re-render
      </button>
    </>
  );
}

// 每次点击按钮，Component 都会重新渲染
// 原因：obj 是新对象，引用不同（浅比较失败）

// 解决方案1：记忆化对象
const obj = useMemo(() => ({ name: 'Alice', age: 25 }), []);

// 解决方案2：自定义比较函数
const Component = React.memo(MyComponent, (prevProps, nextProps) => {
  // 自定义深比较
  return deepEqual(prevProps.obj, nextProps.obj);
});
```

**浅比较 vs 深比较：**

| 特性 | 浅比较 | 深比较 |
|-----|--------|--------|
| 检查层级 | 只检查第一层 | 递归检查所有层 |
| 性能 | O(n)，n 是键数量 | O(n*m)，n 是对象数，m 是平均深度 |
| React 默认 | ✅ 是 | ❌ 否 |
| 适用场景 | props、state | 特殊场景（自定义） |

```javascript
// 浅比较实现
function shallowEqual(a, b) {
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) return false;

  for (let key of keysA) {
    if (!Object.is(a[key], b[key])) { // 只比较一层
      return false;
    }
  }

  return true;
}

// 深比较实现
function deepEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (typeof a !== 'object' || typeof b !== 'object') return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) return false;

  for (let key of keysA) {
    if (!deepEqual(a[key], b[key])) { // 递归比较
      return false;
    }
  }

  return true;
}
```

---

### 误区3：对象内容相同就不会重新渲染 ❌

**为什么错？**

React 比较的是对象**引用**，而不是对象**内容**。即使内容相同，引用不同也会重新渲染。

**为什么人们容易这样错？**

因为在其他框架（如 Vue）中，响应式系统会深入跟踪对象属性，导致混淆 React 的行为。

**正确理解：**

```javascript
// ❌ 错误示例：每次都创建新对象
function App() {
  const [count, setCount] = useState(0);

  const user = { name: 'Alice', age: 25 }; // 每次渲染都是新对象

  return <UserProfile user={user} />;
}

const UserProfile = React.memo(({ user }) => {
  console.log('UserProfile 渲染');
  return <div>{user.name}</div>;
});

// 每次 App 重新渲染，UserProfile 也会重新渲染
// 原因：user 是新对象，Object.is(旧user, 新user) === false

// ✅ 正确示例：记忆化对象
function App() {
  const [count, setCount] = useState(0);

  const user = useMemo(() => ({ name: 'Alice', age: 25 }), []);
  // user 引用不变

  return <UserProfile user={user} />;
}

// 现在 UserProfile 只会渲染一次
// 原因：user 引用相同，Object.is(旧user, 新user) === true
```

**引用 vs 内容：**

```javascript
// 两个内容相同的对象
const obj1 = { name: 'Alice' };
const obj2 = { name: 'Alice' };

// === 和 Object.is 都比较引用
console.log(obj1 === obj2);           // false
console.log(Object.is(obj1, obj2));   // false

// 只有同一个对象才相等
console.log(obj1 === obj1);           // true
console.log(Object.is(obj1, obj1));   // true

// React 的浅比较
shallowEqual(obj1, obj2); // true（会逐个比较属性）
// 但 props 比较用的是 Object.is(obj1, obj2)，返回 false
```

**最佳实践：**

```javascript
// 1. 对于不变的对象，提取到组件外
const CONSTANT_USER = { name: 'Alice', age: 25 };

function App() {
  return <UserProfile user={CONSTANT_USER} />;
}

// 2. 对于可变的对象，使用 useMemo
function App() {
  const [age, setAge] = useState(25);

  const user = useMemo(() => ({ name: 'Alice', age }), [age]);
  // age 变化时才创建新对象

  return <UserProfile user={user} />;
}

// 3. 避免在 render 中创建对象
// ❌ Bad
<Component config={{ theme: 'dark' }} />

// ✅ Good
const config = useMemo(() => ({ theme: 'dark' }), []);
<Component config={config} />
```

---

## 七、【实战代码】

### 基础实现（简化版）

```javascript
// ===== 1. 场景1：Object.is vs === =====
console.log("=== 场景1：Object.is vs === ===\n");

console.log("【NaN 比较】");
console.log(`NaN === NaN: ${NaN === NaN}`);           // false
console.log(`Object.is(NaN, NaN): ${Object.is(NaN, NaN)}`); // true

console.log("\n【±0 比较】");
console.log(`+0 === -0: ${+0 === -0}`);               // true
console.log(`Object.is(+0, -0): ${Object.is(+0, -0)}`); // false
console.log(`Object.is(+0, +0): ${Object.is(+0, +0)}`); // true
console.log(`Object.is(-0, -0): ${Object.is(-0, -0)}`); // true

console.log("\n【其他值比较】");
console.log(`1 === 1: ${1 === 1}`);                   // true
console.log(`Object.is(1, 1): ${Object.is(1, 1)}`);   // true

const obj = {};
console.log(`obj === obj: ${obj === obj}`);           // true
console.log(`Object.is(obj, obj): ${Object.is(obj, obj)}`); // true

console.log(`{} === {}: ${{} === {}}`);               // false
console.log(`Object.is({}, {}): ${Object.is({}, {})}`); // false

// ===== 2. 场景2：区分 +0 和 -0 =====
console.log("\n=== 场景2：区分 +0 和 -0 ===\n");

function identifyZero(value) {
  if (Object.is(value, +0)) {
    return '+0（正零）';
  } else if (Object.is(value, -0)) {
    return '-0（负零）';
  } else {
    return `${value}（非零）`;
  }
}

console.log(identifyZero(0));       // +0（正零）
console.log(identifyZero(-0));      // -0（负零）
console.log(identifyZero(1));       // 1（非零）

// 用除法区分
console.log(`\n1 / +0 = ${1 / +0}`);  // Infinity
console.log(`1 / -0 = ${1 / -0}`);    // -Infinity

// ===== 3. 场景3：React 依赖比较实现 =====
console.log("\n=== 场景3：React 依赖比较实现 ===\n");

function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) {
    console.log('首次渲染，没有旧依赖');
    return false;
  }

  console.log('比较依赖项：');
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    const isEqual = Object.is(nextDeps[i], prevDeps[i]);
    console.log(`  [${i}] ${prevDeps[i]} vs ${nextDeps[i]}: ${isEqual}`);

    if (isEqual) {
      continue;
    }
    return false;
  }

  console.log('所有依赖相等 ✅');
  return true;
}

// 测试1：正常值
console.log('\n【测试1：正常值】');
const deps1_prev = [1, 'a', true];
const deps1_next = [1, 'a', true];
console.log(`结果：${areHookInputsEqual(deps1_next, deps1_prev)}`);

// 测试2：包含 NaN
console.log('\n【测试2：包含 NaN】');
const deps2_prev = [1, NaN, 'a'];
const deps2_next = [1, NaN, 'a'];
console.log(`结果：${areHookInputsEqual(deps2_next, deps2_prev)}`);

// 测试3：值改变
console.log('\n【测试3：值改变】');
const deps3_prev = [1, 2, 3];
const deps3_next = [1, 999, 3];
console.log(`结果：${areHookInputsEqual(deps3_next, deps3_prev)}`);

// ===== 4. 场景4：浅比较实现 =====
console.log("\n=== 场景4：浅比较实现 ===\n");

function shallowEqual(objA, objB) {
  // 1. Object.is 快速判断
  if (Object.is(objA, objB)) {
    console.log('引用相同 ✅');
    return true;
  }

  // 2. 类型检查
  if (
    typeof objA !== 'object' ||
    objA === null ||
    typeof objB !== 'object' ||
    objB === null
  ) {
    console.log('非对象或为 null ❌');
    return false;
  }

  // 3. 键数量检查
  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) {
    console.log(`键数量不同：${keysA.length} vs ${keysB.length} ❌`);
    return false;
  }

  // 4. 逐个比较值
  console.log('逐个比较属性：');
  for (let i = 0; i < keysA.length; i++) {
    const key = keysA[i];

    if (!Object.prototype.hasOwnProperty.call(objB, key)) {
      console.log(`  ${key}: objB 没有此键 ❌`);
      return false;
    }

    const isEqual = Object.is(objA[key], objB[key]);
    console.log(`  ${key}: ${objA[key]} vs ${objB[key]} = ${isEqual}`);

    if (!isEqual) {
      return false;
    }
  }

  console.log('所有属性相等 ✅');
  return true;
}

// 测试1：内容相同，引用不同
console.log('\n【测试1：内容相同，引用不同】');
const obj1 = { a: 1, b: NaN };
const obj2 = { a: 1, b: NaN };
console.log(`结果：${shallowEqual(obj1, obj2)}\n`);

// 测试2：引用相同
console.log('【测试2：引用相同】');
const obj3 = { a: 1 };
console.log(`结果：${shallowEqual(obj3, obj3)}\n`);

// 测试3：嵌套对象（浅比较失败）
console.log('【测试3：嵌套对象】');
const obj4 = { a: { b: 1 } };
const obj5 = { a: { b: 1 } };
console.log(`结果：${shallowEqual(obj4, obj5)}\n`);
console.log('原因：嵌套对象引用不同');

// ===== 5. 场景5：Object.is Polyfill =====
console.log("\n=== 场景5：Object.is Polyfill ===\n");

function objectIsPolyfill(x, y) {
  if (x === y) {
    // 处理 +0 vs -0
    // 1 / +0 = Infinity
    // 1 / -0 = -Infinity
    return x !== 0 || 1 / x === 1 / y;
  } else {
    // 处理 NaN
    // NaN 是唯一 !== 自己的值
    return x !== x && y !== y;
  }
}

console.log('【测试 Polyfill】');
console.log(`objectIsPolyfill(NaN, NaN): ${objectIsPolyfill(NaN, NaN)}`);
console.log(`objectIsPolyfill(+0, -0): ${objectIsPolyfill(+0, -0)}`);
console.log(`objectIsPolyfill(1, 1): ${objectIsPolyfill(1, 1)}`);
console.log(`objectIsPolyfill({}, {}): ${objectIsPolyfill({}, {})}`);
```

### 运行输出示例：

```
=== 场景1：Object.is vs === ===

【NaN 比较】
NaN === NaN: false
Object.is(NaN, NaN): true

【±0 比较】
+0 === -0: true
Object.is(+0, -0): false
Object.is(+0, +0): true
Object.is(-0, -0): true

【其他值比较】
1 === 1: true
Object.is(1, 1): true
obj === obj: true
Object.is(obj, obj): true
{} === {}: false
Object.is({}, {}): false

=== 场景2：区分 +0 和 -0 ===

+0（正零）
-0（负零）
1（非零）

1 / +0 = Infinity
1 / -0 = -Infinity

=== 场景3：React 依赖比较实现 ===

【测试1：正常值】
比较依赖项：
  [0] 1 vs 1: true
  [1] a vs a: true
  [2] true vs true: true
所有依赖相等 ✅
结果：true

【测试2：包含 NaN】
比较依赖项：
  [0] 1 vs 1: true
  [1] NaN vs NaN: true
  [2] a vs a: true
所有依赖相等 ✅
结果：true

【测试3：值改变】
比较依赖项：
  [0] 1 vs 1: true
  [1] 2 vs 999: false
结果：false

=== 场景4：浅比较实现 ===

【测试1：内容相同，引用不同】
逐个比较属性：
  a: 1 vs 1 = true
  b: NaN vs NaN = true
所有属性相等 ✅
结果：true

【测试2：引用相同】
引用相同 ✅
结果：true

【测试3：嵌套对象】
逐个比较属性：
  a: [object Object] vs [object Object] = false
结果：false
原因：嵌套对象引用不同

=== 场景5：Object.is Polyfill ===

【测试 Polyfill】
objectIsPolyfill(NaN, NaN): true
objectIsPolyfill(+0, -0): false
objectIsPolyfill(1, 1): true
objectIsPolyfill({}, {}): false
```

---

### 进阶：React 源码实现

```javascript
// packages/shared/objectIs.js
// React 的 Object.is 实现

const objectIs =
  typeof Object.is === 'function'
    ? Object.is
    : function(x, y) {
        // SameValue 算法
        if (x === y) {
          // 处理 +0 和 -0
          // x !== 0：排除 0 的情况
          // 1 / x === 1 / y：如果都是 0，检查符号
          //   1 / +0 = Infinity
          //   1 / -0 = -Infinity
          return x !== 0 || 1 / x === 1 / y;
        } else {
          // 处理 NaN
          // NaN 是唯一不等于自己的值
          // x !== x：x 是 NaN
          // y !== y：y 是 NaN
          return x !== x && y !== y;
        }
      };

export default objectIs;

// packages/react-reconciler/src/ReactFiberHooks.js
// Hook 依赖比较

function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) {
    return false;
  }

  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    // 使用 objectIs 比较每个依赖项
    if (objectIs(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}

// packages/shared/shallowEqual.js
// 浅比较实现

function shallowEqual(objA, objB) {
  // 1. 快速路径：引用相同
  if (objectIs(objA, objB)) {
    return true;
  }

  // 2. 类型检查
  if (
    typeof objA !== 'object' ||
    objA === null ||
    typeof objB !== 'object' ||
    objB === null
  ) {
    return false;
  }

  // 3. 比较键
  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) {
    return false;
  }

  // 4. 比较值（使用 Object.is）
  for (let i = 0; i < keysA.length; i++) {
    const currentKey = keysA[i];

    if (
      !Object.prototype.hasOwnProperty.call(objB, currentKey) ||
      !objectIs(objA[currentKey], objB[currentKey])
    ) {
      return false;
    }
  }

  return true;
}

// packages/react/src/ReactMemo.js
// React.memo 的实现

function memo(type, compare) {
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
  return elementType;
}

// 默认比较函数（浅比较）
function defaultCompare(prevProps, nextProps) {
  return shallowEqual(prevProps, nextProps);
}
```

**React Object.is 的三个关键应用：**

1. **Hook 依赖比较**：useEffect、useMemo、useCallback 的依赖数组使用 Object.is 逐个比较

2. **props 浅比较**：React.memo 使用 shallowEqual，内部调用 Object.is 比较每个 prop

3. **状态更新判断**：useState 和 useReducer 使用 Object.is 判断新旧状态是否相同，决定是否触发重新渲染

---

## 八、【面试必问】

### 问题1："Object.is 和 === 有什么区别？"

**普通回答（❌ 不出彩）：**

"Object.is 比 === 更严格，能正确比较 NaN。"

**出彩回答（✅ 推荐）：**

> **Object.is 和 === 只有两个差异，其他情况完全相同：**
>
> 1. **NaN 比较**：
>    - `===`：`NaN === NaN` 返回 `false`（遵循 IEEE 754 标准）
>    - `Object.is`：`Object.is(NaN, NaN)` 返回 `true`（值相等性语义）
>
> 2. **零值符号**：
>    - `===`：`+0 === -0` 返回 `true`（不区分符号）
>    - `Object.is`：`Object.is(+0, -0)` 返回 `false`（区分符号）
>
> **实现原理**（React 源码中的 polyfill）：
> ```javascript
> function objectIs(x, y) {
>   if (x === y) {
>     // 处理 ±0：1 / +0 = Infinity，1 / -0 = -Infinity
>     return x !== 0 || 1 / x === 1 / y;
>   }
>   // 处理 NaN：NaN 是唯一不等于自己的值
>   return x !== x && y !== y;
> }
> ```
>
> **为什么 React 选择 Object.is？**
> - **正确性**：处理 NaN 时不会导致无限循环
> - **标准化**：ES6 SameValue 算法，语义明确
> - **性能**：与 === 性能相同（引擎内联优化）
> - **一致性**：在所有场景下行为可预测

**为什么这个回答出彩？**

1. ✅ **具体明确**：只有两个差异，其他相同
2. ✅ **深入原理**：展示了 polyfill 实现
3. ✅ **联系实践**：说明了 React 选择它的原因
4. ✅ **澄清误解**：不是"更严格"，而是"更正确"

---

### 问题2："为什么 React 用 Object.is 而不是 ===？"

**普通回答（❌ 不出彩）：**

"因为 Object.is 能处理 NaN。"

**出彩回答（✅ 推荐）：**

> **React 使用 Object.is 主要出于以下考虑：**
>
> 1. **避免 NaN 导致的无限循环**：
>    ```javascript
>    function Component() {
>      const [value, setValue] = useState(NaN);
>
>      useEffect(() => {
>        console.log('执行');
>      }, [value]);
>
>      // 如果用 ===：
>      //   NaN !== NaN → 依赖变化 → 执行 Effect → 无限循环
>
>      // 使用 Object.is：
>      //   Object.is(NaN, NaN) = true → 依赖不变 → 不执行 ✅
>    }
>    ```
>
> 2. **语义正确性**：
>    - Object.is 代表"值是否相同"，符合开发者直觉
>    - NaN 在数学上是同一个概念，应该相等
>    - React 需要判断"值是否变了"，而非"引用是否变了"
>
> 3. **标准化**：
>    - Object.is 是 ES6 标准（SameValue 算法）
>    - 有明确的规范定义，行为可预测
>    - 避免自己实现比较逻辑的 bug
>
> 4. **性能无损**：
>    - 现代 JavaScript 引擎会内联 Object.is
>    - 性能与 === 几乎相同
>    - 无需为性能而牺牲正确性
>
> **实际影响**：
> - useState/useReducer：判断状态是否变化
> - useEffect/useMemo/useCallback：比较依赖数组
> - React.memo：浅比较 props
>
> 虽然 ±0 的区分在实际中很少用到，但使用标准算法保证了 React 在所有边缘情况下的行为一致性。

**为什么这个回答出彩？**

1. ✅ **举例说明**：用具体代码展示了 NaN 的问题
2. ✅ **多维度分析**：从正确性、标准化、性能多角度解释
3. ✅ **联系实践**：列举了 React 中的实际应用
4. ✅ **深度思考**：说明了为什么不担心 ±0 的区分

---

## 九、【化骨绵掌】

### 卡片1：Object.is 是什么 🎯

**一句话：** Object.is 是 ES6 提供的严格相等判断方法，实现了 SameValue 算法。

**举例：**

```javascript
Object.is(1, 1);       // true
Object.is(NaN, NaN);   // true
Object.is(+0, -0);     // false
```

**应用：** React 所有比较的基础，从依赖数组到 props 浅比较。

---

### 卡片2：SameValue 算法 📏

**一句话：** SameValue 是 ECMAScript 规范定义的相等性语义，比 === 更严格。

**举例：**

```javascript
// === 的两个反直觉行为
NaN === NaN;  // false
+0 === -0;    // true

// SameValue 修正了它们
Object.is(NaN, NaN);  // true
Object.is(+0, -0);    // false
```

**应用：** 理解 Object.is 的行为来源，知道它不是 React 发明的，而是标准。

---

### 卡片3：NaN 比较 🔢

**一句话：** NaN 是唯一不等于自己的值，Object.is 纠正了这一行为。

**举例：**

```javascript
console.log(NaN === NaN);           // false
console.log(Object.is(NaN, NaN));   // true
console.log(NaN !== NaN);           // true
```

**应用：** useEffect 依赖数组包含 NaN 时，用 Object.is 避免无限循环。

---

### 卡片4：±0 区分 ➕➖

**一句话：** JavaScript 中存在 +0 和 -0，Object.is 可以区分它们。

**举例：**

```javascript
console.log(+0 === -0);           // true
console.log(Object.is(+0, -0));   // false
console.log(1 / +0);              // Infinity
console.log(1 / -0);              // -Infinity
```

**应用：** 虽然很少用到，但保证了算法的完整性和标准化。

---

### 卡片5：Polyfill 实现 🔧

**一句话：** Object.is 的 polyfill 只需几行代码，核心是处理 NaN 和 ±0。

**举例：**

```javascript
function objectIs(x, y) {
  if (x === y) {
    return x !== 0 || 1 / x === 1 / y; // ±0
  }
  return x !== x && y !== y; // NaN
}
```

**应用：** React 使用这个 polyfill 确保在旧浏览器中也能正常工作。

---

### 卡片6：依赖比较 🔍

**一句话：** React Hook 的依赖数组使用 Object.is 逐个比较元素。

**举例：**

```javascript
useEffect(() => {
  // ...
}, [dep1, dep2]);

// 等价于
if (!Object.is(dep1, prevDep1) || !Object.is(dep2, prevDep2)) {
  执行Effect();
}
```

**应用：** 理解为什么对象/数组依赖要用 useMemo 包裹。

---

### 卡片7：浅比较 📦

**一句话：** 浅比较只检查对象第一层属性，内部用 Object.is 比较每个值。

**举例：**

```javascript
shallowEqual({ a: 1 }, { a: 1 }); // true
shallowEqual({ a: { b: 1 } }, { a: { b: 1 } }); // false
// 嵌套对象引用不同
```

**应用：** React.memo 默认用浅比较，嵌套对象变化不会检测到。

---

### 卡片8：性能优化 ⚡

**一句话：** Object.is 性能与 === 相同，现代引擎会内联优化。

**举例：**

```javascript
// 两者性能几乎相同
for (let i = 0; i < 1000000; i++) {
  i === i;
  Object.is(i, i);
}
```

**应用：** 不用担心 Object.is 的性能开销，优先考虑正确性。

---

### 卡片9：React.memo 🎭

**一句话：** React.memo 用浅比较判断 props 是否变化，内部基于 Object.is。

**举例：**

```javascript
const Memo = React.memo(Component);
// 等价于
const Memo = React.memo(Component, (prev, next) => {
  return shallowEqual(prev, next);
});
```

**应用：** 传给 memo 组件的 props 要稳定引用，否则会重新渲染。

---

### 卡片10：最佳实践 🎓

**一句话：** 理解 Object.is 是正确使用 React 优化 API 的基础。

**关键技巧：**
- ✅ 依赖数组中的对象用 useMemo 包裹
- ✅ 传给子组件的函数用 useCallback 包裹
- ✅ 理解浅比较的局限性（嵌套对象）
- ✅ 不要过早优化，先保证正确性

**举例：**

```javascript
// ❌ Bad：每次都是新对象
<Child data={{ count }} />

// ✅ Good：记忆化对象
const data = useMemo(() => ({ count }), [count]);
<Child data={data} />
```

**应用：** 避免不必要的重新渲染，提升应用性能。

---

## 十、【一句话总结】

**Object.is 实现 SameValue 算法，正确处理 NaN 和 ±0，在 React 中用于 Hook 依赖比较、props 浅比较和 memo 优化判断，是 React 性能优化和正确性保证的基础。**

---

## ✅ 学习检查清单

### 基础理解
- [ ] 能解释 Object.is 和 === 的区别
- [ ] 理解 SameValue 算法的概念
- [ ] 知道 NaN 的特殊性
- [ ] 了解 +0 和 -0 的区分

### 实现原理
- [ ] 能写出 Object.is 的 polyfill
- [ ] 理解 1 / +0 和 1 / -0 的技巧
- [ ] 知道 NaN !== NaN 的检测方法
- [ ] 了解引擎的优化机制

### React 应用
- [ ] 理解 Hook 依赖比较的实现
- [ ] 知道浅比较的工作原理
- [ ] 会使用 React.memo 优化组件
- [ ] 能正确设置依赖数组

### 进阶知识
- [ ] 理解引用 vs 内容的区别
- [ ] 知道何时用 useMemo/useCallback
- [ ] 了解浅比较 vs 深比较
- [ ] 能读懂 React 比较相关源码

---

## 📚 下一步学习

掌握 Object.is 后，建议学习：

1. **Fiber 架构** - 理解 React 内部的数据结构
2. **Reconciler 协调器** - 深入 Diff 算法和更新机制
3. **Hooks 实现** - 从源码层面理解 useState、useEffect
4. **性能优化** - React.memo、useMemo、useCallback 的综合应用

---

## 🔗 参考资源

- [MDN - Object.is](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/is)
- [ECMAScript - SameValue 算法](https://tc39.es/ecma262/#sec-samevalue)
- [React 源码 - objectIs.js](https://github.com/facebook/react/blob/main/packages/shared/objectIs.js)
- [React 源码 - shallowEqual.js](https://github.com/facebook/react/blob/main/packages/shared/shallowEqual.js)

---

**🎯 记住：Object.is 不是魔法，而是 JavaScript 标准的一部分！**
