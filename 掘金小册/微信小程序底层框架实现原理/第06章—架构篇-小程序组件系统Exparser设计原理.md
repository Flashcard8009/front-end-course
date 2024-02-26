## 前言

无论从温故知新的角度还是体系扩展角度，我认为还是给读者介绍一下WebComponent，因为Exparser与WebComponent还是有着千丝万缕的联系的，了解就要了解的全面，了解的里外通透。

本章节内容分解：
- WebComponent原理
- Custom Element原理
- ShadowDOM思想
- Exparser原理

## 什么是WebComponent?

`WebComponent` 汉语直译过来第一感觉是web组件的意思，但是它只是一套规则、一套API。你可以通过这些API创建自定义的新的组件，并且组件是可以重复使用的，封装好的组件可以在网页和Web应用程序中进行使用。

当前的前端开发环境，Vue、React等都基于组件化开发的形式，但是他们的组件生态并不互通，如果你有过两个平台的开发经验的话，你应该知道最烦恼的就是两个平台的UI组件表现不一致的问题。

我们抽离组件为了提高复用率，提升开发效率。但是脱离了像`Vue、React`这样的框架后，你会发现，原生JS难道就不能开发自定义组件吗？`WebComponent`就是为了解决这个问题。

换一个角度来说，并不是所有的业务场景都需要`Vue\React`这样的框架进行开发、也并是都需要工程化。很多业务场景我们需要原生JS、HTML。

言归正传，`WebComponent`实现的组件可以和HTML原生标签一起使用，有了这个概念之后，我们看一下它的具体表现形式是怎样的。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9efdf9026bc54459bb7bf63133dac0d9~tplv-k3u1fbpfcp-watermark.image)

上面我们看到`<body>`标签还是我们熟悉的标签，但是`<custom-component>`标签就是自定义组件的标签了，它不属于html语义化标签中的任何一个，是自定义的，还是蛮酷的。

接下来我们从这个简单的DEMO入手，对`WebComponent`进行了解。首先就是三大规范：

- Custom Elements规范
- Template规范
- Shadow DOM规范

## Custom Element

所谓自定义元素，即当内置元素无法为问题提供解决方案时，自己动手来创建一个自定义标记来解决，上方的`<custom-component>`就是我们手动创建的自定义`标记`。

这里有个小知识点，为什么要把标签叫做标记呢？前面的文字描述中我一直说的都是标签。还记得HTML的全称吗：`HTML（HyperText Markup Language）超文本标记语言`。对吧，标记语言。讲解知识点的时候要严谨一点，哈哈。

元素的状态是指定义该元素（或者叫做升级该元素）时元素状态的改变，升级过程是异步的。
元素内部的状态有：
- `undefined` 未升级：即自定义元素还未被define。
- `failed` 升级失败：即define过了也实例化开了，但失败了。会自动按HTMLUnknownElement类来实例化。
- `uncustomized` 未定制化：没有define过但却被实例化了，会自动按HTMLUnknownElement类来实例化。
- `custom` 升级成功：define过并且实例化成功了。

我们先从代码结构上来看。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf9bad534b0c4068940193ffd8a2de0c~tplv-k3u1fbpfcp-watermark.image)

首先可以看出，首先有个类的概念。自定义元素类必须继承自window内置的HTML相关类，
这些类位于 `window.<HTML*Element>`，他们都继承自`HTMLElement`类。

然后在`constructor`中定义类一些标记模版，定义模板后，执行`this.appendChild`，其中`this`指向了当前类实例。

最后将自定义组件挂载到`customElements`上，通过`window.customElements.define`方法。这个时候注意了，需要给自定义组件起一个名字，可以看到上面例子中我起的名字为`custom-component`。起名字是有规则的，规则如下：

- 自定义元素的名称，必须包含短横线（-）。它可以确保html解析器能够区分常规元素和自定义元素，还能确保html标记的兼容性。
- 自定义元素只能一次定义一个，一旦定义无法撤回。
- 自定义元素不能单标记封闭。比如`<custom-component />`，必须写一对开闭标记。比如 `<custom-component></custom-component>`。

对于自定义组件挂载的相关API：

- `window.customElement.define('custom-component', CustomComponent, extendsInit)` // 定义一个自定义元素
- `window.customElement.get('custom-component')` // 返回已定义的自定义元素的构造函数
- `window.customElement.whenDefined('custom-component')` // 接收一个promise对象，是当定义自定义元素时返回的，可监听元素状态变化但无法捕捉内部状态值。

