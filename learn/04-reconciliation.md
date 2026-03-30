# 🔄 第四章：协调算法（Reconciliation）

> 协调（Reconciliation）是 React 的核心算法，负责对比新旧组件树，找出最小变化集合。

## 🤔 什么是协调？

当你的组件状态改变时，React 需要：
1. 根据新的 state 和 props，生成新的 ReactElement 树
2. 对比新旧两棵树，找出**最小的差异**
3. 只更新变化了的部分

这个「对比差异」的过程就叫**协调（Reconciliation）**，其中的对比算法就是我们常说的 **Diff 算法**。

> 🌰 **通俗比喻**：想象你有两张「找不同」的图片。React 的 Diff 算法就是帮你快速找出哪些地方不一样，然后只修改那些不同的地方，而不是整张图重新画。

## ⚡ 为什么不直接全量更新？

```mermaid
graph LR
    subgraph "❌ 全量更新"
        A1["新的组件树"] -->|"丢弃旧 DOM<br/>重建全部"| B1["重建整棵 DOM 树<br/>性能极差 💀"]
    end

    subgraph "✅ Diff 更新"
        A2["新的组件树"] -->|"与旧树对比<br/>找出差异"| B2["只更新变化的 DOM<br/>性能优秀 🚀"]
    end
```

完整对比两棵树的最优算法时间复杂度是 **O(n³)**，对于 1000 个节点的树需要 10 亿次比较。React 通过三个策略将复杂度降为 **O(n)**：

## 📐 Diff 的三大策略

### 策略一：同层比较（Tree Diff）

> React 只对比**同一层级**的节点，不会跨层移动。

```mermaid
graph TD
    subgraph "旧树"
        OA["A"] --> OB["B"]
        OA --> OC["C"]
        OB --> OD["D"]
    end

    subgraph "新树"
        NA["A"] --> NB["B"]
        NA --> NC["C"]
        NB --> ND["D"]
    end

    OA <-.->|"✅ 对比"| NA
    OB <-.->|"✅ 对比"| NB
    OC <-.->|"✅ 对比"| NC
    OD <-.->|"✅ 对比"| ND
```

> 💡 **为什么只同层比较？** 在实际的 Web 应用中，跨层级移动 DOM 节点的情况极少发生。React 选择了实用主义：牺牲理论最优解，换取实际性能的大幅提升。

### 策略二：类型比较（Component Diff）

> 类型不同的元素 → 直接销毁重建；类型相同 → 保留节点，只更新属性。

```jsx
// 情况 1：类型相同 → 更新属性
// 旧：<div className="old">Hello</div>
// 新：<div className="new">World</div>
// 结果：保留 div 节点，更新 className 和文本

// 情况 2：类型不同 → 销毁重建
// 旧：<div>Hello</div>
// 新：<span>Hello</span>
// 结果：销毁 div 及其子树，创建新的 span
```

```mermaid
graph TD
    COMPARE["对比两个节点"]
    COMPARE -->|"类型相同"| SAME["保留节点<br/>对比 props<br/>递归对比子节点"]
    COMPARE -->|"类型不同"| DIFF["销毁旧节点及子树<br/>创建新节点及子树"]
    
    style SAME fill:#c8e6c9
    style DIFF fill:#ef9a9a
```

### 策略三：Key 标识（Element Diff）

> 同一层级的子节点通过 `key` 来标识「谁是谁」。

```jsx
// ❌ 没有 key：React 不知道哪个是哪个
<ul>
  <li>苹果</li>    {/* index 0 */}
  <li>香蕉</li>    {/* index 1 */}
  <li>橘子</li>    {/* index 2 */}
</ul>

// ✅ 有 key：React 能精确追踪每个元素
<ul>
  <li key="apple">苹果</li>
  <li key="banana">香蕉</li>
  <li key="orange">橘子</li>
</ul>
```

## 🔍 深入 beginWork —— Diff 的入口

> 📂 **源码位置**：`packages/react-reconciler/src/ReactFiberBeginWork.js`

`beginWork` 是 Render 阶段的核心函数，根据 Fiber 节点的类型（tag）分发到不同的处理逻辑：

