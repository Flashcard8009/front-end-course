大家好，我是小杜杜。我们知道 React
通过多年的迭代，其精妙的设计思想吸引了全世界网页开发者，是如今应用最广泛、最受欢迎的前端开发库之一。它的优势不胜枚举：

  * **有着精简的 UI** ，可以轻而易举地为每一个状态设计出简洁的视图，并且当数据变动时，可以高效快捷地更新渲染合适的组件。
  * **React 的代码风格是自由的，** 在 React 应用中，遵循“一切皆组件”的原则，每一个组件都相当于一个零件，将这些零件组合起来，就是一个完整的应用。
  * **React 具备超强的兼容性和跨平台性** 。在市面中有多种浏览器，每个浏览器的内核都不相同，而 React 通过独有的事件机制，抹平不同浏览器事件对象之间的差异，将不同平台的事件进行模拟合成事件，使其能够跨浏览器执行。
  * **支持 Node 进行服务端渲染** ，可以利用 React Native 进行原生移动应用开发，其中最突出的莫属于 Taro，支持多平台开发。

2019 年 2 月，React 更是在发布的 16.8 版本中引入了 Hooks。Hooks
属于强化组件的一种方式，它带来的全新机制让人耳目一新，因为它拓展了 React 的开发思路，为 React 开发者提供了一种更方便、更简洁的选择。

在引入 Hooks 的概念后，函数组件既保留了原本的简洁，也具备了状态管理、生命周期管理等能力，在原来 Class 组件所具备的能力基础上，还解决了
Class 组件存在的一些代码冗余、逻辑难以复用等问题。因此，在如今的 React 中，Hooks 已经逐渐取代了 Class 的地位，成了主导。

正所谓明其象意，知其本质，要想更好地玩转 Hooks，我们首先要了解组件的通信方式、强化方式，从而明确 Hooks 的优势所在。

## 组件的通信方式

React 将组件分为两大类，一类是类（ Class ）组件，另一类是函数（ Function ）组件。React
中的类和函数，与普通的类和函数，区别为：类和函数组件承载了渲染 UI 和更新 UI 的功能。

每个组件既然是独立的个体，那么就需要“线”将它串联起来，让彼此知道如何运行，这就涉及到组件之间的相互通信问题。

在 React 中一共有五种通信方式，分别是： **props 和 callback、context（跨层级）、Event
事件、ref传递、状态管理（如：mobx 等）** 方式。

这里我们需要了解第一种和第二种最常用的方式，方便我们后续更好的学习。

### props 和 callback 方式

这种方式是 React 中最常见、也是最基本的通讯方式，通常运用在父传子、子传父。

**1\. 父传子**

父组件传递子组件：所有的参数都通过 props 传递，这里要注意一点，组件包裹的内容都在 children 中，如：

    
    
        import { useState } from "react";
        import { Button } from "antd";
    
        const Index: React.FC<any> = () => {
          const [flag, setFlag] = useState<boolean>(true);
    
          return (
            <>
              <div>我是父组件</div>
              <Button type="primary" onClick={() => setFlag((v) => !v)}>
                切换状态
              </Button>
              <Child flag={flag}>大家好，我是小杜杜，一起玩转Hooks吧！</Child>
            </>
          );
        };
    
        const Child: React.FC<any> = (props) => {
          const { flag, children } = props;
          return (
            <div style={{ border: "1px solid #000", padding: 20 }}>
              <div>我是子组件</div>
              <div>父组件传递的flag：{JSON.stringify(flag)}</div>
              <div>父组件传递的children：{children}</div>
            </div>
          );
        };
    
        export default Index;
    

效果：

![](img\1\1.image)

**2\. 子传父**

子组件传父组件：子传父，也称状态提升，通过父组件传递的 callback 函数，通知父组件。如：

    
    
        import { useState } from "react";
        import { Button } from "antd";
    
        const Index: React.FC<any> = () => {
          const [number, setNumber] = useState<number>(0);
    
          return (
            <>
              <div>我是父组件</div>
              <div>子组件的number：{number}</div>
    
              <Child
                getNumber={(v: number) => {
                  setNumber(v);
                }}
              >
                大家好，我是小杜杜，一起玩转Hooks吧！
              </Child>
            </>
          );
        };
    
        const Child: React.FC<any> = ({ getNumber }) => {
          const [number, setNumber] = useState<number>(0);
    
          return (
            <div style={{ border: "1px solid #000", padding: 20 }}>
              <div>我是子组件</div>
              <Button
                type="primary"
                onClick={() => {
                  const res = number + 1;
                  setNumber(res);
                  getNumber(res);
                }}
              >
                点击加一{number}
              </Button>
            </div>
          );
        };
    
        export default Index;
    

