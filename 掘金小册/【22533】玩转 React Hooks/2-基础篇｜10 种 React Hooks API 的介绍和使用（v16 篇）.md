俗话说得好：工欲善其事必先利其器，就是说，我们想要去玩转`React Hooks`，就必须知道 React 官方提供了哪些 Hooks，如何去使用这些
Hooks。

> 问：工作中只用到`useState`、`useEffect`等 Hooks，有必要学习不常见的 Hooks 吗？
>
>
> 答：作为一名前端，应该有属于自己的技术体系，由浅入深，从广度到深度，一步一步地逐渐将架子搭起来，就算在工作中不常使用，但至少要有个概念。知识广度的提升可以让解决方法多一种，所以我们应该熟练掌握这些
> API。

通过本节的阅读，你将学会 `React v16.8` 中提供的
`useState`、`useEffect`、`useContext`、`useReducer`、`useMemo`、`useCallback`、`useRef`、`useImperativeHandle`、`useLayoutEffect`、`useDebugValue`
10 个 API 的使用方法。

![](img\2\1.image)

## 1\. useState

**useState：** 定义变量，使其具备类组件的 `state`，让函数式组件拥有更新视图的能力。

**基本使用：**

    
    
    const [state, setState] = useState(initData)
    

**Params：**

  * initData：默认初始值，有两种情况：函数和非函数，如果是函数，则函数的返回值作为初始值。

**Result：**

  * state：数据源，用于渲染`UI 层`的数据源；
  * setState：改变数据源的函数，可以理解为类组件的 `this.setState`。

**基本用法：** 主要介绍两种`setState`的使用方法。

    
    
    import { useState } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      const [count, setCount] = useState<number>(0);
    
      return (
        <>
          <div>数字：{count}</div>
          <Button type="primary" onClick={() => setCount(count + 1)}>
            第一种方式+1
          </Button>
          <Button
            type="primary"
            style={{ marginLeft: 10 }}
            onClick={() => setCount((v) => v + 1)}
          >
            第二种方式+1
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\2.image)

**注意：** `useState` 有点类似于
`PureComponent`，它会进行一个比较浅的比较，这就导致了一个问题，如果是对象直接传入的时候，并不会实时更新，这点一定要切记。

我们做个简单的对比，比如：

    
    
    import { useState } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      const [state, setState] = useState({ number: 0 });
      let [count, setCount] = useState(0);
    
      return (
        <>
          <div>数字形式：{count}</div>
          <Button
            type="primary"
            onClick={() => {
              count++;
              setCount(count);
            }}
          >
            点击+1
          </Button>
          <div>对象形式：{state.number}</div>
          <Button
            type="primary"
            onClick={() => {
              state.number++;
              setState(state);
            }}
          >
            点击+1
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\3.image)

## 2\. useEffect

**useEffect：** 副作用，这个钩子成功弥补了函数式组件没有生命周期的缺陷，是我们最常用的钩子之一。

**基本使用：**

    
    
    useEffect(()=>{ 
        return destory
    }, deps)
    

**Params：**

  * callback：useEffect 的第一个入参，最终返回 `destory`，它会在下一次 callback 执行之前调用，其作用是清除上次的 callback 产生的副作用；
  * deps：依赖项，可选参数，是一个数组，可以有多个依赖项，通过依赖去改变，执行上一次的 callback 返回的 destory 和新的 effect 第一个参数 callback。

是不是听介绍感觉有点懵，别着急，我们依此来看看。

**模拟挂载和卸载阶段** **：**

