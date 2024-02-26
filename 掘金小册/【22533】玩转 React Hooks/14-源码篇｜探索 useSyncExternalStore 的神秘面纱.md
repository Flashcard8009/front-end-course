大家好，我是小杜杜，在 React v18 中提供了一个全新的 Hooks： **useSyncExternalStore** ，它会通过
**强制的同步状态** 更新，使得外部 `store` 可以支持并发读取。

实际上 useSyncExternalStore 是 useMutableSource 演变而来，主要解决 **外部数据撕裂**
的问题，并且官方明确指出它是提供给三方库（如：`redux`、`mobx`）使用，而非日常开发中使用。

但在 React 文档中（[Subscribing to a browser
API](https://react.dev/reference/react/useSyncExternalStore#subscribing-to-a-
browser-api)）发现这样一段话：

![image.png](img\14\1.image)

大致意思说：添加 useSyncExternalStore 另一个原因是使用浏览器的某些值时，这个值可能在 **将来某个时刻发生变化**
，如：网络连接的状态（[navigator.onLine](https://developer.mozilla.org/en-
US/docs/Web/API/Navigator/onLine)），此时更加推荐使用 useSyncExternalStore。

通过上面这段话，可以得出 useSyncExternalStore 解决外部数据撕裂中的“外部”不仅仅是“第三方库”，也有可能是“浏览器“。当我们需要访问
windows 对象上的一些值时，也需要它的帮助。

![useSyncExternalStore.png](img\14\2.image)

# useSyncExternalStore 使用示例

我们先来看看官网的示例：在 useSyncExternalStore 的基础上封装了 useOnlineStatus，去检查网络连接的状态：

    
    
    // useOnlineStatus
    import { useSyncExternalStore } from "react";
    
    export function useOnlineStatus() {
      const isOnline = useSyncExternalStore(subscribe, getSnapshot);
      return isOnline;
    }
    
    function getSnapshot() {
      return navigator.onLine;
    }
    
    function subscribe(callback: any) {
      window.addEventListener("online", callback);
      window.addEventListener("offline", callback);
      return () => {
        window.removeEventListener("online", callback);
        window.removeEventListener("offline", callback);
      };
    }
    
    
    
    // Index
    import { useOnlineStatus } from "./useOnlineStatus";
    
    function StatusBar() {
      const isOnline = useOnlineStatus();
      return <h1>{isOnline ? "✅ Online" : "❌ Disconnected"}</h1>;
    }
    
    function SaveButton() {
      const isOnline = useOnlineStatus();
    
      function handleSaveClick() {
        console.log("✅ Progress saved");
      }
    
      return (
        <button disabled={!isOnline} onClick={handleSaveClick}>
          {isOnline ? "Save progress" : "Reconnecting..."}
        </button>
      );
    }
    
    const Index = () => {
      return (
        <>
          <SaveButton />
          <StatusBar />
        </>
      );
    };
    
    export default Index;
    

**效果：**

![img1.gif](img\14\3.image)

> useSyncExternalStore API 在第四章中介绍过，这里就不过多赘述。

useSyncExternalStore 解决的问题是 **数据撕裂** 问题，那么什么是数据撕裂呢？一起来看看。

## 什么是数据撕裂？

