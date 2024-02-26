上节我们以 useState 的视角来看 Hooks 的运行机制，但 useEffect 执行机制与 useState 的机制略有不同。

通过本节的学习，你将彻底搞懂 useEffect 的源码，明白 React 如何处理 effect 函数，以及 Hooks 的闭包问题。

# useEffect 原理

众所周知，useEffect 是用来处理副作用函数的，那么什么是副作用呢？

副作用（`Side Effect`）是指 function 做了和本身运算返回值无关的事，如请求数据、修改全局变量，打印、数据获取、设置订阅，以及手动更改
React 组件中的 DOM，这些操作都属于副作用。

useEffect 与 useState 的阶段有所不同，共分为：初始化阶段、更新阶段、commit 阶段，接下来我们围绕这三个阶段全面了解它。

![](img\11\1.image)

## mountEffect（初始化阶段）

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
    function mountEffect(
      create: () => (() => void) | void, // 回调函数，也是副作用函数
      deps: Array<mixed> | void | null, // 依赖项
    ): void {
      mountEffectImpl(
        PassiveEffect | PassiveStaticEffect,
        HookPassive,
        create,
        deps,
      );
    }
    
    function mountEffectImpl(
      fiberFlags: Flags,
      hookFlags: HookFlags,
      create: () => (() => void) | void,
      deps: Array<mixed> | void | null,
    ): void {
      const hook = mountWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      currentlyRenderingFiber.flags |= fiberFlags;
      hook.memoizedState = pushEffect(
        HookHasEffect | hookFlags,
        create,
        undefined,
        nextDeps,
      );
    }
    

从 `mountEffect` 进来，直接走向 `mountEffectImpl` 函数，先来看看 mountEffectImpl 的入参：

  * fiberFlags：有副作用的更新标记，用来标记 hook 在 fiber 中的位置；
  * hookFlags：副作用标记；
  * create：用户传入的回调函数，也是副作用函数；
  * deps：用户传递的依赖项。

**mountEffectImpl 执行流程** ：

  1. 初始化一个 hook 对象，并和 fiber 建立关系；
  2. 判断 deps 是否存在，没有的话则是 null（需要注意下这个参数，后续会详细讲解）；
  3. 给 hook 所在的 fiber 打上副作用的更新标记；
  4. 将副作用的操作存放到 hook.memoizedState 中。

### pushEffect

副作用的操作来到了 `pushEffect`，一起来看看：

    
    
    function pushEffect(tag, create, destroy, deps): Effect {
    
      // 初始化一个effect对象
      const effect: Effect = {
        tag,
        create,
        destroy,
        deps,
        next: (null: any),
      };
      
      let componentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
      
      if (componentUpdateQueue === null) { //第一个effect对象
        componentUpdateQueue = createFunctionComponentUpdateQueue();
        currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
        componentUpdateQueue.lastEffect = effect.next = effect;
      } else { // 存放多个effect对象
        const lastEffect = componentUpdateQueue.lastEffect;
        if (lastEffect === null) {
          componentUpdateQueue.lastEffect = effect.next = effect;
        } else {
          const firstEffect = lastEffect.next;
          lastEffect.next = effect;
          effect.next = firstEffect;
          componentUpdateQueue.lastEffect = effect;
        }
      }
      return effect;
    }
    

别看 `pushEffect` 中有一大坨，但是不是有种似曾相识的感觉呢？没错，它与上节的内容类似，它的作用是创建一个 `effect` 对象，然后形成一个
effect 链表，通过 next 链接 ，最后绑定在 fiber 中的 updateQueue 上。比如下面这段代码：

    
    
      const [name, setName] = useState("小杜杜");
      const [count, setCount] = useState(0);
    
      useEffect(() => {
        console.log(1);
      }, []);
    
      useEffect(() => {
        console.log(2);
      }, [name]);
    
      useEffect(() => {
        console.log(3);
      }, [count]);
    

转化后的 `fiber.updateQueue` 为：

![](img\11\2.image)

### 不同的 Effect