**效果：**

![](img\1\2.image)

### context 方式

这种方式常用于上下文，用于实现祖代组件向后代组件跨层级传值。有些小伙伴可能觉得 context 在工作运用得较少，但实际上， context 是在
React 中一个非常重要的概念，Vue 中的 provide & inject 就来源于 Context。

**Context 的模式**

  1. 创建 Context：React.createContext() 。
  2. Provider：提供者，外层提供数据的组件。
  3. Consumer ：消费者，内层获取数据的组件。

**应用：主题切换**

主题切换是 Context 最经典的应用之一，这里我们利用它来实现一个简单版的主题切换，帮助大家更好地理解 Context。

    
    
        import React, { useState, Component } from "react";
        import { Checkbox, Button, Input } from "antd";
    
        const ThemeContext: any = React.createContext(null); 
    
        //主题颜色
        const theme = {
          dark: {
            color: "#5B8FF9",
            background: "#5B8FF9",
            border: "1px solid #5B8FF9",
            type: "dark",
            buttomType: "primary",
          },
          light: {
            color: "#E86452",
            background: "#E86452",
            border: "1px solid #E86452",
            type: "light",
            buttomType: "default",
          },
        };
    
        const Index: React.FC<any> = () => {
          const [themeContextValue, setThemeContext] = useState(theme.dark);
    
          return (
            <ThemeContext.Provider
              value={{ ...themeContextValue, setTheme: setThemeContext }}
            >
              <Child />
            </ThemeContext.Provider>
          );
        };
    
        class Child extends Component<any, any> {
          static contextType = ThemeContext;
          render() {
            const { border, setTheme, color, background, buttomType }: any =
              this.context;
            return (
              <div style={{ border, color, padding: 20 }}>
                <div>
                  <span> 选择主题： </span>
                  <CheckboxView
                    label="主题1"
                    name="dark"
                    onChange={() => setTheme(theme.dark)}
                  />
                  <CheckboxView
                    label="主题2"
                    name="light"
                    onChange={() => setTheme(theme.light)}
                  />
                </div>
                <div style={{ color, marginTop: 8 }}>
                  大家好，我是小杜杜，一起玩转Hooks吧！
                </div>
                <div style={{ marginTop: 8 }}>
                  <Input
                    placeholder="请输入你的名字"
                    style={{ color, border, marginBottom: 10 }}
                  />
                  <Button type={buttomType}>提交</Button>
                </div>
              </div>
            );
          }
        }
    
        class CheckboxView extends Component<any, any> {
          static contextType = ThemeContext;
    
          render() {
            const { label, name, onChange } = this.props;
            const { color, type }: any = this.context;
    
            return (
              <div
                style={{
                  display: "inline-block",
                  marginLeft: 10,
                }}
              >
                <Checkbox checked={type === name} style={{ color }} onChange={onChange}>
                  {label}
                </Checkbox>
              </div>
            );
          }
        }
    
        export default Index;
    

**效果：**

![](img\1\3.image)

> 问：在 Child 和 CheckboxView 中都有一个静态属性 contextType，后面也没有应用到，这个有什么用？
>
> 答：context 在 React v16 中更新的也较为频繁，static contextType 为新版消费的方式（注意版本在 React
> v16.6），这个静态属性（ contextType ）会指向需要获取的 context（也就是 ThemeContext ），这样就能通过
> this.context 获取 Provider 提供的 contextValue。
>
> 可以看出 staic contextType 只适用于类中，那么函数式中如何消费？React 也提供了一个 Hooks：useContext
> 方式来消费，具体的使用在下节课介绍，这里不做过多赘述。

## 强化组件的四种方式

既然组件在 React 中的地位超然，那么我们就需要掌握如何强化组件，帮助我们更好造轮子。React 提供： mixin 模式、extends
继承模式、高阶组件模式、自定义 Hooks 模式，共四种方式来强化组件。

其中，高阶组件模式和自定义 Hooks 模式是目前最常用的两种模式。

### mixin 模式（已废弃）

mixin 模式也叫混合模式，这种模式是 React 早期的一种强化组件方式，用法与 Vue 中的 mixins 类似，但已被淘汰。

**模型：**

![](img\1\4.image)

要注意的是，如果使用 mixin，必须使用 React.createClass，这种模式类似于滚雪球，会越滚越大，导致复杂度逐渐累加，代码最终难以维护。

所以，在之后的 React 中，取消了 React.createClass，这也就意味 mixin 正式退出了 React
的舞台，所以这里我只是提及一下，并不做过多的讲解。

