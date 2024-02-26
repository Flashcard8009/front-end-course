通过前几节的学习，我们已经掌握了 **React Hooks API** 的使用，掌握了自定义 **Hooks** 的开发流程，接下来将正式进行源码的学习。

我们知道，如果 React 并没有 Hooks，那么函数式组件只能接收 props，渲染 UI，做一个展示组件，所有的逻辑就要在 Class
中书写，这样势必会导致 Class 组件内部错综复杂、代码臃肿。而函数式组件则不然，它能做 Class
组件的功能，拥有属于自己的状态，处理一些副作用，获取目标元素的属性、缓存数据等，所以有必要做一套函数式组件代替类组件的方案，Hooks 也就诞生了。

Hooks 拥有属于自己的状态，提供了 useState 和 useReducer 两个 Hook，解决自身的状态问题，取代 Class 组件的
this.setState。

在我们日常工作中最常用的就是 useState，我们就从它的源码入手，了解函数式组件是如何拥有自身的状态，如何保存数据、更新数据的。全面掌握
useState 的运行流程，就等同于掌握整个 Hooks 的运行机制。

先附上一张今天的知识图谱：

![](img\10\1.image)

# 当引入 useState 后发生了什么？

先举个例子：

    
    
    import { Button } from "antd";
    import { useState } from "react";
    const Index = () => {
      const [count, setCount] = useState(0);
      return (
        <><div>大家好，我是小杜杜，一起玩转Hooks吧！</div><div>数字：{count}</div><Button onClick={() => setCount((v) => v + 1)}>点击加1</Button></>
      );
    };
    
    export default Index;
    

在上述的例子中，我们引入了 useState，并存储 count 变量，通过 setCount 来控制 count。也就是说，count
是函数式组件自身的状态，setCount 是触发数据更新的函数。

在通常的开发中，当引入组件后，会从引用地址跳到对应引用的组件，查看该组件到底是如何书写的。

我们以相同的方式来看看 useState，看看它在 React 中是如何书写的。

文件位置：`packages/react/src/ReactHooks.js`。

    
    
    export function useState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      const dispatcher = resolveDispatcher();
      return dispatcher.useState(initialState);
    }
    

可以看出 useState 的执行就等价于
`resolveDispatcher().useState(initialState)`，那么我们顺着线索看下去：

**resolveDispatcher()** ：

    
    
    function resolveDispatcher() {
      const dispatcher = ReactCurrentDispatcher.current;
      return ((dispatcher: any): Dispatcher);
    }
    

**ReactCurrentDispatcher** ：

文件位置：`packages/react/src/ReactCurrentDispatcher.js`。

    
    
    const ReactCurrentDispatcher = {
      current: (null: null | Dispatcher),
    };
    

通过类型可以看到 `ReactCurrentDispatcher` 不是 null，就是 Dispatcher，而在初始化时
`ReactCurrentDispatcher.current` 的值必为 null，因为此时还 **未进行操作** 。

那么此时就很奇怪了，我们并没有发现 useState 是如何进行存储、更新的，`ReactCurrentDispatcher.current` 又是何时为
`Dispatcher` 的？

既然我们在 useState 自身中无法看到存储的变量，那么就只能从 **函数执行** 开始，一步一步探索 useState 是如何保存数据的。

# 函数式组件如何执行的？

在上节 Fiber 的讲解中，了解到我们写的 JSX 代码，是被 babel 编译成 React.createElement 的形式后，最终会走到
beginWork 这个方法中，而 beginWork 会走到 mountIndeterminateComponent 中，在这个方法中会有一个函数叫
**renderWithHooks** 。

**renderWithHooks 就是所有函数式组件触发函数** ，接下来一起看看：