**撕裂：** 是图形编程中的一个传统术语，是指视觉上的不一致（参考：[#What is
tearing?](https://github.com/reactwg/react-18/discussions/69)）。

在 React v18 中增加 **并发** 机制，换句话说，React 由之前的 **同步渲染** 变为了 **并发渲染**
，接下来一起看看两者在渲染上的区别。

**同步渲染：**

当我们渲染 React 树时，通过 external store 提供数据，如下图：

![image.png](img\14\4.image)

同步渲染流程如下：

  1. 第一张图，当 external store 的数据变为蓝色，React 树开始渲染，对应的组件变为了蓝色。
  2. 第二张图，由于 JS 是单线程的，所以会一直执行下去，此时的组件都会取到 external store 对应的颜色。所以在第三张图中，我们可以看到所有的组件都渲染成了蓝色，UI 显示的状态始终与 external store 的颜色一致。
  3. 第四张图， **当 React 渲染完成后，才允许改变 external store 的值。** 如果 store 在 React 未渲染时更新，此时将进行下一次渲染，继续循环这个过程。

大多数 UI 框架（包括 React v17 版本）都遵从同步渲染的流程，所渲染的 UI 也总是一致的。但在 React v18 上增加了 **并发机制**
，程序并不一定执行下去，会有中断的可能。

**并发渲染：**

在并发模式下，程序并不会一直执行下去，当 external store 渲染组件变为蓝色的过程中，用户也可以改变 store
中的值，让用户感受到页面更加丝滑。如下图：

![image.png](img\14\5.image)

并发渲染流程如下：

  1. 第一张图中，external store 的值为蓝色，渲染的组件也为蓝色。
  2. 在执行的过程中，将 external store 的值改为红色，此时再渲染剩余的组件，因为 store 发生变化，所以剩余的组件也变成了红色。
  3. 第四张图，当渲染完成后，发现一个组件是蓝色，另外两个组件是红色，它们虽然读取相同的数据，但却是不同的值，此时所渲染的 UI 并不是统一的，这种情况就是 **“撕裂”** 。

## 为什么不能用 useState 和 useEffect 代替？

在示例中，我们用 useSyncExternalStore 来监听网络的状态，这种方式明显比较麻烦，为什么不能用 useState 和 useEffect
来代替呢（如：之前介绍的 useNetwork）？

其本质原因跟 React v18 的并发机制有关，也就是并发渲染。因为通过并发渲染，React 会维护不同的 UI，一个是屏幕展示（current
fiber），另一个是准备更新的树（workInProgress fiber），同时为了让用户体验更加丝滑，React
允许暂停优先级低的事件，优先处理优先级高的响应事件。

所以，在一次渲染的过程中，处理事件前后获取的外部 store 有可能不同，如果使用自身的状态，React 无法对此感知，这时就会触发撕裂的情况，即同一个
state 渲染出了不同的值。

而 useSyncExternalStore 就是为了解决这类情况的出现。它会在渲染期间检测外部的 state 是否发生变化，如果展示的 UI
并不统一，会进行 **同步阻塞渲染** ，强制更新，使 UI 保持一致。

接下来，我们一起看看 useSyncExternalStore 的源码，共同揭开它神秘的面纱。

# useSyncExternalStore 原理

useSyncExternalStore 的源码分为两个阶段，分别是：mountSyncExternalStore（初始化阶段）和
updateSyncExternalStore（更新阶段）。

## mountSyncExternalStore（初始化阶段）

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
    function mountSyncExternalStore<T>(
      subscribe: (() => void) => () => void,
      getSnapshot: () => T,
      getServerSnapshot?: () => T,
    ): T {
      const fiber = currentlyRenderingFiber;
      const hook = mountWorkInProgressHook();
    
      let nextSnapshot;
      
      // 是否属于 hydrate 模式
      const isHydrating = getIsHydrating();
      if (isHydrating) {
        // hydrate 模式下
        nextSnapshot = getServerSnapshot();
      } else {
      
        nextSnapshot = getSnapshot();
        const root: FiberRoot | null = getWorkInProgressRoot();
    
        // 并发模式下，一致性检查
        if (!includesBlockingLane(root, renderLanes)) {
          pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);
        }
      }
    
      hook.memoizedState = nextSnapshot;
      
      const inst: StoreInstance<T> = {
        value: nextSnapshot,
        getSnapshot,
      };
      hook.queue = inst;
    
      // useEffect 中的 mountEffect
      mountEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [subscribe]);
    
      fiber.flags |= PassiveEffect;
      
      // 打上对应的标记，与useEffect中一样
      pushEffect(
        HookHasEffect | HookPassive,
        updateStoreInstance.bind(null, fiber, inst, nextSnapshot, getSnapshot),
        undefined,
        null,
      );
    
      return nextSnapshot;
    }
    

