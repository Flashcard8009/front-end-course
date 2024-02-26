在 React v18 中，引入了 useTransition 和 useDeferredValue 两个 Hooks，它们都是用来处理 **数据量大**
的数据，比如百度的搜索框、散点图等。

![useTransition 和  useDeferredValue.png](img\15\1.image)

我们先回顾一下什么是过渡更新任务和紧急更新任务？

  * 紧急更新任务：用户立马能够看到效果的任务，如输入框、按钮等操作，在视图上产生效果的任务。
  * 过渡更新任务：由其他因素引起的任务，导致无法在视图上看到效果的任务，如请求接口数据，需要一个 `loading...` 的状态。

> 这里的任务只是针对单一状态，同一操作可能会有多种任务发生。

为了更好的理解，我们先来看这样一个例子。

假设我们有一个 `input` 输入框，这个输入框的值要维护一个很大列表（假设列表有 2w 条数据），比如说过滤、搜索等情况，这时有两种变化：

  1. input 框内的变化；
  2. 根据 input 的值，1w 条数据的变化。

input 框内的变化是实时获取的，也就是受控的，此时的行为就是 **紧急更新任务** 。

而这 2w 条数据的变化，就会有过滤、重新渲染的情况，此时这种行为被称为 **过渡更新任务** 。

了解完紧急更新任务和过渡更新任务后，正式来看看 useTransition 究竟是如何处理大数据的。

# useTransition 的诞生

在介绍并发的时候提及到 useTransition 内更新的事件会采取 Concurrent 模式，而 Concurrent
模式可以中断，让优先级高的任务先进行渲染，让用户有更好的体验。

换言之，useTransition 是用于一些 **不是很急迫的更新上** ，同时解决并发渲染的问题而诞生的。

> 值得注意的是：useTransition 一定是处理数据量大的数据。

接下来我们模拟一下上述的场景，具体来看看效果。

**模拟案例：**

    
    
    // utils 
    export const count = 20000; // 渲染次数
    
    import { useState } from "react";
    import { Input } from "antd";
    import { count } from "./utils";
    
    // 正常情况
    const Index: React.FC<any> = () => {
      const [list, setList] = useState<string[]>([]);
    
      return (
        <>
          <Input
            onChange={(e) => {
              const res: string[] = [];
              for (let i = 0; i < count; i++) {
                res.push(e.target.value);
              }
              setList(res);
            }}
          />
          {list.map((item, index) => (
            <div key={index}>{item}</div>
          ))}
        </>
      );
    };
    
    export default Index;
    

在案例中，我们有一个输入框，输入内容时会在下方输出 2w 数据。

**正常模式下的效果：**

![img.gif](img\15\2.image)

在正常情况下，输入内容，页面会异常的卡顿，这种体验明显非常不好。

**useTransition 模式下的效果：**

![img2.gif](img\15\3.image)

在 useTransition 中，可以看出在输入数字时，input 框内会正常显示，而列表会滞后，同时 useTransition 提供
isPending 来处理更新是否完成。这种效果明显给用户带来了极好的体验。

## 对比防抖、节流、定时器

可能有小伙伴会问，这不就是防抖和节流嘛，为什么要多出一个 useTransition 呢？是不是有点多此一举？

的确，在 React v18 之前，我们都用防抖、节流去解决，接下来我们先分别看下两种方式的效果。

**防抖模式下的效果：**

防抖 (Debouncing) ：指在一定时间内，多次触发同一个事件，只执行最后一次操作。

![img3.gif](img\15\4.image)

在防抖的示例中，我们发现交互效果得到了明显的改善，因为用户在连续输入时，只在最后一次才做处理，在此之前，浏览器的渲染引擎并没有被阻塞，所以可以看到输入框的内容。

**节流模式下的效果：**

节流（Throttling）：指在一定时间内，多次触发同一个事件，只执行第一次操作。

![img4.gif](img\15\5.image)

在节流的示例中，虽然用户可以看到输入的内容，也能较快地看到对应的结果列表，但在第一次渲染的时候有明显的卡顿，实际上防抖也会在第一次渲染后有卡顿的效果。

我们知道，防抖和节流本质上都是定时器，那么我们现在只用 setTimeout 来进行测试，一起来看看效果：

![img5.gif](img\15\6.image)

在定时器的示例中，效果跟节流的效果类似。但相比于正常情况下的效果要好一些。

## useTransition 与定时器的异同

我们先看看防抖、节流、setTimeout 存在的问题。

  * 防抖：延迟 React 更新操作，换言之，快速长时间输入，列表依旧等不到响应，但列表得到响应后，渲染引擎依旧会出现阻塞，导致页面卡顿。
  * 节流：节流在一段时间内开始处理，渲染引擎也会出现阻塞，页面会卡顿，而节流的时间需要手动配置。
  * setTimeout：setTimeout 也是同理，依旧会出现阻塞、卡顿，所以依然会阻止页面交互。