值得注意的是 fiber.updateQueue 保存的是所有副作用，除了包含 useEffect，同时还包含 `useLayoutEffect` 和
`useInsertionEffect`。

这里会通过不同的 `fiberFlags` 给对应的 effect 打上标记，之后在 updateQueue 链表中的 tag 字段体现出来，最后在
commit 阶段，判断到底是哪种 effect，是同步还是异步等。如：

![](img\11\3.image)

## updateEffect（更新阶段）

    
    
    function updateEffect(
      create: () => (() => void) | void,
      deps: Array<mixed> | void | null,
    ): void {
      updateEffectImpl(PassiveEffect, HookPassive, create, deps);
    }
    
    function updateEffectImpl(
      fiberFlags: Flags,
      hookFlags: HookFlags,
      create: () => (() => void) | void,
      deps: Array<mixed> | void | null,
    ): void {
    
      // 获取更新的hooks
      const hook = updateWorkInProgressHook();
      // 处理deps
      const nextDeps = deps === undefined ? null : deps;
      let destroy = undefined;
    
      if (currentHook !== null) {
        const prevEffect = currentHook.memoizedState;
        destroy = prevEffect.destroy;
        if (nextDeps !== null) {
          const prevDeps = prevEffect.deps;
          
          // 判断依赖是否发生改变，如果没有，只更新副作用链表
          if (areHookInputsEqual(nextDeps, prevDeps)) {
            hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
            return;
          }
        }
      }
    
      // 如果依赖发生改变，则在更新链表的时候，打上对应的标签
      currentlyRenderingFiber.flags |= fiberFlags;
    
      hook.memoizedState = pushEffect(
        HookHasEffect | hookFlags,
        create,
        destroy,
        nextDeps,
      );
    }
    

updateEffect：在更新阶段做的事其实很简单，就是判断 deps 是否发生改变，如果没有发生改变，则直接执行
`pushEffect`，如果发生改变，则附上不同的标签，最后在 commit 阶段，通过这些标签来判断是否执行 effect 函数。

### 不同类型的 deps 对 effect 函数执行的影响

在日常的开发中，有些不熟悉 useEffect 的小伙伴只知道 deps 发生改变，则执行对应的 effect 函数，但对 deps
本身的类型并不了解，这也造就了一些莫名奇怪的 bug，怎么找也找不到，有时候真的有可能是规范所引起的，因此我们看看以下关于 deps 的问题：

  1. deps 不存在时，造成的后果是什么？
  2. deps 是空数组，造成的后果是什么（如：`[]`）？
  3. deps 是数组、对象、函数时，造成的后果是什么（如：`[ [1] ]`、`[{ a: 1 }]`）？

实际上，所有的答案都在 `areHookInputsEqual` 函数中：

    
    
    const nextDeps = deps === undefined ? null : deps;
    
    function areHookInputsEqual(
      nextDeps: Array<mixed>,
      prevDeps: Array<mixed> | null,
    ): boolean {
    
      if (prevDeps === null) {
        return false;
      }
    
      for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
        if (objectIs(nextDeps[i], prevDeps[i])) {
          continue;
        }
        return false;
      }
      return true;
    }
    
    // 存在Object.is，就直接使用，没有的话，手动实现Object.is
    const objectIs: (x: any, y: any) => boolean = typeof Object.is === 'function' ? Object.is : is;
    
    function is(x: any, y: any) {
      return (
        (x === y && (x !== 0 || 1 / x === 1 / y)) || (x !== x && y !== y) 
      );
    }
    

从代码中，共分为三种情况：

  1. 当 deps 不存在时，也就是 `undefined`，则会当作 `null` 处理，所以无论发生什么改变，areHookInputsEqual 的值始终为 `false`，从而每次都会执行；
  2. 当 deps 为空数组时，areHookInputsEqual 返回值为 `true`，此时只更新链表，并没有执行对应的副作用，所以只会走一次；
  3. 当 deps 为对象、数组、函数时，虽然保存了，但在 `objectIs` 做比较时，旧值与新值永远不相等，也就是`[1] !== [1]`、`{a: 1} !== {a: 1}`（指向不同），所以只要当 deps 发生变动，都会触发更新。

