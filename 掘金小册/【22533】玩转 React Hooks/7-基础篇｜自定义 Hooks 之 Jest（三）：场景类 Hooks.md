顾名思义，场景类的 Hooks 需要根据具体的场景去优化封装，这类 Hooks 可能更具有特殊性，所以它们更加贴近我们的业务。

下面着重介绍三个 Hooks 帮助我们去更好地了解。

![场景类Hooks.png](img\7\1.image)

# useSelections

**useSelections：** 封装常见的 Checkbox（多选框）逻辑封装。支持多选、全选等操作。

我们先来看看 Checkbox 常用的场景，如下场景：

![image.png](img\7\2.image)

简要分析下场景：

  1. 所有的按钮可单独点击，第一次点击为选中状态，再次点击为未选中状态；
  2. 1 ～ 9，有一个选中，全选按钮为半选状态，全部选中为全选状态；
  3. 点击全选按钮，控制所有 1～9 按钮为全选状态，再次点击，全部为未选中状态。

首先核心点是全选的按钮与 1～9 按钮的关联，1 ～ 9 的状态要在 useSelections 中，所以 useSelections 的第一个参数是
1～9 的数据。

其次，我们分析下 Checkbox 这个组件，选中通过 `checked` 属性、半选通过`indeterminate`、点击通过 `onClick`
属性。

那么对应的 useSelections 的反参为：

  * **selected** ：选中的数据列表；
  * **toggle** ：1 ～ 9 单个切换的方法，如果 selected 中存在对应的数据，则剔除，如果不存在，则增加；
  * **isSelected** ：判断数据是在 selected 中，可以判断 Checkbox 是否选中状态；
  * **allSelected** ：全选的状态；
  * **toggleAll** ：切换全选的方法；
  * **partiallySelected** ：是否在半选状态。

我们可以发现 useSelections 实际上是对 1～9 状态的一个维护，这里通过 new Set 去处理，原因是 Set
处理数据可以方便点，但要注意的是我们拿数据的时候要通过 **Array.from** 来处理。

此外，我们可以将增加、删除、设置的方法单独拿出来，同时可以直接赋给 useSelections
的第二个参数：initValues，用来设置初始默认值。这里就不过多赘述，我们直接就上代码：

    
    
    import { useSafeState, useCreation } from "..";
    const useSelections = <T,>(lists: T[], initValues: T[] = []) => {
      const [selected, setSelected] = useSafeState<T[]>(initValues);
    
      // 通过new Set去处理选中的数据,转化为数组需要使用Array.from
      const selectedSet = useCreation(() => new Set(selected), [selected]);
    
      const isSelected = (data: T) => selectedSet.has(data);
    
      // 增加
      const selectAdd = (data: T | T[]) => {
        if (Array.isArray(data)) {
          data.map((item) => selectedSet.add(item));
        } else {
          selectedSet.add(data);
        }
        return setSelected(Array.from(selectedSet));
      };
    
      // 删除
      const selectDel = (data: T | T[]) => {
        if (Array.isArray(data)) {
          data.map((item) => selectedSet.delete(item));
        } else {
          selectedSet.delete(data);
        }
        return setSelected(Array.from(selectedSet));
      };
    
      // 设置
      const setSelect = (data: T | T[]) => {
        selectedSet.clear();
        if (Array.isArray(data)) {
          data.map((item) => selectedSet.add(item));
        } else {
          selectedSet.add(data);
        }
        return setSelected(Array.from(selectedSet));
      };
    
      // 状态切换
      const toggle = (data: T) =>
        isSelected(data) ? selectDel(data) : selectAdd(data);
    
      // 全部未选中
      const noneSelected = useCreation(
        () => lists.every((ele) => !selectedSet.has(ele)),
        [lists, selectedSet]
      );
    
      // 全部选中
      const allSelected = useCreation(() => {
        return lists.every((ele) => selectedSet.has(ele));
      }, [lists, selectedSet]);
    
      // 是否半选
      const partiallySelected = useCreation(
        () => !noneSelected && !allSelected,
        [noneSelected, allSelected]
      );
    
      // 全选
      const selectAll = () => {
        lists.map((item) => selectedSet.add(item));
        setSelected(Array.from(selectedSet));
      };
    
      const unSelectAll = () => {
        lists.map((item) => selectedSet.delete(item));
        setSelected(Array.from(selectedSet));
      };
    
      const toggleAll = () => (allSelected ? unSelectAll() : selectAll());
    
      return {
        selected, // 以选择的元素组
        isSelected, // 是否被选中
        selectAdd,
        selectDel,
        toggle,
        setSelect,
        noneSelected,
        allSelected,
        partiallySelected,
        selectAll,
        unSelectAll,
        toggleAll,
      } as const;
    };
    
    export default useSelections;
    

