# 🚀 第九章：并发模式（Concurrent Mode）

> 并发模式是 React 18+ 的核心特性，让 React 能够同时处理多个更新，保证用户体验的流畅性。

## 🤔 什么是并发？

先区分两个概念：

- **并行（Parallel）**：多件事**同时**发生（需要多个 CPU 核心）
- **并发（Concurrent）**：多件事**交替**进行（一个核心快速切换任务）

```mermaid
graph LR
    subgraph "🔀 并行（多线程）"
        P1["任务 A ━━━━━━━"]
        P2["任务 B ━━━━━━━"]
    end
    
    subgraph "🔄 并发（单线程）"
        C1["任务 A ━━ 任务 B ━━ 任务 A ━━ 任务 B ━━"]
    end
```

> 🌰 **比喻**：你在做两道菜。**并行**就是你有两个灶台，两道菜同时做。**并发**就是你只有一个灶台，先炒一会青菜，然后去煮一会汤，再回来继续炒青菜 —— 交替进行，但最终两道菜都做好了。

React 运行在浏览器的**单线程**上，所以它采用的是**并发**模式。

## ❓ 为什么需要并发？

### 没有并发的问题

```jsx
function App() {
  const [input, setInput] = useState('');
  const [list, setList] = useState([]);

  function handleChange(e) {
    setInput(e.target.value);      // 更新输入框
    setList(generateBigList(e.target.value));  // 生成大量列表（耗时 200ms）
  }

  return (
    <div>
      <input value={input} onChange={handleChange} />
      <List items={list} />
    </div>
  );
}
```

```mermaid
graph LR
    subgraph "❌ 没有并发"
        A["用户输入 'R'"] --> B["更新输入框 + 生成列表<br/>🔴 200ms 阻塞"]
        B --> C["用户输入 'e'"] --> D["更新输入框 + 生成列表<br/>🔴 200ms 阻塞"]
        D --> E["用户感觉：卡顿 😤"]
    end
```

### 有并发的解决方案

```jsx
function App() {
  const [input, setInput] = useState('');
  const [list, setList] = useState([]);

  function handleChange(e) {
    setInput(e.target.value);      // ⚡ 高优先级：立即更新

    startTransition(() => {
      setList(generateBigList(e.target.value));  // 🐢 低优先级：可以被中断
    });
  }

  return (
    <div>
      <input value={input} onChange={handleChange} />
      <List items={list} />
    </div>
  );
}
```

```mermaid
graph LR
    subgraph "✅ 有并发"
        A2["用户输入 'R'"] --> B2["⚡ 立即更新输入框"]
        B2 --> C2["🐢 开始生成列表..."]
        C2 -->|"用户又输入了 'e'"| D2["⚡ 中断列表生成<br/>立即更新输入框"]
        D2 --> E2["🐢 用新输入重新生成列表"]
        E2 --> F2["用户感觉：流畅 😊"]
    end
```

## 🛤️ Lanes —— 并发的基础

> 📂 **源码位置**：`packages/react-reconciler/src/ReactFiberLane.js`

Lanes（车道）是 React 并发模式的**优先级模型**。不同的更新被分配到不同的 Lane：

```javascript
// 每个 Lane 是一个二进制位
const SyncLane           = 0b0000000000000000000000000000010;
const InputContinuousLane= 0b0000000000000000000000000001000;
const DefaultLane        = 0b0000000000000000000000000100000;
const TransitionLane1    = 0b0000000000000000000000100000000;
const TransitionLane2    = 0b0000000000000000000001000000000;
// ... 共 16 条 Transition Lane
const IdleLane           = 0b0010000000000000000000000000000;
const OffscreenLane      = 0b0100000000000000000000000000000;
```

```mermaid
graph TD
    subgraph "🏎️ 高速公路上的车道"
        direction LR
        SYNC["🚨 紧急车道<br/>SyncLane<br/>flushSync"]
        INPUT["🚗 快车道<br/>InputContinuousLane<br/>滚动、拖拽"]
        DEFAULT["🚙 普通车道<br/>DefaultLane<br/>setState"]
        TRANS["🚐 慢车道<br/>TransitionLanes<br/>startTransition"]
        IDLE["🚶 步行道<br/>IdleLane<br/>空闲任务"]
    end
    
    SYNC -.->|"优先级递减 →"| INPUT
    INPUT -.->|"→"| DEFAULT
    DEFAULT -.->|"→"| TRANS
    TRANS -.->|"→"| IDLE
```

### Lane 的批处理

多个相同优先级的更新可以合并处理：

```javascript
// 位运算合并多个 Lane
const mergedLanes = TransitionLane1 | TransitionLane2;
// 0b0000000000000000000000100000000 | 0b0000000000000000000001000000000
// = 0b0000000000000000000001100000000

// 检查是否包含某个 Lane
function includesTransitionLane(lanes) {
  return (lanes & TransitionLanes) !== NoLanes;
}

// 获取最高优先级的 Lane
function getHighestPriorityLane(lanes) {
  return lanes & -lanes;  // 取最低位的 1
}
```