文件位置：`packages/react-reconciler/src/ReactFiberHooks`。

    
    
    export function renderWithHooks<Props, SecondArg>(
      current: Fiber | null,
      workInProgress: Fiber,
      Component: (p: Props, arg: SecondArg) => any,
      props: Props,
      secondArg: SecondArg,
      nextRenderLanes: Lanes,
    ): any {
      currentlyRenderingFiber = workInProgress;
    
      // memoizedState: 用于存放hooks的信息，如果是类组件，则存放state信息
      workInProgress.memoizedState = null;
      //updateQueue：更新队列，用于存放effect list，也就是useEffect产生副作用形成的链表
      workInProgress.updateQueue = null;
    
      // 用于判断走初始化流程还是更新流程
      ReactCurrentDispatcher.current =
        current === null || current.memoizedState === null
          ? HooksDispatcherOnMount
          : HooksDispatcherOnUpdate;
    
      // 执行真正的函数式组件，所有的hooks依次执行
      let children = Component(props, secondArg);
    
      finishRenderingHooks(current, workInProgress);
    
      return children;
    }
    
    function finishRenderingHooks(current: Fiber | null, workInProgress: Fiber) {
        
      // 防止hooks乱用，所报错的方案
      ReactCurrentDispatcher.current = ContextOnlyDispatcher;
    
      const didRenderTooFewHooks =
        currentHook !== null && currentHook.next !== null;
    
      // current树
      currentHook = null;
      workInProgressHook = null;
    
      didScheduleRenderPhaseUpdate = false;
    }
    

> PS：展示的代码有稍许加工。

我们先分析下 **renderWithHooks** 函数的入参。

  * **current：** 即 current fiber，渲染完成时所生成的 current 树，之后在 **commit 阶段替换为真正的 DOM 树** ；
  * **workInProgress：** 即 workInProgress fiber，当更新时， **复制 current fiber，从这棵树进行更新，更新完毕后，再赋值给 current 树** ；
  * **Component：** 函数组件本身；
  * **props：** 函数组件自身的 props；
  * **secondArg：** 上下文；
  * **nextRenderLanes：** 渲染的优先级。

> 问：Fiber 架构的三个阶段分别是什么？
>
> 答：总共分为 reconcile、schedule、commit 阶段。
>
>   * **reconcile** 阶段： vdom 转化为 fiber 的过程。
>   * **schedule** 阶段：在 fiber 中遍历的过程中，可以打断，也能再恢复的过程。
>   * **commit** 阶段：fiber 更新到真实 DOM 的过程。
>

## renderWithHooks 的执行流程

  1. 在每次函数组件执行之前，先将 workInProgress 的 memoizedState 和 updateQueue 属性进行清空，之后将新的 Hooks 信息挂载到这两个属性上，之后在 commit 阶段替换 current 树，也就是说 **current 树保存 Hooks 信息；**
  2. 然后通过判断 current 树是否存在来判断走初始化（ HooksDispatcherOnMount ）流程还是更新（ HooksDispatcherOnUpdate ）流程。而 ReactCurrentDispatcher.current 实际上包含所有的 Hooks，简单地讲， **React 根据 current 的不同来判断对应的 Hooks，从而监控 Hooks 的调用情况；**
  3. 接下来调用的 **Component(props, secondArg) 就是真正的函数组件** ，然后依次执行里面的 Hooks；
  4. 最后提供整个的 **异常处理** ，防止不必要的报错，再将一些属性置空，如：currentHook、workInProgressHook 等。

通过 renderWithHooks 的执行步骤，可以看出总共分为三个阶段，分别是： **初始化阶段** 、 **更新阶段** 以及 **异常处理**
三个阶段，同时这三个阶段也是整个 Hooks 处理的 **三种策略** ，接下来我们逐一分析。

# HooksDispatcherOnMount（初始化阶段）

在初始化阶段中，调用的是 `HooksDispatcherOnMount`，对应的 useState 所走的是 `mountState`。

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
     // 包含所有的hooks，这里列举常见的
    const HooksDispatcherOnMount = { 
        useRef: mountRef,
        useMemo: mountMemo,
        useCallback: mountCallback,
        useEffect: mountEffect,
        useState: mountState,
        useTransition: mountTransition,
        useSyncExternalStore: mountSyncExternalStore,
        useMutableSource: mountMutableSource,
        ...
    }
    
    function mountState(initialState){
      // 所有的hooks都会走这个函数
      const hook = mountWorkInProgressHook(); 
      
      // 确定初始入参
      if (typeof initialState === 'function') {
        // $FlowFixMe: Flow doesn't like mixed types
        initialState = initialState();
      }
      hook.memoizedState = hook.baseState = initialState;
      
      const queue = {
        pending: null,
        lanes: NoLanes,
        dispatch: null,
        lastRenderedReducer: basicStateReducer,
        lastRenderedState: (initialState),
      };
      hook.queue = queue;
      
      const dispatch = (queue.dispatch = (dispatchSetState.bind(
        null,
        currentlyRenderingFiber,
        queue,
      ): any));
      return [hook.memoizedState, dispatch];
    }
    

