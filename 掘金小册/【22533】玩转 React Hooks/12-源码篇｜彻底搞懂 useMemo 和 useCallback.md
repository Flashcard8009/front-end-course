在 React Hooks 中，有专门针对优化的两个 Hooks，它们分别是 `useMemo` 和
`useCallback`。同时，它们哥俩也是最具争议的 Hooks，因为如果使用不当，非但达不到优化的效果，还有可能降低性能，让人头大。

通过本章的阅读，将彻底搞懂 useMemo 和 useCallback，同时了解 React 中其他性能优化的方法。

![useMemo和useCallback.png](img\12\1.image)

# useMemo、useCallback 源码

从源码角度上来看，useMemo 和 useCallback 并不复杂，甚至两者的源码十分相似，所以这里我们直接放到一起观看。

## mountMemo/mountCallback（初始化）

    
    
    // mountMemo
    function mountMemo<T>(
      nextCreate: () => T, 
      deps: Array<mixed> | void | null,
    ): T {
      const hook = mountWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      const nextValue = nextCreate();
      hook.memoizedState = [nextValue, nextDeps];
      return nextValue;
    }
    
    // mountCallback
    function mountCallback<T>(
      callback: T,
      deps: Array<mixed> | void | null
    ): T {
      const hook = mountWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      hook.memoizedState = [callback, nextDeps];
      return callback;
    }
    

在初始化中，useMemo 首先创建一个 hook，然后判断 deps 的类型，执行 nextCreate，这个参数是需要缓存的值，然后将 **值与
deps 保存到 memoizedState 上** 。

而 useCallback 更加简单， **直接将 callback和 deps 存入到 memoizedState 里。**

## updateMemo/updateCallback（更新）

    
    
    // updateMemo
    function updateMemo<T>(
      nextCreate: () => T,
      deps: Array<mixed> | void | null,
    ): T {
      const hook = updateWorkInProgressHook();
      // 判断新值
      const nextDeps = deps === undefined ? null : deps;
      const prevState = hook.memoizedState;
      if (prevState !== null) {
        if (nextDeps !== null) {
          //之前保存的值
          const prevDeps: Array<mixed> | null = prevState[1];
          // 与useEffect判断deps一致
          if (areHookInputsEqual(nextDeps, prevDeps)) {
            return prevState[0];
          }
        }
      }
      const nextValue = nextCreate();
      hook.memoizedState = [nextValue, nextDeps];
      return nextValue;
    }
    
    // updateCallback
    function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
      const hook = updateWorkInProgressHook();
      const nextDeps = deps === undefined ? null : deps;
      const prevState = hook.memoizedState;
      if (prevState !== null) {
        if (nextDeps !== null) {
            //之前保存的值
          const prevDeps: Array<mixed> | null = prevState[1];
          // 与useEffect判断deps一致
          if (areHookInputsEqual(nextDeps, prevDeps)) {
            return prevState[0];
          }
        }
      }
      hook.memoizedState = [callback, nextDeps];
      return callback;
    }
    

在更新过程中，useMemo 实际上只做了一件事，就是通过判断两次的 deps 是否发生改变，如果发生改变，则重新执行
`nextCreate()`，将得到的新值重新复制给 memoizedState；如果没发生改变，则直接返回缓存的值。

而 useCallBack 也是同理。通过判断 deps 是否相等的 areHookInputsEqual，与 useEffect
中的一致，所以这里不做过多赘述。

useMemo 和 useCallback 的关系：

从源码角度上来看，无论初始化，亦或者更新，useMemo 比 useCallback 多了一步，即执行 `nextCreate()` 的步骤，那么说明
`useCallback(fn, deps)` 等价于 `useMemo(() => fn, deps)`。

> 注意：useMemo 中的 nextCreate() 中如果引用了 useState
> 等信息，无法被垃圾机制回收（闭包问题），那么访问的属性有可能不是最新的值，所以需要将引用的值传递给 deps，则重新执行 nextCreate()。

# 性能优化的几种方案

我们知道 useMemo、 useCallback 是函数组件提供的优化方案，除此之外，React 还提供其余两种优化方案，接下来一起来看看，有何异同。

## 1.类组件的性能优化

在类组件中主要包含两种方式，分别是 `shouldComponentUpdate` 和`PureComponent`。

**shouldComponentUpdate(nextProps, nextState)** ：生命周期函数，通过比较`nextProps（当前组件的
this.props）` 和 `nextState（当前组件的 this.state）`，来判断当前组件是否有必要继续执行更新过程。

