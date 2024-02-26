在正式进入 React Hooks 源码的章节前，我们需要了解两个核心概念：Fiber 和并发。
因为学习源码必然绕不开这两大核心概念，了解它们，对我们之后学习源码有着莫大的好处。

# 知悉 Fiber

在一个庞大的项目中，如果有某个节点发生变化，就会给 `diff` 带来巨大的压力，此时想要找到真正变化的部分就会耗费大量的时间，也就是说此时，js
会占据主线程去做对比，导致无法正常的页面渲染，此时就会发生页面卡顿、页面响应变差、动画、手势等应用效果差。

为了解决这一问题，React 团队花费两年时间，重写了 React 的核心算法`reconciliation`，在 React v16 中发布，为了区分
`reconciler（调和器）`，将之前的 `reconciler` 称为 `stack reconciler`，之后称作 **fiber
reconciler（简称：Fiber）** 。

简而言之， **Fiber** 实际上是一种核心算法，为了解决 **中断和树庞大** 的问题，也可以认为 **Fiber** 就是 v16 之后的 **虚拟
DOM** 。

为了之后更好的理解，我们先来看看 `element`、`fiber`、`DOM 元素`三者的关系：

  * element 对象就是我们的 `jsx` 代码，上面保存了 `props`、`key`、`children` 等信息；
  * DOM 元素就是最终呈现给用户展示的效果；
  * 而 fiber 就是充当 element 和 DOM 元素的桥梁，简单来说， **只要 element 发生改变，就会通过 fiber 做一次调和，使对应的 DOM 元素发生改变。**

这里附上一张关于 **Fiber** 的简图：

![知悉Fiber.png](img\9\1.image)

## 虚拟 DOM 如何转化为 Fiber 的？

万物始于 `jsx`，那么我们就从 jsx 入手，从而了解 Fiber。

先看看最常见的一段 jsx 代码：

    
    
    const Index = () => {
      return <div>大家好，我是小杜杜，一起玩转hooks吧！</div>;
    }
    

然后到达绑定的结构：

    
    
    import ReactDOM from 'react-dom/client';
    
    const root = ReactDOM.createRoot(
      document.getElementById('root') as HTMLElement
    );
    root.render(
      <App />
    );
    

> 注：React v18 将原先的 `render` 替换为 `createRoot`，也就是将原先的`legacy` 模式转化为
> `concurrent` 模式。

**ReactDOM.createRoot 结构：**

![image.png](img\9\2.image)

实际上，依旧走的是之前的 render 方法。

> 问：既然都走的 render 方法，那么 React 为什么要替换为 createRoot 呢？

> 答：最大的改变点是模式的转化，原先的 **legacy 模式是同步** 的，而转化后的 **concurrent 模式是异步** 的。可以说在
> React v18 的版本中兼容了 **同步渲染** 和 **异步渲染** 。
>
> 其次是对服务端的改变，在新版本中，替换了原有的 `hydrate API`，做成了配置项，而非原有的 `ReactDOM.hydrate`

## beginWork 方法

当普通的 JSX 代码被 babel 编译成 React.createElement 的形式后，最终会走到`beginWork` 这个方法中。

这个方法可以说是 React 整个流程的开始，要特别注意这个方法。beginWork 中有个 tag，而这个 tag 的类型就是判断 element 对应的
fiber，如：

![image.png](img\9\3.image)

> 由于 **beginWork** 这个方法非常重要，因为将这三个参数提及下，我们要存在一个基础的概念。

**beginWork 的入参** ：

  * **current** ：在视图层渲染的树；
  * **workInProgress** ：这个参数尤为重要，它就是在整个内存中所构建的 `Fiber`；树，所有的更新都发生在 `workInProgress` 中，所以这个树是 **最新** 状态的，之后它将替换给 `current`；
  * **renderLanes** ：跟优先级有关。

**element 与 fiber 的对应关系** （这里总结了一些常见的对照表，提供参考）：

fiber| element  
---|---  
`FunctionComponent` = 0| 函数组件  
`ClassComponent` = 1| 类组件  
IndeterminateComponent = 2| 初始化的时候不知道是函数组件还是类组件  
HostRoot = 3| 根元素，通过reactDom.render()产生的根元素  
HostPortal = 4| ReactDOM.createPortal 产生的 Portal  
HostComponent = 5| dom 元素（如`<div>`）  
HostText = 6| 文本节点  
Fragment = 7| `<React.Fragment>`）  
Mode = 8| `<React.StrictMode>`  
ContextConsumer = 9| `<Context.Consumer>`  
ContextProvider = 10| `<Context.Provider>`  
ForwardRef = 11| React.ForwardRef  
Profiler = 12| `<Profiler>`  
SuspenseComponent = 13| `<Suspense>`  
MemoComponent = 14| `React.memo` 返回的组件  
SimpleMemoComponent = 15| `React.memo` 没有制定比较的方法，所返回的组件  
LazyComponent = 16| `<lazy />`  
  
## Fiber 中保存了什么？

那么 Fiber 究竟保存了什么，一起来看看：

![image.png](img\9\4.image)