## mountWorkInProgressHook

整体的流程先走向 `mountWorkInProgressHook()` 这个函数，它的作用尤为重要，因为这个函数的作用是将 **Hooks 与 Fiber
联系起来** ，并且你会发现，所有的 **Hooks 都会走这个函数，只是不同的 Hooks 保存着不同的信息** 。

    
    
    function mountWorkInProgressHook(): Hook {
      const hook: Hook = {
        memoizedState: null,
        baseState: null,
        baseQueue: null,
        queue: null,
        next: null,
      };
    
      if (workInProgressHook === null) { // 第一个hooks执行
        currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
      } else { // 之后的hooks
        workInProgressHook = workInProgressHook.next = hook;
      }
      return workInProgressHook;
    }
    

来看看 **hook** 值的参数：

  * **memoizedState** ：用于保存数据，不同的 Hooks 保存的信息不同，比如 useState 保存 state 信息，useEffect 保存 effect 对象，useRef 保存 ref 对象；
  * **baseState** ：当数据发生改变时，保存最新的值；
  * **baseQueue** ：保存最新的更新队列；
  * **queue** ：保存待更新的队列或更新的函数；
  * **next** ：用于指向下一个 hook 对象。

那么 `mountWorkInProgressHook` 的作用就很明确了， **每执行一个 Hooks 函数就会生成一个 hook 对象，然后将每个
hook 串联起来** 。

> 特别注意：这里的 memoizedState 并不是 Fiber 链表上的 memoizedState，workInProgress
> 保存的是当前函数组件每个 Hooks 形成的链表。

## 执行步骤

了解完 mountWorkInProgressHook 后，再来看看之后的流程。

首先通过 `initialState` 初始值的类型（判断是否是函数），并将初始值赋值给 hook 的`memoizedState` 和
`baseState`。再之后，创建一个 `queue` 对象，这个对象中会保存一些数据，这些数据为：

  * **pending** ：用来调用 dispatch 创建时最后一个；
  * **lanes** ：优先级；
  * **dispatch** ：用来负责更新的函数；
  * **lastRenderedReducer** ：用于得到最新的 state；
  * **lastRenderedState** ：最后一次得到的 state。

最后会定义一个 `dispath`，而这个 dispath 就应该对应最开始的 `setCount`，那么接下来的目的就是搞懂 dispatch 的机制。

## dispatchSetState

dispatch 的机制就是 `dispatchSetState`，在源码内部还是调用了很多函数，所以在这里对 dispatchSetState
函数做了些优化，方便我们更好地观看。

    
    
    function dispatchSetState<S, A>(
      fiber: Fiber, // 对应currentlyRenderingFiber
      queue: UpdateQueue<S, A>, // 对应 queue
      action: A, // 真实传入的参数
    ): void {
    
      // 优先级，不做介绍，后面也会去除有关优先级的部分
      const lane = requestUpdateLane(fiber);
    
      // 创建一个update
      const update: Update<S, A> = {
        lane,
        action,
        hasEagerState: false,
        eagerState: null,
        next: (null: any),
      };
    
       // 判断是否在渲染阶段
      if (fiber === currentlyRenderingFiber || (fiber.alternate !== null && fiber.alternate === currentlyRenderingFiber)) {
          didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
          const pending = queue.pending;
          // 判断是否是第一次更新
          if (pending === null) {
            update.next = update;
          } else {
            update.next = pending.next;
            pending.next = update;
          }
          // 将update存入到queue.pending中
          queue.pending = update;
      } else { // 用于获取最新的state值
        const alternate = fiber.alternate;
        if (alternate === null && lastRenderedReducer !== null){
          const lastRenderedReducer = queue.lastRenderedReducer;
          let prevDispatcher;
          const currentState: S = (queue.lastRenderedState: any);
          // 获取最新的state
          const eagerState = lastRenderedReducer(currentState, action);
          update.hasEagerState = true;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) return;
        }
    
        // 将update 插入链表尾部，然后返回root节点
        const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);
        if (root !== null) {
          // 实现对应节点的更新
          scheduleUpdateOnFiber(root, fiber, lane, eventTime);
        }
      }
    }
    