> **注意：** 如果强行比较对象、数组时，可以通过 `JSON.stringify()` 转化为字符串，当作 deps 的参数。
>
> `useMemo` 和 `useCallback` 中的 deps 也是同理。

## commitRoot（commit 阶段）

commitRoot 是 commit 阶段的入口，一起来看看。

文件位置：`packages/react-reconciler/src/ReactFiberWorkLoop.js`。

    
    
    function commitRoot(
      root: FiberRoot,
      recoverableErrors: null | Array<CapturedValue<mixed>>,
      transitions: Array<Transition> | null,
    ) {
      // 获取优先级
      const previousUpdateLanePriority = getCurrentUpdatePriority();
      const prevTransition = ReactCurrentBatchConfig.transition;
      ...
      commitRootImpl(
          root,
          recoverableErrors,
          transitions,
          previousUpdateLanePriority,
      );
    
      return null;
    }
    

在 commitRoot 中，首先会制定函数的优先级，当执行完毕后，恢复优先级，而这个函数的主体为 `commitRootImpl` 函数。

### commitRootImpl

commitRootImpl 函数非常复杂，这里我们只关注 effect 的逻辑即可，而关于 effect 的逻辑主要是
`scheduleCallback`。

    
    
    function commitRootImpl(root, recoverableErrors, transitions, renderPriorityLevel ) {
      ...
      scheduleCallback(NormalSchedulerPriority, () => {
        // 调度 Effect
        flushPassiveEffects();
        return null;
      });
      ...
      return null;
    }
    

`scheduleCallback` 是 React 调度器（Scheduler）的一个
API，最终通过一个宏任务来异步调度传入的回调函数，使得该回调在下一轮事件循环中执行，此时浏览器已经绘制过一次。同时可以看出，effectlist
的优先级是：普通优先级。

### flushPassiveEffects

    
    
    export function flushPassiveEffects(): boolean {
    
      if (rootWithPendingPassiveEffects !== null) {
        ...
        try {
          ReactCurrentBatchConfig.transition = null;
          // 设置优先级
          setCurrentUpdatePriority(priority);
          // 调用函数
          return flushPassiveEffectsImpl();
        } finally {
          setCurrentUpdatePriority(previousPriority);
          ReactCurrentBatchConfig.transition = prevTransition;
          releaseRootPooledCache(root, remainingLanes);
        }
      }
      return false;
    }
    

到 `flushPassiveEffects` 中，也是一系列跟优先级有关的操作，最终走向 `flushPassiveEffectsImpl`
这个函数。而在这个函数中会执行两个方法，分别是：`commitPassiveUnmountEffects（执行所有 effect 的销毁程序）` 和
`commitPassiveMountEffects（执行所有 effect 的回调函数）`。

接下来逐一进行分析，看看两者的的流向。

### commitPassiveMountEffects 的流向

![](img\11\4.image)

最终的走向为 commitHookEffectListMount 函数，着重看下：

    
    
    function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
      const updateQueue = (finishedWork.updateQueue: any);
      const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
      if (lastEffect !== null) {
        const firstEffect = lastEffect.next;
        let effect = firstEffect;
        do {
          if ((effect.tag & flags) === flags) {
            if (enableSchedulingProfiler) {
              if ((flags & HookPassive) !== NoHookEffect) {
                markComponentPassiveEffectMountStarted(finishedWork);
              } else if ((flags & HookLayout) !== NoHookEffect) {
                markComponentLayoutEffectMountStarted(finishedWork);
              }
            }
    
            // 执行effect函数， 并保存effect函数的结果给destroy
            const create = effect.create;
            effect.destroy = create();
    
            if (enableSchedulingProfiler) {
              if ((flags & HookPassive) !== NoHookEffect) {
                markComponentPassiveEffectMountStopped();
              } else if ((flags & HookLayout) !== NoHookEffect) {
                markComponentLayoutEffectMountStopped();
              }
            }
    
          }
          effect = effect.next;
        } while (effect !== firstEffect);
      }
    }
    