我们知道，防抖和节流的本质都是 **定时器** ，虽然能在一定的程度上改善交互效果，但依旧不能解决卡顿或卡死的情况。因为 **React
的更新不可中断，导致 JS 引擎长时间占据浏览器的主线程，使得渲染引擎被长时间阻塞。**

针对这个问题， React v18 推出 useTransition 来解决这个问题，那么它与定时器有何作用：

  1. 使用 useTransition 会触发 Concurrent 模式，所以渲染进程不会长时间被阻塞，使得其他操作得到及时响应，从而使用户体验得到了极大的提升；
  2. 其次，定时器的本质是 **异步延时执行** ，而 useTransition 属于 **同步执行** ，通过标记 transition 来决定是否完成此次更新。所以 useTransition 要比定时器更新得要早，整体的效果要好很多；
  3. 对于防抖、节流、setTimeout 来说，相当于合并渲染的次数，简单地说，就是控制了 render 的渲染次数，而 useTransition 并没有减少渲染的次数，这点要切记。

> 问：减少 render 的渲染次数不是很好吗？为什么还要用 useTransition 呢？

> 答：在上面的示例中，我们发现无论是防抖还是节流都会出现轻微卡顿的现象，但要特别注意，我们渲染的数据是写死的 2w
> 条，在真实的环境下，我们无法确定实际的数量。
>
> 换言之，我们并不好控制防抖和节流的 **延时时间** ，如果时间过长，导致一种滞后的感觉，如果时间过短，就会出现卡顿的效果。
>
> 而 useTransition 并不需要考虑这些因素，通过中断渲染，让浏览器在空闲时间下执行，达到更佳的效果。

## 设备性能影响