在代码中，我已经将每段代码执行的目的标注出来，为了我们更好地理解，分析一下对应的入参，以及函数体内较重要的参数与步骤。

  1. **分析入参** ：dispatchSetState 一共有三个入参，前两个入参数被 `bind` 分别改为 currentlyRenderingFiber 和 queue，第三个 action 则是我们实际写的函数；
  2. **update 对象** ：生成一个 update 对象，用于记录更新的信息；
  3. **判断是否处于渲染阶段** ：如果是渲染阶段，则将 `update` 放入等待更新的 `pending` 队列中，如果不是，就会获取最新的 `state` 值，从而进行更新。

> 问：bind 的作用是什么？
>
> 答：当函数调用 bind 后，会产生一个新的函数，第一个值会作为新函数的 this，如果第一个参数为 null 或是 undefined 时，会默认指向
> window，其余的参数会依次成为旧函数的参数。

值得注意的是：在更新过程中，也会判断很多，通过调用 `lastRenderedReducer` 获取最新的 `state`，然后进行 **比较（浅比较）**
，如果相等则退出，这一点就是证明 useState 渲染相同值时，组件不更新的原因。

如果不相等，则会将 `update` 插入链表的尾部，返回对应的 `root` 节点，通过 **scheduleUpdateOnFiber
实现对应的更新** ，可见 `scheduleUpdateOnFiber` 是 React 渲染更新的主要函数。

# HooksDispatcherOnUpdate（更新阶段）

在更新阶段时，调用 `HooksDispatcherOnUpdate`，对应的 `useState` 所走的是 `updateState`，如：

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
    const HooksDispatcherOnUpdate: Dispatcher = {
      useRef: updateRef,
      useMemo: updateMemo,
      useCallback: updateCallback,
      useEffect: updateEffect,
      useState: updateState,
      useTransition: updateTransition,
      useSyncExternalStore: updateSyncExternalStore,
      useMutableSource: updateMutableSource,
      ...
    };
    
    function updateState<S>(
      initialState: (() => S) | S,
    ): [S, Dispatch<BasicStateAction<S>>] {
      return updateReducer(basicStateReducer, (initialState: any));
    }
    
    function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
      return typeof action === 'function' ? action(state) : action;
    }
    

在 `updateState` 有两个函数，一个是 `updateReducer`，另一个是 `basicStateReducer`。

`basicStateReducer` 很简单，判断是否是函数，返回对应的值即可。

那么下面主要看 `updateReducer` 这个函数，在 updateReducer 函数中首先调用
`updateWorkInProgressHook`，我们先来看看这个函数，方便后续对 updateReducer 的理解。

## updateWorkInProgressHook

`updateWorkInProgressHook` 跟 `mountWorkInProgressHook` 一样， **当函数更新时，所有的 Hooks
都会执行** 。

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
    function updateWorkInProgressHook(): Hook {
      let nextCurrentHook: null | Hook;
      
      // 判断是否是第一个更新的hook
      if (currentHook === null) { 
        const current = currentlyRenderingFiber.alternate;
        if (current !== null) {
          nextCurrentHook = current.memoizedState;
        } else {
          nextCurrentHook = null;
        }
      } else { // 如果不是第一个hook，则指向下一个hook
        nextCurrentHook = currentHook.next;
      }
    
      let nextWorkInProgressHook: null | Hook;
      // 第一次执行
      if (workInProgressHook === null) { 
        nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
      } else {
        nextWorkInProgressHook = workInProgressHook.next;
      }
    
      if (nextWorkInProgressHook !== null) {
        // 特殊情况：发生多次函数组件的执行
        workInProgressHook = nextWorkInProgressHook;
        nextWorkInProgressHook = workInProgressHook.next;
        currentHook = nextCurrentHook;
      } else {
        if (nextCurrentHook === null) {
          const currentFiber = currentlyRenderingFiber.alternate;
          
          const newHook: Hook = {
            memoizedState: null,
            baseState: null,
            baseQueue: null,
            queue: null,
            next: null,
          };
            nextCurrentHook = newHook;
          } else {
            throw new Error('Rendered more hooks than during the previous render.');
          }
        }
    
        currentHook = nextCurrentHook;
    
        // 创建一个新的hook
        const newHook: Hook = {
          memoizedState: currentHook.memoizedState,
          baseState: currentHook.baseState,
          baseQueue: currentHook.baseQueue,
          queue: currentHook.queue,
          next: null,
        };
    
        if (workInProgressHook === null) { // 如果是第一个函数
          currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
        } else {
          workInProgressHook = workInProgressHook.next = newHook;
        }
      }
      return workInProgressHook;
    }
    

