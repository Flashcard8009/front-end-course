上节中，我们了解了为什么学习单元测试，学习单元测试有哪些好处，并且详细地讲解了 useEventListener，之所以说 useEventListener
是 DOM Hooks 的核心，是因为许多 DOM 操作都会用到这个钩子。

本章将会全面讲解 DOM 类相关的 Hooks，全面了解相关 Hooks 的测试方法。

![DOM类Hooks.png](img\6\1.image)

# useHover

**useHover** ：监听 DOM 元素是否有鼠标悬停。

**分析：** 这个 Hooks 相对简单，既然是监听 DOM 元素，那第一个必然是监听那个 DOM 元素，也就是 **target（目标元素）** 。

那么我们监听到元素要干什么，就是我们的第二个入参，分为： **鼠标进入** 、 **鼠标离开** 、 **状态改变** 这三个函数即可。

    
    
    import useEventListener from "../useEventListener";
    import useSafeState from "../useSafeState";
    import type { BasicTarget } from "../utils";
    
    interface Options {
      onEnter?: () => void;
      onLeave?: () => void;
      onChange?: (isHover: boolean) => void;
    }
    
    const useHover = (target: BasicTarget, options?: Options): boolean => {
      const { onEnter, onLeave, onChange } = options || {};
      const [isHover, setIsHover] = useSafeState<boolean>(false);
    
      useEventListener(
        "mouseenter",
        () => {
          onEnter?.();
          onChange?.(true);
          setIsHover(true);
        },
        target
      );
    
      useEventListener(
        "mouseleave",
        () => {
          onLeave?.();
          onChange?.(false);
          setIsHover(false);
        },
        target
      );
    
      return isHover;
    };
    
    export default useHover;
    

这里，我把 target 的类型单独拿出来，`BasicTarget`为：

    
    
    import type { MutableRefObject } from "react";
    
    type TargetValue<T> = T | undefined | null;
    
    type TargetType = HTMLElement | Element | Window | Document;
    
    type BasicTarget<T extends TargetType = Element> =
      | TargetValue<T>
      | MutableRefObject<TargetValue<T>>;
    

**效果：**

![img.gif](img\6\2.image)

## 单元测试

虽然 useHover 的操作比较简单，但如何进行单元测试呢？如何去模拟鼠标移入、移出事件呢？

在测试 useEventListener 的时候，我们介绍了如何运用`document.createElement` 来创建元素，除了这种方式，再介绍一种：
**@testing-library/react** 进行测试

可能有许多小伙伴喜欢用 `enzyme` 做单元测试，但 enzyme 测试也有很多问题（如：组件触发后，不能改变组件 useState
的值），所以还是使用官方推荐的 `'@testing-library/react'` 测试比较好。

这里我主要介绍下 `'@testing-library/react'` 的 render 和 fireEvent 的方法，掌握这两个，一般的单元测试就 OK
了。

## render

render 主要返回三类，分别是：`getBy...`、`queryBy...`、`findBy...`。

  * `getBy...`：用于定位页面 **已经存在的 DOM 元素** ，如果不存在，则抛出异常。
  * `queryBy...`：用于定位页面 **不存在的 DOM 元素** ，如果不存在，则返回 null，不会抛出异常。
  * `findBy...`：定位页面中的`异常元素`，如果不存在，则抛出异常。

三个的方法基本一致，所以这里就以 **getBy** 为例：

方法| 含义  
---|---  
getByText| 按元素查找文本内容  
getByRole| 按角色去查找  
getByLabelText| 按标签或aria标签文本内容查找  
getByPlaceholderText| 按输入placeholder查找  
getByAltText| 按img的alt属性查找  
getByTitle| 按标题属性或svg标题标记查找  
getByDisplayValue| 按表单元素查找当前值  
getByTestId| 按数据测试查找属性  
  
一般而言，会用到 `getByText` 和 `getByRole` 来获取对应的元素。

## fireEvent

**fireEvent** ：用于实际的操作，也就是模拟点击、键盘、表单等操作。

**使用方式** ：

    
    
    // 两种写法
    fireEvent(node: HTMLElement, event: Event) 
    fireEvent[eventName](node: HTMLElement, eventProperties: Object)
    

**包含的方法：**

    
    
    export type EventType = | 'keyDown' | 'keyPress' | 'keyUp' | 'focus' | 'blur' | 'change' | 'input' ....
    

可以看出 EventType 将所有的方法都包含在内，我们只需要正常使用即可：

    
    
    fireEvent(element, new MouseEvent('click', options?));
    fireEvent.click(element, options?); // 推荐这种写法
    

