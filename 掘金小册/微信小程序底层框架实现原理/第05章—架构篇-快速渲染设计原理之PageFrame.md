## 前言

本章内容分解如下：

- 小程序如何做到快速打开新页面
- 小程序快速渲染流程原理
- webview-pageFrame设计原理

## PageFrame

还记得在前面文章中我们寻找渲染层的时候，找到了个奇怪的`webview`。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f882cd51b5642b6980890c252a591f4~tplv-k3u1fbpfcp-watermark.image)

`pageFrame`这个`webview`是干嘛的呢？我们这章节就来探索一下。

我们在写小程序页面视图时，貌似并不关心`webview`中的html结构，这些都是小程序底层帮我们实现，我们只需要写页面ui和业务逻辑即可。下面我们来看看view视图层小程序帮我们做了什么。先来看一下视图层`pageframe.html`的模板：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75b040d04f4347be9e3cad790f14c323~tplv-k3u1fbpfcp-watermark.image)

为了讲解便利，中间部分代码由标识进行替代。从上图上看来是不是和渲染层的`HTML`非常像。这个`webview`其实就是一个用来新渲染`webview`的模板。

我们已知在小程序中使用 WXML 文件描述页面的结构，WXSS 文件描述页面的样式，WXML文件会被编译为虚拟DOM，WXSS会被编译为js。那么每个独立的页面都会经过这样的编译，如何快速的生成webview，或者说如何快速的打开一个页面。会成为一个问题。我们有了虚拟DOM和样式描述表就可以渲染页面了吗。还远远不够。

先分析一下`pageFrame`的html结构中注入的js资源：

- `./__dev__/wxconfig.js`: 

  小程序默认总配置项，包括用户自定义与系统默认的整合结果。在控制台输入```__wxConfig```可以看出打印结果
  ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98711d7ce0534a49ac267da9c6ed6157~tplv-k3u1fbpfcp-watermark.image)

- `./__dev__/devtoolsconfig.js`

  小程序开发者配置，包括`navigationBarHeight`,标题栏的高度，状态栏高度，等等，控制台输入```__devtoolsconfig```可以看到其对应的信息

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2a4b4b34fbf494c87b9f20e47785f81~tplv-k3u1fbpfcp-watermark.image)
- `./__dev__/deviceinfo.js`

  设备信息，包含```尺寸/像素点pixelRatio```

- `__dev__/jsdebug.js`

  debug工具。

- `./__dev__/WAWebview.js`

   渲染层底层基础库

- `./__dev__/hls.js`

  优秀的视频流处理工具。

- `./__dev__/WARemoteDebug.js`

  底层基础库调试工具

可以看到pageFrame注入的脚本与我们分析`pages/index`渲染层webview是一样的。正式因为pageFrame快速启动技术，就像一个工厂一样，可以快速生成webview的基础格式。在这其中pageFrame就是业务webview的模板。

我们继续往下分析，有一些不一样的地方。

- `<!-- wxappcode -->`

可以发现pageFrame中包含一些注释的单词，这些注释的单词为注释占位符，如果你细心对比就会发现，在前面分析渲染层代码的时候，我们见到过这些注释占位符编译后的样子。

比如wxappcode.js在渲染层中为如下形式（老版本基础库）

`<script src="./__dev__/wxappcode.js" chartset="UTF-8"></script>`

或者是这样（新版本基础库）

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bc422f3ef45495e94bfc334ac6afd90~tplv-k3u1fbpfcp-watermark.image?)

无论新老版本，其中的代码都是一样的。我们先看一下内部结构。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d3418abe1d84490b664bb341ec19b64~tplv-k3u1fbpfcp-watermark.image)

文件中包含了所有页面的编译路径，这个编译路径有什么作用呢，小册后面章节virtualDOM章节中，编译`WXML`后生成的`$gwx`函数需要的参数就是这个路径。此文件中就进行了保存。