其中`window.customElement.whenDefined`方法监听的元素状态为上述讲解的四种元素状态中的：
`failed`升级失败和`custom`升级成功。

讲到这里我们停一下，我们执行一下上述自定义的组件看一下效果。

这里我写了一些style样式为了更美观的体现自定义标签。样式如下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e15059f98c7747269e7d2aa85b8cfe77~tplv-k3u1fbpfcp-watermark.image)

效果图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1869ec709f4747f59c4e2e1041e72436~tplv-k3u1fbpfcp-watermark.image)

可以看到右侧HTML标记中渲染了我们上面自定义的标记`<custom-component>`并且配合style样式，渲染了出来。效果不错是吧。

这里有个问题，我们DEMO里的DOM结构比较简单，所以我们通过`document.createElement`、`appendChild`方法进行构建还不算复杂，如果DOM结构很复杂的组件怎么办呢？一顿使用createElement也不是个办法。这个时候我们就要引入`<template>`标记了。

### Template

`Web Components API` 提供了`<template>`标签，可以在它里面使用HTML定义DOM结构。这样的话我们改版一下我们的自定义组件：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82b461e5dfd8467fadb083dbe7b594ea~tplv-k3u1fbpfcp-watermark.image)

这里有两个需要考虑的地方：

1. 这里因为是DEMO演示所以把`<template>`标签写在了一起，其实可以用脚本把`<template>`注入网页。这样的话，JavaScript 脚本跟`<template>`就能封装成一个 JS 文件，成为独立的组件文件。网页只要加载这个脚本，就能使用`<custom-component>`组件。

2. `<template>`标签内的节点进行操作必须通过`templateElem.content`返回的节点来操作。因为这里获取的`templateElem`并不是一个正常的DOM结构，在控制台打印一下`templateElem.content`得到的结果是`#document-fragment`。它其实是`DocumentFragment`节点，里面才是真正的结构。而且这个模板还要留给其他实例使用，所以不能直接移动它的子元素

这两个地方理解了之后，我们全局审视一下，我们既然是封装组件，那么自定义标记的样式放在外面是不是有些不太合适呢？我自定义了组件样式还要放在外面的话，就失去了一部分组件化开发的概念，而且可能造成样式污染问题。我们平常开发React等组件的时候，样式也是要和组件一起封装起来的。

WebComponent也是可以把样式写在组件中的，我们看一下`WebComponent`中的`style`的语法，修改一下上方DEMO。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f32321ba11104d1bb63de720a3de3ac9~tplv-k3u1fbpfcp-watermark.image)

组件的样式与代码封装在一起，只对自定义元素生效，不会影响外部的全局样式。

那么这个组件样式有了之后，还缺什么呢。还没有开放参数api，我们需要我们的组件可以往里面传递参数。这样我们继续改造我们的DEMO，我们需要使用的时候，自定义组件应该是这个样子的。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5bd33e2741aa40f994dfed375cc66d51~tplv-k3u1fbpfcp-watermark.image)

传入自定义的文本text，如果有text内容那么就展示text，如果没有，那么展示默认值。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12cdf2f775ab48628fb47457f630de72~tplv-k3u1fbpfcp-watermark.image)

首先给需要赋值文本的地方一个类名，方便定位查找。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11c6532222de4ebb814bff4396ae76cd~tplv-k3u1fbpfcp-watermark.image)

然后在自定义标记类中进行查找赋值即可，这里的DOM操作与正常DOM操作方式一致。这样的话页面就展示了我们传入的text值。如果没有的话，会展示默认的文本。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a73906b352c644069d572845c6b07f25~tplv-k3u1fbpfcp-watermark.image)

诶，你看，这样之后就可以传入参数了，但是我们平常使用组件的时候是可以嵌套的，我们不仅仅需要参数注入的形式，还需要嵌套的children形式。继续修改自定义组件。

### slot

WebComponent有一个slot概念，插槽，提供了一个“缺口”让给需要嵌套的DOM。我们先看代码。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bf59727428b4e3f94cf1b62a45b8927~tplv-k3u1fbpfcp-watermark.image)

我们给需要嵌套的结构添加slot属性。在template中使用`<slot>`标记与之呼应。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3440398d9c6a4a89a50efbe2db6fac9f~tplv-k3u1fbpfcp-watermark.image)

这样的就会顺着childNodes中的的slot属性，把与`<template>`中`<slot>`标记name属性相同的标记替换掉。上述我们同时给予了属性值`text`。替换后页面展示结构如下。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1449b42d914947639b1c1ddc6313ed04~tplv-k3u1fbpfcp-watermark.image)

