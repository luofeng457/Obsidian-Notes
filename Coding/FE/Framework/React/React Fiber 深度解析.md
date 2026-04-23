React Fiber 深度解析（2026最新，适配React 18+）

React Fiber 是 React 16 引入的**核心架构重写**，彻底重构了 React 的协调引擎，是 React 实现可中断渲染、优先级调度、并发特性的底层基石。它解决了 React 15 及之前 Stack Reconciler 同步阻塞的核心痛点，为 React 后续所有的性能优化和高级特性提供了架构支撑。

## 一、为什么需要 Fiber？旧架构的致命痛点

在 Fiber 诞生之前，React 使用**Stack Reconciler（栈协调器）**，其核心是基于 JavaScript 调用栈的同步递归更新：

1. **同步阻塞，无法中断**：一旦触发更新，React 会递归遍历整个虚拟 DOM 树进行 Diff 比对，直到整棵树处理完成才会释放主线程。如果组件树层级深、计算量大，这个过程会占用主线程数百毫秒。
    
2. **浏览器主线程卡死**：浏览器的 JS 执行、样式计算、布局绘制、用户交互都在主线程中执行，且互斥。JS 长时间占用主线程，会导致：
    
    1. 动画掉帧（无法达到 60FPS，每帧预算仅 16.6ms）
        
    2. 用户输入无响应（点击、输入、滚动卡顿）
        
    3. 页面交互体验严重下降
        
3. **无优先级区分**：所有更新任务优先级完全相同，用户输入和后台数据渲染会排队执行，无法让高优先级的用户交互优先处理。
    

Fiber 的核心设计目标，就是彻底解决上述问题：

- ✅ 将大的渲染任务拆分为**独立的小工作单元**，实现可中断、可恢复的渲染
    
- ✅ 为不同任务分配优先级，高优先级任务可打断低优先级任务
    
- ✅ 与浏览器的渲染节奏配合，在帧空闲时间执行任务，避免阻塞主线程
    
- ✅ 为并发渲染、Suspense、流式 SSR 等高级特性提供底层架构支持
    

## 二、Fiber 的核心定义：三层含义

Fiber 不是单一的概念，而是一套完整的架构体系，包含三层核心含义：

### 1. 作为数据结构的 Fiber

每个 React 组件（包括原生 DOM 节点）都会对应一个**Fiber 节点**，它是一个普通的 JavaScript 对象，存储了组件的类型、属性、状态、DOM 引用、以及与其他 Fiber 节点的关联关系。

与传统的虚拟 DOM 树不同，Fiber 节点通过**链表指针**连接，而非嵌套的树形结构，这是实现可中断遍历的核心基础。核心属性如下：

```javascript
// 简化的Fiber节点结构
const FiberNode = {
  // 1. 节点标识属性
  tag: 0, // 节点类型：函数组件/类组件/原生DOM/Portal等
  type: 'div' | Component, // 组件类型/原生DOM标签名
  key: 'key', //  Diff算法的唯一标识
  stateNode: domElement | componentInstance, // 真实DOM节点/组件实例

  // 2. 链表结构指针（核心！实现可中断遍历）
  return: FiberNode | null, // 指向父Fiber节点
  child: FiberNode | null, // 指向第一个子Fiber节点
  sibling: FiberNode | null, // 指向右边的兄弟Fiber节点
  alternate: FiberNode | null, // 双缓存指针，指向另一棵树的对应节点

  // 3. 状态与更新属性
  pendingProps: {}, // 新的props
  memoizedProps: {}, // 上一次渲染的props
  memoizedState: {}, // 上一次渲染的state（类组件）/hooks链表（函数组件）
  updateQueue: {}, // 更新队列，存储setState产生的更新

  // 4. 优先级与副作用属性
  lanes: 0b0000000000000000000000000000000, // 优先级车道（React17+）
  effectTag: 0, // 副作用标记：插入/更新/删除等
  firstEffect: null, // 副作用链表头
  lastEffect: null, // 副作用链表尾
  nextEffect: null, // 下一个有副作用的Fiber节点
}
```

### 2. 作为工作单元的 Fiber

每个 Fiber 节点对应一个**最小工作单元**，React 的渲染循环会逐个处理 Fiber 节点，完成该节点的 Diff 比对、副作用标记等工作。处理完一个单元后，React 会检查是否还有剩余时间片、是否有更高优先级任务，决定是否继续执行还是让出主线程。

### 3. 作为架构体系的 Fiber

Fiber 是一套完整的**Fiber Reconciler（纤维协调器）** 架构，包含了调度器（Scheduler）、协调器（Reconciler）、渲染器（Renderer）三层完整体系，实现了从更新触发到 DOM 渲染的全流程管控。

## 三、Fiber 的核心基石：双缓存机制