### 更新如何被分配 Lane

```javascript
function requestUpdateLane(fiber) {
  // 1. 如果在 Transition 中，使用 TransitionLane
  if (currentEventTransitionLane !== NoLane) {
    return currentEventTransitionLane;
  }
  
  // 2. 根据当前执行环境确定
  const updateLane = getCurrentUpdatePriority();
  if (updateLane !== NoLane) {
    return updateLane;
  }
  
  // 3. 根据事件类型确定
  const eventLane = getCurrentEventPriority();
  return eventLane;
}

// 不同事件对应不同优先级
function getEventPriority(domEventName) {
  switch (domEventName) {
    case 'click':
    case 'keydown':
    case 'keyup':
      return DiscreteEventPriority;   // → SyncLane
      
    case 'scroll':
    case 'mousemove':
    case 'drag':
      return ContinuousEventPriority; // → InputContinuousLane
      
    default:
      return DefaultEventPriority;    // → DefaultLane
  }
}
```

## 🔄 useTransition —— 标记低优先级更新

> 📂 **源码位置**：`packages/react-reconciler/src/ReactFiberHooks.js`

```javascript
function mountTransition() {
  const [isPending, setPending] = mountState(false);
  
  const start = (callback) => {
    setPending(true);
    
    // 🔑 关键：在 Transition 上下文中执行回调
    const prevTransition = ReactSharedInternals.T;
    ReactSharedInternals.T = {};  // 标记进入 Transition
    
    try {
      setPending(false);
      callback();  // callback 中的 setState 会被标记为 TransitionLane
    } finally {
      ReactSharedInternals.T = prevTransition;
    }
  };
  
  return [isPending, start];
}
```

### 使用方式

```jsx
function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('home');

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);  // 这个更新是 TransitionLane（低优先级）
    });
  }

  return (
    <div>
      <TabBar 
        tabs={['home', 'about', 'contact']} 
        onSelect={selectTab} 
      />
      {isPending ? <Spinner /> : <TabPanel tab={tab} />}
    </div>
  );
}
```

```mermaid
sequenceDiagram
    participant User as 👤 用户
    participant App as 📱 应用
    participant React as 🧠 React

    User->>App: 点击 "About" 标签
    App->>React: startTransition(() => setTab('about'))
    
    React->>React: 1. isPending = true（高优先级 SyncLane）
    React->>React: 2. setTab('about')（低优先级 TransitionLane）
    
    Note over React: 先渲染高优先级更新
    React->>App: 显示 Spinner 🔄
    
    Note over React: 在后台渲染低优先级更新
    React->>React: 渲染新的 TabPanel...
    
    alt 用户又点了别的标签
        User->>App: 点击 "Contact"
        React->>React: 中断 "About" 的渲染 ⏹️
        React->>React: 开始渲染 "Contact"
    else 渲染完成
        React->>App: 显示 About 页面<br/>isPending = false
    end
```

## ⏳ useDeferredValue —— 延迟值

```javascript
function mountDeferredValue(value) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = value;
  return value;
}

function updateDeferredValue(value) {
  const hook = updateWorkInProgressHook();
  const prevValue = hook.memoizedState;
  
  // 如果值没变，直接返回
  if (Object.is(value, prevValue)) {
    return value;
  }
  
  // 如果当前正在进行高优先级渲染，返回旧值
  // 然后在低优先级的渲染中返回新值
  if (isCurrentTreeHiddenPriority()) {
    hook.memoizedState = value;
    return value;
  } else {
    // 标记需要一个低优先级的重渲染
    const deferredLane = requestDeferredLane();
    markSkippedUpdateLanes(deferredLane);
    hook.memoizedState = prevValue;  // 先返回旧值
    return prevValue;
  }
}
```

### 使用场景

```jsx
function SearchResults({ query }) {
  // query 改变时，deferredQuery 会「延迟」更新
  const deferredQuery = useDeferredValue(query);
  
  // 用 deferredQuery 做耗时计算
  const results = useMemo(() => 
    search(deferredQuery), // 只在 deferredQuery 变化时重新搜索
    [deferredQuery]
  );
  
  // query !== deferredQuery 时，说明正在「过渡」
  const isStale = query !== deferredQuery;
  
  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      {results.map(item => <Result key={item.id} item={item} />)}
    </div>
  );
}
```

## 🌊 Suspense —— 异步的优雅处理

Suspense 让组件可以「等待」异步数据，在等待期间显示 fallback：

```jsx
function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile />  {/* 可能还在加载数据 */}
    </Suspense>
  );
}
```

### Suspense 的工作原理

```mermaid
sequenceDiagram
    participant React as 🧠 React
    participant Component as 📦 组件
    participant Suspense as 🌊 Suspense
    participant Network as 🌐 网络

    React->>Component: 渲染 UserProfile
    Component->>Network: 发起数据请求
    Component->>React: throw Promise ⚡
    
    Note over React: 抛出 Promise！
    React->>Suspense: 捕获 Promise
    Suspense->>React: 显示 fallback（<Loading />）
    
    Network->>Component: 数据返回
    Component->>React: Promise resolve
    React->>React: 重新渲染 UserProfile
    React->>Suspense: 替换 fallback 为真实内容
```