## 测试用例

经过 render 和 fireEvent 的加持，我们书写 useHover 的测试用例也就简单了很多，首先我们通过 render 创建一个
div，再利用 fireEvent 模拟 **移入和移出** 即可。

    
    
    import { render, fireEvent, renderHook, act } from "@testing-library/react";
    import useHover from ".";
    
    describe("useHover", () => {
      it("should be defined", () => {
        expect(useHover).toBeDefined();
      });
    
      it("测试Hover", () => {
        const { getByText } = render(<div>Hover</div>);
    
        const { result } = renderHook(() => useHover(getByText("Hover")));
        
        // void 的目的是告诉fireEvent方法返回的是undefined
        act(() => void fireEvent.mouseEnter(getByText("Hover")));
        expect(result.current).toBe(true);
        
        act(() => void fireEvent.mouseLeave(getByText("Hover")));
        expect(result.current).toBe(false);
      });
    
      it("测试功能", () => {
        const { getByText } = render(<div>Hover</div>);
        let count = 0;
        let flag = false;
        const { result } = renderHook(() =>
          useHover(getByText("Hover"), {
            onEnter: () => {
              count++;
            },
            onChange: (isFlag) => {
              flag = isFlag;
            },
            onLeave: () => {
              count++;
            },
          })
        );
    
        expect(result.current).toBe(false);
        
        act(() => void fireEvent.mouseEnter(getByText("Hover")));
        expect(result.current).toBe(true);
        expect(count).toBe(1);
        expect(flag).toBe(true);
        
        act(() => void fireEvent.mouseLeave(getByText("Hover")));
        expect(result.current).toBe(false);
        expect(count).toBe(2);
        expect(flag).toBe(false);
      });
    });
    
    

**测试结果：**

![企业微信截图\\_ebbc78ed-5b58-4256-87bc-3cd8e9564696.png](img\6\3.image)

> 问：useEventListener 和 useHover 的测试方法有什么不同？一定要按这种模式进行测试吗？

> 答：测试 useEventListener 的方式是通过 document.createElement 去模拟对应的 div，而 useHover
> 是通过 render 模拟 div，从本质上来说两者的方法一样，只是通过 render 更加直观，要说两者最大的区别就是，render 创建 div
> 的方式是 tsx 文件，document.createElement 创建 div 的方式是 ts文件，这点切记即可。
>
> 其次，测试的方式肯定不止于这两种，但我们创建自己的 Hooks，这两种方式就足够了，如果想要彻底学习 Jest，还需要大量的时间单独了解，这里主要是扩展
> Hooks 的使用。

# useDocumentVisibility

**useDocumentVisibility**
：监听页面是否可见，即当前可见元素的上下文环境。由此可以知道当前文档（即为页面）是在背后，或是不可见的隐藏的标签页。

这个 Hooks 非常的简单，通过 useEventListener 去监听 visibilitychange
即可，但这个钩子却很有趣，问题也比较多，接下来一起看看。

    
    
    import { useSafeState, useEventListener } from "..";
    import { isBrowser } from "../utils";
    
    type VisibilityProps = "hidden" | "visible" | undefined;
    
    const getVisibility = () => {
      if (!isBrowser) {
        return "visible";
      }
      return document.visibilityState;
    };
    
    const useDocumentVisibility = (): VisibilityProps => {
      const [visibility, setVisibility] = useSafeState(() => getVisibility());
    
      useEventListener(
        "visibilitychange",
        () => {
          setVisibility(getVisibility());
        },
        document
      );
    
      return visibility;
    };
    
    export default useDocumentVisibility;
    

**效果：**

![img1.gif](img\6\4.image)

## visibilitychange 问题

判断页面是否可见需要通过 `document.visibilityState`
去判断，但有趣的就在这里，我们分别看下[中文](https://developer.mozilla.org/zh-
CN/docs/Web/API/Document/visibilityState)和[英文](https://developer.mozilla.org/en-
US/docs/Web/API/Document/visibilityState)的版本：

![image.png](img\6\5.image)

可以看到，中文和英文关于 visibilityState 缺少了个 prerender 的状态，那么就好奇了，visibilityState 到底有没有
prerender 这个状态呢？

这时我们主要看看 ts 的类型是什么：

![image.png](img\6\6.image)

很明显，状态并没有 prerender，可以说是一个漏洞。

那么有小伙伴问了，为什么不直接用 VisibilityState，而是要自己去写呢？

看看这张图（我同事上的 **visibilityState** 类型）：