**效果：**

![img.gif](img\7\3.image)

**useSelections 的单元测试：**

关于 useSelections 的单元测试实际非常简单，只需要简单地对其方法进行测试就行了，这里就没有必要讲述，感兴趣的小伙伴可以直接看代码，

![image.png](img\7\4.image)

> 实际上，useSelection 的实现并不难，只是它跟之前的 Hooks 略有不同，它是以实际场景为条件所创建的，依赖度相对较高。但这里有一个提醒：
> **Hooks 是基于逻辑的，而非 View 层面** 。

# useCountDown

**useCountDown：** 用于管理倒计时的 Hooks。在日常工作中我们时常需要倒计时的帮助，但处理时间总是比较麻烦的事，而
useCountDown 可以帮助我们解决这类困难。

我们先思考一下实际的场景，假设我们要两天后的倒计时，那么我们要知道两天后距离现在的时间戳，然后通过对应的时间戳转换为对应的 **天、时、分、秒**
，完成倒计时。

可看出，要想计算倒计时，就得具备两个条件：

  * **targetDate** ：目标时间，如：上述示例的两天后；
  * **interval** ：变化的时间，通常为 1s === 1000 ms。

返参只要返回目标时间距离当前时间的时间戳（`remainTime`），和转化后的天、时、分等（`formattedTime`）即可。