mountSyncExternalStore 对应三个入参，分别是：

  * **subscribe** ：订阅函数，用于 **注册一个回调函数，当存储值发生更改时被调用** ；
  * **getSnapshot** ：返回当前存储值的函数；
  * **getServerSnapshot** ：返回服务端（`hydration` 模式下）渲染期间使用的存储值的函数（这里我们绕过 hydration 模式下的处理）。

**mountSyncExternalStore 整体流程：**

  1. 首先拿到对应的 fiber 节点，创建一个 hook 对象，React 先判断当前的环境是不是 hydration 模式；
  2. 接下来生成 store 的快照，获取当前 store 的状态值，只是 hydration 模式下通过 getServerSnapshot 获取，否则通过 getSnapshot 获取。并将获取的状态值存储在对应的 memoizedState 中；
  3. 对 render 阶段结束时会对 store 进行一致性检查；
  4. 最后执行 mountEffect 和 pushEffect，这两步与 useEffect 的初始化步骤对应，打上对应的标记，在 commit 阶段进行一致性检查，防止 store 的状态不一致。

阅读完 mountSyncExternalStore，核心点有
pushStoreConsistencyCheck、subscribeToStore、updateSyncExternalStore
三个函数，接下来我们逐一进行分析。

### pushStoreConsistencyCheck

**pushStoreConsistencyCheck** ：检查一致性，如果是并发模式，会创建一个 check 对象，并添加到 fiber 中的
updateQueue 对象的 store 数组中。

    
    
    function pushStoreConsistencyCheck<T>(
      fiber: Fiber,
      getSnapshot: () => T,
      renderedSnapshot: T,
    ): void {
      fiber.flags |= StoreConsistency;
      
      const check: StoreConsistencyCheck<T> = {
        getSnapshot,
        value: renderedSnapshot,
      };
      
      let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
      
      if (componentUpdateQueue === null) { // 第一个 check 对象
        componentUpdateQueue = createFunctionComponentUpdateQueue();
        currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
        
        componentUpdateQueue.stores = [check];
      } else { // 多个 check 对象
        const stores = componentUpdateQueue.stores;
        
        if (stores === null) {
          componentUpdateQueue.stores = [check];
        } else {
          stores.push(check);
        }
      }
    }
    

从源码可以看出，收集 check 的过程和 useEffect 中收集 effect 对象类似，
createFunctionComponentUpdateQueue() 用来创建一个更新队列，最终放入 stores 数组中。

### subscribeToStore

**subscribeToStore：** 通过 store 提供的 subscribe 方法订阅对应的状态变化，如果发生变化，则会采用同步阻塞模式渲染。

    
    
    function subscribeToStore<T>(
      fiber: Fiber,
      inst: StoreInstance<T>,
      subscribe: (() => void) => () => void,
    ): any {
     // 通过 store 的 dispatch 方法修改 store 会触发
     const handleStoreChange = () => {
        if (checkIfSnapshotChanged(inst)) {
          forceStoreRerender(fiber);
        }
      };
      return subscribe(handleStoreChange);
    }
    
    // 判断 store 的值是否发生变化
    function checkIfSnapshotChanged<T>(inst: StoreInstance<T>): boolean {
      const latestGetSnapshot = inst.getSnapshot;
      
      // 旧值
      const prevValue = inst.value;
      try {
        // 新值
        const nextValue = latestGetSnapshot();
        // 与 useEffect 中的一致，进行浅比较
        return !is(prevValue, nextValue);
      } catch (error) {
        return true;
      }
    }
    
    // 使用阻塞模式渲染
    function forceStoreRerender(fiber: Fiber) {
      const root = enqueueConcurrentRenderForLane(fiber, SyncLane);
      if (root !== null) {
        scheduleUpdateOnFiber(root, fiber, SyncLane, NoTimestamp);
      }
    }
    