![企业微信截图_b8bfdfa4-102a-4c2d-bb7b-8c1918308524.png](img\6\7.image)

可以发现 visibilityState 存在两种版本的定义，为了比较稳妥的方式，采用内置属性，这样就不会依赖 ts 中的 lib 了。

## 单元测试：jest.mock

我们之前讲了两种创建 div 的方式，那么该如何模拟整个网页的状态呢？如何模拟对应的环境又变成了一个有趣的点。

这时可以通过 Mock 去模拟对应的环境，主要讲解下 `jest.fn()`、`jest.mock()`、`jest.spyOn()` 三个方法。

  * **jest.fn(implementation)：** 这种方式是 Mock 函数最简单的方式，如果没有定义函数内部的实现，则会返回 undefined。
  * **jest.mock(moduleName, factory, options)：** 用来 mock 一些模块或者文件。
  * **jest.spyOn(object, methodName)：** 返回一个 mock function，是 jest.fn 的语法糖，而且能够追踪对应的调用信息。

然后我们需要通过这三个函数来模拟页面即可，通过 mockReturnValue 控制当前页面的状态。

## 测试用例

    
    
    import { renderHook, act } from "@testing-library/react";
    import useDocumentVisibility from ".";
    const mockIsBrowser = jest.fn();
    const mockVisibilityState = jest.spyOn(document, 'visibilityState', 'get');
    
    jest.mock('../utils/isBrowser', () => {
      return {
        __esModule: true,
        get default() {
          return mockIsBrowser();
        },
      };
    });
    
    afterAll(() => {
      jest.clearAllMocks();
    });
    
    describe("useDocumentVisibility", () => {
      it("should be defined", () => {
        expect(useDocumentVisibility).toBeDefined();
      });
    
      it('模拟浏览器更新是否影响', () => {
        mockVisibilityState.mockReturnValue('hidden');
        mockIsBrowser.mockReturnValue(false);
        const { result } = renderHook(() => useDocumentVisibility());
        expect(result.current).toEqual('visible');
      });
    
      it('模拟页面是否隐藏', () => {
        mockVisibilityState.mockReturnValue('hidden');
        mockIsBrowser.mockReturnValue(true);
        const { result } = renderHook(() => useDocumentVisibility());
        expect(result.current).toEqual('hidden');
        mockVisibilityState.mockReturnValue('visible');
        act(() => {
          document.dispatchEvent(new Event('visibilitychange'));
        });
        expect(result.current).toEqual('visible');
      });
    });
    

> 无论是 useEventListener 中模拟 useRef，还是 useDocumentVisibility
> 中的浏览器，都是模拟对应的场景，而不是真正去做 useRef
> 和浏览器，不要局限于代码的写法与作用，而是应该去模拟对应的场景，这点是我认为单元测试与逻辑代码的主要区别。

**结果：**

![企业微信截图\\_80e73e37-5bf5-4718-a050-ba756b7cec80.png](img\6\8.image)

# useInViewport

**useInViewport：** 用于观察元素是否可见区域，以及元素可见的比例，这里需要 **IntersectionObserver**
，它可以帮助我们实现这个 Hook，具体可查看：[Intersection Observer
API](https://developer.mozilla.org/en-
US/docs/Web/API/Intersection_Observer_API)。

我们根据官网来确定 useInViewport 的出入参。

  * 入参：target（监听目标的元素）、可配置项（threshold、rootMargin、root），其中要注意下 threshold，它的范围必须是 **0 ～ 1 的数字** 。
  * 出参：inViewport（元素是否可见）、ratio（出现的比例，但必须配合 threshold 触发）。

**代码演示** ：

    
    
    import { useEffect } from "react";
    import useSafeState from "../useSafeState";
    import { getTarget } from "../utils";
    
    interface Options {
      root?: any;
      rootMargin?: string;
      threshold?: number | number[];
    }
    
    const useInViewport = (target: any, options?: Options) => {
      const [inViewport, setInViewport] = useSafeState<boolean>();
      const [ratio, setRatio] = useSafeState<number>();
    
      useEffect(() => {
        const element = getTarget(target);
    
        const observer = new IntersectionObserver(
          (entries) => {
            for (const entry of entries) {
              setInViewport(entry.isIntersecting);
              setRatio(entry.intersectionRatio);
            }
          },
          {
            ...options,
            root: options?.root ? getTarget(options.root) : null,
          }
        );
    
        observer?.observe(element);
    
        return () => {
          observer?.disconnect();
        };
      }, [target]);
    
      return [inViewport, ratio] as const;
    };
    
    export default useInViewport;
    
    

**效果：**