```javascript
// 简化的 beginWork
function beginWork(current, workInProgress, renderLanes) {
  // current !== null 表示这是更新（不是首次渲染）
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    // 🚀 优化：如果 props 和 type 都没变，可以跳过
    if (oldProps === newProps && !hasContextChanged()) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }

  // 根据节点类型分发
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(current, workInProgress, renderLanes);
    case ClassComponent:
      return updateClassComponent(current, workInProgress, renderLanes);
    case HostComponent:  // DOM 元素如 div, span
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:       // 文本节点
      return updateHostText(current, workInProgress);
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
    case SuspenseComponent:
      return updateSuspenseComponent(current, workInProgress, renderLanes);
    // ...更多类型
  }
}
```

### 函数组件的处理

```javascript
function updateFunctionComponent(current, workInProgress, renderLanes) {
  // 1. 调用组件函数，执行 Hooks
  const nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes
  );
  
  // 2. 如果组件没有变化，可以复用（bailout 优化）
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  
  // 3. 协调子节点（Diff 的核心）
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  
  return workInProgress.child;
}
```

## 👶 子节点 Diff —— reconcileChildFibers

> 📂 **源码位置**：`packages/react-reconciler/src/ReactChildFiber.js`

这是 Diff 算法最核心的部分，处理子节点的增删改：

```javascript
function reconcileChildFibers(returnFiber, currentFirstChild, newChild, lanes) {
  // 根据新子节点的类型，走不同的 Diff 路径
  
  // 1️⃣ 新子节点是对象（单个元素）
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return reconcileSingleElement(returnFiber, currentFirstChild, newChild, lanes);
      case REACT_PORTAL_TYPE:
        return reconcileSinglePortal(returnFiber, currentFirstChild, newChild, lanes);
      case REACT_LAZY_TYPE:
        // 处理 lazy 组件
    }
    
    // 新子节点是数组（多个元素）
    if (isArray(newChild)) {
      return reconcileChildrenArray(returnFiber, currentFirstChild, newChild, lanes);
    }
  }
  
  // 2️⃣ 新子节点是文本
  if (typeof newChild === 'string' || typeof newChild === 'number') {
    return reconcileSingleTextNode(returnFiber, currentFirstChild, '' + newChild, lanes);
  }
  
  // 3️⃣ 新子节点为空 → 删除所有旧子节点
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

### 单节点 Diff（reconcileSingleElement）

当新的子节点只有一个元素时：

```mermaid
graph TD
    START["新的子节点是单个元素"]
    START --> CHECK["遍历旧的子节点"]
    CHECK --> KEY_MATCH{"key 相同？"}
    KEY_MATCH -->|"是"| TYPE_MATCH{"type 相同？"}
    KEY_MATCH -->|"否"| DELETE_OLD["标记删除该旧节点<br/>继续检查下一个兄弟"]
    DELETE_OLD --> CHECK
    TYPE_MATCH -->|"是"| REUSE["✅ 复用旧 Fiber<br/>更新 props"]
    TYPE_MATCH -->|"否"| DELETE_ALL["❌ 删除该旧节点及其余兄弟<br/>创建新 Fiber"]

    style REUSE fill:#c8e6c9
    style DELETE_ALL fill:#ef9a9a
