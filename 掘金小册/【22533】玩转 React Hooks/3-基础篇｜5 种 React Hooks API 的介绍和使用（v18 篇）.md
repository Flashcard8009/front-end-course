本节主要介绍`v18`中提供的
`useSyncExternalStore`、`useTransition`、`useDeferredValue`、`useInsertionEffect`、`useId`
5 个 API 的使用方法。

![](img\3\1.image)

## 1\. useSyncExternalStore

**useSyncExternalStore：** 会通过强制的同步状态更新，使得外部 `store` 可以支持并发读取。

**注意：** 这个 Hooks 并不是在日常开发中使用的，而是给第三方库 `redux`、`mobx` 使用的，因为在 React v18 中，主推的
Concurrent（并发）模式可能会出现状态不一致的问题（比如在 `react-redux 7.2.6` 的版本），所以官方给出
useSyncExternalStore 来解决此类问题。

简单地说，useSyncExternalStore 能够让 React 组件在 Concurrent
模式下安全、有效地读取外接数据源，在组件渲染过程中能够检测到变化，并且在数据源发生变化的时候，能够调度更新。

当读取到外部状态的变化，会触发强制更新，以此来保证结果的一致性。

**基本使用：**

    
    
    const state = useSyncExternalStore(
        subscribe,
        getSnapshot,
        getServerSnapshot
    )
    

**Params：**

  * subscribe：订阅函数，用于注册一个回调函数，当存储值发生更改时被调用。 此外，useSyncExternalStore 会通过带有记忆性的 getSnapshot 来判断数据是否发生变化，如果发生变化，那么会强制更新数据；
  * getSnapshot：返回当前存储值的函数。必须返回缓存的值。如果 getSnapshot 连续多次调用，则必须返回相同的确切值，除非中间有存储值更新；
  * getServerSnapshot：返回服务端（`hydration` 模式下）渲染期间使用的存储值的函数。

**Result：**

  * state：数据源，用于渲染 `UI 层`的数据源。

