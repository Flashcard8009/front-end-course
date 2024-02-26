我们知道 React 之间的通信方式有 props 和 callback、context（跨层级）、event bus 事件总线、ref
传递、状态管理五种方式。其中，状态管理可以 **无视组件之间的层级关系** ，通过 **集中式存储**
管理应用的状态，使数据流更加清晰，以此来解决大型复杂应用中的组件通信问题。如：

![image.png](img\16\1.image)

状态管理的方式有很多种，如：redux、mobx、recoil 等，除了这些三方库之外，我们是否可以使用自定义 Hooks 去帮助我们实现呢？

答案是肯定的。我们可以通过 useCreateStore、useConnect 两个自定义 Hooks，再配合 createContext
就可以实现一个简易版的状态库。

在状态管理的库中，redux 是我们在工作中最常用库，所以我们先来熟悉下 redux，然后再用自定义 Hooks
去模拟对应的功能，以此帮助我们更深层次地理解自定义 Hooks 的实践。

# react-redux 基本使用

提到 redux，就不得不提及到 react-redux 库，它的作用是将 redux 接入到 React 中，实现在 React 中使用 redux
进行状态管理。

整个渲染的流程共有三个部分，分别是：

  * Store：所有的状态存储在一个单一的 store （JavaScript 对象）中，并且对应的状态不允许改变；
  * Action：用于更新状态，当我们要改变 store 的值，就需要通过 dispatch 函数来帮助我们完成更新操作，通常而言 dispatch 中包含一个 type 属性，type 的值决定我们要执行的操作；
  * Reducer：用于更新状态的纯函数，它接收先前的状态和一个 action，然后返回最新的状态。

> 在网上找到了这张图，很形象，可帮助我们迅速了解 redux 的数据流。

![image.png](img\16\2.image)

## 具体使用

首先，在 react-redux 中提供了一个名为 Provider 的组件，它接收一个 store，用于将 store 传递给应用程序的所有组件，如：

    
    
       <Provider store={store}>
          <View /> // 视图组件
       </Provider>
    

其中 store 需要通过 redux 库提供的 createStore 方法来创建，createStore 接收一个参数：reducers，也就是对应的
action，如：

    
    
    const store = createStore(reducers);
    
    // reducers 对应 action，多个 action 可用 combineReducers 处理
    // initialState 为默认值
    export default function action(state = initialState, action: any) {
    
      // 通过 type 去判断
      switch (action.type) {
        case xxx:
        ...
        default:
          return state;
      }
    }
    

之后，我们需要通过 react-redux 库中的 connect 函数去将组件与 redux store 连接起来，去使用即可。

在 connect 函数中接收两个参数，分别是：mapStateToProps 和 mapDispatchToProps。

  * **mapStateToProps：** 用于更新 props，返回 store 中的值，作为 props，传入对应的组件中。
  * **mapDispatchToProps：** 用于更新 action，会返回一个 dispatch，用来触发 action，如果没有第二个参数，则将 dispatch 作为 props 传入对应的组件中。