![img.gif](img\6\9.image)

## 单元测试：模拟 IntersectionObserver

在 useInViewport 中，核心是使用 **IntersectionObserver API** ，但在 Jest 中是不存在的，所以我们要根据
mock 去模拟这个 API。

我们主要使用了 IntersectionObserver 的 observe 和 disconnect，所以我们只需要模拟这两个方法即可，如：

    
    
        mockIntersectionObserver = jest.fn();
        mockIntersectionObserver.mockReturnValue({
          observe: () => null,
          disconnect: () => null,
        });
        window.IntersectionObserver = mockIntersectionObserver;
    

要注意，这里并没有去模拟 observe 和 disconnect 具体的方法，那么究竟如何实现浏览器中的 IntersectionObserver API
的方法呢？

为了解决这一问题，我们可以使用 `mockFn.mock.calls`
方法。它包含对此模拟函数进行的所有调用的调用参数的数组。数组中的每一项都是调用期间传递的参数数组。简单地说，它会将每次执行的方法进行 **截取** 。

回过头来再看看 useInViewport 中使用了 isIntersecting 和 intersectionRatio 两个属性，所以我们通过
**mock.calls** 的配合，直接改变对应的值即可完成模拟，如：

    
    
        const [onChange] = calls[calls.length - 1];
    
        act(() => {
          onChange([
            {
              container,
              isIntersecting: true,
              intersectionRatio: 0.5,
            },
          ]);
        });
    

## 测试用例

    
    
    import { renderHook, act } from "@testing-library/react";
    import useInViewport from ".";
    
    describe("useInViewport", () => {
      it("should be defined", () => {
        expect(useInViewport).toBeDefined();
      });
    
      let container: HTMLDivElement;
      let mockIntersectionObserver: jest.Mock<any, any>;
    
      beforeEach(() => {
        container = document.createElement("div");
        document.body.appendChild(container);
    
        mockIntersectionObserver = jest.fn();
        mockIntersectionObserver.mockReturnValue({
          observe: () => null,
          disconnect: () => null,
        });
        window.IntersectionObserver = mockIntersectionObserver;
      });
    
      afterEach(() => {
        document.body.removeChild(container);
      });
    
      it("测试基础功能", () => {
        const { result } = renderHook(() => useInViewport(container));
        const calls = mockIntersectionObserver.mock.calls;
    
        const [onChange] = calls[calls.length - 1];
    
        act(() => {
          onChange([
            {
              container,
              isIntersecting: true,
              intersectionRatio: 0.5,
            },
          ]);
        });
        const [inViewport, ratio] = result.current;
        expect(inViewport).toBeTruthy();
        expect(ratio).toBe(0.5);
      });
    
      it("测试第二参数节点", () => {
        const { result } = renderHook(() =>
          useInViewport(container, { root: container })
        );
        const calls = mockIntersectionObserver.mock.calls;
    
        const [onChange] = calls[calls.length - 1];
        act(() => {
          onChange([
            {
              container,
              isIntersecting: true,
            },
          ]);
        });
        const [inViewport] = result.current;
        expect(inViewport).toBeTruthy();
      });
    });
    

> useInViewport 在进行单元测试的时候并不用具体模拟对应的 API，而是通过 mock
> 去模拟对应的方法，并且此方法不用管内部实现，只需要模拟值即可，以 **结果** 为导向，完成整个测试。
>
> 另外，在模拟 IntersectionObserver、div 节点时，最好通过 beforeEach
> 去创建，不要直接在函数外部去写，以免造成环境污染。

# useNetWork

**useNetWork：** 用于查看网络状态的
Hook，具体可查看：[NetworkInformation](https://developer.mozilla.org/en-
US/docs/Web/API/NetworkInformation)。

由于每个浏览器的内核不同，在这里就不做兼容，以谷歌浏览器为主，除了官网提供的属性外，多加一个 `online`：是否连接网络的字段。

我们只需要通过 useEventListener 监听 online（有网状态下触发）和 offonline（无网状态下触发）即可。