通过slot我们就可以进行嵌套使用自定义组件了，进行子组件或者DOM结构拓展。要想获得嵌套在里面的DOM节点怎么办呢。通过结构上看，slot替换后的DOM结构并没有在Shadow Tree中，而是与ShadowTree同级。我们可以通过document上的DOM操作方法进行操作，也可以通过CustomComponent类中的this进行DOM操作。

到这里回顾一下上述讲解的，有人可能会有疑问。自定义组件`<template>`中，我为了操作DOM，起的class名称和外面其他业务class名称重复了怎么办呢？其实没关系的，shadow DOM中的结构是与外界隔离的，外界是无法获取到内部DOM的，就像前面讲的一样，它可以理解为一颗单独的DOM树，隐藏的DOM树。

### 事件

有了参数之后不能离开事件Event，对吧，我们想添加一个文本的点击事件。继续来改造升级。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08fa558b6edc4037b413a80a520ba4a5~tplv-k3u1fbpfcp-watermark.image)

和我们平常使用的DOM操作是一样的，使用`content`可以获取到`Shadow Tree`上的DOM节点，通过document则可以获取到`HTML Tree`上的DOM节点。

说到事件，就不得不提一下事件冒泡。我们知道自定义组件树是一颗隐藏起来的树，但是内部的事件冒泡是可以一层层冒上去的，可以看到上面DEMO中使用了`event.stopPropagation()`方法阻止了事件冒泡。这里有个知识点，自定义事件 `new Event()`中，options有几个参数可以设置冒泡行为方式，其中就有关于`Shadow DOM`的。我们来看一下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75cbdeb2246a43a5b2f90d2d17be2b0d~tplv-k3u1fbpfcp-watermark.image)

下方讲解Exparser冒泡事件的时候会与这里进行呼应。

这样封装完后是不是觉得非常完美了呢？其实还差一点，细心的同学应该发现了。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dda65ce4df00416e8facacaba7b672e1~tplv-k3u1fbpfcp-watermark.image)

看一下右侧的HTML结构，我们可以展开`<custom-component>`标记看到里面的结构。是不是有种白封装了的感觉。如果只有这样的效果的话，跟模板引擎渲染组件的效果是一样的。所以我们不希望用户能够看到`<custom-component>`的内部代码，WebComponent 允许内部代码隐藏起来，这叫做 Shadow DOM，即这部分 DOM 默认与外部 DOM 隔离，内部任何代码都无法影响外部。

下面我们就来看一下这个`Shadow DOM`到底是什么。

> 诶，重点来了，Exparser的组件模型与WebComponents标准中的ShadowDOM高度相似

## Shadow DOM

Shadow DOM 允许将隐藏的 DOM 树附加到常规的 DOM 树中——它以 shadow root 节点为起始根节点，在这个根节点的下方，可以是任意元素，和普通的 DOM 元素一样。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f00049e551b243759f3a14a73c906463~tplv-k3u1fbpfcp-watermark.image)

把本来DOM树中的一部分封装起来，并且隐藏起来，隐藏起来的树概念为Shadow Tree。把它理解成DOM上一棵特殊的子树，称之为shadow tree或影子树。也是树，但是很特殊，树里面也是DOM，就像我们上面用document.createElement创建的DOM一样。

影子树的根节点，我们称之为`shadow root`或`影子根`。

影子根的父节点，我称之为宿主`shadow host`

我们先看一下隐藏起来的`Shadow Tree`是什么样子的呢，我们把我们的自定义组件进行改版，改为`Shadow DOM`规则。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e16768603e34e9ca1dd034ef00d4b49~tplv-k3u1fbpfcp-watermark.image)

首先实例化一个影子根，挂载到宿主上，这里的宿主是this。上面说过，this指向`CustomComponent`。然后我们把创建的DOM结构，或者`<template>`结构挂载到影子根上即可。看一下HTML结构展示。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7843efa3b4d4834b7feae672792f24c~tplv-k3u1fbpfcp-watermark.image)

这个时候我们可以看到我们自定义元素里面的结构已经变成了Shadow DOM。顺带说下`attachShadow`中的mode参数有两种值“open”、“closed”；

- `open`： 表示可以通过页面内的 JavaScript 方法来获取 Shadow DOM，例如使用 Element.shadowRoot 属性：

```javascript
let myShadowDom = myCustomElem.shadowRoot;
```
- `closed`： 那么就不可以从外部获取`Shadow DOM了`。`myCustomElem.shadowRoot` 将会返回 null