### 源码中的 Suspense 处理

```javascript
// 在 beginWork 中处理 SuspenseComponent
function updateSuspenseComponent(current, workInProgress, renderLanes) {
  const nextProps = workInProgress.pendingProps;
  
  let showFallback = false;
  const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags;
  
  if (didSuspend) {
    showFallback = true;
    workInProgress.flags &= ~DidCapture;  // 清除标记
  }
  
  if (showFallback) {
    // 渲染 fallback
    const fallbackChildren = nextProps.fallback;
    return mountSuspenseFallbackChildren(workInProgress, fallbackChildren, renderLanes);
  } else {
    // 渲染主内容
    const primaryChildren = nextProps.children;
    return mountSuspensePrimaryChildren(workInProgress, primaryChildren, renderLanes);
  }
}

// 当组件 throw Promise 时
function throwException(root, value, sourceFiber, renderLanes) {
  if (typeof value === 'object' && typeof value.then === 'function') {
    // 这是一个 Promise（Suspense 触发）
    const wakeable = value;
    
    // 找到最近的 Suspense 边界
    const suspenseBoundary = getNearestSuspenseBoundaryToCapture(sourceFiber);
    
    if (suspenseBoundary !== null) {
      // 标记 Suspense 需要显示 fallback
      suspenseBoundary.flags |= ShouldCapture;
      
      // 注册 Promise 的回调（resolve 后重新渲染）
      attachPingListener(root, wakeable, renderLanes);
    }
  }
}
```

## 🔄 并发渲染的中断与恢复

### 高优先级中断低优先级

```mermaid
graph TD
    subgraph "时间轴"
        T1["T=0ms<br/>开始 Transition 渲染"]
        T2["T=3ms<br/>处理 Fiber A ✅"]
        T3["T=6ms<br/>处理 Fiber B ✅"]
        T4["T=8ms<br/>⚡ 用户点击！<br/>高优先级中断"]
        T5["T=8ms<br/>开始处理高优先级更新"]
        T6["T=10ms<br/>高优先级渲染完成 ✅"]
        T7["T=10ms<br/>恢复 Transition 渲染"]
        T8["T=13ms<br/>处理 Fiber C ✅"]
        T9["T=16ms<br/>Transition 渲染完成 ✅"]
    end
    
    T1 --> T2 --> T3 --> T4
    T4 --> T5 --> T6 --> T7 --> T8 --> T9

    style T4 fill:#ef9a9a
    style T5 fill:#ef9a9a
    style T6 fill:#ef9a9a
```

```javascript
// 简化的中断逻辑
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
  // shouldYield() 返回 true 时循环退出
  // workInProgress 保留了当前位置
  // 下次执行时从这里继续
}

function ensureRootIsScheduled(root) {
  const existingCallbackPriority = root.callbackPriority;
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  
  // 如果有更高优先级的更新来了
  if (newCallbackPriority > existingCallbackPriority) {
    // 取消当前正在执行的低优先级任务
    cancelCallback(existingCallbackNode);
    
    // 安排新的高优先级任务
    const newCallbackNode = scheduleCallback(
      higherPriority,
      performWorkOnRoot.bind(null, root)
    );
    root.callbackNode = newCallbackNode;
    root.callbackPriority = newCallbackPriority;
  }
}
```

## 📝 本章小结

```mermaid
graph TD
    TITLE["并发模式核心"]
    
    TITLE --> A["Lanes 优先级"]
    A --> A1["不同更新分配不同 Lane"]
    A --> A2["高优先级可中断低优先级"]
    A --> A3["位运算高效管理"]
    
    TITLE --> B["并发 API"]
    B --> B1["useTransition<br/>标记低优先级更新"]
    B --> B2["useDeferredValue<br/>延迟非紧急值"]
    B --> B3["Suspense<br/>异步数据的优雅等待"]
    
    TITLE --> C["核心机制"]
    C --> C1["时间切片：5ms 让出"]
    C --> C2["可中断渲染"]
    C --> C3["优先级调度"]
```

**核心要点**：
1. 🔄 并发 ≠ 并行，React 在单线程上通过**交替执行**实现并发
2. 🛤️ **Lanes** 模型提供精细的优先级控制，不同更新分配到不同车道
3. ⚡ `useTransition` 将更新标记为低优先级，不阻塞用户交互
4. ⏳ `useDeferredValue` 让值延迟更新，避免频繁重渲染
5. 🌊 `Suspense` 通过 throw Promise 机制，优雅处理异步加载
6. 🔀 高优先级更新可以**中断**低优先级更新的渲染

---

📖 **下一章**：[服务端组件（Server Components）→](./10-server-components.md)

> 在最后一章，我们将探索 React 最新的服务端组件技术，理解它如何改变前后端的协作方式。