**updateWorkInProgressHook 执行流程** ：如果是首次执行 Hooks 函数，就会从已有的 **current**
树中取到对应的值，然后声明 `nextWorkInProgressHook`，经过一系列的操作，得到更新后的 Hooks 状态。

在这里要注意一点，大多数情况下，`workInProgress` 上的 `memoizedState` 会被置空，也就是
`nextWorkInProgressHook` 应该为 null。但 **执行多次** 函数组件时，就会出现循环执行函数组件的情况，此时
`nextWorkInProgressHook` 不为 null。

## updateReducer

掌握了 `updateWorkInProgressHook`执行流程后， 再来看 `updateReducer` 具体有哪些内容。

    
    
    function updateReducer<S, I, A>(
      reducer: (S, A) => S,
      initialArg: I,
      init?: I => S,
    ): [S, Dispatch<A>] {
    
      // 获取更新的hook，每个hook都会走
      const hook = updateWorkInProgressHook();
      const queue = hook.queue;
    
      queue.lastRenderedReducer = reducer;
    
      const current: Hook = (currentHook: any);
    
      let baseQueue = current.baseQueue;
     
      // 在更新的过程中，存在新的更新，加入新的更新队列
      const pendingQueue = queue.pending;
      if (pendingQueue !== null) {
        // 如果在更新过程中有新的更新，则加入新的队列，有个合并的作用，合并到 baseQueue
        if (baseQueue !== null) {
          const baseFirst = baseQueue.next;
          const pendingFirst = pendingQueue.next;
          baseQueue.next = pendingFirst;
          pendingQueue.next = baseFirst;
        }
        current.baseQueue = baseQueue = pendingQueue;
        queue.pending = null;
      }
    
      if (baseQueue !== null) {
        const first = baseQueue.next;
        let newState = current.baseState;
    
        let newBaseState = null;
        let newBaseQueueFirst = null;
        let newBaseQueueLast = null;
        let update = first;
        
        // 循环更新
        do {
          // 获取优先级
          const updateLane = removeLanes(update.lane, OffscreenLane);
          const isHiddenUpdate = updateLane !== update.lane;
    
          const shouldSkipUpdate = isHiddenUpdate
            ? !isSubsetOfLanes(getWorkInProgressRootRenderLanes(), updateLane)
            : !isSubsetOfLanes(renderLanes, updateLane);
    
          if (shouldSkipUpdate) {
            const clone: Update<S, A> = {
              lane: updateLane,
              action: update.action,
              hasEagerState: update.hasEagerState,
              eagerState: update.eagerState,
              next: (null: any),
            };
            if (newBaseQueueLast === null) {
              newBaseQueueFirst = newBaseQueueLast = clone;
              newBaseState = newState;
            } else {
              newBaseQueueLast = newBaseQueueLast.next = clone;
            }
            
            // 合并优先级（低级任务）
            currentlyRenderingFiber.lanes = mergeLanes(
              currentlyRenderingFiber.lanes,
              updateLane,
            );
            markSkippedUpdateLanes(updateLane);
          } else {
             // 判断更新队列是否还有更新任务
            if (newBaseQueueLast !== null) {
              const clone: Update<S, A> = {
                lane: NoLane,
                action: update.action,
                hasEagerState: update.hasEagerState,
                eagerState: update.eagerState,
                next: (null: any),
              };
              
              // 将更新任务插到末尾
              newBaseQueueLast = newBaseQueueLast.next = clone;
            }
    
            const action = update.action;
            
            // 判断更新的数据是否相等
            if (update.hasEagerState) {
              newState = ((update.eagerState: any): S);
            } else {
              newState = reducer(newState, action);
            }
          }
          // 判断是否还需要更新
          update = update.next;
        } while (update !== null && update !== first);
    
        // 如果 newBaseQueueLast 为null，则说明所有的update处理完成，对baseState进行更新
        if (newBaseQueueLast === null) {
          newBaseState = newState;
        } else {
          newBaseQueueLast.next = (newBaseQueueFirst: any);
        }
    
        // 如果新值与旧值不想等，则触发更新流程
        if (!is(newState, hook.memoizedState)) {
          markWorkInProgressReceivedUpdate();
        }
    
        // 将新值，保存在hook中
        hook.memoizedState = newState;
        hook.baseState = newBaseState;
        hook.baseQueue = newBaseQueueLast;
    
        queue.lastRenderedState = newState;
      }
    
      if (baseQueue === null) {
        queue.lanes = NoLanes;
      }
    
      const dispatch: Dispatch<A> = (queue.dispatch: any);
      return [hook.memoizedState, dispatch];
    }
    