**代码演示** ：

    
    
    import { useEventListener, useSafeState } from "..";
    
    interface NetWorkProps {
      online?: boolean; // 网络是否为在线
      rtt?: number; // 当前连接下评估的往返时延
      type?:
        | "bluetooth"
        | "cellular"
        | "ethernet"
        | "none"
        | "wifi"
        | "wimax"
        | "other"
        | "unknown"; // 设备使用与所述网络进行通信的连接的类型
      saveData?: boolean; // 用户代理是否设置了减少数据使用的选项
      downlinkMax?: number; // 最大下行速度（单位：兆比特/秒）
      effectiveType?: "slow-2g" | "2g" | "3g" | "4g"; // 网络连接的类型
    }
    
    const getConnection = (): NetWorkProps | undefined => {
      if (navigator && typeof navigator === "object") {
        const nav = navigator as any;
        return {
          rtt: nav.connection?.rtt,
          type: nav.connection?.type,
          saveData: nav.connection?.saveData,
          downlinkMax: nav.connection?.downlinkMax,
          effectiveType: nav.connection?.effectiveType,
        };
      }
    };
    
    const useNetwork = (): NetWorkProps => {
      const [net, setNet] = useSafeState(() => ({
        online: navigator?.onLine,
        ...getConnection(),
      }));
    
      useEventListener("online", () => {
        setNet((v) => ({
          ...v,
          online: true,
        }));
      });
    
      useEventListener("offline", () => {
        setNet((v) => ({
          ...v,
          online: false,
        }));
      });
    
      return net;
    };
    
    export default useNetwork;
    

**效果：**

![image.png](img\6\10.image)

## 单元测试：dispatchEvent

那么我们想想这个钩子该如何进行测试呢？首先来想想主要面临的两个问题：

  1. 不同的浏览器对应的字段不同，难道要模拟不同的浏览器？将所有的字段都进行模拟？
  2. 如何模拟有网和无网的状态？

针对第一个问题，如果模拟对应的浏览器，即模拟内核是很麻烦的，像这种测试就没有必要去模拟，当然也不是所有的字段都要模拟，只要走代码的顺序即可

至于如何去模拟对应的状态，这里就要使用 dispatchEvent 这个方法。

**dispatchEvent：** 自定义触发事件。

**使用方法：**

    
    
    window.dispatchEvent(event) // event：事件
    

dispatchEvent 相当于 fireEvent 的超集， fireEvent 只能模拟对应的事件（如：点击），而 dispatchEvent
不但可以模拟 fireEvent 中的事件，还能够模拟 navigator 中的事件，比如：网络、浏览器尺寸等。

> 问：那为什么不直接用 dispatchEvent，还推荐使用 fireEvent 呢？

> 答：dispatchEvent 确实可以模拟各种事件，功能要比 fireEvent 强大，之所以推荐 fireEvent，主要因为
> dispatchEvent 的使用比较复杂，有些不存在 ts 的提示，并且在日常开发 Hooks
> 的过程中，很少会用到，所以推荐使用：fireEvent。

**dispatchEvent 的使用：**

    
    
    // 模拟点击事件
    const ele = document.getElementById("id");
    ele.dispatchEvent(new MouseEvent("click", { ... }))
    
    //模拟鼠标移动
    document.dispatchEvent(
      new MouseEvent('mousemove', {
        clientX: x,
        clientY: y,
        screenX: x,
        screenY: y,
      })
    )
    

## 测试用例

    
    
    import { renderHook, act } from "@testing-library/react";
    import useNetwork from "./";
    
    describe("useNetwork", () => {
      it("should be defined", () => {
        expect(useNetwork).toBeDefined();
      });
    
      it("切换网络状态", () => {
        const { result } = renderHook(() => useNetwork());
        expect(result.current.online).toBeTruthy();
        act(() => {
          window.dispatchEvent(new Event("offline")); //关闭
        });
        expect(result.current.online).toBeFalsy();
        act(() => {
          window.dispatchEvent(new Event("online"));  //开启
        });
        expect(result.current.online).toBeTruthy();
      }a);
    });
    

**结果：**

![image.png](img\6\11.image)

# 小结

本节介绍了 **useHover** 、 **useDocumentVisibility** 、 **useNetWork** 、
**useInViewport** 四个 Hooks。

通过本章和上章我们了解到各种测试的方法，简单总结下：

  1. 如何利用 renderHook 和 arc 测试自定义 Hooks；
  2. 创建 DOM 进行测试主要有两种方式： **document.createElement** 和 **render** 方式；
  3. 测试点击、鼠标移动推荐使用 **fireEvent** ，如果是测试网络、浏览器等事件可通过 **dispatchEvent** ；
  4. 比较难的测试需要通过 Jest 中的 mock 进行对应的模拟，注意模拟的是所需的环境；
  5. 单元测试的思想与代码实现的思想不太相同，不要拥有固有的逻辑进行测试，如：useEventListener 的 useRef 的测试、useCss 的单元测试。

学习完本节，我们已经完全掌握有关 DOM 类 Hooks 的开发与测试方法，下章进行场景类 Hooks 的讲解。