> 源码位置：`packages/react-reconciler/src/ReactFiber.js`中的`FiberNode`。

为了更直观地查看 `FiberNode` 的属性，我们直接看对应的 **type** （位置在同目录下的 `ReactInternalTypes`）。

将 **FiberNode** 内容简单化为四个部分，分别是 **Instance** 、 **Fiber** 、 **Effect** 、
**Priority** 。

**Instance** ：这个部分是用来存储一些对应 `element` 元素的属性。

    
    
    export type Fiber = {
      tag: WorkTag,  // 组件的类型，判断函数式组件、类组件等（上述的tag）
      key: null | string, // key
      elementType: any, // 元素的类型
      type: any, // 与fiber关联的功能或类，如<div>,指向对应的类或函数
      stateNode: any, // 真实的DOM节点
      ...
    }
    

**Fiber** ：这部分内容存储的是关于 `Fiber` 链表相关的内容和相关的 `props`、`state`。

    
    
    export type Fiber = {
      ...
      return: Fiber | null, // 指向父节点的fiber
      child: Fiber | null, // 指向第一个子节点的fiber
      sibling: Fiber | null, // 指向下一个兄弟节点的fiber
      index: number, // 索引，是父节点fiber下的子节点fiber中的下表
      
      ref:
        | null
        | (((handle: mixed) => void) & {_stringRef: ?string, ...})
        | RefObject,  // ref的指向，可能为null、函数或对象
        
      pendingProps: any,  // 本次渲染所需的props
      memoizedProps: any,  // 上次渲染所需的props
      updateQueue: mixed,  // 类组件的更新队列（setState），用于状态更新、DOM更新
      memoizedState: any, // 类组件保存上次渲染后的state，函数组件保存的hooks信息
      dependencies: Dependencies | null,  // contexts、events（事件源） 等依赖
    
      mode: TypeOfMode, // 类型为number，用于描述fiber的模式 
      ...
    }
    

**Effect** ：副作用相关的内容。

    
    
    export type Fiber = {
      ...
       flags: Flags, // 用于记录fiber的状态（删除、新增、替换等）
       subtreeFlags: Flags, // 当前子节点的副作用状态
       deletions: Array<Fiber> | null, // 删除的子节点的fiber
       nextEffect: Fiber | null, // 指向下一个副作用的fiber
       firstEffect: Fiber | null, // 指向第一个副作用的fiber
       lastEffect: Fiber | null, // 指向最后一个副作用的fiber
      ...
    }
    

**Priority** ：优先级相关的内容。

    
    
    export type Fiber = {
      ...
      lanes: Lanes, // 优先级，用于调度
      childLanes: Lanes,
      alternate: Fiber | null,
      actualDuration?: number,
      actualStartTime?: number,
      selfBaseDuration?: number,
      treeBaseDuration?: number,
      ...
    }
    

## 链表之间如何连接的？

我们知道了 Fiber 中保存的属性，那么我们要知道标签之间是如何连接的。`Fiber` 中通过 `return`、`child`、`sibling`
这三个参数来进行连接，它们分别指向父级、子级、兄弟，也就是说每个 `element` 通过这三个属性进行连接，同时通过 tag 的值来判断对应的
element 是什么。如：

    
    
    const Index = (props)=> {
    
      return (
        <div>
          大家好，我是小杜杜，一起玩转Hooks吧！
          <div>知悉Fiber</div>
          <p>更好的了解Hooks</p>
        </div>
      );
    }
    

那么按照之前讲的就会转化为：

![image.png](img\9\5.image)

**Fiber** 结构的创建和更新都是 **深度优先遍历** ，遍历顺序为：

  1. 首先会判断当前组件是类组件还是函数式组件，类组件 tag 为 1，函数式为 0；
  2. 然后发现 div 标签，标记 tag 为 5；
  3. 发现 div 下包含三个部分，分别是，文本：`大家好，我是小杜杜，一起玩转hooks吧！`、`div标签`、`p标签`；
  4. 遍历文本：`大家好，我是小杜杜，一起玩转 hooks 吧！`，下面无节点，标记 `tag` 为 6；
  5. 在遍历 div 标签，标记 tag 为 5，此时下面有节点，所以对节点进行遍历，也就是文本 `知悉 fiber`，标记 `tag` 为 6；
  6. 同理最后遍历`p标签`。

整个的流程就是这样，通过 `tag` 标记属于哪种类型，然后通过 `return`、`child`、`sibling` 这三个参数来判断节点的位置。

# React v18 并发机制

在 React v18 中，最重要的一个概念就是 **并发（concurrency）** 。其中 `useTransition`
、`useDeferredValue` 的内部原理都是基于并发的，可见并发的重要性。

## 什么是并发？

**并发：** 在操作系统中，是指一个时间段中有几个程序都处于已启动运行到运行完毕之间，且这几个程序都是在同一个处理机上运行，但
**任一个时刻点上只有一个程序在处理机上运行** 。

