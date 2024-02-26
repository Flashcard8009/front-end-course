## 前言

这一章节中我们将对渲染层的底层基础库WAWebview.js进行一个分析，看看里面都有哪些模块。

本章内容分解：

- 渲染层基础库源码解析
- Exparser系统源码解析。

## WAWebview.js注入

我们重温一下在哪里看到过渲染层基础库的注入。在最开始章节找寻小程序渲染层代码中我们找寻到了渲染层基础库的注入：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55b9ccafa73c4edfa3fddd4087b6f73e~tplv-k3u1fbpfcp-watermark.image)

## 模块分析

首先我们进行`WAWebview.js`文件的解析。在我们`2.17.3`版本的基础库包中，渲染层基础库代码大概有8万多行。那么我们怎么进行分析呢？

我这边使用了比较传统的方法，就是折叠法，折叠成代码块，然后进行分析。这里折叠使用了Subline编辑器，因为我的VsCode插件较多，每次修改或者变动文件会有点卡，使用没有插件的Subline比较轻量，如果你也是跟着这本小册同步的去做，建议你也下载一个Subline去折叠代码。

我们从上往下进行分析，第一个模块是`Foundation`模块。

### core-js模块

最开始有一段4000多行的模块，为`core-js`模块。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ccaea45cb264781aeefe233b6b0e531~tplv-k3u1fbpfcp-watermark.image)

core-js负责初始化框架js代码，编译js，加载业务逻辑js等功能。

### Foundation

Foundation是基础模块。我们来看一下声明的核心位置，在文件的4275行左右。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2b564f729aa40c7bea254a091a9a8c9~tplv-k3u1fbpfcp-watermark.image)

在源码的基础上我添加了一些备注，方便阅读理解。本身是没有备注的。

我们可以从其中看到里面包含了一些api，有`EventEmitter`事件的发布订阅，配置的`Ready`事件，基础库`Ready`事件，`Bridge Ready`事件，`env`、`global`环境变量。

其中`EventEmitter` 的核心就是事件触发与事件监听器功能的封装。小程序应该是自己在内部实现了一套。如果你想自己尝试一下`Node.js`环境下可以使用`require('events')`包，js环境下也有类似的实现[event-emitter](https://www.npmjs.com/package/event-emitter)，值得了解一下。

### WeixinJSBridge

WeixinJSBridge通讯模块，在4528行左右的。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f112217c9da4e0fac492460d9354c29~tplv-k3u1fbpfcp-watermark.image)

可以看到`WeixinJSBridge`通讯模块中包含了我们在通讯系统章节讲解的通讯api，包含有`on`、`publish`、`invoke`、`subscribe`、`invokeCallbackHandler`、`subscribeHandler`。 

通讯系统的具体实现在通讯系统章节有讲解，这里只是对Native注入通讯api的封装，便于内部调用。

### 异常监听模块

基础库内针对`promise`或者`js`等异常事件也是做了监听处理。在文件4872行左右

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70595fc2b26c4e1daa9cea7e01d13ea2~tplv-k3u1fbpfcp-watermark.image)

### 日志打印模块

日志打印模块在文件5027行左右。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cabece8b84274219b26966e2ce3ee6b2~tplv-k3u1fbpfcp-watermark.image)

日志打印模块包含`wxNativeConsole`、`__webviewConsole__`、`wxConsole`、`wxPerfConsole`等，打印日志类型非常丰富。日志打印API在基础库源码中导出充斥着，因为边界函数特别的多，恨不得每个逻辑判断都加上`try catch`

### 系统函数和第三方函数模块

系统函数和第三方函数模块在文件5294行左右。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bcc63ef06ca149568aff7f2a5876ed6f~tplv-k3u1fbpfcp-watermark.image)

### Report信息上报模块

Report信息上报模块在文件5488行左右。内部包含了非常多种类的上报api及异常监听api。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f482423c4d0b41c78a77bfd367df0eb9~tplv-k3u1fbpfcp-watermark.image)

### Exparser模块