主要作用是：遍历所有的 `effect list`，然后依次执行对应的 effect 副作用函数，并将其结果保留在 `destroy` 函数中。

> effect list 是一个用于 effectTag 副作用列表容器，包含第一个节点：firstEffect， 和最后一个节点
> lastEffect，通过 next 链接，在 commit 阶段，根据这些 effectTag 来判断执行的时机，从而对相应的 DOM 树进行更改。

### commitPassiveUnmountEffects 的流向

![img.png](img\11\5.image)

最终的走向为 commitHookEffectListUnmount 函数， 来看看：

    
    
    function commitHookEffectListUnmount(
      flags: HookFlags,
      finishedWork: Fiber,
      nearestMountedAncestor: Fiber | null,
    ) {
      const updateQueue= finishedWork.updateQueue;
      const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
      if (lastEffect !== null) {
        const firstEffect = lastEffect.next;
        let effect = firstEffect;
        do {
          if ((effect.tag & flags) === flags) {
            // Unmount
            const destroy = effect.destroy;
            effect.destroy = undefined;
            if (destroy !== undefined) {
              if (enableSchedulingProfiler) {
                if ((flags & HookPassive) !== NoHookEffect) {
                  markComponentPassiveEffectUnmountStarted(finishedWork);
                } else if ((flags & HookLayout) !== NoHookEffect) {
                  markComponentLayoutEffectUnmountStarted(finishedWork);
                }
              }
               // 调用销毁逻辑
              safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
              if (enableSchedulingProfiler) {
                if ((flags & HookPassive) !== NoHookEffect) {
                  markComponentPassiveEffectUnmountStopped();
                } else if ((flags & HookLayout) !== NoHookEffect) {
                  markComponentLayoutEffectUnmountStopped();
                }
              }
            }
          }
          effect = effect.next;
        } while (effect !== firstEffect);
      }
    }
    

主要通过 `safelyCallDestroy` 走对应的销毁逻辑，这里要注意下，effect 的执行需要保证所有组件的 effect
的销毁函数执行完才能够执行。

因为多个组件可能共用一个 ref，如果不将销毁函数执行，改变 ref 的值，有可能会影响其他 effect 执行。

# 经典的闭包问题

阅读完 useEffect 源码后，再来看最为经典的 React Hooks
的闭包问题，就会变得异常简单。相信各位小伙伴一定都踩过坑，我们解决一下这个问题，同时巩固之前所学的知识。

先看下面这段代码：

    
    
    import { useState, useEffect } from "react";
    import { Button, message } from "antd";
    const Index = () => {
      const [count, setCount] = useState(0);
    
      useEffect(() => {
        const timer = setTimeout(() => {
          setCount((v) => v + 1);
        }, 2000);
        return () => {
          clearTimeout(timer);
        };
      }, []);
    
      useEffect(() => {
        const timer = setTimeout(() => {
          message.info(`当前的count为：${count}`);
        }, 3000);
        return () => {
          clearTimeout(timer);
        };
      }, []);
    
      return (
        <div style={{ padding: 20 }}>
          <div>数字：{count}</div>
          <Button onClick={() => setCount((v) => v + 1)}>加1</Button>
        </div>
      );
    };
    
    export default Index;
    

例子很简单，进入页面，创建 count，在第一个 useEffect 中过 `2s` 给 count 加 1，并且在这 `2s`
中点击按钮两次，之后在第二个 useEffect 中过 `3s` 进行获取 count 值。

那么你觉得 info 中的 count 是 `0`、`1` 还是 `3` ？

![](img\11\6.image)

思考下，在页面中可以看到，渲染的值变成了 3，但为什么在 info 中是 0 呢？这种情况就是最经典的闭包问题。

首先，在绝大多数的场景下，并不会出现闭包问题，只有在延迟调用的场景下（如：`setTimeout`、`setInterval`、`Promise.then`
等），才会出现闭包问题。接下来一起看看该如何解决。