关于宿主（shadow host）还有一点注意，内置的控件元素不能成为宿主，比如：img、button、input、textarea、select、radio、checkbox，video等等，通常我们使用继承自HTMLElement类的自定义元素作为宿主。那么为什么这些标签不可以成为宿主呢。我们考虑一下video标签，我们使用video标签的时候只需要设置一下src即可在页面中渲染出一个完整的播放器，那么video标签是不是一个`Shadow DOM`呢?我们来看一下。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53495f7b1d5d420eb64b2631d7af1a41~tplv-k3u1fbpfcp-watermark.image)

可以看到默认我们是看不到里面的DOM结构的，是因为某些的浏览器默认关闭了查看user agent shadow  DOM。打开控制台，点击右上角的setting图标后，可以看到`Show user agent shadow  DOM`选项

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b1113c90d714d18af31dc85e093ac3d~tplv-k3u1fbpfcp-watermark.image)

打开这个选项之后，我们再来看HTML结构。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11070404c53640c496fffceefb1015d2~tplv-k3u1fbpfcp-watermark.image)

这时候可以看到video标签可以展开了，展开后明晃晃的 `#shadow-root` 展现在了我们面前，对比一下我们的自定义组件`<custom-component>`标签展开，都是`#shadow-root`。有没有一种终于解开了多年的疑惑的感觉。video标签内部的DOM结构展示了出来，就算不去细看里面的结构，我们肯定可以知道一定会有的结构：开始播放按钮、暂停按钮、音量、全屏等等。

这样就都说得通了。浏览器自带了一些Shadow DOM组件，那么想当然的，img、button、input、textarea、select、radio、checkbox，video等等这些标签是不可以作为宿主的，因为他们已经`有主`了。

## Exparser框架原理

`Exparser`是微信小程序的组件组织框架，内置在小程序基础库中，为小程序提供各种各样的组件支撑。内置组件和自定义组件都有Exparser组织管理。

上方我们讲到的Shadow DOM模型，Exparser参照Shadow DOM模型实现，并且进行了一些修改。那我们就来详细说一下有哪些相似或者不相似的地方。

Exparser的组件模型与WebComponents标准中的Shadow DOM高度相似。Exparser会维护整个页面的节点树相关信息，包括节点的属性、事件绑定等，相当于一个简化版的Shadow DOM实现。Exparser的主要特点包括以下几点：

- 基于Shadow DOM模型：模型上与WebComponents的Shadow DOM高度相似，但不依赖浏览器的原生支持，也没有其他依赖库；实现时，还针对性地增加了其他API以支持小程序组件编程。
- 可在纯JS环境中运行：这意味着逻辑层也具有一定的组件树组织能力。
- 高效轻量：性能表现好，在组件实例极多的环境下表现尤其优异，同时代码尺寸也较小。

在理解了`WebComponent`的概念之后，再理解Exparser就会变得简单。就有了一些可以与之对照的一些概念，比如`Shadow DOM`模型，属性、事件绑定、slot等等。并且拥有了与WebComponent一样的优秀表现。

在`Exparser`的组件模型中，组件的节点树称为`Shadow Tree`，即组件内部的实现；最终拼接成的页面节点树被称为`Composed Tree`，即将页面所有组件节点树合成之后的树。这里与我们理解`WebComponent`时候是有一点点区别的。可以想像小程序自带的组件都是自定义组件，而Web中我们通过`WebComponent`写的才是自定义组件。并且在Web中，页面所有组件节点合成的树为`HTML Tree`。这里的概念要进行区分。

### 内置组件

Exparser内置了很多基础组件，比如说视图容器类、表单类、导航类、媒体类、开放类等几十种组件。有了这么丰富的组件，再配合WXSS，我们可以搭建出任何效果的界面。在功能层面上，也满足绝大部分需求。

把一个组件内置到小程序框架里的一个重要原则是：这个组件是基础的。换句话说，没有这个组件的话，在小程序架构里无法实现或者实现不好某类功能。

比如像一些开放类组件，有open-data组件提供展示群名称、用户信息等微信体系下的隐私信息，有`button`组件里`open-type`属性所提供分享、跳转App等敏感操作的能力。

还有比如像视图容器类组件`movable-view`这种因双线程模型导致手势识别不好实现的组件，这是因为手势识别需要高频率捕捉手指的触摸事件，而在双线程模型中，触摸事件从渲染层发出，派发到逻辑层，这中间是有一定的延时而导致视图跟随手指运动这类交互变得有些卡顿。

