# 前言

在之前的小节中，我们已经完全掌握如何开发一个自定义 Hooks，以及相关的单元测试，接下来我们介绍一下在面试中、工作中常用的
Hooks，以此来帮助我们更好地了解 Hooks、完善 Hooks 的体系。

> 为了更好地展示自定义 Hooks，之后就不将 Hooks 效果代码展现出来了，但还是会贴张效果图，展示效果，如果好奇如何使用的代码，可以去 GitHub
> 上查看对应的代码。

![常用Hooks.png](img\8\1.image)

# 1\. useDebounceFn

**useDebounceFn** ：用来处理防抖函数的 Hooks，我们主要通过 [Lodash](https://www.lodashjs.com/)
来处理[防抖](https://www.lodashjs.com/docs/lodash.debounce)。

**确定出入参** ：参考 **Lodash** 中的防抖函数中的 **debounce** 。

    
    
     `_.debounce(func, [wait=0], [options=])`
    

可以确定入参共有 5 个，分别是： **func（防抖函数）** 、 **wait（超时时间/s）** 、
**leading（是否延迟开始前调用的函数）** 、 **trailing（是否在延迟开始后调用函数）** 、 **maxWait（最大等待时间）** 。

其中， **func（防抖函数）** 是最主要的，所以把它单独拆开，其他的放在一起。

出参：触发防抖的函数，官方提供的 cancel（取消延迟）和 flush（立即调用），这里只返回了触发防抖的函数即可。

**优化方案：** 使用 useLatest 处理对应的 func，保持函数最新值，利用 useCreation 优化整个 debounce
即可，另外，需要 useUnmount 在卸载的时候调用 cancel 方法卸载组件。

**代码展示** ：

    
    
    import { useLatest, useUnmount, useCreation } from "..";
    import debounce from "lodash/debounce";
    
    type noop = (...args: any[]) => any;
    
    interface DebounceOptions {
      wait?: number;
      leading?: boolean;
      trailing?: boolean;
      maxWait?: number;
    }
    
    const useDebounceFn = <T extends noop>(fn: T, options?: DebounceOptions) => {
      const fnRef = useLatest(fn);
    
      const debounced = useCreation(
        () =>
          debounce(
            (...args: Parameters<T>): ReturnType<T> => fnRef.current(...args),
            options?.wait ?? 1000,
            options
          ),
        []
      );
    
      useUnmount(() => {
        debounced.cancel();
      });
    
      return debounced;
    };
    
    export default useDebounceFn;
    

> 问：在 debounce 中使用 options?.wait ?? 1000 中的 ”??“ 是什么？

> 答：?? 是 ES11 的新语法： **空值合并运算符** ，只会在左边的值严格等于 null 或 undefined 时起作用，一起来看看与 ||
> 的区别：
    
    
    const a = 0
    const b = a || 7 //b = 7
    const c = a ?? 7 // c = 0
    

> 也就是说 ?? 可以处理值为 0 的情况，在这里我们如果用 || ，没有办法处理 wait 为 0 的情况，但实际上这种情况是存在，所以使用 ??。
>
> 有许多小伙伴可能觉得更新的新特点很多都是鸡肋，但实际上有些特性非常好，我们不管能否运用到，至少要知道，提升思维，是非常有必要的。

**使用方式** ：

    
    
    const run = useDebounceFn(
        fn:(...args: any[]) => any,
        options?: Options
    );
    

**效果：**

![img2.gif](img\8\2.image)

# 2\. useDebounce

**useDebounce** ：用来处理防抖值的 Hooks，既然学了处理函数的防抖，那么处理值的防抖就简单多了，我们只需要利用
useDebounceFn 即可。

**代码演示：**

    
    
    import { useDebounceFn, useSafeState, useCreation } from "..";
    
    import type { DebounceOptions } from "../useDebounceFn";
    
    const useDebounce = <T,>(value: T, options?: DebounceOptions) => {
      const [debounced, setDebounced] = useSafeState(value);
    
      const run = useDebounceFn(() => {
        setDebounced(value);
      }, options);
    
      useCreation(() => {
        run();
      }, [value]);
    
      return debounced;
    };
    
    export default useDebounce;
    
    

**使用方式** ：

    
    
    const debouncedValue = useDebounce(
      value: any,
      options?: Options
    );
    

**效果：**

![img3.gif](img\8\3.image)

# 3\. useThrottleFn

**useThrottleFn** ：用来处理节流函数的 Hooks，同样的，我们使用
[Lodash](https://www.lodashjs.com/) 中的
[节流](https://www.lodashjs.com/docs/lodash.throttle)来处理。

节流与防抖基本一致，只不过缺少 maxWait（最大等待时间）字段，其余的都一样，所以代码就不过多赘述了，直接看看效果即可：

![img4.gif](img\8\4.image)

# 4\. useThrottle

**useThrottle** ：用来处理节流值的 Hooks，跟 useDebounce 同理。

**效果** ：

![img5.gif](img\8\5.image)

# 5.useLockFn

**useLockFn** ：竞态锁，防止异步函数并发执行。

我们在表单中或者各种按钮中，都需要与后端进行交互，这个钩子的作用是防止用户重复点击，重复调取接口（特别是订单的提交），所以这个钩子适用场景非常多，也很重要。

**确定出入参** ：入参应该是执行函数的效果，出参则是何时执行的函数。

既然 useLockFn 是防止异步函数并发执行，那么我们所接受的 fn 必然返回 Promise 形式，同时，接口也会有各种各样的情况，必须通过 try
catch 包一层。

那么我们只需要一个状态来判定是否执行对应的函数即可，由于处理的是函数，直接使用 useCallback 包裹即可。

**代码演示：**

    
    
    import { useRef, useCallback } from "react";
    
    const useLockFn = <P extends any[] = any[], V extends any = any>(
      fn: (...args: P) => Promise<V>
    ) => {
      const lockRef = useRef(false);
    
      return useCallback(
        async (...args: P) => {
          if (lockRef.current) return;
          lockRef.current = true;
          try {
            const ret = await fn(...args);
            lockRef.current = false;
            return ret;
          } catch (e) {
            lockRef.current = false;
            throw e;
          }
        },
        [fn]
      );
    };
    
    export default useLockFn;
    

**使用方式** ：

    
    
    const run = useLockFn<P extends any[] = any[], V extends any = any>(
       fn: (...args: P) => Promise<V>
    ): (...args: P) => Promise<V | undefined>
    

**效果：** 可配合 loading 处理按钮的效果。

![img6.gif](img\8\6.image)

# 6\. useFullscreen

**useFullscreen** ：设置 DOM 元素是否全屏，有的时候，页面信息过多，我们希望去除无关的模块，更好展示所需的模块，就可以使用这个钩子。

**确定出入参** ：这里我们使用 [screenfull](https://github.com/sindresorhus/screenfull)
库进行封装。

参数首先为：target（目标 DOM 元素），其次是进入全屏所触发的方法和退出全屏的方法。

返参提供：当前是否全屏的状态（ isFullscreen ），进入（ enterFullscreen ）/ 退出（ exitFullscreen
）触发的函数，以及是否可全屏的状态（ isEnabled ）即可。

优化手段使用：useLatest 和 useCallback 即可。

**getTarget** ：获取 DOM 目标。在 React 中，除了使用 document.getElementById 等，还可以通过
**useRef** 获取节点信息，所以我们做个兼容：

    
    
    import type BasicTarget from "./BasicTarget";
    type TargetType = HTMLElement | Element | Window | Document;
    
    const getTarget = <T extends TargetType>(target: BasicTarget<T>) => {
      let targetElement: any;
    
      if (!target) {
        targetElement = window;
      } else if ("current" in target) {
        targetElement = target.current;
      } else {
        targetElement = target;
      }
    
      return targetElement;
    };
    export default getTarget;
    

**代码展示：**

    
    
    import screenfull from "screenfull";
    import { useLatest, useSafeState } from "..";
    import { getTarget } from "../utils";
    import type { BasicTarget } from "../utils";
    import { useCallback } from "react";
    
    interface Options {
      onEnter?: () => void;
      onExit?: () => void;
    }
    
    const useFullscreen = (target: BasicTarget, options?: Options) => {
      const { onEnter, onExit } = options || {};
    
      const [isFullscreen, setIsFullscreen] = useSafeState(false);
    
      const onExitRef = useLatest(onExit);
      const onEnterRef = useLatest(onEnter);
    
      const onChange = () => {
        if (screenfull.isEnabled) {
          const ele = getTarget(target);
          if (!screenfull.element) {
            onExitRef.current?.();
            setIsFullscreen(false);
            screenfull.off("change", onChange);
          } else {
            const isFullscreen = screenfull.element === ele;
            if (isFullscreen) {
              onEnterRef.current?.();
            } else {
              onExitRef.current?.();
            }
            setIsFullscreen(isFullscreen);
          }
        }
      };
    
      const enterFullscreen = useCallback(() => {
        const ele = getTarget(target);
        if (!ele) return;
        if (screenfull.isEnabled) {
          screenfull.request(ele);
          screenfull.on("change", onChange);
        }
      }, []);
    
      const exitFullscreen = useCallback(() => {
        const ele = getTarget(target);
        if (screenfull.isEnabled && screenfull.element === ele) {
          screenfull.exit();
        }
      }, []);
    
      return {
        isFullscreen,
        isEnabled: screenfull.isEnabled,
        enterFullscreen,
        exitFullscreen,
      };
    };
    
    export default useFullscreen;
    

> 注：这里要注意一点，有些浏览器在点击全屏后，背景会是黑色，而非白色，这是因为浏览器默认全屏没有背景色，所以是黑色，所以此时需要在整个项目下设置颜色，如：
>
> `*:-webkit-full-screen { background: #fff; }`

**使用方式** ：

    
    
    const { 
        isFullscreen,
        isEnabled,
        enterFullscreen,
        exitFullscreen } = useFullscreen(target, {
           onEnter?: () => void,
           onExit?: () => void
        });
    
    

**效果：**

![企业微信截图_7b9c3a7a-8a06-45b5-b60d-0d32b41db810.png](img\8\7.image)

![image.png](img\8\8.image)

# 7\. useCopy

**useCopy**
：用于复制信息，在平常的开发中，为了用户操作方便，会设置复制按钮，将复制好的数据自动回传到选项的值，或是粘贴板，此时这个钩子就派上了用场。

使用：[copy-to-clipboard 库](https://github.com/sudodoki/copy-to-clipboard)。

**确定出入参** ：很明显，这个钩子并不需要入参，出参是复制后的文字，以及触发复制的方法。

**代码演示：**

    
    
        import writeText from "copy-to-clipboard";
        import { useSafeState } from "..";
        import { useCallback } from "react";
    
        type copyTextProps = string | undefined;
        type CopyFn = (text: string) => void; // Return success
    
        const useCopy = (): [copyTextProps, CopyFn] => {
          const [copyText, setCopyText] = useSafeState<copyTextProps>(undefined);
    
          const copy = useCallback((value?: string | number) => {
            if (!value) return setCopyText("");
            try {
              writeText(value.toString());
              setCopyText(value.toString());
            } catch (err) {
              setCopyText("");
            }
          }, []);
    
          return [copyText, copy];
        };
    
        export default useCopy;
    

**使用方式** ：

    
    
        const [copyText, copy] = useCopy();
    

**效果：**

![img1.gif](img\8\9.image)

# 8\. useTextSelection

**useTextSelection** ：实时获取用户当前选取的文本内容及位置。当我们要实时获取用户所选择的文字、位置等，这个钩子会有很好的效果。

**确定出入参** ：

  * **入参：** 选取文本的范围，可以是指定节点下的文字，当没有指定的节点，应该监听全局的，也就是 document。

  * **出参：** 首先是选取的文字，以及文字距离屏幕的间距，除此之外，还有文字本身的宽度和高度。这里推荐使用 window.getSelection() 方法。

getSelection()：表示用户选择的文本范围或光标的当前位置。

如果有值的话，getSelection() 返回的值进行 toString() 则是选取的值，否则为空。

然后使用 selection.getRangeAt(index) 来获取 Range
对象，主要包含选取文本的开始索引（startOffset）和结束索引（endOffset）。

最后通过 Range 的 getBoundingClientRect() 方法获取对应的宽、高、屏幕的距离等信息。

至于监听事件，我们可以利用 useEventListener 去监听对应的鼠标事件：mousedown（鼠标按下）、mouseup（鼠标松开）去完成。

**代码演示：**

    
    
    import useEventListener from "../useEventListener";
    import useSafeState from "../useSafeState";
    import useLatest from "../useLatest";
    import type { BasicTarget } from "../utils";
    
    interface RectProps {
      top: number;
      left: number;
      bottom: number;
      right: number;
      height: number;
      width: number;
    }
    
    interface StateProps extends RectProps {
      text: string;
    }
    
    const initRect: RectProps = {
      top: NaN,
      left: NaN,
      bottom: NaN,
      right: NaN,
      height: NaN,
      width: NaN,
    };
    
    const initState: StateProps = {
      text: "",
      ...initRect,
    };
    
    const getRectSelection = (selection: Selection | null): RectProps | {} => {
      const range = selection?.getRangeAt(0);
      if (range) {
        const { height, width, top, left, right, bottom } =
          range.getBoundingClientRect();
        return { height, width, top, left, right, bottom };
      }
      return {};
    };
    
    const useTextSelection = (
      target: BasicTarget | Document = document
    ): StateProps => {
      const [state, setState] = useSafeState(initState);
      const lastRef = useLatest(state);
    
      useEventListener(
        "mouseup",
        () => {
          if (!window.getSelection) return;
          const select = window.getSelection();
          const text = select?.toString() || "";
          if (text) setState({ ...state, text, ...getRectSelection(select) });
        },
        target
      );
    
      useEventListener(
        "mousedown",
        () => {
          if (!window.getSelection) return;
          if (lastRef.current.text) setState({ ...initState });
          const select = window.getSelection();
          select?.removeAllRanges();
        },
        target
      );
    
      return state;
    };
    
    export default useTextSelection;
    

**使用方式** ：

    
    
     const state = useTextSelection(target?)
    

**效果：**

![img.gif](img\8\10.image)

> 除此之外，useTextSelection 可配合 Popover 做划词翻译的效果。

# 9\. useResponsive

**useResponsive：** 获取相应式信息，当屏幕尺寸发生改变时，返回的尺寸信息不同，换言之，useResponsive
可以获取浏览器窗口的响应式信息。

**确定出入参** ：

**入参：** 设定屏幕的尺寸范围，这里我们使用栅格布局（bootstrap）的范围，如：

  * xs：0px，最小尺寸；
  * sm：576px，设备：平板；
  * md：768px，设备：桌面显示屏；
  * lg：992px，设备：大桌面显示器；
  * xl：1200px 超大屏幕显示器

**出参：** 尺寸范围是否符合条件，如果符合则为 true，否则为 false。

但这里要注意下，我们默认的入参是栅格的范围，但在真实情况下，入参是允许改变，而出参根据入参的范围而来，所以我们并不知道 useResponsive
具体参数，但可以确定出入参的类型，所以我们需要 Record 的帮助。如：

    
    
    // 入参
    type ResponsiveConfig = Record<string, number>;
    
    // 出参
    type ResponsiveInfo = Record<string, boolean>;
    

解决了 ts 问题后，再来看看另一个问题，对于整个系统而言，所有的布局应该相同，如果把入参放入 useResponsive 中，那么每次调用
useResponsive 都要进行配置，那样会很麻烦，所以我们把入参提取出来，再额外封装个方法，用来设置 responsiveConfig。

最后，我们用 useEventListener 来监听尺寸即可。

**代码演示：**

    
    
    import useSafeState from "../useSafeState";
    import useEventListener from "../useEventListener";
    import isBrowser from "../utils/isBrowser";
    
    type ResponsiveConfig = Record<string, number>;
    type ResponsiveInfo = Record<string, boolean>;
    
    // bootstrap 对应的四种尺寸
    let responsiveConfig: ResponsiveConfig = {
      xs: 0,
      sm: 576,
      md: 768,
      lg: 992,
      xl: 1200,
    };
    
    let info: ResponsiveInfo = {};
    
    export const configResponsive = (config: ResponsiveConfig) => {
      responsiveConfig = config;
    };
    
    const clac = () => {
      const width = window.innerWidth;
      const newInfo = {} as ResponsiveInfo;
      let shouldUpdate = false;
      for (const key of Object.keys(responsiveConfig)) {
        newInfo[key] = width >= responsiveConfig[key];
        // 如果发生改变，则出发更新
        if (newInfo[key] !== info[key]) {
          shouldUpdate = true;
        }
      }
      if (shouldUpdate) {
        info = newInfo;
      }
      return {
        shouldUpdate,
        info,
      };
    };
    
    const useResponsive = () => {
      if (isBrowser) {
        clac();
      }
    
      const [state, setState] = useSafeState<ResponsiveInfo>(() => clac().info);
    
      useEventListener("resize", () => {
        const res = clac();
        if (res.shouldUpdate) setState(res.info);
      });
    
      return state;
    };
    
    export default useResponsive;
    

在这里我们简单做个处理，用 shouldUpdate 来判断是否更新 info，如果 newInfo 和 info
不等，则证明需要更新视图，防止视图不断刷新。

**使用方式** ：

    
    
     // 配置
     configResponsive({
       small: 0,
       middle: 800,
       large: 1200,
     });
    
     // 使用
     const responsive = useResponsive();
    

**效果：**

![img2.gif](img\8\11.image)

# 10\. useTrackedEffect

**useTrackedEffect：** 可监听 useEffect 中的 deps 中的那个发生变化，用法与 useEffect 基本一致。

我们都知道， useEffect 可以监听 deps 的变化，而触发对应的函数，但如果变量值存在多个值时， useEffect 并无法监听是哪个 deps
发生了改变。

如：useEffect 同时监听了 A 和 B，我们想要的效果是 A 改变触发对应的函数，B 改变触发对应的函数，A 和 B
共同触发一个函数，针对这种情况，使用 useEffect 就会变得很麻烦，而 useTrackedEffect 可以完美地解决这个问题，并且还会
**记录上次的值** ，方便我们更好地操作。

**确定出入参** ：useTrackedEffect 的结构应该与 useEffect 的结构保持一致，所以并不存在出参，只需要涉及入参即可。

**入参参数：**

  1. effect：对应 useEffect 的第一个参数，执行函数；
  2. deps：对应 useEffect 的第二个参数，发生改变的函数依赖；
  3. type_list：增加第三个参数，对应 deps 的名称，注意，要和 deps 一一对应，否则结果会有所差异。

确定完入参，那么 useTrackedEffect 中的第一个参数 effect 应该返回哪些信息呢，一起来看看：

  1. changes：改变对应 deps 的索引，通过索引去判断哪个 deps 发生改变；
  2. previousDeps：上一次改变的 deps 值；
  3. currentDeps：改变后的 deps 值；
  4. type_changes：改变对应 deps 的索引，不过对应于中文，而非索引。

除此之外，我们需要记录上一次的值，需要利用 useRef 的特性来帮助我们完成。

**代码演示：**

    
    
    import type { DependencyList } from "react";
    import { useEffect, useRef } from "react";
    
    type Effect = (
      changes?: number[], // 改变的 deps 参数
      previousDeps?: DependencyList, // 上一次的 deps 集合
      currentDeps?: DependencyList, // 本次最新的 deps 集合
      type_changes?: string[] // 返回匹配的字段名
    ) => void | (() => void);
    
    // 判断改变的effect
    const onChangeEffect = (deps1?: DependencyList, deps2?: DependencyList) => {
      if (deps1) {
        return deps1
          .map((_, index) =>
            !Object.is(deps1[index], deps2?.[index]) ? index : -1
          )
          .filter((v) => v !== -1);
      } else if (deps2) {
        return deps2.map((_, index) => index);
      } else return [];
    };
    
    const useTrackedEffect = (
      effect: Effect,
      deps?: DependencyList,
      type_list?: string[]
    ) => {
      const previousDepsRef = useRef<DependencyList>();
    
      useEffect(() => {
        const changes = onChangeEffect(previousDepsRef.current, deps);
        const previousDeps = previousDepsRef.current;
        previousDepsRef.current = deps;
        const type_changes = (type_list || []).filter((_, index) =>
          changes.includes(index)
        );
        return effect(changes, previousDeps, deps, type_changes);
      }, deps);
    };
    
    export default useTrackedEffect;
    

这里有个关键点是：onChangeEffect 函数，用来判断哪一个 deps 发生改变，deps1 为旧的 deps， deps2 为新的
deps，但要注意，deps1 和 deps2 应该一一对应，总共分为三种情况。

  1. dep1 不存在：第一次，改变的应该是 deps2，所以改动点为 deps2 的索引；
  2. dep1 存在：说明存在旧值，然后依次比较 dep1 和 deps2 的值，如果不想等，则更新最新值的索引，想等的话，则返回 -1， 之后再整体过滤一遍不等于 -1 的值，所得到的就是更新的索引；
  3. 特别要注意，useEffect 存在为空数组的情况，说明 dep1 、dep2 都不存在。

**使用方式** ：

    
    
    useTrackedEffect(
      effect: (changes: [], previousDeps: [], currentDeps: [], type_changes: [) => (void | (() => void | undefined)),
      deps?: deps,
      type_list?: string[]
    )
    

**效果：**

![img3.gif](img\8\12.image)

在入参的时候，多加入了一个 type_list，其实这个加不加并没有多大的必要，毕竟我们可以通过索引去判断，没有必要再多设置一个参数去维护。

假设我们不加入 type_list，但我获取的时候仍然想要字段，而不是索引，此时该如何做？

以上述的案例来说，我们可不可以这样更改：

    
    
     useTrackedEffect(() => {           useTrackedEffect(() => {
         ...                      =>     ...
     },[count, count1],                 },[{count}, {count1}], 
    

我们把 deps 变成对象，这样我们就可以拿到对应的名称和值，也不需要再去维护一个字段了。

实际上，这种方式确实可以，但这种方式却打破了 useEffect 常用的规则，因为 useTrackedEffect 本来就是 useEffect
的扩展，使用上应该尽量保持一致，对使用者来说并不友好。

其次，我们改变结构，也就意味着 useTrackedEffect
的内部也要将结构转化，而转化的目的仅仅是为了更好的区分，这种行为非常“多此一举”。所以这种行为的价值不大。

# 思考

在 useTrackedEffect 中，我们谈及到将 deps 的数组结构改成对象结构，那么我们顺便思考一下，useEffect 中的 deps
每一项的类型是否有区别？也就是基本类型和引用类型的效果是否一样？举个例子：

    
    
    const [count, setCount] = useState(0)
    const [count1, setCount1] = useState([0])
    
    // 第一种
    useEffect(() => {...}, [count])
    
    // 第二种
    useEffect(() => {...}, [count1])
    
    // 执行
    setCount(0)
    setCount([0])
    

当我们执行时就会发现第一个 useEffect 不会执行，但第二个 useEffect 还是会执行，其实原理非常简单，useTrackedEffect
的实现也给出了答案，感兴趣小伙伴可以自行思考下（在源码篇中进行详细讲解）。

# 小结

本小节我们共学习了 10 个自定义 Hooks，这些 Hooks 都是在工作中常用的，希望各位小伙伴能够亲自去实现一遍，以此来更好地了解 Hooks。

同时，这也是我们基础篇的最后一篇，下节我们正式学习有关 Hooks 的源码，从根本上去知悉 Hooks，让它成为工作中得力助手！