### extends 继承模式

extends 继承模式就是通过继承，一步一步地将组件强化，React 中的类本身也是继承， 如 React.Component 、
React.PureComponent 都是继承，但这种模式需要对组件进行足够的掌握，否则可能会发生一些奇怪的情况。

模型：

![](img\1\5.image)

    
    
        import React from "react";
        import { Button } from "antd";
    
        class Child extends React.Component<any, any> {
          constructor(props: any) {
            super(props);
            this.state = {
              msg: "大家好，我是小杜杜，一起玩转Hooks吧！",
            };
          }
    
          speak() {
            console.log("Child中的speak");
          }
    
          render() {
            return (
              <>
                <div>{this.state.msg}</div>
                <Button type="primary" onClick={() => this.speak()}>
                  查看控制台
                </Button>
              </>
            );
          }
        }
    
        class Index extends Child {
          speak() {
            console.log("extends 模式，强化后会替代Child的speak方法");
          }
        }
    
        export default Index;
    
    

**效果：**

![](img\1\6.image)

在原本的 Child 中，点击按钮后应该打印出 Child 中的 speak 方法，可经过 extends 继承后，替换成 Index 中的 speak
方法。可见 extends 也是一种强化组件的手段。

### 高阶组件（HOC）模式

高级组件模式也就是 HOC，是现如今最常见的强化组件方式，无论是面试还是日常的开发都有它的影子。

但 HOC 本身并不是 React API 的一部分，而是基于 React 的组合特性而形成的设计模式。

> 问：什么样的组件被称为高阶组件？
>
> 答：如果一个组件接收的参数是一个组件，并且返回也是一个组件，那么这个组件就是高阶组件（HOC）。

**模型：**

![](img\1\7.image)

从模型看出，Hoc 模式与 extends 模式类似，都是逐渐强化组件，使组件越来越强大、健壮，但 extends 继承模式需要考虑的因素很多，HOC
模式适应性更强。举个例子：

    
    
        import React, { useState } from "react";
        import { Button } from "antd";
    
        const HOC = (Component: any) => (props: any) => {
          return (
            <Component
              name={"大家好，我是小杜杜，一起玩转Hooks吧！"}
              {...props}
            ></Component>
          );
        };
    
        const Index: React.FC<any> = (props) => {
          const [flag, setFlag] = useState<boolean>(false);
    
          return (
            <div>
              <Button type="primary" onClick={() => setFlag(true)}>
                获取props
              </Button>
              {flag && <div>{JSON.stringify(props)}</div>}
            </div>
          );
        };
    
        export default HOC(Index);
    

**效果：**

![](img\1\8.image)

> 问：在上面的例子中， HOC = (Component: any) => (props: any) => {} 这种写法是什么意思？
>
> 答：这种写法实际是一种简写方式，如果不好理解，可以看看这种方式转化为 ES5 是什么样子：
    
    
        var HOC = function (Component) { 
          return function (props) {
              return React.createElement(Component, __assign({ name: "大家好，我是小杜杜，一起玩转Hooks吧！" }, props));
          }; 
        };
    