在 subscribeToStore 中，会进行一层判断：checkIfSnapshotChanged 函数，它会判断 store
是否发生变化，判断的依据也跟 useEffect 中的一致，通过 `is` 进行 **浅比较** ，如果发生了变化，则会执行
forceStoreRerender 方法，手动触发 Sync 阻塞渲染，处理优先级和挂载更新节点。

简单点说，我们通过 store 的 dispatch 修改内容时，store 会遍历依赖列表，按照顺序依次执行回调函数。

### updateStoreInstance

**updateStoreInstance：** 在 commit 阶段中，会统一处理 render 阶段的所有 effect，此时会再次检查 store
是否发生变化，防止 store 的状态不一致。

    
    
    function updateStoreInstance<T>(
      fiber: Fiber,
      inst: StoreInstance<T>,
      nextSnapshot: T,
      getSnapshot: () => T,
    ): void {
      inst.value = nextSnapshot;
      inst.getSnapshot = getSnapshot;
    
      // 在 commit 阶段中，检查 store 是否发生变化
      if (checkIfSnapshotChanged(inst)) {
        // 触发同步阻塞渲染
        forceStoreRerender(fiber);
      }
    }
    

## updateSyncExternalStore（更新阶段）

    
    
    function updateSyncExternalStore<T>(
      subscribe: (() => void) => () => void,
      getSnapshot: () => T,
      getServerSnapshot?: () => T,
    ): T {
      const fiber = currentlyRenderingFiber;
      
      // 获取更新的hooks
      const hook = updateWorkInProgressHook();
      
      // 获取新的 store 状态
      const nextSnapshot = getSnapshot();
      const prevSnapshot = (currentHook || hook).memoizedState;
      const snapshotChanged = !is(prevSnapshot, nextSnapshot);
      if (snapshotChanged) {
        hook.memoizedState = nextSnapshot;
        markWorkInProgressReceivedUpdate();
      }
      const inst = hook.queue;
    
      updateEffect(subscribeToStore.bind(null, fiber, inst, subscribe), [
        subscribe,
      ]);
    
      if (
        inst.getSnapshot !== getSnapshot ||
        snapshotChanged ||
        (workInProgressHook !== null &&
          workInProgressHook.memoizedState.tag & HookHasEffect)
      ) {
        fiber.flags |= PassiveEffect;
        pushEffect(
          HookHasEffect | HookPassive,
          updateStoreInstance.bind(null, fiber, inst, nextSnapshot, getSnapshot),
          undefined,
          null,
        );
    
        const root: FiberRoot | null = getWorkInProgressRoot();
    
        if (!includesBlockingLane(root, renderLanes)) {
          pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);
        }
      }
    
      return nextSnapshot;
    }
    

可以看出 updateSyncExternalStore 和 mountSyncExternalStore 的步骤基本类似，来看看对应的流程：

  1. 获取更新的 hooks 对象、新的 store 状态，存储到 memoizedState 中；
  2. 通过 updateEffect 方法在节点更新后执行对应的 subscribe 方法。与 useEffect 的更新方法对应，只不过这里检查的并不是 deps，而是 subscribe。也就是说，如果 subscribe 不发生改变，则不会执行；
  3. 接下来操作与 mountSyncExternalStore 一致，在 render 阶段结束时，commit 阶段会分别对 store 进行一致性检查，防止 store 的状态不一致。

# 实现 useSyncExternalStore

实际上 useSyncExternalStore 的原理并没有那么难懂，从源码的角度来看，就是在渲染前后去检查 store
的值是否发生改变，如果发生改变，则更新值。你可以认为 useSyncExternalStore 就是 useState、useEffect、
useLayoutEffect 配合形成的。在 React 源码中也有对应的实现。