> 在这里，我们简单写下示例，具体的代码可在 [GitHub](https://github.com/DomeSy/domesy-hooks) 中查看。
    
    
    // 文件位置：example/ReduxView/view
    
    // Father
    const Index = ({ count, msg, onAdd, onSub }: any) => {
      return (
         ...
      );
    };
    
    // 第一个用于传递 props， 第二个参数用于传递 action, 如果 第二个参数不传，会把 dispatch 当作 props 传递过去
    export default connect(
      (state: any) => ({ count: state.count, msg: state.msg }),
      (dispatch: any) => {
        return {
          onAdd: () => dispatch({ type: "add" }),
          onSub: () => dispatch({ type: "sub" }),
        };
      }
    )(Index);
    
    // Clear
    const Index = ({ count, dispatch }: any) => {
      return (
          ...
          <Button
            style={{ marginLeft: 8 }}
            onClick={() => dispatch({ type: "clear" })}
          >
            清除
          </Button>
        ... 
      );
    };
    
    export default connect((state) => state)(Index);
    

## 效果展示

![img.gif](img\16\3.image)

在这个案例中，有父组件：Father，以及它的子组件 Child 和孙组件 Son，此外还有 Father 的兄弟组件 Sibling，和毫无关联的清除组件
Clear。

其中 Child 组件内有一个输入框，用于改变 msg 的值，其他组件用于接收 msg，Clear 组件接收 count 值，同时可以将 store
中的值还原为初始状态和控制 Son 组件的展示状态。

# 设计揣摩：实现跨层级通信

在我们了解完 react-redux 后，简单从使用维度上做下总结： **随时存，随时取。**

这六个字非常简单，其意义是：可以在任意组件中使用 store 中的值，也可以在任意的组件中存储对应的值，无视对应的层级关系，实现状态共享。

想要实现 react-redux 的功能，首先就要解决 **通信问题** ，让状态得到共享，使每个组件都能获得 store 中的状态，并且可以去改变它。

所以，我们可以利用 context（跨层级）来实现跨层级的通信方式，也就是通过 useContext 来获取共有状态，所以我们需要
createContext 的帮助，用它来替代 Provider。

然后需要去实现以下两个自定义 Hooks 来实现 react-redux。

  * **useCreateStore：** 类比 createStore，用于生成一个 Store，并提供对应的实例方法，帮助 useConnect 获取状态属性；
  * **useConnect：** 类比 connect，让每个组件都能获取到 store 中的状态，并且提供 dispatch 方法，以此来订阅 state，如果 state 发生改变，被订阅的组件发生更新。

参照 react-redux 的流程来一起看看实现的思路：

  1. 存储一个公共的 store，用于全局管理 state，当 state 发生变化，通知对应的组件更新；
  2. 收集使用 useConnect 的组件信息，用于后续的更新和销毁；
  3. 维护负责更新的 dispatch，当值发生更新的时候，更新对应的组件；
  4. 当组件销毁时，对应 store 内的数据也应当清除。

明确思路后，我们接下来围绕以上四点去实现 useCreateStore 和 useConnect 即可。

# 实现步骤

## useCreateStore 实现

首先，我利用 createContext 来替代 Provider，如：

    
    
    // createRedux.ts
    import { createContext } from "react";
    const ReduxContext = createContext(null);
    export default ReduxContext;
    
    // index.ts
    const Index = () => {
      const store = useCreateStore(reducers, initialState);
    
      return (
        <ReduxContext.Provider value={store}>
          <View />
        </ReduxContext.Provider>
      );
    };
    

那么，ReduxContext.Provider 所接收的 store 需要 useCreateStore 进行处理即可。我们进行如下设计：

    
    
    // useCreateStore.ts
    const useCreateStore = (reducer: any, initState: any) => {
      let store = useRef<any>(null);
    
      if (!store.current) {
        store.current = new ReduxHooksStore(reducer, initState);
      }
    
      return store.current;
    };
    

useCreateStore 的入参数分为两个：

  * **reducer：** 对应 createStore 的 reducers，也就是 action；
  * **initState：** 初始值，这里将初始化的值拆分出来，方便后续的操作。

跟以往的自定义 Hooks 一样，我们需要通过 useRef 取存储对应的值，用于保存对应的实例帮助我们处理这些事，也就是 ReduxHooksStore。

至于 ReduxHooksStore 具体内部的实现，我们一步一步根据场景去实现。

## useConnect

useConnect 是模拟 connect 方法，可以让任意组件做到随时存，随时取。所以，它涉及两个功能：

  * 初始化：可以拿到 store 中的任意数据，提供给视图；
  * 更新：提供 dispatch 方法，如果 store 中的数据发生改变，则通知对应的视图组件发生更新。

所以，useConnect 返回的参数应当为 `[state, dispatch]`。

## 初始化场景

在整个案例中，共有 3 个初始化变量，分别是 count（数字）、msg（Child 中的消息）和 flag（控制 Son 组件展示的条件）。

在初始化的场景中，我们什么都没处理，所以 useConnect 对应的第一个参数 state 就应该是 useCreteStore 中的
initState，所以在 ReduxHooksStore 中只需要提供一个初始化方法即可，如：

    
    
    class ReduxHooksStore {
      reducer: any;
      state: any;
    
      constructor(reducer: any, initState: any) {
        this.reducer = reducer;
        this.state = initState;
      }
    
      // 初始化方法
      getInitState = () => {
        return this.state;
      };
     }
    

然后，通过 useContext 获取到实例方法，用 useRef 存储即可。

    
    
    import ReduxContext from "./createRedux";
    
    const useConnect = () => {
      // 获取对应的值
      const contextValue: any = useContext(ReduxContext);
      const { getInitState } = contextValue;
    
      const stateValue = useRef(getInitState());
      return [stateValue.current, dispatch];
    };
    

### 定制化入参

通过上述的处理，我们拿到的 state 为全量的数据，要想拿到特定的数据，只需要给 useConnect 一个入参即可，让用户手动获取状态。

    
    
    const useConnect = (mapStoreToState?: (data: any) => void) => {
      ...
      const stateValue = useRef(getInitState(mapStoreToState));
      ...
    };
    
    // useCreateStore.ts
    class ReduxHooksStore{
      ...
      getInitState = (mapStoreToState?: (data: any) => void) => {
        return mapStoreToState ? mapStoreToState(this.state) : this.state;
      };
    }
    

此时，useConnect 就支持以下两种方式：

    
    
    // 全量
    const [state, dispatch] = useConnect();
    
    //  定制化
    const [state, dispatch] = useConnect((data) => ({ count: data.count }));
    

## 更新场景

在更新场景中，我们希望通过 dispatch 触发改变 store 中的值，以及刷新使用 useConnect 的组件。

所以在更新场景中存在两个步骤：

  1. 统计组件：统计使用 useConnect 的组件个数，当 store 发生变化时，更新对应的组件，组件销毁时，移除该组件；
  2. 更新组件：驱动组件更新的一定是 Hooks 所创建的变量，所以与 useReactive 中的更新一样，直接使用 useUpdate 即可。

### 统计组件

统计组件的个数，我们通过一个对象去存储，然后保持每个存储的组件唯一即可，所以我们在 ReduxHooksStore 设置
components_connect，然后比较旧值（oldState）与新值（newState）是否 相等（用 id 区分组件）， 来帮助我们实现功能。

    
    
    class ReduxHooksStore {
      id: number;
      components_connect: any;
    
      // 注册
      subscribe = (connectCurrent: any) => {
        const connectName = `domesy_redux_` + ++this.id;
        this.components_connect[connectName] = connectCurrent;
        return connectName;
      };
    
      // 卸载
      unSubscribe = (connectName: any) => {
        delete this.components_connect[connectName];
      };
    }
    

在 subscribe 中接收一个参数 connectCurrent，connectCurrent
是保存信息，同时我们返回对应的组件名称，方便后续的卸载即可。

当使用 useConnect 的时候触发注册，所以触发的条件为保存的值 connectValue，而 connectValue 的变化取决于
contextValue（ useContext(ReduxContext)），这里我们直接使用 useCreation 即可。

    
    
    const useConnect = () => {
      ...
      const connectValue = useCreation(() => {
        const state = {
          oldState: stateValue.current,
          mapStoreToState,
          /* 更新函数 */
          update: (newState: any) => {
            state.oldState = newState;
            stateValue.current = newState;
          },
        };
        return state;
      }, [contextValue]); // 将 contextValue 作为依赖项。
    
      useEffect(() => {
        const name = subscribe(connectValue);
        return function () {
          // 卸载
          unSubscribe(name);
        };
      }, [connectValue]);
    
      ...
    };
    

关于保存的数据，我们需要一个旧值（oldState），以及更新函数（update），而 mapStoreToState 则是针对定制化入参的兼容处理。

### 更新组件

当我们统计完组件的个数时，我们只需要触发 dispatch 时，去遍历
components_connect，然后比较旧值（oldState）与新值（newState）是否发生改变即可，如果发生改变，则触发对应的 update
方法，刷新视图即可。

    
    
      dispatch = (action: any) => {
        this.state = this.reducer(this.state, action);
    
        /* 批量更新 */
        Object.keys(this.components_connect).forEach((name) => {
          const { update, oldState, mapStoreToState } =
            this.components_connect[name];
          const newState = mapStoreToState
            ? mapStoreToState(this.state)
            : this.state;
    
          // 如果不一致，则触发更新函数
          if (!shallowEqual(oldState, newState)) update(newState);
        });
      };
    

> 其中 shallowEqual 用来比较旧值和新值是否相等的方法。这里偷了个懒，把之前的 shallowEqual（[源码篇｜彻底搞懂 useMemo
> 和
> useCallback](https://juejin.cn/book/7230622711905517605/section/7231328469697691704)
> ） 直接复制过来～

最后，我们在 update 的方法使用 useUpdate 即可。

    
    
    const useConnect = () => {
      ...
      const {  dispatch } = contextValue;
      
      const update = useUpdate();
    
      const connectValue = useCreation(() => {
        const state = {
          ...
          update: (newState: any) => {
            ...
            // 更新
            update();
          },
        };
        return state;
      }, [contextValue]); // 将 contextValue 作为依赖项。
    
      ...
      return [stateValue.current, dispatch];
    };
    
    

## 整体效果

![img2.gif](img\16\4.image)

此时，我们就用两个自定义 Hooks，实现了一个简易版的 react-redux。

# 扩展：批量更新

在更新的步骤中，我们通常会使用 unstable_batchedUpdates 去优化更新的流程，它的作用是优化异步场景。

unstable_batchedUpdates 是 react-dom 提供的方法，它一般用于状态库，并非是日常的开发中使用。

但在这里我们并不需要用 unstable_batchedUpdates 单独处理更新流程，原因是 React v18 中将会自动进行批处理，而 v18
版本以下，则不会进行批处理，需要依靠 unstable_batchedUpdates 去实现。

## unstable_batchedUpdates 使用示例

为了更好的理解，我们模拟一下对应的场景，来看看具体效果：

    
    
    export default class Index extends React.Component {
      state = {
        number: 0,
      };
    
      render() {
        return (
          <>
            <div>number: {this.state.number}</div>
            <Button color="primary" onClick={() => {
              this.setState({
                number: this.state.number + 1,
              });
              this.setState({
                number: this.state.number + 2,
              });
              this.setState({
                number: this.state.number + 3,
              });
    
              setTimeout(() => {
                this.setState({
                  number: this.state.number + 4,
                });
                this.setState({
                  number: this.state.number + 5,
                });
                this.setState({
                  number: this.state.number + 6,
                });
              });
            }}>
              点击
            </Button>
          <>
        );
      }
    }
    

当你看到这种面试题时，一定要注意 React 的环境，React v18 和 v17 完全是两个答案，所以这里需要特别注意～

问：点击一次按钮，number 是多少？

**React v17 答案** ：18。

![img3.gif](img\16\5.image)

原因：3 + 4 + 5 + 6 = 18；

在正常情况下会进行批量处理，所以只会执行最后一个，因此是 3； 而在异步（定时器）下，每个 state 都会进行，因此为：4 + 5 + 6.

**React v18 答案** ：9。

![img4.gif](img\16\6.image)

原因： 3 + 6 = 9

而在 v18 中，异步环境下也会自动合并，也就是执行最后一次，因此第一次执行为 3，第二次为 6。

要想在 v17 中的异步场景中具有批量更新的效果，此时只需要用 unstable_batchedUpdates 包裹一下。

## flushSync

那么可能会有小伙伴好奇，如果我不想在 v18 的异步场景下合并批量操作该怎么办？

为此，官方提供 flushSync 方法，如果处理的函数被它包裹，则不会进行批量更新。

## 函数组件中的影响

上述我们举的例子是类组件，但我们的 Hooks 是运用在函数组件中，那么还有必要加上 unstable_batchedUpdates 吗？

实际上，在异步场景中并不会影响函数组件中的值，但会造成多次渲染的问题，我们直接来看看在 v17 中的效果：

    
    
    const Index = () => {
      const [flag, setFlag] = useState(false);
      
      console.log('触发渲染')
    
      return (
        <>
          <div>flag: {JSON.stringify(flag)}</div>
          <Button
            color='primary'
            onClick={() => {
              setTimeout(() => {
                // unstable_batchedUpdates(() => {
                //   setCount(count + 1);
                //   setCount(count + 1);
                //   setFlag(v => !v)
                // })
    
                setFlag(v => !v)
                setFlag(v => !v)
                setFlag(v => !v)
              });
            }}
          >
            点击
          </Button>
        </>
      );
    };
    
    export default Index;
    

正常效果：

![img5.gif](img\16\7.image)

使用 unstable_batchedUpdates 的效果：

![img6.gif](img\16\8.image)

可见，使用 unstable_batchedUpdates 后每次只会更新一次，如果是未使用的话，则会触发三次。

> 要特别注意：unstable_batchedUpdates 虽然在 v18 中并未移除，但它本身并不算一个正式的
> API，同时在工作中并不需要使用，所以这里只是增加个扩展知识（面试～），在日常工作中，我们并不需要它。

# 小结

本节课，我们学习了 react-redux 的基本使用，并且用两个自定义 Hooks 去实现 react-redux 的基本功能，通过具体场景去实现
useCreateStore 和 useConnect。

下节我们继续学习如用自定义 Hooks 设计表单。