说到这里可以提及一下小程序js动画效果问题，如果是纯css动画可以在视图层进行处理。如果业务场景为手势识别之类的，监听事件不断的触发，数据不断的改变。这样的业务场景中，我们可以想像，如果坐标值不断改变的话，在逻辑与视图分开的双线程架构中，线程与线程之间的通讯是非常频繁的，会有很大的性能问题。所以我们可以看到微信开放了一个标记`<WXS>`，可以在渲染层写部分js逻辑。这样话就可以在渲染层单独处理频繁改变的数据，这样的话就避免了线程与线程之间频繁通讯导致的性能和延时问题。

### 自定义组件

Exparser支持用户自定义组件。我们可以看一下官方给予的自定义组件的例子，通过例子来看一下与WebComponent有什么区别。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc6595e6a4b54831a87a0089572a80db~tplv-k3u1fbpfcp-watermark.image)

这里如果将`input-with-label`抽象成一个自定义组件，那么可以将整个节点树拆分成两部分。构建Shadow Tree，如下

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/159ba131a8db49d59da9163002992ff6~tplv-k3u1fbpfcp-watermark.image)

看到了熟悉的`<slot>`标记，插槽，一看就是上面我们讲的插槽式写法，那里不一样呢？外面少了template标记包裹。

那么来看一下，生成Shadow Tree之后，`<input-with-label>`这个组件怎么调用了呢？

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbf2bf137f31423a8d3241da452e0b75~tplv-k3u1fbpfcp-watermark.image)

这里就可以理解了，`TEXT`文本会插入到`Shadow Tree`的`<slot>`标记处。这里的原理和逻辑与WebComponent是一至的。

在自定义组件的概念基础上，我们可以把所有组件都进行分离，这样，各个组件也将具有各自独立的逻辑空间。每个组件都分别拥有自己的独立的数据、setData调用。

整个页面节点树实质上被拆分成了若干个ShadowTree（页面的body实质上也是一个组件，因而也是一个ShadowTree）最终组成了小程序中的`Composed Tree`。

在这个时候可以看一下官方的这句话，就非常好理解了。

> 小程序中，所有节点树相关的操作都依赖于Exparser，包括WXML到页面最终节点树的构建、createSelectorQuery调用和自定义组件特性等。

### 组件间通信

不同组件实例间的通信有WXML属性值传递、事件系统、selectComponent和relations等方式。其中，WXML属性值传递是从父组件向子组件的基本通信方式，而事件系统是从子组件向父组件的基本通信方式。

Exparser的事件系统完全模仿Shadow DOM的事件系统。在通常的理解中，事件可以分为冒泡事件和非冒泡事件，但在`Shadow DOM`体系中，冒泡事件还可以划分为在`Shadow Tree上`冒泡的事件和在`Composed Tree`上冒泡的事件。如果在`Shadow Tree`上冒泡，则冒泡只会经过这个组件`Shadow Tree上`的节点，这样可以有效控制事件冒泡经过的范围。

我们还是通过一个例子理解事件冒泡。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/159ba131a8db49d59da9163002992ff6~tplv-k3u1fbpfcp-watermark.image)

slot一个button

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2262f8a88ebc4518bbd81c860b9f4136~tplv-k3u1fbpfcp-watermark.image)

用上面的例子来说，当在` button `上触发一个事件时：

- 如果事件是非冒泡的，那只能在 button 上监听到事件。
- 如果事件是在 Shadow Tree 上冒泡的，那 button 、 input-with-label 、view 可以依次监听到事件。
- 如果事件是在 Composed Tree 上冒泡的，那 button 、 slot 、label 、 input-with-label 、 view 可以依次监听到事件。

在自定义组件中使用triggerEvent触发事件时，可以指定事件的bubbles、composed和capturePhase属性，用于标注事件的冒泡性质。这一点和前面讲的自定义事件相互呼应，triggerEvent可以理解为小程序中的自定义事件createEvent。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8337ac81330416b92258ec86239e605~tplv-k3u1fbpfcp-watermark.image);

小程序基础库自身也会通过这套事件系统提供一些用户事件，如tap、touchstart和form组件的submit等。其中，tap等用户触摸引发的事件是在`Composed Tree`上的冒泡事件，其他事件大多是非冒泡事件。

## 总结

文章前半部分表面上讲的是WebComponent，但由于Exparser的组件模型与WebComponents标准中的Shadow DOM高度相似，其中大家也可以看到像事件系统完全复刻，slot插槽，属性传递等等。可以通过WebComponent设计模式去更好的了解Exparser组件系统，在脑海中有个原理与使用的思维概念。

具体的Exparser运行原理、编译原理等在后面底层基础库章节进行解析。