**基本用法：**

    
    
    import { useSyncExternalStore } from "react";
    import { Button } from "antd";
    import { combineReducers, createStore } from "redux";
    
    const reducer = (state: number = 1, action: any) => {
      switch (action.type) {
        case "ADD":
          return state + 1;
        case "DEL":
          return state - 1;
        default:
          return state;
      }
    };
    
    /* 注册reducer,并创建store */
    const rootReducer = combineReducers({ count: reducer });
    const store = createStore(rootReducer, { count: 1 });
    
    const Index: React.FC<any> = () => {
      //订阅
      const state = useSyncExternalStore(
        store.subscribe,
        () => store.getState().count
      );
    
      return (
        <>
          <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>
          <div>数据源： {state}</div>
          <Button type="primary" onClick={() => store.dispatch({ type: "ADD" })}>
            加1
          </Button>
          <Button
            style={{ marginLeft: 8 }}
            onClick={() => store.dispatch({ type: "DEL" })}
          >
            减1
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\3\2.image)

当我们点击按钮后，会触发 store.subscribe（订阅函数），执行 getSnapshot 后得到新的 count，此时 count
发生变化，就会触发更新。

## 2\. useTransition

**useTransition：** 返回一个状态值表示过渡更新任务的等待状态，以及一个启动该过渡更新任务的函数。

> 问：什么是过渡更新任务？
>
> 答：过渡任务是对比紧急更新任务所产生的。
>
> 紧急更新任务指，输入框、按钮等任务需要在视图上立即做出响应，让用户立马能够看到效果的任务。
>
> 但有时，更新任务不一定那么紧急，或者说需要去请求数据，导致新的状态不能够立马更新，需要一个 `loading...` 的状态，这类任务称为过渡任务。

我们再来举个比较常见的例子帮助理解紧急更新任务和过渡更新任务。

当我们有一个 `input` 输入框，这个输入框的值要维护一个很大列表（假设列表有 1w 条数据），比如说过滤、搜索等情况，这时有两种变化：

  1. input 框内的变化；
  2. 根据 input 的值，1w 条数据的变化。

input 框内的变化是实时获取的，也就是受控的，此时的行为就是紧急更新任务。而这 1w
条数据的变化，就会有过滤、重新渲染的情况，此时这种行为被称为过渡更新任务。

**基本使用：**

    
    
    const [isPending, startTransition] = useTransition();
    

**Result：**

  * isPending：布尔值，过渡状态的标志，为 true 时表示等待状态；
  * startTransition：可以将里面的任务变成过渡更新任务。

**基本用法：**

    
    
    import { useState, useTransition } from "react";
    import { Input } from "antd";
    
    const Index: React.FC<any> = () => {
      const [isPending, startTransition] = useTransition();
      const [input, setInput] = useState("");
      const [list, setList] = useState<string[]>([]);
    
      return (
        <>
          <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>
          <Input
            value={input}
            onChange={(e) => {
              setInput(e.target.value);
              startTransition(() => {
                const res: string[] = [];
                for (let i = 0; i < 10000; i++) {
                  res.push(e.target.value);
                }
                setList(res);
              });
            }}
          />
          {isPending ? (
            <div>加载中...</div>
          ) : (
            list.map((item, index) => <div key={index}>{item}</div>)
          )}
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\3\3.image)

从上述的代码可以看到，我们通过 input 去维护了 1w 条数据，通过 isPending 的状态来控制是否展示完成。

可以看出，useTransition 是为了处理大量数据而存在的，那么有些小伙伴可能会问，这种情况不应该用防抖吗？为什么还会出现 useTransition
呢？

实际上防抖的本质是 `setTimeout`，也就是减少了渲染的次数，而 useTransition
并没有减少其渲染的次数，至于具体的区别，在之后的源码篇中专门介绍，这里我们只要清楚 useTransition 的用法即可。

## 3\. useDeferredValue

useDeferredValue：可以让状态滞后派生，与 useTransition 功能类似，推迟屏幕优先级不高的部分。

在一些场景中，渲染比较消耗性能，比如输入框。输入框的内容去调取后端服务，当用户连续输入的时候会不断地调取后端服务，其实很多的片段信息是无用的，这样会浪费服务资源，
React 的响应式更新和 JS 单线程的特性也会导致其他渲染任务的卡顿。而 useDeferredValue 就是用来解决这个问题的。

> 问：useDeferredValue 和 useTransition 怎么这么相似，两者有什么异同点？
>
> 答：useDeferredValue 和 useTransition 从本质上都是标记成了过渡更新任务，不同点在于 useDeferredValue
> 是将原值通过过渡任务得到新的值， 而 useTransition 是将紧急更新任务变为过渡任务。
>
> 也就是说，useDeferredValue 用来处理数据本身，useTransition 用来处理更新函数。

**基本使用：**

    
    
    const deferredValue = useDeferredValue(value);
    

**Params：**

  * value：接受一个可变的值，如`useState`所创建的值。

**Result：**

  * deferredValue：返回一个延迟状态的值。

**基本用法：**

    
    
    import { useState, useDeferredValue } from "react";
    import { Input } from "antd";
    
    const getList = (key: any) => {
      const arr = [];
      for (let i = 0; i < 10000; i++) {
        if (String(i).includes(key)) {
          arr.push(<li key={i}>{i}</li>);
        }
      }
      return arr;
    };
    
    const Index: React.FC<any> = () => {
      //订阅
      const [input, setInput] = useState("");
      const deferredValue = useDeferredValue(input);
      console.log("value：", input);
      console.log("deferredValue：", deferredValue);
    
      return (
        <>
          <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>
          <Input value={input} onChange={(e: any) => setInput(e.target.value)} />
          <div>
            <ul>{deferredValue ? getList(deferredValue) : null}</ul>
          </div>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\3\4.image)

上述的功能类似于搜索，从 1w 个数中找到输入框内的数。

> 问：什么场景下使用`useDeferredValue` 和 `useTransition` ？
>
> 答：通过上面的两个例子介绍我们知道，useDeferredValue 和 useTransition
> 实际上都是用来处理数据量大的数据，比如，百度输入框、散点图等，都可以使用。它们并不适用于少量数据。
>
> 但在这里更加推荐使用 useTransition，因为 useTransition 的性能要高于 useDeferredValue，除非像一些第三方的
> Hooks 库，里面没有暴露出更新的函数，而是直接返回值，这种情况下才去考虑使用 useDeferredValue。
>
> 这两者可以说是一把双刃剑，在数据量大的时候使用会优化性能，而数据量低的时候反而会影响性能。

## 4\. useInsertionEffect

**useInsertionEffect：** 与 useEffect 一样，但它在所有 DOM 突变之前同步触发。

**注意：**

  * useInsertionEffect 应限于 css-in-js 库作者使用。在实际的项目中优先考虑使用 useEffect 或 useLayoutEffect 来替代；
  * 这个钩子是为了解决 `CSS-in-JS` 在渲染中注入样式的性能问题而出现的，所以在我们日常的开发中并不会用到这个钩子，但我们要知道如何去使用它。

**基本使用：**

    
    
    useInsertionEffect(callback,deps)
    

**基本用法：**

    
    
    import { useInsertionEffect } from "react";
    
    const Index: React.FC<any> = () => {
      useInsertionEffect(() => {
        const style = document.createElement("style");
        style.innerHTML = `
          .css-in-js{
            color: blue;
          }
        `;
        document.head.appendChild(style);
      }, []);
    
      return (
        <div>
          <div className="css-in-js">大家好，我是小杜杜，一起玩转Hooks吧！</div>
        </div>
      );
    };
    
    export default Index;
    

**效果：**

![](img\3\5.image)

**执行顺序：** 在目前的版本中，React 官方共提供三种有关副作用的钩子，分别是 useEffect、useLayoutEffect 和
useInsertionEffect，我们一起来看看三者的执行顺序：

    
    
    import { useEffect, useLayoutEffect, useInsertionEffect } from "react";
    
    const Index: React.FC<any> = () => {
      useEffect(() => console.log("useEffect"), []);
    
      useLayoutEffect(() => console.log("useLayoutEffect"), []);
    
      useInsertionEffect(() => console.log("useInsertionEffect"), []);
    
      return <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>;
    };
    
    export default Index;
    

**效果：**

![](img\3\6.image)

从效果上来看，可知三者的执行的顺序为：useInsertionEffect > useLayoutEffect > useEffect。

## 5\. useId

**useId：** 是一个用于生成横跨服务端和客户端的稳定的唯一 ID ，用于解决服务端与客户端产生 ID 不一致的问题，更重要的是保证了 React
v18 的 `streaming renderer （流式渲染）`中 id 的稳定性。

这里我们简单介绍一下什么是 `streaming renderer`。

在之前的 React ssr 中，hydrate（ 与 render 相同，但作用于 ReactDOMServer 渲染的容器中
）是整个渲染的，也就是说，无论当前模块有多大，都会一次性渲染，无法局部渲染。但这样就会有一个问题，如果这个模块过于庞大，请求数据量大，耗费时间长，这种效果并不是我们想要看到的。

于是在 React v18 上诞生出了 streaming renderer
（流式渲染），也就是将整个模块进行拆分，让加载快的小模块先进行渲染，大的模块挂起，再逐步加载出大模块，就可以就解决上面的问题。

此时就有可能出现：服务端和客户端注册组件的顺序不一致的问题，所以 `useId` 就是为了解决此问题而诞生的，这样就保证了 `streaming
renderer` 中 ID 的稳定性。

**基本使用：**

    
    
    const id = useId();
    

**Result：**

  * id：生成一个服务端和客户端统一的`id`。

**基本用法：**

    
    
    import { useId } from "react";
    
    const Index: React.FC<any> = () => {
      const id = useId();
    
      return <div id={id}>大家好，我是小杜杜，一起玩转Hooks吧！</div>;
    };
    
    export default Index;
    

**效果：**

![](img\3\7.image)

## 小结

至此，我们已经搞懂了 React 截止目前为止提供的 Hooks API，建议不熟悉 React
的小伙伴亲自动手试一试，毕竟眼过千遍，不如手过一遍，当你亲自实现后才会加强自己的理解，加深 Hooks 的使用。

下一节，我们将介绍自定义 hooks 的实现。