再接着看的话会看到WXSS章节讲到的编译后的WXSS，就跟我们在渲染层看到的一样，`eval(setCssToHead....)`。

内部记录的属性有：

- `decodePathName`
- `.json`配置
- `.wxml`编译后的`$gwx`函数。`$gwx`函数会在virtualDOM章节重点讲解。
- `.wxss`编译后的`eval`函数。

这里就可以想象出来，如果小程序需要打开某个页面的时候，只需要从这里提取出页面特有的这几个属性，配合`pageFrame`模板就可以快速生成一个新的webview。

那么具体快速启动的方案是怎样的呢？我们继续探索

## 如何快速启动

我们看一下官方给予的一段描述：

在视图层内，小程序的每一个页面都独立运行在一个页面层级上。小程序启动时仅有一个页面层级，每次调用`wx.navigateTo`，都会创建一个新的页面层级；相对地，`wx.navigateBack`会销毁一个页面层级。
对于每一个新的页面层级，视图层都需要进行一些额外的准备工作。在小程序启动前，微信会提前准备好一个页面层级用于展示小程序的首页。除此以外，每当一个页面层级被用于渲染页面，微信都会提前开始准备一个新的页面层级，使得每次调用`wx.navigateTo`都能够尽快展示一个新的页面。
页面层级的准备工作分为三个阶段。第一阶段是启动一个`WebView`，在iOS和`Android`系统上，操作系统启动WebView都需要一小段时间。第二阶段是在`WebView`中初始化基础库，此时还会进行一些基础库内部优化，以提升页面渲染性能。第三阶段是注入小程序`WXML`结构和`WXSS`样式，使小程序能在接收到页面初始数据之后马上开始渲染页面（这一阶段无法在小程序启动前执行）。

对于`wx.redirectTo`，这个调用不会打开一个新的页面层级，而是将当前页面层级重新初始化：重新传入页面的初始数据、路径等，视图层清空当前页面层级的渲染结果然后重新渲染页面。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f201e6d2de9e41c4b01c3dc589abdc8a~tplv-k3u1fbpfcp-watermark.image)

我们对官方的这段描述进行一个解释：

视图层内，每个页面我们已经知道了都是一个webview，当小程序启动的时候只有一个首页webview,我们前面的例子中为`pages/index`页面。

我们在去寻找index页面的渲染层的时候做过这样一件事情，寻找webview标签，进入渲染层的控制台。在寻找webview渲染层的时候，我们找到了四个webview

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/036782085ddf4c5d86a3fb9483c2ef15~tplv-k3u1fbpfcp-watermark.image)

- pages/index
- pages/pageFrame
- appService.js // 逻辑层
- 微信开发者工具

`wx.navigateTo`为新开一个页面，在微信小程序中的表现形式为从右往左划出来的一个页面，为什么如此的丝滑，想必与`pageFrame`这层`webview`密不可分。在执行`wx.navigateTo`新开一个页面的时候，就是创建一个新的webview并插入到视图层中。这里忘记的可以返回到`webview`章节，有一个部分专门解释了五个`webview`的部分，并且创建新`webview`并插入这个操作是有最大值限制的，有关于性能方面的考虑。

`wx.navigateBack`则为销毁webview。

我们在打开`pages/logs/logs`视图页面时，发现dom中多加载了一个`__pageframe__/pageframe.html`的视图层，其模板内容正如上方描述的。这个视图层的作用正是为了小程序提前为一个新的页面层准备的。

小程序每个视图层页面内容都是通过`pageframe.html`模板来生成的，包括小程序启动的首页；下面来看看小程序为快速打开小程序页面做的技术优化：

- 首页启动时，即第一次通过`pageframe.html`生成内容后，后台服务会缓存`pageframe.html`模板首次生成的html内容。
- 非首次新打开页面时，页面请求的`pageframe.html`内容直接走后台缓存
- 非首次新打开页面时，`pageframe.html`页面引入的外链js资源(如上图所示)走本地缓存