React 使用**双缓存（Double Buffering）** 技术管理 Fiber 树，这是实现安全中断、高效渲染的核心基础。React 在内存中同时维护两棵完整的 Fiber 树：

### 1. 两棵核心 Fiber 树

- **current 树**：当前屏幕上显示的 UI 对应的 Fiber 树，与真实 DOM 完全一一对应，所有用户可见的内容都来自这棵树。
    
- **workInProgress 树**：正在内存中构建的新 Fiber 树，所有的更新计算、Diff 比对、副作用标记都在这棵树上完成，整个过程对用户完全不可见。
    

两棵树的对应节点通过`alternate`指针互相引用，实现节点复用，避免重复创建对象带来的性能损耗。

### 2. 双缓存的工作流程

1. 触发更新（setState/useState），React 基于最新的状态，以 current 树为蓝本，开始构建 workInProgress 树；
    
2. Render 阶段，在 workInProgress 树上完成所有节点的 Diff 比对、副作用标记，整个过程可中断、可重启，不会影响屏幕显示；
    
3. workInProgress 树构建完成后，进入 Commit 阶段，React 将根节点的`current`指针直接指向 workInProgress 树，完成两棵树的切换；
    
4. 切换完成后，原来的 current 树变为下一次更新的 workInProgress 树，两棵树循环交替使用。
    

**核心优势**：所有的计算都在内存中的草稿树（workInProgress）上完成，只有最终构建完成后才会一次性提交到 UI，彻底避免了渲染一半的内容显示到屏幕上，同时实现了任务中断后，直接丢弃草稿树重新构建，不会影响当前显示的 UI。

## 四、Fiber 的完整工作流程

Fiber 架构将整个更新流程分为**三个核心阶段**：调度阶段（Scheduler）、渲染阶段（Render / 协调阶段）、提交阶段（Commit 阶段）。其中调度和渲染阶段可中断，提交阶段同步不可中断。

### 第一阶段：调度阶段（Scheduler）—— 决定 “什么时候执行任务”

调度阶段由 React 独立的**Scheduler 包**实现，核心职责是：为更新任务分配优先级，根据浏览器主线程的空闲情况，决定任务的执行时机，实现时间切片和优先级抢占。

#### 1. 优先级分配：Lanes 车道模型

React 17 + 使用**Lanes（车道模型）** 替代了旧的 expirationTime，用 32 位二进制数表示优先级，每一位代表一个独立的车道，数字越小，优先级越高。

核心优先级分类（从高到低）：

|   |   |   |
|---|---|---|
|优先级 Lane|二进制标识|适用场景|
|SyncLane|0b0000000000000000000000000000001|同步更新，最高优先级，如 ReactDOM.render、用户点击事件|
|InputContinuousLane|0b0000000000000000000000000000100|连续用户输入，如输入框打字、滚动、拖拽|
|DefaultLane|0b0000000000000000000000000010000|默认更新，如网络请求后的状态更新、useState 常规更新|
|TransitionLane|0b0000000000000000000000011100000|过渡更新，如 startTransition/useTransition 包裹的更新|
|IdleLane|0b0000000000000000000001000000000|最低优先级，浏览器完全空闲时才会执行|

Lanes 模型的核心优势：

- 位运算速度极快，性能开销极低；
    
- 支持多优先级并发，多个更新可以在不同车道并行处理；
    
- 可灵活实现优先级的合并、抢占、过滤，完美适配 React 18 的并发渲染。
    

#### 2. 时间切片的实现

时间切片是 Fiber 实现非阻塞渲染的核心：将长任务拆分为多个小任务，每个任务在一个时间片内执行，时间片耗尽后，让出主线程给浏览器，等待下一帧再继续执行。

**核心实现细节**：

- React 默认的时间片长度为**5ms**，保证浏览器有足够的时间完成剩余的渲染工作；
    
- 放弃使用`requestIdleCallback`：因为该 API 触发频率不稳定，在用户交互时（如滚动）几乎不会触发，兼容性也有问题；
    
- React 基于`MessageChannel`（降级方案为`setTimeout`）实现了自己的时间切片：在每帧的宏任务中执行任务，时间片耗尽后，通过 MessageChannel 发送消息，让出主线程，等待下一次循环继续执行。
    

#### 3. 优先级抢占

当低优先级任务正在执行时，如果触发了更高优先级的更新，React 会：

1. 立即中断当前低优先级的 Render 阶段任务；
    
2. 丢弃当前正在构建的 workInProgress 树；
    
3. 基于最新的状态和最高优先级，重新启动调度和 Render 流程；
    
4. 高优先级任务完成 Commit 后，再重新调度被中断的低优先级任务。
    

### 第二阶段：Render 阶段（协调阶段）—— 决定 “要更新什么”