Exparser组件系统模块的实现就是在基础库中，文件9820行左右。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3b73eecf47b47e2af86c3ee8d937b11~tplv-k3u1fbpfcp-watermark.image)

还记得exparser组件系统章节中渲染层经过编译后的自定义HTML标记吗？忘记没关系，我们回顾一下

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e8ffa924f2642da9b14ab1870dd41b7~tplv-k3u1fbpfcp-watermark.image)

上图是渲染层中编译后的Exparser自定义组件标记，`wx-text`、`wx-view`等。我们在WXML中写的结构是这样的：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0764d973e7e4b11b668f2e42849f775~tplv-k3u1fbpfcp-watermark.image)

这是经过几层编译后的结果，前面章节我们讲到 `WXML`文件经过`WCC`编译器编译成`js`文件，生成`$gwx()`函数, `$gwx()`函数接收文件路径和动态数据生成`virtualDOM`。这里看一下编译后的`virtualDOM`中的`<text>`标记。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91d1dee0ea5d4d5f982f60e9c20d7987~tplv-k3u1fbpfcp-watermark.image)

可以看到`virtualDOM`中已经把WXML中的`<text>`标记编译成了`wx-text`，通过`Exparser`组件系统章节我们可知这是`Exparser`自定义组件。

那么`Exparser`组件系统就是将`virtualDOM`转化成`HTML自定义标`记的最后环节。

顺着上面的线索我们看一下渲染层基础库`Exparser`模块中的`wx-text`这个组件的定义是什么样的。为了方便讲解我把代码进行了拆分，我们按照模块逐一分解。

首先进行元素的注册，并且声明模版`template`。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/308b5228a01a41bd8a9616e737ef8048~tplv-k3u1fbpfcp-watermark.image)

与`WebComponent`中的`tempalte`一致，并且设计了插槽`slot`。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4655f1788d9417f87de59f6a0e73e93~tplv-k3u1fbpfcp-watermark.image)

接下来就是可配置的一些属性，并且这些属性是开放出去的。与小程序组件文档上的属性进行对应。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c9b42ea132e4f1592df8527ceef23e5~tplv-k3u1fbpfcp-watermark.image)

接下里就是方法模块：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/953f6c0221f647cc91de5c04cd620f96~tplv-k3u1fbpfcp-watermark.image)

可以看到`method`里面就是自定义的一些方法, 其中`_htmlDecode`处理了`space`的合法值及一些语法的转换实现，对应文档中的：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/686492d8b44b4d29a9341f0343d06b78~tplv-k3u1fbpfcp-watermark.image)

文档中`tips`描述的也很明确。

可以看到与method同层级的还有`created`、`attached`方法，其中`attached`为组件扩展附加方法，比如给予一些提示或者边界检测。上面截图中的`attached`方法内部包含一条警告日志：

> 对于开发人员：[text] selectable已弃用，请改用user select。

`created`方法即为初始化组件方法，内部触发`method._update`函数，那我们就看一下`_update`方法内部：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd935662dfde4296944303739fe9ec28~tplv-k3u1fbpfcp-watermark.image)

内部实现首先是判断文本在不可选的状况，纯展示文本如果过长那么提示

> For developer: 提示： text 组件包含了长文本，可以考虑增加 user-select 属性，方便用户复制。

然后就是进行上面讲到的`_htmlDecode`方法对文本进行一些转译处理，最后插入到slot插槽中去。

这样的话一个完整的`wx-text`类型的exparser组件就注册完成了。

### virtualDOM模块

VirtualDOM 模块和Vue、React中virtualDOM实现相似，但这里它主要模拟了DOM 接口上面的element() 对象，在文件78322行左右。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72e458aff8d548bb8066c5e0231b2fde~tplv-k3u1fbpfcp-watermark.image)

截图是此模块中关于元素样式属性的配置。

### 默认样式注入模块

最后的模块就是默认样式注入，在文件83694行左右。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6752070c629d46808499d1e259d38dc6~tplv-k3u1fbpfcp-watermark.image)

## 总结

这一章节分析了渲染层基础库WAWebview的模块分析。下一节我们进行逻辑层基础库WAService的模块分析。