这样在后续新打开页面时，都会走缓存的`pageframe`的内容，避免重复生成，快速打开一个新页面。

那么视图层新打开页面的流程是怎样的呢？

其实在小程序开发者工具实现中，在创建每个视图层页的webview时，都会为其绑定了`onLoadCommit`事件(它会在页面加载完成后触发，包含当前文档的导航和副框架的文档加载)。初始时`webview`的src会被指定为空页面地址`http://127.0.0.1:${global.proxyPort}/aboutblank?${c}`，其中`c`为对应`webview`的`id`。`webview`从空页面到具体页面视图的过程如下：

1. 空页面地址webview加载完毕后执行事件中的`reload`方法，即设置`webview`的`src`为`pageframe`地址
2. 加载完成后，设置其src为`pageframe.html`, 新的src内容加载完成后再次触发`onLoadCommit`事件但根据条件不会执行`reload`方法。
3. `pageframe.html`页面在`dom ready`之后触发注入并执行具体页面相关的代码，此时通过`history.pushState`方法修改webview的src但是webview并不会发送页面请求。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81398349fb3d41cc8d58acfe078b17ee~tplv-k3u1fbpfcp-watermark.image)

`pageframe.html`模板生成的内容除小程序基础库视图层的底层功能之外，还包括小程序所有页面的模板信息、配置信息以及样式内容，这些在上方讲解的`wxappcode.js`文件中已经了解过了。

那么，既然每个视图层页面由`pageframe`模板生成，那么小程序每个页面独有的页面内容如`dom`和样式等如何生成呢，这主要是利用`nw.js`的`executeScript`方法来执行一段js脚本来注入只与当前页面相关的代码，包括当前页面的配置，注入当前页的css以及当前页面的`virtual dom`的生成.

最终生成的js代码（拿`pages/index/index`为例）如下图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54791d0a3f514a309af3b23f705c89e1~tplv-k3u1fbpfcp-watermark.image)

其中：

`history.pushState('','', 'http://127.0.0.1:63444/__pageframe__/pages/index/index')`
这句代码的作用修改当前webview的src，因为视图层的webview的src为`pageframe.html`，通过这句代码将其变更为具体的页面地址。

另外，需要注意的是`nw.js`的`executeScript`方法注入的代码是需要时机的，需要等到视图层的初始化工作准备`ready`之后才行，那么这个时机如何知道呢？细心的读者可能发现，在`pageframe`模板的最后一个`script`的内容:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc953d61447d4a379643c338b84f79cd~tplv-k3u1fbpfcp-watermark.image)

这个从字面意思可以看出此时应该是页面`dom ready`的一个时机，通过`alert`来进行通知。

`alert`能通知消息？

当然可以的，在`nw.js`的`webview`中`alert`、`prompt`对应的弹框是被会阻止的，那么通过为`webview`绑定`dialog`事件来知道是那种弹框类型，以及提示内容.恍然大悟了没有？

如果你尝试直接打开在`webview`层的src地址，你会发现只有一个弹窗出现，你会很疑惑。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d54d32840ee9475ba9f8b2a4196d6167~tplv-k3u1fbpfcp-watermark.image)

这样方法`loadPage`就会触发nw注入并执行页面相关的代码。最终生成的页面视图对应`dom`结构如下图：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/965776ccc3444b45b56a3af5ff61048a~tplv-k3u1fbpfcp-watermark.image)

可以看出，视图页面生成的`dom`结构中，`document.body`已无`pageframe.html`模板中对应body中的`script`内容，这是因为视图层的`WAWebview.js`在通过`virtual dom`生成真实dom过程中，它会挂载到页面的`document.body`上，覆盖掉`pageframe.html`模板中对应`document.body`的内容。

这样的话就完成了一套快速渲染过程。



