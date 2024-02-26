通过前两节的学习，我们已经知道如何使用 Hooks API 了，在 v16.8 中，官方还提出了一个 Hooks 我们没有讲到，就是自定义
Hooks。顾名思义，自定义 Hooks 是我们自己开发的。

通过本节课的学习，你将完全了解什么是自定义 Hooks、一个优秀的自定义 Hooks 该如何设计，以及如何设计一个具备响应式的 useState。

![](img\4\1.image)

## 自定义 Hooks 究竟是什么？

react-hooks 是 React16.8
的产物，目的是增加代码的可复用性、逻辑性，并且解决函数式组件无状态的问题，这样既保留了函数式的简单，又解决了没有数据管理状态的缺陷。

而自定义 Hooks 是 react-hooks 基础上的一个扩展，它可以根据实际的业务场景、需求制定相应的 Hooks，
将对应的逻辑进行封装，从而具备逻辑性、复用性。

从本质而言， **Hooks 就是一个函数** ，可以简单地认为 Hooks 是用来处理一些通用性数据、逻辑的。

> 问：既然 Hooks 是函数，那么它与普通函数又什么关系，与函数组件又有什么关系？
>
> 答：其实三者的关系十分紧密，也是一个比较容易混淆的点。先看下图

![](img\4\2.image)

> 从图中可看出，普通函数加入 html（JSX 语法）就是函数组件，但这个组件无状态，也就是没有数据管理状态，而 Hooks
> 的作用就是让函数组件具备数据管理的能力。如果说函数组件是一辆车，那么 Hooks 就是油，驱动这辆车跑起来的燃料。

  * **Hooks 的驱动条件**

所谓驱动条件，就是会改变数据源，从而驱动整个数据状态。通常用 useState、useReducer 为驱动条件，驱动整个自定义 Hooks。

  * **通用模式**

自定义 Hooks 的名称通常以 use 开头，我们设计为：

> const [ xxx, ...] = useXXX(参数一，参数二...)

正所谓光说不练假把式，接下来我们正式进入自定义 Hooks 的讲解，直接开搞！！！

## useLatest

useLatest：永远返回最新的值，可以避免闭包问题。

> 问：什么是闭包？
>
> 答：闭包是指有权访问另一个函数作用域的变量的函数。
>
> 另外，关于 Hooks 闭包的问题，在之后讲解 useEffect 的时候详细介绍，届时会用到这个钩子。

**示例：**

    
    
    import { useState, useEffect } from "react";
    
    export default () => {
      const [count, setCount] = useState(0);
    
      useEffect(() => {
        const interval = setInterval(() => {
          console.log("count:", count);
          setCount(count + 1);
        }, 1000);
        return () => clearInterval(interval);
      }, []);
    
      return (
        <>
          <div>自定义Hooks：useLatestt</div>
          <div>count: {count}</div>
        </>
      );
    };
    

大家可以猜想下，打印出的 count 值是什么？以及页面中的 count 值是多少？

答案：打印出的 count 为 0，页面中的 count 为 1（具体原因我们在讲 useEffect 源码篇时提及，这里先看解决方法），效果如下：

![](img\4\3.image)

**解决方法：**

利用 useRef 的高级用法： **缓存数据** 去解决，并且这种方式在`react-redux`源码中进行应用，而不止是获取元素属性。

    
    
    import { useRef } from "react";
    
    const useLatest = <T,>(value: T): { readonly current: T } => {
      const ref = useRef(value);
      ref.current = value;
    
      return ref;
    };
    
    export default useLatest;
    

> 一起来看下这段代码 <T,>(value: T): { readonly current: T }。
>
> 从作用来看，这个钩子返回的永远是最新值，也就是说，这个钩子的入参与出参都是这个值，但这个值我们却不知道是 string、number
> 还是其他类型的值，这时，我们就希望它传入的值与返回的值是同种类型。
>
> 简单来说， **无论传入什么类型，都要返回对应的类型** ，这种情况必是泛型。
>
> :{readonly current: T} 代表返回结果的类型，由于我们使用的为 useRef ，所以，返回的值都在 current 内，那么
> current 的类型就是 T。
>
> 至于 readonly 则是代表的只读不可修改，因为固定模式为 current 对象，所以这里使用 readonly 。