如果 shouldComponentUpdate 返回的结果为 true，则继续执行对应的更新；如果为
false，则代表停止更新，用于减少组件的不必要渲染，从而优化性能。

**PureComponent** ：与 Component 的用法基本一致，但 PureComponent 会对props 和 state
进行浅比较，从而跳过不必要的更新（减少 render 的次数），提高组件性能。

那么浅比较是什么呢？先举个例子来看看：

    
    
    import { PureComponent } from "react";
    import { Button } from "antd";
    
    class Index extends PureComponent<any, any> {
      constructor(props: any) {
        super(props);
        this.state = {
          data: {
            number: 0,
          },
        };
      }
    
      render() {
        const { data } = this.state;
        return (
          <>
            <div>大家好，我是小杜杜，一起玩转Hooks吧！</div>
            <div> 数字： {data.number}</div>
            <Button
              type="primary"
              onClick={() => {
                const { data } = this.state;
                data.number++;
                this.setState({ data });
              }}
            >
              数字加1
            </Button>
          </>
        );
      }
    }
    
    export default Index;
    

**效果：**

![img.gif](img\12\2.image)

当点击按钮时，对应的数字并没有变化，这是因为 PureComponent 会比较两次的 data 对象，它会认为这种写法并没有改变原先的
data，所以不会改变。

解决方法：

    
    
    this.setState({ data: {...data} })
    

### 对比 shouldComponentUpdate 和 PureComponent

首先要特别明确 **shouldComponentUpdate 是生命周期的方法，而 PureComponent 是组件。**

换言之，在 PureComponent 也可以调取 shouldComponentUpdate 函数，如果调取，则会对新旧 props、state 进行
`shallowEqual` 浅比较，另外 **shouldComponentUpdate 的权重要高于 PureComponent** 。

### shallowEqual 浅比较

我们可以简单地看下对应的源码，其中有一个专门检查是否更新的函数：`checkShouldComponentUpdate`。

文件位置：`packages/react-reconciler/src/ReactFiberClassComponent.js`

    
    
    function checkShouldComponentUpdate(workInProgress, ctor, oldProps, newProps, oldState, newState, nextContext) {
      const instance = workInProgress.stateNode;
      if (typeof instance.shouldComponentUpdate === 'function') {
        // shouldComponentUpdate 更新
        let shouldUpdate = instance.shouldComponentUpdate(
          newProps,
          newState,
          nextContext,
        );
        return shouldUpdate;
      }
    
       // 判断原型链是否存在isPureReactComponent
      if (ctor.prototype && ctor.prototype.isPureReactComponent) {
        return (
          !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState)
        );
      }
    
      return true;
    }
    

在 PureComponent 组件的原型链包含 `isPureReactComponent` 属性，同时也是通过这个属性来判断是否要进行浅比较。

**shallowEqual：**

    
    
    function shallowEqual(objA: mixed, objB: mixed): boolean {
      // 这里的is和useEffect源码中的is一致，不做过多的介绍
      if (is(objA, objB)) {
        return true;
      }
    
      if (
        typeof objA !== 'object' ||
        objA === null ||
        typeof objB !== 'object' ||
        objB === null
      ) {
        return false;
      }
    
      const keysA = Object.keys(objA);
      const keysB = Object.keys(objB);
    
      if (keysA.length !== keysB.length) {
        return false;
      }
    
      for (let i = 0; i < keysA.length; i++) {
        const currentKey = keysA[i];
        if (
          !hasOwnProperty.call(objB, currentKey) ||
          !is(objA[currentKey], objB[currentKey])
        ) {
          return false;
        }
      }
    
      return true;
    }
    

**shallowEqual 浅比较流程：**

  1. 首先比较新旧 `props/state` 是否相等，如果相等，则返回 true，不更新组件；
  2. 接下来判断新旧 `props/state` 是否为对象，如果不是对象或为 `null` 的情况，则返回 false，更新组件；
  3. 然后将新旧 `props/state` 通过 `Object.keys` 转化为数组，如果不相等，则证明有新增或减少，返回 false，更新组件；
  4. 最后进行遍历（浅比较），如果有不相同的话，则返回 false 更新组件。

总的来说，PureComponent 通过自带的 props 和 state 的浅比较实现了 shouldComponentUpdate() ，这点是
Component 所不具备的。