文件位置：`packages/use-sync-external-store/src/useSyncExternalStoreShimClient.js`。

    
    
    import { useState, useEffect, useLayoutEffect } from "react";
    
    const useSyncExternalStore = <T,>(
      subscribe: any,
      getSnapshot: () => T,
      getServerSnapshot?: () => T
    ) => {
      const value = getSnapshot();
      const [{ inst }, forceUpdate] = useState({ inst: { value, getSnapshot } });
    
      // 同步执行
      useLayoutEffect(() => {
        inst.value = value;
        inst.getSnapshot = getSnapshot;
    
        if (checkIfSnapshotChanged(inst)) {
          forceUpdate({ inst });
        }
      }, [subscribe, value, getSnapshot]);
    
      // 异步执行
      useEffect(() => {
        if (checkIfSnapshotChanged(inst)) {
          forceUpdate({ inst });
        }
        const handleStoreChange: any = () => {
          if (checkIfSnapshotChanged(inst)) {
            forceUpdate({ inst });
          }
        };
        // 取消订阅
        return subscribe(handleStoreChange);
      }, [subscribe]);
    
      return value;
    };
    
    // 检查 store 是否发生变化
    function checkIfSnapshotChanged<T>(inst: {
      value: T;
      getSnapshot: () => T;
    }): boolean {
      const latestGetSnapshot = inst.getSnapshot;
      const prevValue = inst.value;
      try {
        const nextValue = latestGetSnapshot();
        // 对应 is 方法
        return !Object.is(prevValue, nextValue);
      } catch (error) {
        return true;
      }
    }
    
    export default useSyncExternalStore;
    

实现流程：

  1. 首先，通过 getSnapshot 方法生成快照，并保存在 value 中；
  2. 然后使用 useState 创建一个变量 inst，将 value 和 getSnapshot 作为初始化值；
  3. 之后分别用 useLayoutEffect 和 useEffect 创建一个副作用，通过 checkIfSnapshotChanged 检查外部状态管理工具的状态快照是否发生变化，如果发生变化，则通过 forceUpdate 去更新状态；
  4. 最后通过 useDebugValue 将 value 展示在 React 开发者工具中。

这里将副作用分为 useLayoutEffect 和 useEffect，也就是分为同步、异步两种模式，这样可以更好地控制组件的生命周期，避免出现意外。

> 上述代码与源码略有不同，感兴趣的可以自己尝试一下。
>
> 此外，在 SSR 中，如果使用 useSyncExternalStore，必须定义 getServerSnapshot，否则会引发错误。
>
> 如果在服务端渲染时不能提供一个初值，可以将组件转换成一个只在客户端渲染的组件，方法是在服务端渲染时抛出一个异常通过 `<Suspense>` 展示
> fallback 的 UI（具体可参照：[useSyncExternalStore First
> Look](https://julesblom.com/writing/usesyncexternalstore)）。

# 小结

在本章中，我们首先了解到 useSyncExternalStore 为什么存在、解决了什么问题，主要明确 useSyncExternalStore
并不止适用于三方库，像网络、尺寸、媒体查询等外部因素都有可能影响 UI，产生数据撕裂，而 useSyncExternalStore 可有效地避免 UI
中的视觉不一致问题。

之后我们对 useSyncExternalStore 的源码进行分析，为了保证 store 状态一致，会通过 **同步阻塞渲染，强制更新**
，共采取了三道保险：

  1. 通过 dispatch 方式修改 store，强制使用 Sync 同步，期间不可中断渲染；
  2. 在并发模式下，协调结束后进行一致性检查，如果不同，则强制使用 Sync 同步；
  3. 在 commit 阶段再进行一致性检查，如果不同，则强制使用 Sync 同步。

可以说 useSyncExternalStore 通过三道强力的保险，保证了 UI 的一致。

最后，我们通过 useState、useEffect 和 useLayoutEffect 简易地实现了
useSyncExternalStore，帮助我们更好地理解这个 Hooks。

总的来说，useSyncExternalStore 的实现原理并没有那么复杂，但它使用的场景要比想象中的更加广泛，所以掌握
useSyncExternalStore 是非常有必要的。

下一章，我们继续 React v18 提供的 Hooks：useTransition 和 useDeferredValue。