**验证：**

我们用 useLatest 包一下 count，如：

    
    
     const ref = useLatest(count);
     
        useEffect(() => {
            const interval = setInterval(() => {
              console.log("count:", count);
              console.log("ref:", ref);
              setCount(ref.current + 1);
            }, 1000);
            return () => clearInterval(interval);
        }, []);
    

**最终效果：**

![](img\4\4.image)

此时就能拿到 count 的最新值。

## useMount 和 useUnmount

**useMount：** 只在组件初始化执行的 hook。

**useUnmount** ：只在组件卸载时的 hook。

两者都是根据 useEffect 演化而来，而 useUnmount 需要注意一下，这里传入的函数需要保持最新值，直接使用 useLatest 即可：

    
    
    // useMount
    import { useEffect } from "react";
    
    const useMount = (fn: () => void) => {
      useEffect(() => {
        fn?.();
      }, []);
    };
    
    export default useMount;
    
    // useUnmount
    import { useEffect } from "react";
    import useLatest from "../useLatest";
    
    const useUnmount = (fn: () => void) => {
      const fnRef = useLatest(fn);
    
      useEffect(
        () => () => {
          fnRef.current();
        },
        []
      );
    };
    
    export default useUnmount;
    

**示例：**

    
    
    import { useState } from "react";
    import { useMount, useUnmount } from "../../hooks";
    
    import { Button, message } from "antd";
    
    const Child = () => {
      useMount(() => {
        message.info("首次渲染");
      });
    
      useUnmount(() => {
        message.info("组件已卸载");
      });
    
      return <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>;
    };
    
    const Index = () => {
      const [flag, setFlag] = useState<boolean>(false);
    
      return (
        <div>
          <Button type="primary" onClick={() => setFlag((v) => !v)}>
            切换 {flag ? "unmount" : "mount"}
          </Button>
          {flag && <Child />}
        </div>
      );
    };
    
    export default Index;
    

**效果：**

![](img\4\5.image)

## useUnmountedRef

**useUnmountedRef：** 获取当前组件是否卸载，这个钩子的思路也很简单，只需要利用 useEffect 的状态，来保存对应的值就 ok 了。

    
    
    import { useEffect, useRef } from "react";
    
    const useUnmountedRef = (): { readonly current: boolean } => {
      const unmountedRef = useRef<boolean>(false);
    
      useEffect(() => {
        unmountedRef.current = false;
        return () => {
          unmountedRef.current = true;
        };
      }, []);
    
      return unmountedRef;
    };
    
    export default useUnmountedRef;
    

**示例：**

    
    
    import { useState } from "react";
    import { useUnmountedRef, useUnmount, useMount } from "../../hooks";
    import { Button } from "antd";
    
    const Child = () => {
      const unmountedRef = useUnmountedRef();
    
      useMount(() => {
        console.log("初始化：", unmountedRef);
      });
      useUnmount(() => {
        console.log("卸载：", unmountedRef);
      });
    
      return <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>;
    };
    
    const Index = () => {
      const [flag, setFlag] = useState<boolean>(false);
    
      return (
        <div>
          <Button type="primary" onClick={() => setFlag((v) => !v)}>
            切换 {flag ? "卸载" : "初始化"}
          </Button>
          {flag && <Child />}
        </div>
      );
    };
    
    export default Index;
    

**效果：**

![](img\4\6.image)

## useSafeState

useSafeState：使用方法与 useState 的用法完全一致，但在组件卸载后异步回调内的 setState
不再执行，这样可以避免因组件卸载后更新状态而导致的内存泄漏。