```

```javascript
function reconcileSingleElement(returnFiber, currentFirstChild, element, lanes) {
  const key = element.key;
  let child = currentFirstChild;
  
  // 遍历旧的子节点链表
  while (child !== null) {
    if (child.key === key) {
      // key 相同
      if (child.elementType === element.type) {
        // type 也相同 → 复用！
        deleteRemainingChildren(returnFiber, child.sibling); // 删除多余兄弟
        const existing = useFiber(child, element.props);      // 复用 Fiber
        existing.return = returnFiber;
        return existing;
      } else {
        // key 相同但 type 不同 → 不可能复用任何节点
        deleteRemainingChildren(returnFiber, child);
        break;
      }
    } else {
      // key 不同 → 标记删除，继续看下一个兄弟
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }
  
  // 没有可复用的 → 创建新 Fiber
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.return = returnFiber;
  return created;
}
```

### 多节点 Diff（reconcileChildrenArray）—— 最复杂的部分

当新的子节点是数组时（列表渲染），需要处理：**节点更新、新增、删除、移动**。

React 使用**两轮遍历**来高效处理：

```mermaid
graph TD
    START["多节点 Diff 开始"]
    
    subgraph "第一轮遍历"
        R1["从左到右逐个对比<br/>旧: [A, B, C, D]<br/>新: [A, B, E, F]"]
        R1 --> R1C{"key 和 type 都相同？"}
        R1C -->|"是"| R1Y["复用，继续下一个"]
        R1Y --> R1
        R1C -->|"否"| R1N["停止第一轮 ⏹️"]
    end
    
    subgraph "第二轮遍历"
        R2A{"旧节点遍历完？"}
        R2A -->|"是"| R2A1["剩余新节点全部创建<br/>（纯新增场景）"]
        R2A -->|"否"| R2B{"新节点遍历完？"}
        R2B -->|"是"| R2B1["剩余旧节点全部删除<br/>（纯删除场景）"]
        R2B -->|"否"| R2C["构建旧节点 Map<br/>key → Fiber"]
        R2C --> R2D["遍历剩余新节点<br/>在 Map 中查找可复用的"]
        R2D --> R2E["未匹配的旧节点标记删除"]
    end
    
    R1N --> R2A

    style R1Y fill:#c8e6c9
    style R2A1 fill:#bbdefb
    style R2B1 fill:#ef9a9a
    style R2C fill:#ffe0b2
```

#### 用具体例子理解

**场景 1：节点更新（最常见）**
```
旧: A → B → C → D
新: A → B → C → D  （props 变了）

第一轮：A✅ B✅ C✅ D✅  全部匹配，全部复用更新
```

**场景 2：尾部新增**
```
旧: A → B → C
新: A → B → C → D → E

第一轮：A✅ B✅ C✅  旧节点遍历完
第二轮：D 和 E 是新增，直接创建
```

**场景 3：头部删除**
```
旧: A → B → C → D
新: B → C → D

第一轮：A 和 B 的 key 不同，停止
第二轮：
  - 旧节点建 Map: {A: fiberA, B: fiberB, C: fiberC, D: fiberD}
  - 新节点 B → 在 Map 中找到 → 复用
  - 新节点 C → 在 Map 中找到 → 复用
  - 新节点 D → 在 Map 中找到 → 复用
  - Map 中剩余 A → 删除
```

**场景 4：节点移动**
```
旧: A → B → C → D
新: A → C → B → D

第一轮：A✅, 然后 C 和 B 的 key 不同，停止
第二轮：
  - 旧节点建 Map: {B: fiberB, C: fiberC, D: fiberD}
  - 新节点 C → 在 Map 中找到 → 复用
  - 新节点 B → 在 Map 中找到 → 复用，但位置变了 → 标记移动
  - 新节点 D → 在 Map 中找到 → 复用
```

### 移动判断的关键：lastPlacedIndex

```javascript
// 简化的移动判断逻辑
let lastPlacedIndex = 0;  // 上一个可复用节点在旧树中的位置

for (let newIdx = 0; newIdx < newChildren.length; newIdx++) {
  const newChild = newChildren[newIdx];
  const oldFiber = existingChildren.get(newChild.key);
  
  if (oldFiber) {
    // 找到可复用的旧节点
    if (oldFiber.index < lastPlacedIndex) {
      // 旧位置在 lastPlacedIndex 左边 → 需要向右移动
      newFiber.flags |= Placement;
    } else {
      // 旧位置在右边 → 不需要移动
      lastPlacedIndex = oldFiber.index;
    }
  } else {
    // 没有可复用的 → 新建
    newFiber.flags |= Placement;
  }
}
```

> 🌰 **比喻**：想象一排学生站队。老师要求按新顺序重排。不需要所有人都动——只需要让「位置不对」的人移动到正确位置。`lastPlacedIndex` 就是老师的记忆：「我已经排好了前几个人，在我记忆中的位置之前的人才需要移动」。

```mermaid
graph LR
    subgraph "旧顺序（index）"
        OA["A (0)"] --> OB["B (1)"] --> OC["C (2)"] --> OD["D (3)"]
    end
    
    subgraph "新顺序"
        NA["A"] --> NC["C"] --> NB["B"] --> ND["D"]
    end

    OA -.->|"index 0 ≥ lastPlaced(0)<br/>不移动 ✅<br/>lastPlaced=0"| NA
    OC -.->|"index 2 ≥ lastPlaced(0)<br/>不移动 ✅<br/>lastPlaced=2"| NC
    OB -.->|"index 1 < lastPlaced(2)<br/>需要移动 ⚠️"| NB
    OD -.->|"index 3 ≥ lastPlaced(2)<br/>不移动 ✅<br/>lastPlaced=3"| ND
```

## ✅ completeWork —— 完成节点处理

> 📂 **源码位置**：`packages/react-reconciler/src/ReactFiberCompleteWork.js`

当一个 Fiber 节点的所有子节点都处理完毕后，会调用 `completeWork`：

```javascript
function completeWork(current, workInProgress, renderLanes) {
  const newProps = workInProgress.pendingProps;
  
  switch (workInProgress.tag) {
    case HostComponent: {  // DOM 元素
      if (current !== null && workInProgress.stateNode != null) {
        // 🔄 更新：对比新旧 props，计算需要更新的属性
        updateHostComponent(current, workInProgress, type, newProps);
      } else {
        // 🆕 首次渲染：创建 DOM 节点
        const instance = createInstance(type, newProps, workInProgress);
        appendAllChildren(instance, workInProgress);  // 将子 DOM 挂载
        workInProgress.stateNode = instance;
        finalizeInitialChildren(instance, type, newProps);  // 设置初始属性
      }
      
      // 冒泡副作用标记
      bubbleProperties(workInProgress);
      return null;
    }
    
    case HostText: {  // 文本节点
      if (current !== null && workInProgress.stateNode != null) {
        // 更新文本
        const oldText = current.memoizedProps;
        const newText = newProps;
        if (oldText !== newText) {
          markUpdate(workInProgress);
        }
      } else {
        // 创建文本节点
        workInProgress.stateNode = createTextInstance(newText);
      }
      return null;
    }
    // ...
  }
}
```

### 副作用冒泡（bubbleProperties）

`completeWork` 的一个关键操作是将子树的副作用标记**冒泡**到父节点：

```mermaid
graph BT
    D["div<br/>flags: Update<br/>subtreeFlags: Placement | Update"]
    H["h1<br/>flags: NoFlags<br/>subtreeFlags: NoFlags"]
    P["p<br/>flags: Placement<br/>subtreeFlags: Update"]
    SPAN["span<br/>flags: Update<br/>subtreeFlags: NoFlags"]

    H -->|"冒泡"| D
    P -->|"冒泡"| D
    SPAN -->|"冒泡"| P
```

> 💡 **为什么要冒泡？** 在 Commit 阶段，React 需要快速找到所有有副作用的节点。通过 `subtreeFlags`，React 可以在遍历时**跳过整棵没有副作用的子树**，大大提升性能。

## 🔑 为什么 Key 如此重要？

### 不用 Key 的后果

```jsx
// ❌ 没有 key
{items.map(item => <li>{item.name}</li>)}
```

React 会用 **index 作为默认 key**。当列表重排时：

```
旧: li(index=0, "苹果") → li(index=1, "香蕉") → li(index=2, "橘子")
删除第一个后:
新: li(index=0, "香蕉") → li(index=1, "橘子")

React 对比:
index=0: "苹果" → "香蕉"  → 更新文本（不必要的更新！）
index=1: "香蕉" → "橘子"  → 更新文本（不必要的更新！）
index=2: "橘子" → 无      → 删除
```

### 用 Key 的效果

```jsx
// ✅ 有 key
{items.map(item => <li key={item.id}>{item.name}</li>)}
```

```
旧: li(key="apple") → li(key="banana") → li(key="orange")
删除 apple 后:
新: li(key="banana") → li(key="orange")

React 对比:
key="apple": 旧有新无 → 删除
key="banana": 匹配 → 复用 ✅
key="orange": 匹配 → 复用 ✅
```

## 📝 本章小结

```mermaid
graph TD
    TITLE["协调算法核心"]
    
    TITLE --> S1["三大策略"]
    S1 --> S1A["同层比较 → O(n)"]
    S1 --> S1B["类型比较 → 同类型复用"]
    S1 --> S1C["Key 比较 → 精确追踪"]
    
    TITLE --> S2["处理流程"]
    S2 --> S2A["beginWork → 向下递归处理"]
    S2 --> S2B["reconcileChildren → Diff 核心"]
    S2 --> S2C["completeWork → 向上完成"]
    
    TITLE --> S3["多节点 Diff"]
    S3 --> S3A["第一轮：从左到右比较"]
    S3 --> S3B["第二轮：Map 查找 + 移动判断"]
```

**核心要点**：
1. 🎯 三大 Diff 策略将 O(n³) 降为 O(n)
2. 🔍 `beginWork` 根据节点类型分发处理
3. 👶 `reconcileChildFibers` 是 Diff 的核心入口
4. 🔄 多节点 Diff 使用两轮遍历 + Map 查找
5. 🔑 正确使用 `key` 对列表性能至关重要

---

📖 **下一章**：[调度器（Scheduler）原理 →](./05-scheduler.md)

> 在下一章，我们将深入 React 的调度系统，理解它如何实现「优先级调度」和「时间切片」。