**其他：** 实际上，HOC 可以做很多事情，比如强化
props、条件渲染、性能优化、事件赋能、反向继承等，这块内容本身也比较大，感兴趣的话。可以看看以下两篇的内容，帮助大家更好掌握 HOC：

  * [作为一名 React，我是这样理解 HOC 的](https://juejin.cn/post/7103345085089054727)；
  * [花三个小时，完全掌握分片渲染和虚拟列表～](https://juejin.cn/post/7121551701731409934)。

### 自定义 Hooks 模式

Hooks 是 React v16.8 以后新增的
API，目的是增加代码的可复用性、逻辑性，最主要的目的是解决了函数式组件无状态的问题，这样既保留了函数式的简单，又解决了没有数据管理状态的缺陷。

**模型：**

![](img\1\9.image)

从模型中我们看到，自定义 Hooks 实际上是在辅助组件，让其开发更加丝滑、简洁、维护性更高。

Hooks 可以说是现如今最常用、最主流的强化组件方式，学好 Hooks 是非常重要的，接下来我们一起来看看 Hooks 在 React 中的地位。

## 大势所趋之 Hooks

在如今的环境中，已经有越来越多的人认可了 Hooks 能力，并且加入到 Hooks 的大家庭中：

  * 2019 年 2 月，React 正式发布 v16.8 版本，使其具备 Hooks 能力（最新的 v18 中，多了 5 个 Hooks API）；
  * 2019 年 6 月，尤雨溪提出了关于 Vue3 Composition API 的提案（ Vue3 中也可以使用 Hooks）；
  * Ant Design Pro V5 等框架是以 Hooks 为主体的前端框架；
  * `Solid.js`、 `Preact` 等框架，也开始选择加入 `Hooks`；
  * 很多优秀的开源项目（如 Ant Design ），都从原先的 Class 升级到 Hooks；
  * ......

虽然目前的环境中，Class Component 与 Hooks API 并驾齐驱，但从整体趋势来看，Hooks 将会彻底取代 Class
Component，成为真正的主流。那么为什么 Hooks 会被越来越多的人推崇？凭什么取代 Class 呢？

在这里，我认为 Hooks 的兴起是大势所趋，是必然的结果，主要原因有以下几点。

### 1\. Class 组件的缺陷

在 React v16.8 之前，我们都使用 Class 组件，很少去用函数组件，根本原因是函数组件虽然简洁，但没有数据状态管理，这个致命的缺陷使
Class 组件成为了主流。

但当 React v16.8 的出现，带来了全新的 Hooks API，它彻底解决了函数式的这个缺陷。这里我们简要列举出 Class 组件的 3
点主要缺陷，看看为何会出现 Hooks。

  * **缺陷一：super 的传递**

在讲解 extends 继承模式的时候有这样一段代码：

![](img\1\10.image)

红框中的代码可以说用到 Class 就会看到，那么为什么要在 constructor 中调用 super 呢，如果直接调用(如 `super()`
)的结果又是什么？

实际上 super 的作用等于执行 Component 函数，如果不使用 super() 就会导致 Component 函数内的 props
找不到，也就是在代码中使用 `this.props` 打印出 `undefined`，所以这段代码是必要的。

  * **缺陷二：奇怪的 this 指向**

在 Class 的写法中，随处可见的就是 `.bind(this)`，为什么所有的方法都要进行绑定呢？不绑定的话，为什么找不到对应的 this
呢？有些小伙伴可能会问，看你使用的是箭头函数，并没有绑定 this 啊，但要记住箭头函数是 ES6 的产物，如果没有箭头函数，还是需要进行` bind`
绑定。

之所以 Class 组件需要 bind 的根本原因是在 React 的事件机制中，dispatchEvent 调用的
`invokeGuardedCallback` 是直接使用的 `func`，并没有指定调用的组件，所以在 Class 组件中的方法都需要手动绑定 this。

而箭头函数本身并不会创建自己的 this，它会继承上层的 this，所以不需要进行绑定，this 本身就是指向的组件。

  * **缺陷三：繁琐的生命周期**

在 Class 组件中有很多关于生命周期的 API，以此用来数据管理，主要的版本分为 v16.0 和
v16.4，如：`componentDidMount`、`getDerivedStateFromProps(prevProps, prevState)`
等，大概有 9 个 API，我们要完全掌握生命周期的用法，显然并不是一件容易的事。

在了解到 Class 组件的缺陷后，我们反观函数式组件是否存在这些问题。

从代码中看到函数式组件并没有 super，更不用关心 this 的指向，相对于类中繁琐的生命周期，都可以使用 `useState`、`useEffect`等
Hooks 代替，极大地降低了代码数量、行数，使其更容易上手，代码更加简洁、规整、维护性更高。

### 2\. 更好的状态复用

从强化组件的模型中，我们可以看出自定义 Hooks 的模式与 mixin 模式更相近。

但为什么 mixin 会被废弃，其根本原因是 mixin 的弊端太多了，并且 React 官方也明确表示不建议我们去使用 mixins：

![](img\1\11.image)

而 Hooks 可以完美避开 mixin 的弊端，并且简单，高度聚合，阅读方便，给开发人员带来效率提升，Bug 数减少，这样的 Hooks，有谁不爱呢？

### 3\. 友好的替代

当 React 官方提供了 Hooks API 后，并没有强制开发人员去使用它，而是将优势与劣势摆出来，是否使用的最终决定权交给大众选择。

这样在项目中，即可以使用熟悉的 Class，又能尝试新颖的 Hooks。当项目的逐渐迭代，开发人员在开发过程中逐渐了解 Hooks
的优势。这种悄无声息的改变，使越来越多的人熟悉 Hooks、加入 Hooks。

> 综上 3 点，可见掌握 Hooks 已经是一门必修课，而且 Hooks 是与我们开发中息息相关的技能，所以掌握它，是非常有必要的。

## 小结

本节课我们了解 React 的组件的通信方式、强化方式，并了解到 Hooks 为什么能够取代 Class 成为大势所趋。

接下来，我们将正式走进 Hooks 的大门，从官方给出的 Hooks API 进军，全面了解 Hooks API 的使用方法和场景。