这里要注意的是卸载后的异步条件，所以直接使用 useUnmountedRef 即可，代码如下：

    
    
    import { useCallback, useState } from "react";
    import type { Dispatch, SetStateAction } from "react";
    import useUnmountedRef from "../useUnmountedRef";
    
    function useSafeState<S>(
      initialState: S | (() => S)
    ): [S, Dispatch<SetStateAction<S>>];
    function useSafeState<S = undefined>(): [
      S | undefined,
      Dispatch<SetStateAction<S | undefined>>
    ];
    function useSafeState<S>(initialState?: S | (() => S)) {
      const unmountedRef: { current: boolean } = useUnmountedRef();
      const [state, setState] = useState(initialState);
      const setCurrentState = useCallback((currentState: any) => {
        if (unmountedRef.current) return;
        setState(currentState);
      }, []);
    
      return [state, setCurrentState] as const;
    }
    
    export default useSafeState;
    

> 之所以讲 useSafeState，是因为虽然它本身的实现并不难，但在真实的环境中，我们经常使用。所以在这里，主要介绍一下
> ts，让不熟悉的小伙伴尽快熟悉起来，方便我们后面的代码的阅读。

  1. 首先，这个钩子和 useState 的用法完全一致，所以我们的入参和出参保持一致。这里使用泛型与 useLatest 的原因一样；
  2. 入参：initialState，这个参数并不是一定必需的，所以存在两种情况，一种是 S（传入什么就是什么类型）、另一种是 undefined，其中 S 还分为是否为函数。所以标准的写法是函数重载，简单理解为：可以在同一个函数下定义多种类型值，最后汇总到一块；
  3. 返参：`[state, setCurrentState] as const`，这种写法叫做断言，所谓断言，通过 as 这个参数告诉编辑器，就是这种类型，不用你再次校验。是不是很任性，像极了你的女朋友，而 `as const` 是标记为不可变，即这个数组的长度与成员类型均不可再进行修改，可翻译为 readonly `[S, Dispatch<SetStateAction<S>>]`，这样可能更加好理解一点；
  4. 至于 `Dispatch<SetStateAction<S>>` 这种写法是固定的，就是对应 useState 的第二个参数。

> 除此之外，这里还用到了 useCallback。在之前的介绍中，我说要配合使用 React.Memo，那么这里为什么要用呢？
>
> 其实这里要特意说明下，如果是在开发自定义 Hooks 的时候，可直接使用 useCallback，而在具体的业务场景中，useCallback 需要配合
> React.Memo 使用，具体为何，在之后介绍 useCallBack 源码篇中进行说明，现在只需要记住即可。

## useUpdate

**useUpdate：** 强制组件重新渲染，最终返回一个函数。

这就回到开头所说的问题，是什么驱动函数式的更新：用 useState、useReducer 作为更新条件，这里以 useReducer 做演示，毕竟大家对
useState 都很熟悉。

具体的做法是：搞个累加器，无关的变量，触发一次，就累加 1，这样就会强制刷新。

    
    
    import { useReducer } from "react";
    
    function useUpdate(): () => void {
      const [, update] = useReducer((num: number): number => num + 1, 0);
    
      return update;
    }
    
    export default useUpdate;
    

**测试：**

    
    
    import { useUpdate } from "../../hooks";
    import { Button, message } from "antd";
    
    const Index = () => {
      const update = useUpdate();
    
      return (
        <div>
          <div>时间：{Date.now()}</div>
          <Button
            type="primary"
            onClick={() => {
              update();
            }}
          >
            更新
          </Button>
        </div>
      );
    };
    
    export default Index;
    

**效果：**

![](img\4\7.image)

## useCreation

**useCreation** ：强化 useMemo 和 useRef，用法与 useMemo 一样，一般用于性能优化。