updateReducer 的作用是将待更新的队列 pendingQueue 合并到 baseQueue
上，之后进行循环更新，最后进行一次合成更新，也就是批量更新，统一更换节点。

这种行为解释了 useState 在更新的过程中为何传入相同的值，不进行更新，同时多次操作，只会执行最后一次更新的原因了。

## 更新 state 值

为了更好理解更新流程，我们做一个简单的例子来说明：

    
    
    function Index() {
      const [count, setCount] = useState(0);
    
      return (
        <div style={{ padding: 20 }}>
          <div>数字：{count}</div>
          <Button
            onClick={() => {
              // 第一种方式
              setCount((v) => v + 1);
              setCount((v) => v + 2);
              setCount((v) => v + 3);
    
              // 第二种方式
              setCount(count + 1);
              setCount(count + 2);
              setCount(count + 3);
            }}
          >
            批量执行
          </Button>
        </div>
      );
    }
    
    export default Index;
    

案例中就是普通的点击按钮，触发 count 变化的操作，那么大家可以猜想下，这两种方式点击按钮后的 count 的值究竟是多少？

答案：

  * 第一种 count 等于： **6** ；
  * 第二种 count 等于： **3** 。

出现这种原因也非常简单，当 setCount 的参数为函数时，此时的返参 v 就是 `baseQueue` 链表不断更新的值，所以为 0 + 1 + 2 +
3 = 6。

![图片1.png](img\10\2.image)

而第二种的 count 为渲染后的值，也就是说，三个 setCount 全部执行完成，合并之后，count 才会变，在合并前为 0 + 1， 0 + 2，
0 + 3。最后一次为 3，所以 count 为 3。

# ContextOnlyDispatcher 异常处理阶段

在 `renderWithHooks` 流程最后，调用了 `finishRenderingHooks` 函数，这个函数中用到了
`ContextOnlyDispatcher`，那么它的作用是什么呢？看看代码：

![](img\10\3.image)

**throwInvalidHookError：**

    
    
    function throwInvalidHookError() {
      throw new Error(
        'Invalid hook call. Hooks can only be called inside of the body of a function component. This could happen for' +
          ' one of the following reasons:\n' +
          '1. You might have mismatching versions of React and the renderer (such as React DOM)\n' +
          '2. You might be breaking the Rules of Hooks\n' +
          '3. You might have more than one copy of React in the same app\n' +
          'See https://reactjs.org/link/invalid-hook-call for tips about how to debug and fix this problem.',
      );
    }
    

可以看到，`ContextOnlyDispatcher` 是判断所需 `Hooks` 是否在函数组件内部，有 **捕获并抛出异常的作用** ，这也就解释了
Hooks 无法在 React 之外运行的原因。

# useState 运行流程

我们以 **useState** 为例，讲解了对应的初始化和更新，简单回顾一下运行流程：

![](img\10\4.image)

# Hooks 规则：时序问题

了解完 useState 源码，以及存储、更新、异常的处理方案，我们发现其中有一个问题点，在我们多次调用 useState 的时候，React
是如何知道我们要改变的 useState 就是想要改变的 useState 呢？如：

    
    
    const [name, setName] = useState("小杜杜")
    const [age, setAge] = useState(0)
    
    useEffect(() => {}, [])
    