事实上，destory 会用在组件卸载阶段上，把它当作组件卸载时执行的方法就 ok，通常用于监听 `addEventListener` 和
`removeEventListener` 上，如：

    
    
    import { useState, useEffect } from "react";
    import { Button } from "antd";
    
    const Child = () => {
      useEffect(() => {
        console.log("挂载");
    
        return () => {
          console.log("卸载");
        };
      }, []);
    
      return <div>大家好，我是小杜杜，一起学习hooks吧！</div>;
    };
    
    const Index: React.FC<any> = () => {
      const [flag, setFlag] = useState(false);
    
      return (
        <>
          <Button
            type="primary"
            onClick={() => {
              setFlag((v) => !v);
            }}
          >
            {flag ? "卸载" : "挂载"}
          </Button>
          {flag && <Child />}
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\4.image)

**依赖变化：**

`dep`的个数决定`callback`什么时候执行，如：

    
    
    import { useState, useEffect } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      const [number, setNumber] = useState(0);
      const [count, setCount] = useState(0);
    
      useEffect(() => {
        console.log("count改变才会执行");
      }, [count]);
    
      return (
        <>
          <div>
            number: {number} count: {count}
          </div>
          <Button type="primary" onClick={() => setNumber((v) => v + 1)}>
            number + 1
          </Button>
          <Button
            type="primary"
            style={{ marginLeft: 10 }}
            onClick={() => setCount((v) => v + 1)}
          >
            count + 1
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\5.image)

**无限执行：**

当 useEffect 的第二个参数 deps
不存在时，会无限执行。更加准确地说，只要数据源发生变化（不限于自身中），该函数都会执行，所以请不要这么做，否则会出现不可控的现象。

    
    
    import { useState, useEffect } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      const [count, setCount] = useState(0);
      const [flag, setFlag] = useState(false);
    
      useEffect(() => {
        console.log("大家好，我是小杜杜，一起学习hooks吧！");
      });
    
      return (
        <>
          <Button type="primary" onClick={() => setCount((v) => v + 1)}>
            数字加一：{count}
          </Button>
          <Button
            type="primary"
            style={{ marginLeft: 10 }}
            onClick={() => setFlag((v) => !v)}
          >
            状态切换：{JSON.stringify(flag)}
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\6.image)

## 3\. useContext

**useContext：** 上下文，类似于 `Context`，其本意就是设置全局共享数据，使所有组件可跨层级实现共享。

useContext 的参数一般是由 `createContext` 创建，或者是父级上下文 `context`传递的，通过
`CountContext.Provider` 包裹的组件，才能通过 `useContext` 获取对应的值。我们可以简单理解为 `useContext`
代替 `context.Consumer` 来获取 `Provider` 中保存的 `value` 值。

**基本使用：**

    
    
    const contextValue = useContext(context)
    

**Params：**

  * context：一般而言保存的是 context 对象。

**Result：**

  * contextValue：返回的数据，也就是`context`对象内保存的`value`值。

**基本用法：** 子组件 Child 和孙组件 Son，共享父组件 Index 的数据 count。

    
    
    import { useState, createContext, useContext } from "react";
    import { Button } from "antd";
    
    const CountContext = createContext(-1);
    
    const Child = () => {
      const count = useContext(CountContext);
    
      return (
        <div style={{ marginTop: 10 }}>
          子组件获取到的count: {count}
          <Son />
        </div>
      );
    };
    
    const Son = () => {
      const count = useContext(CountContext);
    
      return <div style={{ marginTop: 10 }}>孙组件获取到的count: {count}</div>;
    };
    
    const Index: React.FC<any> = () => {
      let [count, setCount] = useState(0);
    
      return (
        <>
          <div>父组件中的count：{count}</div>
          <Button type="primary" onClick={() => setCount((v) => v + 1)}>
            点击+1
          </Button>
          <CountContext.Provider value={count}>
            <Child />
          </CountContext.Provider>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\7.image)

## 4\. useReducer

useReducer： 功能类似于 `redux`，与 redux 最大的不同点在于它是单个组件的状态管理，组件通讯还是要通过
props。简单地说，useReducer 相当于是 useState 的升级版，用来处理复杂的 state 变化。

**基本使用：**

    
    
    const [state, dispatch] = useReducer(
        (state, action) => {}, 
        initialArg,
        init
    );
    

**Params：**

  * reducer：函数，可以理解为 redux 中的 reducer，最终返回的值就是新的数据源 state；
  * initialArg：初始默认值；
  * init：惰性初始化，可选值。

**Result：**

  * state：更新之后的数据源；
  * dispatch：用于派发更新的`dispatchAction`，可以认为是`useState`中的`setState`。

> 问：什么是惰性初始化？
>
> 答：惰性初始化是一种延迟创建对象的手段，直到被需要的第一时间才去创建，这样做可以将用于计算 state 的逻辑提取到 reducer
> 外部，这也为将来对重置 state 的 action 做处理提供了便利。换句话说，如果有 `init`，就会取代 `initialArg`。

**基本用法：**

    
    
    import { useReducer } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      const [count, dispatch] = useReducer((state: number, action: any) => {
        switch (action?.type) {
          case "add":
            return state + action?.payload;
          case "sub":
            return state - action?.payload;
          default:
            return state;
        }
      }, 0);
    
      return (
        <>
          <div>count：{count}</div>
          <Button
            type="primary"
            onClick={() => dispatch({ type: "add", payload: 1 })}
          >
            加1
          </Button>
          <Button
            type="primary"
            style={{ marginLeft: 10 }}
            onClick={() => dispatch({ type: "sub", payload: 1 })}
          >
            减1
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\8.image)

**特别注意：** 在 reducer 中，如果返回的 state 和之前的 state 值相同，那么组件将不会更新。

比如这个组件是子组件，并不是组件本身，然后我们对上面的例子稍加更改，看看这个问题：

    
    
    const Index: React.FC<any> = () => {
      console.log("父组件发生更新");
      ...
      return (
        <>
            ...
          <Button
            type="primary"
            style={{ marginLeft: 10 }}
            onClick={() => dispatch({ type: "no", payload: 1 })}
          >
            无关按钮
          </Button>
          <Child count={count} />
        </>
      )
    };
    
    const Child: React.FC<any> = ({ count }) => {
      console.log("子组件发生更新");
      return <div>在子组件的count：{count}</div>;
    };
    

**效果：**

![](img\2\9.image)

可以看到，当 count 无变化时，子组件并不会更新，这点还希望大家铭记。

## 5\. useMemo

**场景：** 在每一次的状态更新中，都会让组件重新绘制，而重新绘制必然会带来不必要的性能开销，为了防止没有意义的性能开销，React Hooks 提供了
useMemo 函数。

useMemo：理念与 `memo` 相同，都是判断是否满足当前的限定条件来决定是否执行`callback`
函数。它之所以能带来提升，是因为在依赖不变的情况下，会返回相同的引用，避免子组件进行无意义的重复渲染。

**基本使用：**

    
    
    const cacheData = useMemo(fn, deps)
    

**Params：**

  * fn：函数，函数的返回值会作为缓存值；
  * deps：依赖项，数组，会通过数组里的值来判断是否进行 fn 的调用，如果发生了改变，则会得到新的缓存值。

**Result：**

  * cacheData：更新之后的数据源，即 fn 函数的返回值，如果 deps 中的依赖值发生改变，将重新执行 fn，否则取上一次的缓存值。

**问题源泉：**

可能光听概念，各位小伙伴觉得太过于枯燥，也不是很明白，我们举个小案例来看看：

    
    
    import { useState } from "react";
    import { Button } from "antd";
    
    const usePow = (list: number[]) => {
      return list.map((item: number) => {
        console.log("我是usePow");
        return Math.pow(item, 2);
      });
    };
    
    const Index: React.FC<any> = () => {
      let [flag, setFlag] = useState(true);
    
      const data = usePow([1, 2, 3]);
    
      return (
        <>
          <div>数字集合：{JSON.stringify(data)}</div>
          <Button type="primary" onClick={() => setFlag((v) => !v)}>
            状态切换{JSON.stringify(flag)}
          </Button>
        </>
      );
    };
    
    export default Index;
    

从例子中来看， 按钮切换的 flag 应该与 usePow 的数据毫无关系，那么可以猜一猜，当我们切换按钮的时候，usePow 中是否会打印
`我是usePow` ？

我们直接看看运行后的效果：

![](img\2\10.image)

可以看到，当我们点击按钮后，会打印`我是usePow`，这样就会产生开销。毫无疑问，这种开销并不是我们想要见到的结果，所以有了 `useMemo`。
并用它进行如下改造：

    
    
    const usePow = (list: number[]) => {
      return useMemo(
        () =>
          list.map((item: number) => {
            console.log(1);
            return Math.pow(item, 2);
          }),
        []
      );
    };
    

**优化后的效果：**

![](img\2\11.image)

## 6\. useCallback

useCallback：与 useMemo 极其类似，甚至可以说一模一样，唯一不同的点在于，useMemo 返回的是值，而 useCallback
返回的是函数。

**基本使用：**

    
    
    const resfn = useCallback(fn, deps)
    

**Params：**

  * fn：函数，函数的返回值会作为缓存值；
  * deps：依赖项，数组，会通过数组里的值来判断是否进行 fn 的调用，如果依赖项发生改变，则会得到新的缓存值。

**Result：**

  * resfn：更新之后的数据源，即 fn 函数，如果 deps 中的依赖值发生改变，将重新执行 fn，否则取上一次的函数。

**基础用法：**

    
    
    import { useState, useCallback, memo } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      let [count, setCount] = useState(0);
      let [flag, setFlag] = useState(true);
    
      const add = useCallback(() => {
        setCount(count + 1);
      }, [count]);
    
      return (
        <>
          <TestButton onClick={() => setCount((v) => v + 1)}>普通点击</TestButton>
          <TestButton onClick={add}>useCallback点击</TestButton>
          <div>数字：{count}</div>
          <Button type="primary" onClick={() => setFlag((v) => !v)}>
            切换{JSON.stringify(flag)}
          </Button>
        </>
      );
    };
    
    const TestButton = memo(({ children, onClick = () => {} }: any) => {
      console.log(children);
      return (
        <Button
          type="primary"
          onClick={onClick}
          style={children === "useCallback点击" ? { marginLeft: 10 } : undefined}
        >
          {children}
        </Button>
      );
    });
    
    export default Index;
    

简要说明下，`TestButton` 里是个按钮，分别存放着有无 useCallback 包裹的函数，在父组件 Index 中有一个 flag
变量，这个变量同样与 count 无关，那么，我们切换按钮的时候，`TestButton` 会怎样执行呢？效果如下：

![](img\2\12.image)

可以看到，我们切换 flag 的时候，没有经过 useCallback
的函数会再次执行，而包裹的函数并没有执行（点击“普通点击”按钮的时候，useCallbak 的依赖项 count 发生了改变，所以会打印出
useCallback 点击）。

> 问：为什么在 TestButton 中使用了 React.memo，不使用会怎样？
>
> 答：useCallback 必须配合 React.memo
> 进行优化，如果不配合使用，性能不但不会提升，还有可能降低。至于为什么，容我在这里先卖个关子，在后面讲解 useCallback
> 源码中详细说明，这节我们只要学会使用即可。

## 7\. useRef

**useRef：** 用于获取当前元素的所有属性，除此之外，还有一个高级用法：缓存数据（后面介绍`自定义Hooks`的时候会详细介绍）。

**基本使用：**

    
    
    const ref = useRef(initialValue);
    

**Params：**

  * initialValue：初始值，默认值。

**Result：**

  * ref：返回的一个 current 对象，这个 current 属性就是 ref 对象需要获取的内容。

**基本用法：**

    
    
    import { useState, useRef } from "react";
    
    const Index: React.FC<any> = () => {
      const scrollRef = useRef<any>(null);
      const [clientHeight, setClientHeight] = useState<number>(0);
      const [scrollTop, setScrollTop] = useState<number>(0);
      const [scrollHeight, setScrollHeight] = useState<number>(0);
    
      const onScroll = () => {
        if (scrollRef?.current) {
          let clientHeight = scrollRef?.current.clientHeight; //可视区域高度
          let scrollTop = scrollRef?.current.scrollTop; //滚动条滚动高度
          let scrollHeight = scrollRef?.current.scrollHeight; //滚动内容高度
          setClientHeight(clientHeight);
          setScrollTop(scrollTop);
          setScrollHeight(scrollHeight);
        }
      };
    
      return (
        <>
          <div>
            <p>可视区域高度：{clientHeight}</p>
            <p>滚动条滚动高度：{scrollTop}</p>
            <p>滚动内容高度：{scrollHeight}</p>
          </div>
          <div
            style={{ height: 200, border: "1px solid #000", overflowY: "auto" }}
            ref={scrollRef}
            onScroll={onScroll}
          >
            <div style={{ height: 2000 }}></div>
          </div>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\13.image)

## 8\. useImperativeHandle

useImperativeHandle：可以通过 ref 或 forwardRef 暴露给父组件的实例值，所谓的实例值是指值和函数。

实际上这个钩子非常有用，简单来讲，这个钩子可以让不同的模块关联起来，让父组件调用子组件的方法。

举个例子，在一个页面很复杂的时候，我们会将这个页面进行模块化，这样会分成很多个模块，有的时候我们需要在`最外层的组件上`控制其他组件的方法，希望最外层的点击事件同时执行`子组件的事件`，这时就需要
useImperativeHandle 的帮助（在不用`redux`等状态管理的情况下）。

**基本使用：**

    
    
    useImperativeHandle(ref, createHandle, deps)
    

**Params：**

  * ref：接受 useRef 或 forwardRef 传递过来的 ref；
  * createHandle：处理函数，返回值作为暴露给父组件的 ref 对象；
  * deps：依赖项，依赖项如果更改，会形成新的 ref 对象。

**父组件是函数式组件：**

    
    
    import { useState, useRef, useImperativeHandle } from "react";
    import { Button } from "antd";
    
    const Child = ({cRef}:any) => {
    
      const [count, setCount] = useState(0)
    
      useImperativeHandle(cRef, () => ({
        add
      }))
    
      const add = () => {
        setCount((v) => v + 1)
      }
    
      return <div>
        <p>点击次数：{count}</p>
        <Button onClick={() => add()}> 子组件的按钮，点击+1</Button>
      </div>
    }
    
    const Index: React.FC<any> = () => {
      const ref = useRef<any>(null)
    
      return (
        <>
          <div>大家好，我是小杜杜，一起学习hooks吧！</div>
          <div></div>
          <Button
            type="primary"
            onClick={() =>  ref.current.add()}
          >
            父组件上的按钮，点击+1
          </Button>
          <Child cRef={ref} />
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\2\14.image)

**当父组件是类组件时：**

如果当前的父组件是 Class 组件，此时不能使用 useRef，而是需要用 forwardRef 来协助我们处理。

forwardRef：引用传递，是一种通过组件向子组件自动传递引用 ref
的技术。对于应用者的大多数组件来说没什么作用，但对于一些重复使用的组件，可能有用。

听完 forwardRef 的概念是不是有点云里雾里的感觉，什么是引用传递呢？是不是感觉很陌生，官方中，也对 forwardRef
的介绍很少，别纠结，先来思考下。

> 问：在上述的例子中，我们通过 cRef 传递 ref，为什么不能直接用 ref 传递 ref 呢（毕竟我们平常传递的参数都会尽可能保持一致）？
>
> 简化一下问题：函数式组件中允许 ref 通过 props 传参吗？
>
> 答：在函数式组件中不允许 ref 作为参数，除了 ref，key 也不允许作为参数，原因是在 React 内部中，ref 和 key 会形成单独的
> key 名。

回过头来看 forwardRef，所谓引用传递就是为了解决无法传递 ref 的问题。

经过 forwardRef 包裹后，会将 props（其余参数）和 ref 拆分出来，ref 会作为第二个参数进行传递。如：

    
    
    import { useState, useRef, useImperativeHandle, Component, forwardRef } from "react";
    import { Button } from "antd";
    
    const Child = (props:any, ref:any) => {
    
      const [count, setCount] = useState(0)
    
      useImperativeHandle(ref, () => ({
        add
      }))
    
      const add = () => {
        setCount((v) => v + 1)
      }
    
      return <div>
        <p>点击次数：{count}</p>
        <Button onClick={() => add()}> 子组件的按钮，点击+1</Button>
      </div>
    }
    
    const ForwardChild = forwardRef(Child)
    
    class Index extends Component{
      countRef:any = null
      render(){
        return   <>
          <div>大家好，我是小杜杜，一起学习hooks吧！</div>
          <div></div>
          <Button
            type="primary"
            onClick={() => this.countRef.add()}
          >
            父组件上的按钮，点击+1
          </Button>
          <ForwardChild ref={node => this.countRef = node} />
        </>
      }
    }
    
    export default Index;
    

## 9\. useLayoutEffect

**useLayoutEffect：** 与 useEffect 基本一致，不同点在于它是同步执行的。简要说明：

  * 执行顺序：useLayoutEffect 是在 DOM 更新之后，浏览器绘制之前的操作，这样可以更加方便地修改 DOM，获取 DOM 信息，这样浏览器只会绘制一次，所以 useLayoutEffect 的执行顺序在 useEffect 之前；
  * useLayoutEffect 相当于有一层防抖效果；
  * useLayoutEffect 的 callback 中会阻塞浏览器绘制。

**基本使用：**

    
    
    useLayoutEffect(callback,deps)
    

**防抖效果：**

    
    
    import { useState, useEffect, useLayoutEffect } from "react";
    
    const Index: React.FC<any> = () => {
      let [count, setCount] = useState(0);
      let [count1, setCount1] = useState(0);
    
      useEffect(() => {
        if(count === 0){
          setCount(10 + Math.random() * 100)
        }
      }, [count])
    
      useLayoutEffect(() => {
        if(count1 === 0){
          setCount1(10 + Math.random() * 100)
        }
      }, [count1])
    
      return (
        <>
          <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>
          <div>useEffect的count:{count}</div>
          <div>useLayoutEffect的count:{count1}</div>
        </>
      );
    };
    
    export default Index;
    

在这个例子中，我们分别设置 count 和 count1 两个变量，初始值都为 0，然后分别通过 useEffect 和 useLayout
控制，通过随机值来变更两个变量的值。也就是说，count 和 count1 连续变更了两次。

**效果：**

![](img\2\15.image)

从效果上来看，count 要比 count1 更加抖动（效果可能感觉不到，建议自己试试，刷新的快点就能看到效果）。

这是因为两者的执行顺序，简要分析下：

  * useEffect 执行顺序：setCount 设置 => 在 DOM 上渲染 => useEffect 回调 => setCount 设置 => 在 DOM 上渲染。
  * useLayoutEffect 执行顺序：setCount 设置 => useLayoutEffect 回调 => setCount 设置 => 在 DOM 上渲染。

可以看出，useEffect 实际进行了两次渲染，这样就可能导致浏览器再次回流和重绘，增加了性能上的损耗，从而会有闪烁突兀的感觉。

> 问：既然 useEffect 会执行两次渲染，导致回流和重绘，相比之下， useLayoutEffect 的效果要更好，那么为什么都用
> useEffect 而不用 useLayoutEffect 呢？
>
> 答：根本原因还是同步和异步，虽然 useLayoutEffect 只会渲染一次，但切记，它是同步，类比于 Class 组件中，它更像
> componentDidMount，因为它们都是同步执行。既然是同步，就有可能阻塞浏览器的渲染，而 useEffect 是异步的，并不会阻塞渲染。
>
> 其次，给用户的感觉不同，对比两者的执行顺序，useLayoutEffect 要经过本身的回调才设置到 DOM 上，也就是说， useEffect
> 呈现的速度要快于 useLayoutEffect，让用户有更快的感知。
>
> 最后，即使 useEffect 要渲染两次，但从效果上来看，变换的时间非常短，这样情况下，也无所谓，除非闪烁、突兀的感觉非常明显，才会去考虑使用
> useLayoutEffect 去解决。

## 10\. useDebugValue

**useDebugValue：** 可用于在 React 开发者工具中显示自定义 Hook 的标签。这个 Hooks 目的就是检查自定义 Hooks。

**注意：** 这个标签并不推荐向每个 hook 都添加 debug 值。当它作为共享库的一部分时才最有价值。（也就是自定义 Hooks
被复用的值）。因为在一些情况下，格式化值可能是一项开销很大的操作，除非你需要检查 Hook，否则没有必要这么做。

**基本使用：**

    
    
    useDebugValue(value, (status) => {})
    

**Params：**

  * value：判断的值；
  * callback：可选，这个函数只有在 Hook 被检查时才会调用，它接受 debug 值作为参数，并且会返回一个格式化的显示值。

**基本用法：**

    
    
    function useFriendStatus(friendID) {
      const [isOnline, setIsOnline] = useState(null);
    
      // ...
    
      // 在开发者工具中的这个 Hook 旁边显示标签  
      // e.g. "FriendStatus: Online"  useDebugValue(isOnline ? 'Online' : 'Offline');
      return isOnline;
    }
    

## 小结

本节主要讲 `React v16` 中的 Hooks，共有 useState、useEffect 等 10 个 Hooks API。

相信大家已经完全清楚这些钩子的用法，至于源码方面，我们留到后面去介绍，还是先以使用为目的，之后再一步一步进行探索。

下节讲解 `React v18` 中的 Hooks，带你全面了解 React v18 的官方 Hooks API 的使用方式。