**useCreation 如何增强：**

  * useMemo 的第一个参数 fn，会缓存对应的值，那么这个值就有可能拿不到最新的值，而 useCreation 拿到的值永远都是最新值；
  * useRef 在创建复杂常量的时候，会出现潜在的性能隐患（如：实例化 `new Subject`），但 useCreation 可以有效地避免。

来简单分析一下如何实现 useCreation:

  1. 明确出参入参：useCreation 主要强化的是 useMemo，所以出入参应该保持一致。出参返回对应的值，入参共有两个，第一个对应函数，第二个对应数组（此数组可变触发）；
  2. 最新值处理：针对 useMemo 可能拿不到最新值的情况，可直接依赖 useRef 的高级用法来保存值，这样就会永远保存最新值；
  3. 触发更新条件：比较每次传入的数组，与之前对比，若不同，则触发、更新对应的函数。

    
    
    import { useRef } from "react";
    import type { DependencyList } from "react";
    
    const depsAreSame = (
      oldDeps: DependencyList,
      deps: DependencyList
    ): boolean => {
      if (oldDeps === deps) return true;
    
      for (let i = 0; i < oldDeps.length; i++) {
        if (!Object.is(oldDeps[i], deps[i])) return false;
      }
    
      return true;
    };
    
    const useCreation = <T,>(fn: () => T, deps: DependencyList) => {
      const { current } = useRef({
        deps,
        obj: undefined as undefined | T,
        initialized: false,
      });
    
      if (current.initialized === false || !depsAreSame(current.deps, deps)) {
        current.deps = deps;
        current.obj = fn();
        current.initialized = true;
      }
    
      return current.obj as T;
    };
    
    export default useCreation;
    

**分析 useRef：**

useRef 的保存值应该有哪些？其中 `deps` 和 `obj` 不必多说，一个是数组，一个是数据，是必须要保存的，除此之外，还需要保存
`initialized（初始化条件）`，这个参数的作用是应对首次保存值，之后判断是否保存，根据 deps 判断即可。