useCountDown 的 targetDate 可能存在多种形式，比如字符串、数字、日期等格式，转化起来相对麻烦，所以我们这里直接用 [dayjs
库](https://dayjs.fenxianglu.cn/category/)，来帮助我们解决这个问题。

**目标时间与当前时间的时间差：**

    
    
    const calcRemain = (target?: TDate) => {
      if (!target) return 0;
      const remain = dayjs(target).valueOf() - Date.now();
      return remain < 0 ? 0 : remain;
    };
    

**时间戳进行转化：**

    
    
    const calcFormat = (milliseconds: number): FormattedRes => {
      return {
        days: Math.floor(milliseconds / 86400000),
        hours: Math.floor(milliseconds / 3600000) % 24,
        minutes: Math.floor(milliseconds / 60000) % 60,
        seconds: Math.floor(milliseconds / 1000) % 60,
        milliseconds: Math.floor(milliseconds) % 1000,
      };
    };
    

**时间变化的条件：targetDate、interval。** 每秒都会变化，所以这里我们依靠 **setInterval** 的即可。

    
    
      useEffect(() => {
        if (!targetDate) return setRemainTime(0);
        setRemainTime(calcRemain(targetDate));
    
        const timer = setInterval(() => {
          const remain = calcRemain(targetDate);
          setRemainTime(remain);
          if (remain === 0) {
            clearInterval(timer);
          }
        }, interval);
    
        return () => clearInterval(timer);
      }, [targetDate, interval]);
    

**看看效果：**

![img.gif](img\7\5.image)

在效果图中，我们发现时间戳在无规律地变化，尽管设置 interval 为 1000 ms 也不管用，这是因为每次变化的时间并不一定是 1000ms，而是
1000ms 左右，加之程序本身有一定的延迟 ，所以会有无规律变化的感觉。我可以通过 `Math.round()` （四舍五入）来辅助我们完成转化。

## targetTime 和 onEnd

useCountDown 除了上述功能外，我们可以扩展些额外的功能，让 useCountDown 更加完美，如：

  * **targetTime：** 剩余时间，当前时间 + 剩余时间 = 目标时间；
  * **onEnd：** 当倒计时结束后，触发回调函数。

可以看出 targetTime 相当于 targetDate 是个简化的版本，所以我们只需要做个兼容即可：

    
    
      const target = useCreation(() => {
        if (targetTime) {
          return targetTime > 0 ? Date.now() + targetTime : undefined;
        } else {
          return targetDate;
        }
      }, [targetTime, targetDate]);
    

而 onEnd 触发的时机为 remain === 0 时即可，效果如下：

![img1.gif](img\7\6.image)

## 单元测试：日期测试

在 useCountDown 的单元测试中，涉及到了时间的测试，在这里，我们需要 **jest.useFakeTimers** 的帮助。

**jest.useFakeTimers** ：模拟假计时器，当我们需要 **日期、性能、时间、计时器**
的功能，如：Date、setTimeout()、clearTimeout()、setInterval()、clearInterval()
等，都可以通过它来实现。

但在 Jest 之前的版本中，jest.useFakeTimers 的使用比较麻烦，但在 Jest 26 中，加入 modern
方式来激活定时器的配置（[参考文档](https://jestjs.io/blog/2020/05/05/jest-26#new-fake-
timers)），如：

    
    
       jest.useFakeTimers("modern")
    

但调用 jest.useFakeTimers 会对文件中的所有测试使用假计时器，这种行为是一个 **全局操作** ，会影响同一文件的其他测试。

所以，在使用 jest.useFakeTimers 的时候必须配合使用 **jest.useRealTimers** ，它的作用是
**恢复全局日期、性能、时间和计时器 API 的原始实现。**

比如：

    
    
      beforeAll(() => {
        jest.useFakeTimers("modern");
      });
    
      afterAll(() => {
        jest.useRealTimers();
      });
    

## 定时器测试

当我们设置好环境后，还需要掌握 Jest 中测试定时器的方法： **jest.advanceTimersByTime(msToRun)**
，它可以执行宏任务队列。

换句话说，我们在开发中使用的 setTimeout() 、setInterval()、setImmediate() 都可通过
jest.advanceTimersByTime 进行对应的模拟操作。

测试用例：

    
    
      it("测试 targetTime", () => {
        const { result } = renderHook(
          (
            props: any = {
              targetTime: 3000,
            }
          ) => useCountDown(props)
        );
        expect(result.current[0]).toBe(3000);
        expect(result.current[1].seconds).toBe(3);
    
        act(() => {
          jest.advanceTimersByTime(1000);
        });
        expect(result.current[0]).toBe(2000);
        expect(result.current[1].seconds).toBe(2);
      });
    

简要地说明下：首先我们通过 renderHook 设置 useCountDowen 的剩余时间还剩 3s，然后执行
`jest.advanceTimersByTime(1000)` 模拟定时器执行一秒，所以此时剩余的时间还剩 2s。

> 注意：jest.advanceTimersByTime 模拟是定时器的操作，所以它依然要放入 act 中才会有效果。

## 设置系统时间

在 useCountDown 的设计中，有可能目标的时间小于当前时间，此时返回的应该为
0，但对应的测试中，我们想要自由地去设置当前的时间，这种情况下就可以使用 `jest.setSystemTime` 的帮助。

**jest.setSystemTime(now?: number | Date)：**
模拟程序中运行时的系统时钟，会影响当前时间，但它本身不会触发定时器等。

举个小例子：

    
    
      beforeAll(() => {
        jest.useFakeTimers("modern");
        jest.setSystemTime(new Date("2020-01-01").getTime());
      });
      
        it("测试 targetDate 小于当前时间", () => {
        const { result } = renderHook(() =>
          useCountDown({
            targetDate: new Date("2021-01-01").getTime(),
          })
        );
        expect(result.current[0]).toBe(0);
      });
    

在测试 targetDate 小于当前时间时，我们给的目标时间是 2021-01-01，是小于 2023 年的，但我们通过
jest.setSystemTime 更改后变为了 2020-01-01，此时测试的时间就会大于当前时间，也就是时间戳并不为 0。如：

![image.png](img\7\7.image)

所以我们要想测试这个用例，比 2020-01-01 小就 OK 了，当然，如果不设置 jest.setSystemTime 会默认为当前时间。

# useCss

**useCss** ：用于动态地修改 CSS，是一种具备 **Css-in-JS** （在 JSX/TSX 中书写 CSS）的 Hook。

在实际开放中，我们的 css 通常是与 ts 分开的，并且每个组件的样式并不好复用，如果把 css 也弄成 js，那么在一定程度上也能帮助我们快速开发。

> 问：在 JSX 中通过行内样式不也一样可以修改 CSS 吗？那么 useCss 的优势在哪？

> 答：首先 useCss 最终产出的是一个字符串，通过 `className` 来设定 CSS，而非`style`，优先级仍是 **style >
> useCSS** 。
>
> 抛开区别而言，我们知道行内样式只能控制自身颜色，当我们需要 **媒体查询、伪选择器、控制子节点的 CSS** 等的操作都没有办法实现，而 useCss
> 可以帮助我们完美地实现。
>
> 除此之外，还有以下几点比较重要的原因。
>
>   1. **减少项目编译依赖** ：当我们的样式全部通过 **ts/js** 书写，那么就不需要`css/less/sass`
> 等文件的介入，从而减少项目的依赖、编译，这样我们的项目就变成了纯 js/ts 项目。
>
>   2. **组件化思想** ：useCss 的入参是一个对象，也就是说，我们可以将 css 也当成组件，模块化抽离出，这样无疑带来了莫大的好处。
>
>   3. **代码共享** ：可以非常轻松地在 JS 和 CSS 间共享常量、函数等。
>
>   4. **动态的设置前缀** ：可以有效避免臃肿的 CSS 代码。
>
>   5. **功能实现** ：动态变化的 **主题** 等。
>
>   6. ……
>
>

>
> 总而言之，引入 useCSS 是非常有必要的。

在这里，通过 [nano-css](https://github.com/streamich/nano-css) 去实现 useCss 的功能，通过
useCreation 去优化，`cssToTree` 方法去生产对应的样式即可。

**代码演示：**

    
    
    import { useEffect } from "react";
    import { create, NanoRenderer } from "nano-css";
    import { addon as addonCSSOM, CSSOMAddon } from "nano-css/addon/cssom";
    import { addon as addonVCSSOM, VCSSOMAddon } from "nano-css/addon/vcssom";
    import { cssToTree } from "nano-css/addon/vcssom/cssToTree";
    import { useCreation } from "..";
    
    type NoneType = NanoRenderer & CSSOMAddon & VCSSOMAddon;
    const nano = create() as NoneType;
    addonCSSOM(nano);
    addonVCSSOM(nano);
    
    let counter = 0;
    // CSSProps 在下方介绍
    const useCss = (css: CSSProps): string => {
      const className = useCreation(
        () => "domesy-hooks-css-" + (counter++).toString(36),
        []
      );
      const sheet = useCreation(() => new nano.VSheet(), []);
    
      useEffect(() => {
        const tree = {};
        cssToTree(tree, css, "." + className, "");
        sheet.diff(tree);
    
        return () => {
          sheet.diff({});
        };
      }, []);
    
      return className;
    };
    
    export default useCss;
    

## TS 类型

在 useCss 中，需要特别注意一点，就是 TS 的类型问题。因为它接收的参数是一个 css 对象，但它支持多层嵌套，换句话说，不管 css
这个下面有多少个对象，其类型都应该是 React.CSSProperties。

所以，CSSProps 的类型应该为：

    
    
    type CSSKey = keyof React.CSSProperties;
    
    type CSSProps =
      | React.CSSProperties
      | {
          [key: Exclude<string, CSSKey>]: CSSProps;
        };
    

**keyof 关键字：** 可以获取一个对象接口的所有 `key` 值，用于检查对象上的键是否存在。如：

    
    
    interface Props { 
        name: string;
        age: number;
    }
    
    type PropsKey = keyof Props; //包含 name， age
    
    const res:PropsKey = 'name' // ok
    const res1:PropsKey = 'tel' // error
    

**Exclude <T, U> :** 将 T 类型中的 U 类型剔除。如：

    
    
    type info = "name" | "age" | "sex";
    type info1 = "name" | "age";
    type infoProps = Exclude<info, info1> // "sex"
    

了解完 keyof 和 Exclude 后，回过头来看 CSSProps 的类型就会明了很多，如此一来就能解决 css 的类型问题。

## 使用示例

    
    
    import { useCss } from "../../hooks";
    
    const Index = () => {
      const classDiv = useCss({
        color: "red",
        "&:hover": {
          color: "blue",
        },
      });
    
      const classP = useCss({
        p: {
          color: "green",
          "&:nth-of-type(2)": {
            color: "rebeccapurple",
          },
        },
      });
    
      return (
        <>
          <div className={classDiv}>
            鼠标放上来： 大家好，我是小杜杜，一起玩转Hooks吧！
          </div>
          <div className={classP}>
            <p>CSS-in-JS</p>
            <p>控制div下p标签的字体颜色</p>
            <p style={{ color: "pink" }}>我是行内样式</p>
          </div>
        </>
      );
    };
    
    export default Index;
    

**效果：**

![img.gif](img\7\8.image)

**useCss 的单元测试：**

针对 CSS 的这种 Hooks 该如何测试，是不是需要设置一个 div，设置完后，再去验证 div 的字体颜色呢？

不，测试是代码的逻辑，我们只需要传入 useCss 对应的参数，测试返回的类名即可，无需太过麻烦。

    
    
    import { renderHook } from "@testing-library/react";
    import useCss from ".";
    
    describe("useCss", () => {
      it("should be defined", () => {
        expect(useCss).toBeDefined();
      });
    
      it("测试css", () => {
        const { result } = renderHook(() => useCss({ color: "red" }));
        expect(result.current).toBe("domesy-hooks-css-0");
      });
    });
    

**结果：**

![image.png](img\7\9.image)

# 小结

本小节介绍 useSelections、useCountDown、useCss 三个自定义 Hooks。首先通过单选、全选的实际场景封装出
useSelections，之后了解如何测试时间、定时器、修改系统时间等操作，最后将 CSS 与 JS 结合，以 css-in-js
的方式去书写项目中样式。

通过三节的内容，相信 90% Hooks 的单元测试都能轻松解决，总体来说，单元测试的逻辑与自定义 Hooks
的逻辑不同，单元测试更趋向于“步骤”，需要去模拟各种场景，从而达到预期的效果。

下一节，我们进行常用的 Hooks 开发。