这两个 useState 只有参数上的区别，那么 React 是如何区分是 name 还是 age 呢？

答案其实很简单，就是 **时序** ，React 相当于做了一个合并操作，当我们第一次调用 **useState** 时，保存了 **name**
，第二次调用时保存了 **age** ，相当于类中的结构。

    
    
    this.setState({
        name: "小杜杜",
        age: 7
    })
    

当然，在 `mountWorkInProgressHook` 讲解中说过，所有的 Hooks 在创建时，都会产生对应的 hook 对象，当有多个 Hooks
时会以 `next` 连接起来。

当初始化完成后，对应的结构应该是：

![图片1 \(2\).png](img\10\5.image)

同时，在 Hooks 的规则中有这么一条： **只在最顶层使用 Hook，不要在循环、条件或嵌套函数中使用 Hook** 。

那如果就把它放在条件中，会发生什么变化呢？

    
    
    let age, setAge
    if(name! == "小杜杜"){
       [age, setAge] = useState(0)
    }
    
    useEffect(() => {}, [])
    

在初始化中 name 为小杜杜，但当 name 改变时，便没有了 age，看看此时的报错：

![](img\10\6.image)

造成这样的结果，是因为在新的中缺少了 `useState2`，换句话说，在更新状态的时候，表的结构遭到了 **破坏，让原本指向 useState2
的，指向到 useEffect** 。

从源码的角度来讲，`current` 树的 `memoizedState` 缓存 `hook` 信息，和当前的 `workInProgress`
不一致，此时就会发生异常。这也是 **不能在条件语句中创建的原因。**

> 注：另外可以在 Fiber 中的 `_debugHookTypes` 属性中查看调用 Hooks 的顺序。

# Hooks 的实现与 Fiber 有必然联系吗？

最终 Hooks 存储的数据保存在 Fiber 中，Hooks 的产生也确实在 Fiber 的基础上，那么 Hooks 与 Fiber 的关系是必然的吗？

从 React 的角度出发，整个的渲染流程中是通过 Fiber 去进行转化的，流程为： **jsx => vdom => Fiber => 真实 DOM**
。

而 **Hooks 对应 Fiber 中的 memorizedState 链表，依靠 next 链接，只是不同的 hooks 对应保存的值不同而已。**
换言之，可以把 Fiber 当作保存 Hooks 值的容器，但这与本身是否依赖 Fiber 并没有太大的联系。

就好比 `preact` 中的 Hooks，它并没有实现 Fiber 架构，但也同样实现了 Hooks，它把 Hooks 链表放在了
`vnode._component._hooks` 属性上。

总的来说 ：实现 Hooks 与 Fiber 并没有必然的联系，相反，只要有对应保存的地方就 ok 了。

# 小结

在本节中，我们首先发现在 useState 自身并不存在对应的存储关系，所以只能从函数执行的过程中去寻找答案，最终发现所有的答案都在
`renderWithHooks` 中，并发现 React 主要通过三大策略去处理 Hooks。

这三大策略分别是：HooksDispatcherOnMount（ **初始化阶段，与 Fiber 之间产生关联**
）、HooksDispatcherOnUpdate（ **更新阶段，与 Fiber 的桥梁已建立，对桥梁的维护** ） 和
ContextOnlyDispatcher（ **异常处理阶段，判断是否在 React 内部调用，如果不是，则抛出异常** ）。

其中，初始化阶段和更新阶段分别调用：`mountWorkInProgressHook` 和 `updateWorkInProgressHook`
函数，同时也是所有 `Hooks` 所走的函数。

在文章开头提过 Hooks 是依靠 useState 或 useReducer 进行自身状态的管理，而在 useState 的更新中运用到了
`updateReducer`， 这是因为两者的原理非常相似，感兴趣的同学再去看看 useReducer 的源码，相信阅读起来不会有任何问题。

从三大策略整体来看， **所有的 Hooks 都遵从上面的逻辑** ，只是在三大策略中执行的方法有所出入，因此从 useState 的视角来看，就是整个
Hooks 的运行机制。

总的来说，源码内容还是有难度的，如果你是第一次接触，建议多读几遍、沉下心，一步一步去看，不要着急，切记： **一口吃不吃胖子** 。