**测试：**

    
    
    import React, { useState } from "react";
    import { Button } from "antd";
    import { useCreation } from "../../hooks";
    
    const Index: React.FC<any> = () => {
      const [flag, setFlag] = useState<boolean>(false);
    
      const getNowData = () => {
        return Math.random();
      };
    
      const nowData = useCreation(() => getNowData(), []);
    
      return (
        <>
          <div>正常的函数： {getNowData()}</div>
          <div>useCreation包裹后的： {nowData}</div>
          <Button
            type="primary"
            onClick={() => {
              setFlag((v) => !v);
            }}
          >
            切换状态{JSON.stringify(flag)}
          </Button>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![](img\4\8.image)

可以看到，当无关的变量刷新时，会影响 getNowData 产生的随机值，但不会影响经过 useCreation 包裹后的值，从而增强渲染时的性能问题。

## useReactive

**背景：** 当我们开发组件或做功能复杂的页面时，会有大量的变量，再来看看 useState 的结构`const [count, setCount] =
useState<number>(0)`，假设要设置 10 个变量，那么我们是不是要设置 10 个这样的结构？

有的小伙伴会说，值设置成对象不就好了吗？但我们设置的时候必须 setCount(v => {...v, count: 7}) 这样去写，也是比较麻烦的。

其次，我们用的值和设置的值分成了两个，这样也带来了不便，因此：useReactive 可以帮我们解决这个问题。

**useReactive** ：一种具备响应式的 useState，用法与 useState 类似，但可以动态地设置值。

> 问：什么是响应式？
>
> 答：响应式是指能及时更新的数据，比如 let count = 0; count = 7；那么此时 count 的值就为 7，这样的数据就是响应式。
>
> 这里尤其注意一点，函数式组件是无法通过上面这种方式来更新视图，要牢记，有的小伙伴可能看着看着就懵了，觉得本来就这样，实际不然，所以一定要熟记概念。

**useReactive 如何设计：**

  1. useReactive 的入参、出参怎么设定？
  2. 如何制作成响应式数据？
  3. 如何使用 Ts 类型？如何优化 useReactive ？

**分析：**

useReactive 整体结构：`const state = useReactive({ count: 0 })` 。

  * 使用：state.count；
  * 设置：state.count = 7。

**数据制作成响应式，实际上分两个步骤:**

  1. 如何检测值的改变，就是当 state.count = 7 设置后，怎样让 state.count 成为 7 ？
  2. 如何刷新视图，让页面看到效果？

刷新视图用上述的 useUpdate 即可，而监测数据、改变数据可以使用 ES6 提供的 Proxy 和 Reflect 来处理。

Proxy 用于修改某些操作的默认行为，等同于在语言层面做出修改，所以属于一种“元编程”（meta
programming），即对编程语言进行编程。可以这样理解，Proxy
就是在目标对象之前设置的一层`拦截`，外界想要访问都要经过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。

而 Reflect 与 Proxy 的功能类似，但只能保持 Object 的默认行为。

至于优化，直接用 useCreation 即可，配合 useLatest 来处理存放 initialState（初始值），用来确保值永远是最新值。

    
    
    import { useUpdate, useCreation, useLatest } from "../index";
    
    const observer = <T extends Record<string, any>>(
      initialVal: T,
      cb: () => void
    ): T => {
      const proxy = new Proxy<T>(initialVal, {
        get(target, key, receiver) {
          const res = Reflect.get(target, key, receiver);
          return typeof res === "object"
            ? observer(res, cb)
            : Reflect.get(target, key);
        },
        set(target, key, val) {
          const ret = Reflect.set(target, key, val);
          cb();
          return ret;
        },
      });
    
      return proxy;
    };
    
    const useReactive = <T extends Record<string, any>>(initialState: T): T => {
      const ref = useLatest<T>(initialState);
      const update = useUpdate();
    
      const state = useCreation(() => {
        return observer(ref.current, () => {
          update();
        });
      }, []);
    
      return state;
    };
    
    export default useReactive;
    

**Ts 分析：** 这里使用了一个新的的知识点：`Record<string, any>`。

**Record 的语法介绍：** `Record<K extends keyof any, T>`。

**作用：** 将 `K` 中所有的属性转化为 `T` 类型。

**举个例子：**

    
    
        interface Props {
            name: string,
            age: number
        }
    
        type InfoProps = 'JS' | 'TS'
        
        const Info: Record<InfoProps, Props> = {
            JS: {
                name: '小杜杜',
                age: 7
            },
            TS: {
                name: 'TypeScript',
                age: 11
            }
        }
    

也就是说，InfoProps 的每一项属性都包含 Props 的属性。

> 问：为什么 Proxy 和 Reflect 联合使用？单独使用 Proxy 不行吗？
>
> 答：ES6 中的 Proxy 和 Reflect 在平常的开发中可能运用的比较少，很多小伙伴可能并不了解，Proxy 和 Reflect
> 一般是对数据的劫持，有点类似于 ES5 中的 `Object.defineProperty()`，但功能要更加强大。
>
> 两者联合使用的根本原因是：this 的指向，至于具体为什么，这里就不做过多的介绍，但要强调一点，Proxy 和 Reflect 的作用非常大，在
> Vue/corejs 的源码中有大量的应用，掌握这两个 API 非常有必要。

**验证：**

    
    
    import { useReactive } from "../../hooks";
    import { Button, Input } from "antd";
    
    const Index = () => {
      const state = useReactive<any>({
        count: 0,
        name: "大家好，我是小杜杜，一起玩转Hooks吧！",
        flag: true,
        arr: [],
        bugs: ["小杜杜", "react", "hook"],
        addBug(bug: string) {
          this.bugs.push(bug);
        },
        get bugsCount() {
          return this.bugs.length;
        },
      });
    
      return (
        <div>
          <div style={{ fontWeight: "bold" }}>基本使用：</div>
          <div style={{ marginTop: 8 }}> 对数字进行操作：{state.count}</div>
          <div
            style={{
              margin: "8px 0",
              display: "flex",
              justifyContent: "flex-start",
            }}
          >
            <Button type="primary" onClick={() => state.count++}>
              加1
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => state.count--}
            >
              减1
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => (state.count = 7)}
            >
              设置为7
            </Button>
          </div>
          <div style={{ marginTop: 8 }}> 对字符串进行操作：{state.name}</div>
          <div
            style={{
              margin: "8px 0",
              display: "flex",
              justifyContent: "flex-start",
            }}
          >
            <Button type="primary" onClick={() => (state.name = "小杜杜")}>
              设置为小杜杜
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => (state.name = "Domesy")}
            >
              设置为Domesy
            </Button>
          </div>
          <div style={{ marginTop: 8 }}>
            {" "}
            对布尔值进行操作：{JSON.stringify(state.flag)}
          </div>
          <div
            style={{
              margin: "8px 0",
              display: "flex",
              justifyContent: "flex-start",
            }}
          >
            <Button type="primary" onClick={() => (state.flag = !state.flag)}>
              切换状态
            </Button>
          </div>
          <div style={{ marginTop: 8 }}>
            {" "}
            对数组进行操作：{JSON.stringify(state.arr)}
          </div>
          <div
            style={{
              margin: "8px 0",
              display: "flex",
              justifyContent: "flex-start",
            }}
          >
            <Button
              type="primary"
              onClick={() => state.arr.push(Math.floor(Math.random() * 100))}
            >
              push
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => state.arr.pop()}
            >
              pop
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => state.arr.shift()}
            >
              shift
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => state.arr.unshift(Math.floor(Math.random() * 100))}
            >
              unshift
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => state.arr.reverse()}
            >
              reverse
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => state.arr.sort()}
            >
              sort
            </Button>
          </div>
          <div style={{ fontWeight: "bold", marginTop: 8 }}>计算属性：</div>
          <div style={{ marginTop: 8 }}>数量：{state.bugsCount} 个</div>
          <div style={{ margin: "8px 0" }}>
            <form
              onSubmit={(e) => {
                state.bug ? state.addBug(state.bug) : state.addBug("domesy");
                state.bug = "";
                e.preventDefault();
              }}
            >
              <Input
                type="text"
                value={state.bug}
                style={{ width: 200 }}
                onChange={(e) => (state.bug = e.target.value)}
              />
              <Button type="primary" htmlType="submit" style={{ marginLeft: 8 }}>
                增加
              </Button>
              <Button style={{ marginLeft: 8 }} onClick={() => state.bugs.pop()}>
                删除
              </Button>
            </form>
          </div>
          <ul>
            {state.bugs.map((bug: any, index: number) => (
              <li key={index}>{bug}</li>
            ))}
          </ul>
        </div>
      );
    };
    
    export default Index;
    

**效果：**

![](img\4\9.image)

## 小结

本节共讲解了
useLatest、useMount、useUnmount、useUnmountedRef、useSafeState、useUpdate、useCreation、useReactive
共 8 个 Hooks。

了解自定义 Hooks 的驱动条件和通用模式，配合 ts，讲解如何实现自定义 Hooks，大体步骤如下：

  1. 确定自定义 Hooks 的入参和出参；
  2. 优化方面优先考虑 useCreation，函数优化使用 useCallback；
  3. 保存值使用 useLatest，如果复杂的话，依旧用 useRef 的高级用法处理，如 useCreation 的实现；
  4. 最后使用 ts 来优化类型，因为是通用的 Hooks，所以基本都会使用泛型这一概念，如果对 ts 不清楚，建议多看看 ts 的语法，因为 ts 也是主流，在现在的项目中基本都会使用 ts。

最后再说一句：当我们自己想实现一个这样的功能，该如何思考？方法有的时候比结果更为重要，所以建议看文章的小伙伴都能亲自实现一遍，加深对 Hooks 的感悟。