> 注意：PureComponent 可能会因深层的数据不一致而产生错误的否定判断，从而导致 shouldComponentUpdate 结果返回
> false，界面得不到更新，要谨慎使用。

## 2.React.memo 高阶组件

**React.memo** ：结合了 pureComponent 和 componentShouldUpdate 功能，会对传入的 props
进行一次对比，然后根据第二个函数返回值来进一步判断哪些 props 需要更新。

要注意 React.memo 是一个 **高阶组件** ，函数式组件和类组件都可以使用。

**`React.memo`接收两个参数**：

  * 第一个参数：组件本身，也就是要优化的组件；
  * 第二个参数：(pre, next) => boolean， pre：之前的数据，next：现在的数据，返回一个`布尔值`，若为 true 则不更新，为 false 更新。

> 注意：如果 React.memo 的第二个参数不存在时，则按照浅比较的方式进行比较。

举个例子：

    
    
    import { Component, memo } from "react";
    import { Button } from "antd";
    
    const Child = ({ number, msg = "" }: any) => {
      return (
        <>
          {console.log(`${msg}子组件渲染`)}
          <p>
            {msg}数字：{number}
          </p>
        </>
      );
    };
    
    const HOCChild = memo(Child, (pre, next) => {
      if (pre.number === next.number) return true;
      if (next.number < 7) return false;
      return true;
    });
    
    class Index extends Component<any, any> {
      constructor(props: any) {
        super(props);
        this.state = {
          flag: true,
          number: 1,
        };
      }
    
      render() {
        const { flag, number } = this.state;
        return (
          <div>
            大家好，我是小杜杜，一起玩转Hooks吧！
            <Child number={number} />
            <HOCChild number={number} msg="被memo包的" />
            <Button type="primary" onClick={() => this.setState({ flag: !flag })}>
              状态切换{JSON.stringify(flag)}
            </Button>
            <Button
              type="primary"
              style={{ marginLeft: 8 }}
              onClick={() => this.setState({ number: number + 1 })}
            >
              数字加一：{number}
            </Button>
          </div>
        );
      }
    }
    
    export default Index;
    

例子很简单，在 **Index** 中，我们定义了 number 和 flag 两个变量，number 传入对应的 Child，而 flag 与 Child
并没有直接的关联，接下来看看效果：

![img.gif](img\12\3.image)

从图中可以看到，当我们变更无关变量：flag 时，没有被 memo 包裹的子组件 Child 也会进行渲染，而包裹的则不会。同时 memo
的第二个参数可以主动控制是否渲染，当数字大于等于 7 时，则对包裹的组件停止渲染。

> 注意：memo 的第二个参数的返回值与 `shouldComponentUpdate` 的返回值是相反的。这点要注意下。

## 3.优化方案的区别

优化方式| 服务对象| 返回结果  
---|---|---  
PureComponent| 类组件| true：不渲染，false：渲染  
memo| 类组件或函数组件| true：渲染，false：不渲染  
useMemo| 函数组件| -  
  
# useCallback 的性能问题

在所有的 Hooks 中，useCallback 可能是最具争议的一个 hook，根本原因还是性能问题。

首先 useCallback 可以记住函数，避免函数的重复生成，缓存后的函数传递给子组件时，可以避免子组件的重复渲染，从而提升性能。

那是不是说只要是函数，都加入 useCallback，性能都会得到提升呢？

实际不然，性能的提升还有一个前提： **其子组件必须通过 React.memo 包裹，或者必须使用 shouldComponentUpdate 处理** 。

那么如果不进行配套使用，单独使用 useCallback，这种情况下性能不但没有提升，反而还会影响性能。

这是因为当一个函数执行完毕后，就会从调用函数的栈中被弹出，里面的内存也会被回收，即便在函数的内部再创建多个函数，最终也会被释放掉。

而 **函数式组件的性能本身是非常快的** ，它不同于 Class 组件，本身并没有`renderProps` 等额外层级技术，所以相对轻量，而我们使用
useCallack 的时候，这本身就有一定的代价，相当于在原本的基础上增加了 **闭包的使用** 、 **deps 对比的逻辑**
，因此，盲目的使用反而会造成组件的负担。