Render 阶段的核心职责是：**构建 workInProgress 树，完成新旧 Fiber 树的 Diff 比对，标记所有需要执行的 DOM 变更（副作用），生成副作用链表**。

这个阶段完全在内存中执行，不会操作真实 DOM，**可中断、可重启、无副作用**，是 Fiber 可中断渲染的核心阶段。

#### 1. 核心遍历方式：循环 + 链表，替代递归

Stack Reconciler 使用递归遍历虚拟 DOM 树，调用栈无法中断；而 Fiber 使用**while 循环 + 链表指针**遍历 Fiber 树，只需保存当前处理的 Fiber 节点指针，就能随时中断和恢复遍历。

核心遍历逻辑（简化）：

```javascript
// 当前正在处理的工作单元
let workInProgress = null;

// 工作循环（并发模式）
function workLoopConcurrent() {
  // 还有工作单元，且时间片未耗尽，继续执行
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

// 处理单个Fiber工作单元
function performUnitOfWork(unitOfWork) {
  // 1. 递阶段：处理当前节点，创建子节点
  const next = beginWork(unitOfWork.alternate, unitOfWork);
  
  // 2. 没有子节点，进入归阶段
  if (next === null) {
    completeUnitOfWork(unitOfWork);
  } else {
    // 有子节点，继续处理子节点
    return next;
  }
}
```

#### 2. 递阶段：beginWork

`beginWork`是 Render 阶段的核心函数，负责处理单个 Fiber 节点，核心工作：

1. 根据 Fiber 节点的`tag`类型，区分处理函数组件、类组件、原生 DOM 节点等；
    
    1. 类组件：调用生命周期、render 函数，获取子节点；
        
    2. 函数组件：执行组件函数，调用 hooks，获取返回的子节点；
        
    3. 原生 DOM 节点：根据标签类型，生成子节点列表；
        
2. 执行**Diff 算法**，对比新旧子节点，生成新的子 Fiber 节点；
    
3. 为有变更的 Fiber 节点标记`effectTag`（副作用类型），比如：
    
    1. `Placement`：需要插入新的 DOM 节点；
        
    2. `Update`：需要更新 DOM 属性 / 文本；
        
    3. `Deletion`：需要删除 DOM 节点；
        
4. 返回第一个子 Fiber 节点，继续向下遍历；如果没有子节点，返回 null，进入归阶段。
    

**Diff 算法的核心优化**：Fiber 架构的 Diff 算法依然遵循 React 经典的同层比较规则，并未改变核心逻辑：

- 只对同一层级的节点进行比对，不跨层级比较；
    
- 用 key 唯一标识节点，实现列表节点的复用、重排、删除；
    
- 相同 type 和 key 的节点，复用 Fiber 对象，只更新属性；
    
- 不同 type 的节点，直接销毁旧节点，创建新节点。
    

#### 3. 归阶段：completeWork

当一个 Fiber 节点的所有子节点都处理完成后，会进入`completeWork`函数，核心工作：

1. 对于原生 DOM 节点，创建真实 DOM 元素，设置 DOM 属性、样式、事件监听；
    
2. 收集当前节点和子节点的所有副作用，构建**effect 链表**；
    
3. 向上回溯，处理兄弟节点，直到根节点，完成整棵 workInProgress 树的构建。
    

#### 4. 核心产物：effect 副作用链表

Render 阶段最终的产物，就是一个串联了所有有变更的 Fiber 节点的**effect 链表**。

Commit 阶段不需要遍历整棵 Fiber 树，只需要遍历这个线性的 effect 链表，就能一次性执行所有 DOM 变更，极大提升了 Commit 阶段的执行效率（Commit 阶段不可中断，必须越快越好）。

### 第三阶段：Commit 阶段（提交阶段）—— 执行 “真正的 DOM 更新”

Commit 阶段的核心职责是：**遍历 effect 链表，执行所有 DOM 变更，触发生命周期和 hooks 副作用，完成 UI 的最终更新**。

这个阶段**同步、不可中断**，一旦开始就必须执行完成，否则会出现 UI 与数据不一致、部分 DOM 更新部分未更新的错乱问题。

Commit 阶段分为三个严格按顺序执行的子阶段：

#### 1. Before Mutation 阶段（DOM 变更前）

- 核心工作：执行 DOM 变更前的准备工作，读取当前 DOM 的快照；
    
- 类组件：执行`getSnapshotBeforeUpdate`生命周期，获取更新前的 DOM 状态（如滚动位置）；
    
- 函数组件：执行 useLayoutEffect 的清理函数；
    
- 不会修改 DOM，仅做读取和准备工作。
    

#### 2. Mutation 阶段（DOM 变更执行）

- 核心工作：**执行所有真实 DOM 操作**，是唯一会修改 DOM 的阶段；
    