我们先温习下上节的内容。当进行初始化后，Hooks 信息在 Fiber 中的`memorizedState` 链表中，通过 `next`
链接，直到没有，则为 `null`。

案例中对应 `useState`、`useEffect`、`useEffect` 3 个 `hook`，自然对应链表中的 3 个
`memorizedState`，如：

![](img\11\7.image)

当执行 useEffect 时，一直拿最初的 `count = 0` 来记录引用关系。再加上 `deps`
为空数组，此时只执行一次，所以无论点击多少次按钮，其结果都为 0。

## 设置 deps 为 count

既然在引用时一直拿 `count = 0` 为引用条件，那么我们将 deps 的参数设置为 count，去监听 count，从而初始化定时器，是不是就
ok了呢？

来看下效果：

![](img\11\8.image)

> 问：为什么要在 useEffect 的 return 中清空定时器呢？
>
> 答：useEffect 对应的 return 就是源码中的 `destroy 函数`，如果不清空，那么还会执行之前的定时器。setTimeout
> 的效果可能不是很清晰，感兴趣的小伙伴可以换成 setInterval 试试。

## useLatest 解决

那么 deps 设置为 count 真的能够解决闭包问题吗？

我认为并不能彻底解决。在上述的问题中，useEffect 函数本身执行了 3 遍（2 次点击，1 次count+1），换句话说，只要 count
发生变化，就会执行 useEffect 函数。

如果现在的需求变为只想在 3s 后获取到最新值，之后再点击按钮，不获取最新值，该怎么办？

其实答案很简单，利用 ref 的高级用法——缓存数据，也就是 useLatest 去解决就 ok 了。如：

    
    
      const countRef = useLatest(count);
      useEffect(() => {
        const timer = setTimeout(() => {
          message.info(`当前的count为：${countRef.current}`);
        }, 3000);
    
      }, []);
    

**效果：**

![](img\11\9.image)

## 结果何时为 1？

我在一开始的问题中写了 3 个答案，分别是：0、1、3。0 和 3 讲解了原因，那么 1 是怎么出现的呢？答案是将 `setCount((v) => v +
1)` 替换为 `setCount(count + 1)`。

如果不知道为什么，那就再看看上节的内容，做个简单的巩固吧～

# 小结

本节通过 useEffect 的源码，来了解 useEffect 的执行流程，大体分三个阶段：初始化、更新和执行阶段，同时讲解了不同 effect
的连接方式、deps 参数问题等，最后通过经典的闭包问题，巩固 useState、useEffect。

现在对 useEffect 的三个阶段做出简要总结。

**1\. useEffect 初始化阶段**

建立一个 hook 对象，与 fiber 建立关联，并给 hook 所在的 fiber 打上副作用更新标记，最后存储在 fiber 中的
`memoizedState`，同时在 fiber 中的 updateQueue 存储了相关的副作用（这些副作用是闭环链表）。

**2\. useEffect 更新阶段**

这个阶段最主要的是：判断 deps 是否发生改变，如果没有改变，则更新副作用链表；如果发生改变，则在更新链表的时候，打上对应的副作用标签。之后在
commit 阶段，根据对应的标签，重新执行对应的副作用。

**3\. useEffect 执行阶段**

在 commit 阶段中，入口为 `commitRoot`，走到 `commitRootImpl`，用 `scheduleCallback` 进行调度，同时
effectlist 属于普通调度，最终走向：`commitPassiveMountEffects` 和
`commitPassiveUnmountEffects`，回调和销毁的过程类似。值得注意的是：不同的 effect 通过 effectTag 来判断。

实际上，执行阶段是非常复杂的，主要是有优先级的部分，需要判断执行的时机，是同步或异步，何时执行，在文节中只是简要地介绍下关于 effect list
的部分，并不涉及麻烦、难懂的知识点。

总的来说，源码确实比较头疼，但当你逐渐掌握后，会发现越来越有趣。下一节我们将学习 useMemo 和 useCallback 的源码及详解～