来看看这个例子：

    
    
    import { useState, useCallback, memo } from "react";
    import { Button } from "antd";
    
    const Index: React.FC<any> = () => {
      let [count, setCount] = useState(0);
      let [number, setNumber] = useState(0);
      let [flag, setFlag] = useState(true);
    
      const add = useCallback(() => {
        setCount(count + 1);
      }, [count]);
    
      return (
        <>
          <div>数字number：{number}</div>
          <div>数字count：{count}</div>
          <TestButton onClick={() => setNumber((v) => v + 1)}>普通点击</TestButton>
          <TestButton onClick={add}>useCallback点击</TestButton>
          <Button
            style={{ marginLeft: 10 }}
            type="primary"
            onClick={() => setFlag((v) => !v)}
          >
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
    

在父组件（ Index ）中共有三个变量：number、count、flag，子组件（TestButton）封装了一个按钮，控制 number 和
count 的变化，其中 count 的变化 add 被 useCallback 包裹，那么看下效果：

**效果：**

![img1.gif](img\12\4.image)

简要地分析下：flag 这个变量与 count 和 number 没有关系，同时也和 TestButton 没有关系，但它的更改，却能让没有被
useCallBack 包裹的组件刷新。

这是因为子组件认为两个函数并非相等，所以会触发更新；相反，用 usecallBack 包裹的组件传递的 onClick 还是之前缓存的
add，没有发生改变，所以不会触发更新。

同理，如果没有 `memo/shouldComponentUpdate` 的协助，就没有浅比较的逻辑，不管有没有 useCallck
的缓存，都会重新执行子组件。

所以说，useCallback 一定要配合 **memo/shouldComponentUpdate** 的协助，才能起到优化作用。

## useCallback 不推荐使用

对于 useCallback，我的建议是： **绝大部分场景不使用** 。原因有以下几点：

  1. **很难看到优化后的效果** ：从效果上来讲，useCallBack 配合 `memo/shouldComponentUpdate` 确实能够阻止子组件的无关渲染，但这个渲染是 **render 的渲染，并非浏览器渲染，** 但 js 的运行要远远快于浏览器的 `Rendering` 和 `Painting`，再加上 React 本身提供 diff 算法，所以很难看到优化后的价值。
  2. **使用起来较为麻烦：** 当判断是否使用时，要先考虑其价值是否值得，如果是案例中的场景，那么使用 useCallback 就完全没有必要。除非是特别复杂的组件，才会考虑单独使用。
  3. **对新手不友好：** 要让 useCallback 起到优化作用，必须配合`memo/shouldComponentUpdate`，也就是说你要了解对应的 API，否则很容易出现 bug，其次 useCallback 本身存在闭包问题，很容易入坑。
  4. **代码可读性变差** ：使用 useCallback 的时候很容易出现“无限套娃”的情况，引用维护依赖关系时要变得小心翼翼，修改时要考虑的要素很多，一点没考虑到，就会出现 bug。

## useCallback 使用场景

  1. 当设计一个 **极其复杂的组件** ，其函数体非常复杂时，优先考虑 **useCallback** 。
  2. **自定义 Hooks 的设计** ，因为在自定义 Hooks 里面的函数，不会依赖于引用它的组件里面的数据，同时如果函数传递给第三方使用，可以规避第三方组件的重复渲染。

# useMemo 适当使用

相对于 useCallback，useMemo 的收益就显而易见了，但 useMemo 也并不是无限制使用，在简单的场景下同样也不建议使用，比如：

    
    
    a = 1
    b = 2
    
    c = a + b;
    d = useMemo(() => a + b, [a, b])
    

很明显 c 是只计算 `a + b`，而 d 还要记录 a 和 b 的值，还要比较是否更改，这种情况下，c 的消耗明显小于 d 的消耗。

综上所述， **useMemo 推荐适当使用** 。

# 小结

在本小节中，我们首先学习了 useMemo 和 useCallback 的源码，两者的源码可以说是一模一样，只是 useMemo 比 useCallback
多了执行了 `nextCreate()` 函数，可以说 **useMemo 是 useCallback 的语法糖** 。

学习完源码后，我们继续深究 React 的其他性能优化的方案，搞懂在类组件中的 shouldComponentUpdate 生命周期和
PureComponent 组件的优化原理，同时弄懂 React.memo 高阶组件，要注意它本身就支持 Class 组件和函数式组件。

最后，我们知道了为什么 useCallback 一定要配合 **memo/shouldComponentUpdate** 的协助，明确了 useMemo 和
useCallBack 的使用场景。

总的来说，useMemo 和 useCallBack 是两个非常重要的 Hooks，我们一定不要乱用它们，合理地使用才会让项目事半功倍。

下一章，我们一起走进 useRef 的世界，探索 Ref 的奥秘。