- 按严格顺序执行副作用：
    
    - 先执行 Deletion 删除操作：销毁需要删除的 DOM 节点，执行类组件的`componentWillUnmount`，解绑 ref，清理 hooks；
        
    - 再执行 Update 更新操作：更新 DOM 的属性、样式、文本内容；
        
    - 最后执行 Placement 插入操作：将新创建的 DOM 节点插入到父容器中；
        
- 完成所有 DOM 变更后，将 workInProgress 树切换为 current 树，完成 UI 的最终更新。
    

#### 3. Layout 阶段（DOM 变更后）

- 核心工作：DOM 已经更新完成，执行所有依赖最新 DOM 的操作；
    
- 类组件：同步执行`componentDidMount`、`componentDidUpdate`生命周期；
    
- 函数组件：同步执行`useLayoutEffect`的回调函数；
    
- 更新 ref 引用，指向最新的 DOM 节点 / 组件实例；
    
- 这个阶段可以同步读取最新的 DOM 布局信息，不会触发页面闪烁。
    

**注意**：我们常用的`useEffect`回调，会在 Layout 阶段完成后，浏览器绘制完成，再异步执行，不会阻塞浏览器的渲染。

## 五、Fiber 架构对开发的核心影响与最佳实践

### 1. 生命周期的废弃与规范

Render 阶段可中断、可重启，意味着 Render 阶段的生命周期可能会被**多次执行**，如果在其中包含副作用（如 API 请求、修改全局变量、setState），会导致重复执行、内存泄漏、数据异常等问题。

这也是 React 废弃以下生命周期的核心原因：

- `componentWillMount`
    
- `componentWillReceiveProps`
    
- `componentWillUpdate`
    

**最佳实践**：

- 所有副作用都放在`componentDidMount`/`componentDidUpdate`/`useEffect`/`useLayoutEffect`中执行；
    
- Render 阶段的代码（组件函数体、render 函数、shouldComponentUpdate）必须保持纯净，无副作用。
    

### 2. 并发渲染的使用规范

React 18 通过`createRoot`开启并发渲染后，Fiber 的优先级调度、可中断渲染能力才会完全生效。此时需要注意：

- 被标记为低优先级的更新（如`startTransition`包裹的内容），可能会被高优先级更新打断、多次重启，对应的组件函数可能会多次执行；
    
- 不要依赖更新的执行顺序，不要在组件函数体中执行不可重复的操作。
    

**最佳实践**：

- 用`useTransition`/`useDeferredValue`将非紧急的更新（如大数据列表渲染、搜索结果过滤）标记为低优先级，避免阻塞用户输入；
    
- 保持组件函数的纯净性，确保多次执行不会产生异常。
    

### 3. 性能优化的核心变化

Fiber 架构下，React 的性能优化重点从 “减少 Diff 次数”，转变为 “减少单个 Fiber 节点的执行时间”：

- 单个组件的 render 函数执行时间过长，依然会占用主线程，导致时间片耗尽，影响响应速度；
    
- 对于大列表，使用虚拟列表（react-window/react-virtualized）减少 Fiber 节点的数量，是最有效的优化方式；
    
- `memo`/`useMemo`/`useCallback`依然有效，核心作用是减少不必要的 Fiber 节点重建和 Diff 计算。
    

## 六、常见误区澄清

1. **Fiber 不是新的 Diff 算法**：Fiber 是整个协调架构的重写，Diff 算法只是 Render 阶段的一部分，核心的同层比较、key 优化规则并未改变。
    
2. **Fiber 不是默认开启并发渲染**：React 16-17 的 Legacy 模式下，Render 阶段依然是同步执行的，只有 React 18 的`createRoot`开启的并发模式，才会真正启用可中断的并发渲染。
    
3. **Fiber 不会让更新速度更快**：Fiber 的核心目标不是让更新更快，而是让更新更流畅，避免主线程阻塞，提升用户交互的响应性。极端情况下，并发更新的总执行时间会比同步更新更长，但用户感知会更流畅。
    
4. **Fiber 不能破解 DRM 加密内容**：这里补充之前的流媒体相关，Fiber 是 React 的渲染架构，和流媒体下载无关，之前的内容是独立的。
    

## 七、总结

React Fiber 的本质，是一套**基于链表结构的、可中断、带优先级的任务调度与执行架构**。它通过将大的渲染任务拆分为最小工作单元，配合双缓存机制、时间切片、Lanes 优先级调度，彻底解决了同步渲染的阻塞问题，让 React 能够与浏览器的渲染节奏完美配合，优先响应用户交互，实现更流畅的 UI 体验。

同时，Fiber 架构也为 React 后续的所有高级特性（并发渲染、Suspense、Server Components、流式 SSR）提供了底层支撑，让 React 从一个 UI 库，进化为一套完整的跨平台渲染架构体系。