通俗来讲， **并发是具备处理多个任务的能力，但不是同时执行，而是交替执行，每次依旧只能执行一个** 。比如：你此时在工作，
三体（电视剧）更新了一集，如果你：

  1. 把工作干完，再去看三体，这种情况说明你既不支持并发也不支持并行；
  2. 把工作先扔到一边，先去看三体，再去工作，这种情况说明你支持并发，但并不支持并行；
  3. 边工作边看三体，两不耽误，这种情况说你支持并行。

看完并发的概念后，再来看看 React 中的并发是什么，以及它的作用有什么。

## React 中的并发

首先，js 是单线程语言，也就是说 js 在同一时间只能干一件事情。但这样就会产生一个问题，如果当前的事情非常耗时，那么后续的事情就会被延后（阻塞）。

比如用户点击按钮后，先执行一个非常耗时的操作（大约 500ms），再进行其他操作，但在这 500ms
中，界面是属于卡死的状态，也就是说用户是无法进行其他操作，这种行为是非常影响用户体验的。而并发就是为了解决这类事件。

在并发的情况下，React
会先点击这个耗时任务，当其他操作发生时（如滚动），先执行滚动的任务，然后再执行耗时任务，这样既能保持耗时任务的进行，又能让用户进行交互。

虽然想法是好的，但实现起来就比较困难了。比如在更新中又触发了其他更新条件，怎么区分哪个是耗时任务？在更新的时候如何中断耗时任务？又该如何去恢复呢？...

来看看 React 官网是如何描述的：

![image.png](img\9\6.image)

官网描述 **并发属于新的一种幕后机制** ，它允许在同一时间内，准备多个版本的 UI，即多个版本的更新。

## 时间分片

首先，我们要知道一个前置知识点：[window.requestIdleCallback](https://developer.mozilla.org/zh-
CN/docs/Web/API/Window/requestIdleCallback)。

它的作用是： **插入一个函数，这个函数将在浏览器空闲时期被调用。**
这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。函数一般会按先进先调用的顺序执行，然而，如果回调函数指定了执行超时时间
`timeout`，则有可能为了在超时前执行函数而打乱执行顺序。

整个页面的内容是一帧一帧绘制出来的，通常来讲，1s 内绘制的帧数越多，就代表变现的画面更加细腻。大多数浏览器绘制一帧在 16.6ms 左右，执行步骤为：

![image.png](img\9\7.image)

  1. 用户的操作：如 click、input 等；
  2. 执行 JS 代码，宏任务和微任务；
  3. 渲染前执行 resize/scroll 等操作，再执行 requestAnimationFrame 回调；
  4. 渲染页面，绘制 html、css 等；
  5. 执行 RIC（requestIdleCallback 回调函数），如果前面的步骤执行完成了，一帧还有剩余时间，就会执行该函数。

而 React 是将任务进行拆解，然后放到 requestIdleCallback 中执行。比如一个 300ms 的更新，拆解为 6 个 50ms
的更新，然后放到 requestIdleCallback 中，如果一帧之内有剩余就会去执行，这样的话更新一直在继续，也可以达到交互的效果。

但 requestIdleCallback 的兼容性非常差，React 团队并不打算使用，而是自己去实现一个类似 requestIdleCallback
的功能，也就是： **时间分片** 。

## 优先级

优先级是 React 中非常重要的模块，分为两种方式：

  1. **紧急更新（Urgent updates）：** 用户的交互，如：点击、输入等，直接影响用户体验的行为都属于紧急情况；
  2. **过渡更新（Transition updates）：** 页面跳转等操作属于非紧急情况。

优先级的模块非常大，这里不做过多的介绍。我们只需要知道，所有的操作都有对应优先级，React 会先执行紧急的更新，其次才会执行非紧急的更新。

## 并发模式的实现

关于并发模式，整体可分为三步，分别是：

  1. 每个更新，都会分配一个优先级（lane），用于区分紧急程度；
  2. 将不紧急的更新拆解成多段，并通过宏任务的方式将其合理分配到浏览器的帧当中，使得紧急任务可以插入进来；
  3. 优先级高的更新会打断优先级低的更新，等优先级高的更新执行完后，再执行优先级低的任务。

## Concurrent 模式是否默认开启？

并发机制是 React v18 中的新功能，那么很多小伙伴会产生这样的疑问，Concurrent 模式需要手动开启吗？还是说所有的代码都转化成
Concurrent 模式了呢？

实际上，在 React v18 中，Concurrent 并不需要手动开启，而是默认开启，换句话说，Concurrent 模式无法关闭，而是一直存在的。

但要注意，并不是所有的代码都执行 Concurent 模式，比如事件更新在 event、setTimeout、网络请求等，React 依旧采用 legacy
（同步阻塞）模式，但如果事件更新与 Suspense、useTransition、useDeferredValue 相关，React 则会采用
Concurent 模式。

总的来说，React 的 Concurrent 模式是否启用取决于 **触发更新的上下文** ，这点要特别注意。

# 小结

本章讲解了 Fiber
与并发机制的基础知识，并不涉及比较难的部分，所以阅读起来相对轻松一点。同时，相对于更高级的模块，应该先把架子搭起来，由浅入深，一点一点地慢慢啃。

下章我们将正式进入 React Hooks 的源码部分。