关于渲染，也与本身的机器有关，关于具体的影响可参考：[Real world example: adding startTransition for slow
renders](https://github.com/reactwg/react-18/discussions/65)。

> 注：useTransition 用于函数组件，在类中使用 startTransition。但要注意，startTransition
> 也可以用在函数组件中。

我们直接来看看效果，注意滑块：

  1. 正常情况下：

![img6.gif](img\15\7.image)

  2. 定时器效果下：

![img7.gif](img\15\8.image)

  3. useTransition 效果下：

![img9.gif](img\15\9.image)

# useTransition 源码

## mountTransition（初始化）

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
    function mountTransition(): [
      boolean,
      (callback: () => void, options?: StartTransitionOptions) => void,
    ] {
      const [isPending, setPending] = mountState(false);
      const start = startTransition.bind(null, setPending);
      const hook = mountWorkInProgressHook();
      hook.memoizedState = start;
      return [isPending, start];
    }
    

在 mountTransition 中，首先由 isPending 来定义状态，然后会走 startTransition 方法，返回的 start 会保存在
memoizedState 中，那么我们一起看看 startTransition 做了哪些事。

### startTransition

    
    
    function startTransition(
      setPending: boolean => void,
      callback: () => void,
      options?: StartTransitionOptions,
    ): void {
    
      // 获取优先级
      const previousPriority = getCurrentUpdatePriority();
      
      // 将当前任务重新设置优先级，并且等级要低于 ContinuousEventPriority
      setCurrentUpdatePriority(
        higherEventPriority(previousPriority, ContinuousEventPriority),
      );
    
      setPending(true);
    
      // 标记一个过渡位
      const prevTransition = ReactCurrentBatchConfig.transition;
      ReactCurrentBatchConfig.transition = ({}: BatchConfigTransition);
      const currentTransition = ReactCurrentBatchConfig.transition;
    
      if (enableTransitionTracing) {
        if (options !== undefined && options.name !== undefined) {
          ReactCurrentBatchConfig.transition.name = options.name;
          ReactCurrentBatchConfig.transition.startTime = now();
        }
      }
    
     
      try {
        setPending(false);
        callback();
      } finally {
        setCurrentUpdatePriority(previousPriority);
        ReactCurrentBatchConfig.transition = prevTransition;
    
      }
    }
    
    // higherEventPriority
    export function higherEventPriority(
      a: EventPriority,
      b: EventPriority,
    ): EventPriority {
      return a !== 0 && a < b ? a : b;
    }
    

在 startTransition 中的流程为：

  1. 首先通过 getCurrentUpdatePriority 获取优先级，通过 higherEventPriority 方法重新给 ContinuousEventPriority（连续事件优先级）设置优先级，如果该任务的优先级低于 ContinuousEventPriority，则继续使用该任务的优先级。
  2. 之后通过 setPending 将 isPending 设置为 true， 然后会设置一个标记位，此时更新会 **优先处理** 。
  3. 然后再将 isPending 改为 false，并在 callback 中触发定义的更新，此过程会触发 setPending， 最终设置回原来的优先级。

### isPending 工作原理

首先我们要知道，mountTransition 中用 mountState 定义的 isPending 就是 useTransition
中的第一个参数，也就是中间状态。

但在 startTransition 中连续调用了三次 setPending，换言之，调用了三次 useState，而在实际的效果中，只触发了两次
React 更新呢？

我们很容易想到， useState 具有批量更新的机制，但应该将三次触发更新合并成一次更新，为什么是两次呢？

实际原因是：

    
    
    ReactCurrentBatchConfig.transition = ({}: BatchConfigTransition);
    

将 transition 设置为空，使得前后逻辑中的上下文不一致，导致采用的模式不同，分别采用 legacy（同步阻塞）模式和
concurrent（并发）模式。 而后面的两次更新会触发批量更新，合并为一次。所以，一共会触发两次更新。

## updateTransition（更新）

    
    
    function updateTransition(): [
      boolean,
      (callback: () => void, options?: StartTransitionOptions) => void,
    ] {
      const [isPending] = updateState(false);
      const hook = updateWorkInProgressHook();
      const start = hook.memoizedState;
      return [isPending, start];
    }
    

可以看出，useTransition 在更新过程中并没有什么特殊的逻辑，只是调用 `updateState` 去更新 isPending 的状态。

## 对比 startTransition

对比 Hooks 中的 useTransition，我们顺便看看类中的 startTransition，两者有何区别。

![img10.gif](img\15\10.image)

在 startTransition 中，当用户连续输入时，会出现轻微的卡顿，可以看出 startTransition
并没有防抖的效果，具体原因下文介绍，我们先来看看对应的源码：

文件位置：`packages/react/src/ReactStartTransition.js`。

    
    
    export function startTransition(
      scope: () => void,
      options?: StartTransitionOptions,
    ) {
      const prevTransition = ReactCurrentBatchConfig.transition;
      
      // 设置状态
      ReactCurrentBatchConfig.transition = ({}: BatchConfigTransition);
    
      try {
        // 执行更新
        scope();
      } finally {
        // 恢复原来的状态
        ReactCurrentBatchConfig.transition = prevTransition;
      }
    }
    
    

在 startTransition 源码中，我们发现并没有 isPending 的逻辑，这是直接导致 startTransition 不具备防抖效果的原因。

要知道，在 Concurrent 模式下， **低优先级更新会被高优先级中断，此时，低优先级更新已经开始的协调会被清除，并且会被重置为未开始的状态。**

当被重置后，导致 transition 更新只有在用户停止输入（或超过 5s）时才会得到有效的处理。

通过设置 isPending 为 true 时可以形成中断，形成类似防抖的作用；而 startTransition 本身并没有中断，连续的输入并不会重置
transition 更新，然后开始浏览器渲染过程，因此没有防抖的作用。

通过源码的阅读，我们发现 useTransition 实际上是 useState + startTransition 的结合体，而 isPending
的状态通过 ReactCurrentBatchConfig.transition 的变化进行更新，以此来捕获过渡时间。

# useDeferredValue

当我们介绍完 useTransition 后，我们再一起看看它的“兄弟”：useDeferredValue。

之所以称为“兄弟”，是因为这两个 Hooks 极为相似，有点类似于 useMemo 和 useCallback 的关系，useTransition
用来处理更新函数，而 useDeferredValue 用来处理数据本身。

useDeferredValue 可以让 **状态滞后派生** ，推迟屏幕优先级不高的部分。

## 使用示例

useDeferredValue 是趋向于值的维护，当我们存在批量查找的时候，它会是一个好帮手，举个例子：

    
    
    import { useState, useDeferredValue } from "react";
    import { Input } from "antd";
    
    const getList = (key: any) => {
      const arr = [];
      for (let i = 0; i < 20000; i++) {
        if (String(i).includes(key)) {
          arr.push(<li key={i}>{i}</li>);
        }
      }
      return arr;
    };
    
    const Index: React.FC<any> = () => {
      const [input, setInput] = useState("");
      const deferredValue = useDeferredValue(input);
    
      return (
        <>
          <div>寻找2w以内匹配的数据：</div>
          <Input value={input} onChange={(e: any) => setInput(e.target.value)} />
          <div>
            <ul>{deferredValue ? getList(deferredValue) : null}</ul>
          </div>
        </>
      );
    };
    
    export default Index;
    

效果：

![img.gif](img\15\11.image)

在案例中，我们通过 useDeferredValue 去维护 Input 中的值，从两万条数据中去查询包含的值，然后输出到列表中。

了解完 useDeferredValue 的使用，再来看看它的源码，同样分为：mountDeferredValue（初始化）和
updateDeferredValue（更新）两个步骤。

## mountDeferredValue（初始化）

文件位置：`packages/react-reconciler/src/ReactFiberHooks.js`。

    
    
    function mountDeferredValue<T>(value: T): T {
      const hook = mountWorkInProgressHook();
      hook.memoizedState = value;
      return value;
    }
    

mountDeferredValue 的功能很简单，只是进行了一个初始化 hook，将值保存在 memoizedState 中。

## updateDeferredValue（更新）

    
    
    function updateDeferredValue<T>(value: T): T {
      const hook = updateWorkInProgressHook();
      const resolvedCurrentHook: Hook = (currentHook: any);
      const prevValue: T = resolvedCurrentHook.memoizedState;
      return updateDeferredValueImpl(hook, prevValue, value);
    }
    
    function updateDeferredValueImpl<T>(hook: Hook, prevValue: T, value: T): T {
      const shouldDeferValue = !includesOnlyNonUrgentLanes(renderLanes); // 对比优先级
      
      if (shouldDeferValue) {
        if (!is(value, prevValue)) {
          // 设置优先级
          currentlyRenderingFiber.lanes = mergeLanes(
            currentlyRenderingFiber.lanes,
            deferredLane,
          );
          markSkippedUpdateLanes(deferredLane);
          hook.baseState = true;
        }
    
        return prevValue;
      } else {
        // 如果 baseState 存在，则会触发更新流程
        if (hook.baseState) {
          hook.baseState = false;
          markWorkInProgressReceivedUpdate();
        }
    
        hook.memoizedState = value;
        return value;
      }
    }
    

在 updateDeferredValue 中，首先拿到上一次记录的值（prevValue），然后走向 updateDeferredValueImpl
函数。

updateDeferredValueImpl 函数首先会对比优先级，如果优先级高于当前优先级，shouldDeferValue 则为 true，通过 is
去比较新值（value）与旧值（prevValue）是否相等，如果不相等，则更新优先级，并且将 baseState 设置为
true，用作后续是否更新视图的依据。

此时，baseState 为 true 代表新值与旧值不同，则会触发 markWorkInProgressReceivedUpdate() 函数（与
useState 的 updateReducer 一致），触发更新渲染流程，最终返回最新值。

# useTransition 与 useDeferredValue 的使用场景

通过上面的源码，我们发现 useTransition 和 useDeferredValue 都是将包裹的任务标记成 **过渡更新任务**
。换言之，它们包裹的数据都属于 **优先级比较低** 的，所以在渲染的时候会有一定的滞后性，从而用更多的资源去渲染优先级更高的更新。

同时，它们都适合 **大数据** 处理的优化，如案例中 2w 条数据的处理、百度输入框、散点图等，除此之外，一般的场景没有必要去使用这两个
hooks，因为它们本身会带来一定的性能损耗， 只有处理 **数据量大的数据** 时，才去考虑去使用它们。

最后，对同一个资源优化时，只需要用它们两个的其中一个即可，因为它们优化的效果一致，如果两个都使用，肯定会带来一定的损耗，所以两者并不建议同时使用。

> 问：既然 useTransition 与 useDeferredValue 这么相似，那我们如何更好地区分它们呢？

> 答：能使用 useTransition 的时候就使用 useTransition，除非不能用 useTransition，才去考虑
> useDeferredValue。
>
> 因为 useTransition 用来处理函数，也就是说它可以一次性处理几个更新函数，并且在大多数场景下 useTransition 要比
> useDeferredValue 的性能更好，所以这里更加推荐 useTransition。
>
> 但我们使用一些三方库的时候，比如 ahooks，它的更新函数并没有直接暴露给我们，只返回对应的值给我们，这种情况下可使用
> useDeferredValue 来做优化。

# 小结

通过本章的阅读，我们了解了 useTransition 和 useDeferredValue 的使用方法和对应的源码，通过它们可实现 **中断**
，使优先级更高的事件优先执行，同时学习了两者的关联和使用场景。

并且通过案例详细分析 useTransition 与防抖、节流、定时器的区别，然后对比 startTransition 来说明 useTransition
的优势， 总而言之，useTransition 的效果更佳，极大地提升用户体验。

> 注：startTransition 在函数组件和类组件中都能使用，而 useTransition 只能用在函数组件。

除此之外，我们要特别铭记不同的更新上下文，导致的模式不同。用 useTransition、useDeferredValue 包裹的更新才会走
Concurent 模式，一般情况依旧会走 legacy 模式。

最后，关于源码篇的章节就此结束，接下来，我们通过具体的实践，来帮助我们在工作中更好地去应用这些 Hooks。

