## 前言

本章内容分解：
- 寻找渲染线程文件
- webview容器设计
- 渲染层文件解析

## Webview及基础模块

上面在用小程序架构图的时候一直说渲染层、逻辑层。下面带大家来找一下这两个文件。

首先我们在微信开发者工具上默认看到的界面是调试我们开发代码的地方，真正渲染编译的文件并不是长这个样子的。比如说在调试器中看到的```WXML```选项卡中，看到的代码和我们项目中的```WXML```代码是一致的，这样的话我们很方便的就可以调试。

话不多说，我们在左上角选择，```微信开发者工具 ->调试 ->调试微信开发者工具```。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7d8f6b66f7604fd9940a4cb45acf3da2~tplv-k3u1fbpfcp-watermark.image)

然后你会发现弹出了另一个调试界面，与谷歌的调试界面不能说相似吧，简直一摸一样。打开```Element```页签看到结构如下。这是微信开发者工具的结构。并不是渲染层。

可以看到微信开发者工具的界面的代码竟然如此熟悉，就是```html```。就像调试网页一样，鼠标滑过页面```DOM```结构的时候，微信开发者工具相应的模块就会高亮起来，可以简单看一下微信开发者工具的页面结构。不得不感叹一下混合开发这样的开发模式，不仅仅在移动端应用起来，在PC端也是完全没有问题的，性能用起来也是没什么问题的，能用```javascript```写的终究会用```javascript```来重写。大前端666.

我们在```Element```页签中的DOM结构中来找一找左侧我们看到页面的模块是怎样展现的呢。找到的话如下。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43c44ed2efe548f3b56dba6acc9d4bd4~tplv-k3u1fbpfcp-watermark.image)

我们可以看到一些`webview`标签，其中第一个```webview```标签的src路径看起来是不是很眼熟，这个就是当前页面在项目中的路径，以```webview```的形式渲染在微信开发者工具界面中，可以说是嵌入在里面。

这个才是真正的```渲染层```，在这里是无法展开```webview```标签看到里面页面的具体结构的，要用一些特殊的方式。

在开发者工具调试界面，也就是单独的调试界面中切换到```console```页签，这里的```console```页签和我们平常使用的浏览器的是一样的。我们找到```webview所有的DOM节点```。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/036782085ddf4c5d86a3fb9483c2ef15~tplv-k3u1fbpfcp-watermark.image)

这里可以看到有四个```webview```，如果跳转页面了大于4个也是正常的，如果打开新页面，则会创建并插入一个新```webview```。我这里是只打开了默认页面，所以只有四个，下面我会演示大于4个点时候。基础的四个```webview```分别是什么呢？他们分别是```视图层的webview```，```业务逻辑层webview```，```调试器的webview```和```编辑区的webview```。重点分析视图层和业务逻辑层webview。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a1d61009c6144ac9d4781cd2e585007~tplv-k3u1fbpfcp-watermark.image)

上方webview层src路径为```pageframe/pages/index/index``` 是渲染层视图，渲染的就是我们进来时候开到的初始页面。
下方webview层src路径为```appservice/appservice?***``` 是逻辑层视图。

由上面可以看出，小程序开发者工具业务逻辑层是在webview中执行的。这里可能有人有些绕，这里举例子的一切环境都是我们电脑上的微信开发者工具。微信开发者工具模拟了在移动端的双线程结构。每个webview可以理解为在移动端跑的一条线程。

因为小程序开发者工具是HTML嵌入写的，所以看代码示例可能看着看着就以为是小程序了，缕清这一点，上面我们看到的HTML结构不是小程序，只其中几个webview中才是小程序的逻辑。其他的都是开发者工具为了模拟微信端的环境。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7734e6b3a045430a8af39f6ef607a466~tplv-k3u1fbpfcp-watermark.image)

所以我们重点关注视图层和逻辑层的webview。

如果我们回到开发者工具的操作页面进行页面跳转一下。点击一下index页面的头像会跳转到logs日志页面后，回到开发者工具的```devTools工具```后再寻找webview层，这时候会发现多了一个webview

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bffc537165df4180872cd59b04878a6a~tplv-k3u1fbpfcp-watermark.image)

从图中可以看出，多了一个视图层webview:```pages/logs/logs```，可以见得，页面得载入是通过创建并插入webview来实现的，

这个过程中会涉及到一个问题，就是打开页面的数量。在某些设计下，比如一个商品详情页面中会推荐其他的商品，这些商品点进去就是详情页面。详情页面中又有推荐，又可以点进去。这样重复下去打开的页面就会越来越多。

如果不做限制，会有严重的```内存```问题。所以在这方面，微信小程序做了限制，在微信小程序中打开的页面不能超过10个，达到10个页面后，就不能再打开新的页面。这时候有可能会卡死，不知道的话会以为是```bug```。

我们话接上文，从上方我们自己寻找webview代码节点可以知道第一个就是```page/index/index```路径相对应的webview。通过```showdevTools```方法来打开调试此webview界面的调试器。命令如下。

```
document.getElementsByTagName('webview')[0].showDevTools(true, null)
```

上述命令复制粘贴的时候有时候打不开调试器，复制粘贴打不开的话，从头手打即可。打开的调试器页面如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e015bfe620c4b72969436e966633e77~tplv-k3u1fbpfcp-watermark.image)

是不是很多调试器一层套一层的。不过不必慌张，我们慢慢看下去，看到最后你一定会豁然开朗的。我们继续。

这样的话就打开了当前显示的页面的调试界面。可以看到这里的title显示的就是小程序渲染层。

我们看一下小程序渲染层的结构，从head区域开始分析，从上到下。

第一个```style标签```中包含了一些```基础style标签```。展开的部分截图如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef7ad3f99cf1484a958ce5c2e5e10f94~tplv-k3u1fbpfcp-watermark.image)

第一个style标签处理了webview层的基础组件的样式初始化。

上面我们提到过wxml中的标签使用的是```Exparser框架```，其中这些自定义组件的样式也是需要一起渲染在webview中的。

可以看一下截图，黄色区域的base64两张图片我也贴在这里，是```wx-audio-button标签```的```开始切图```和```暂停切图```。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09807c69ad6d43d2b51a88675816985b~tplv-k3u1fbpfcp-watermark.image)

看的仔细的同学肯定看出不太一样的地方，这里没有```rpx```，这里就是单纯的css，css选择器，css的px。为什么没有rpx呢继续往下看。

再往下面这个区域，首先可以看到几个全局变量，从英文名称上就可以知道意思：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b5f903e1f31419d9feb102efb29a646~tplv-k3u1fbpfcp-watermark.image)

- `webviewId`,从这个id又可以再次印证webview层不止一个。
- `wxAppCode` 整个页面的json wxss wxml编译之后都存储在这里，下面有个script标签就是它，下面会讲到。
- `Vd_version_info` 版本信息

后面的js文件如果想看的话，把鼠标放在路径上右键`open in new tap`即可

`./dev/wxconfig.js`是小程序默认总配置项，包括用户自定义与系统默认的整合结果。在控制台输入`__wxConfig`可以看出打印结果

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98711d7ce0534a49ac267da9c6ed6157~tplv-k3u1fbpfcp-watermark.image)

`./dev/devtoolsconfig.js`小程序开发者配置，包括navigationBarHeight,标题栏的高度，状态栏高度，等等，控制台输入`__devtoolsConfig`可以看到其对应的信息

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0fbcbe4305be4badbab6218a3ba17267~tplv-k3u1fbpfcp-watermark.image)

`./dev/deviceinfo.js` 设备信息，包含`尺寸/像素点pixelRatio`。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f81ed35483174e2f8e81f18cab655ddb~tplv-k3u1fbpfcp-watermark.image)

`./dev/jsdebug.js` debug工具。

`./dev/WAWebview.js` 渲染层底层基础库，底层基础库后面会重点讲到。

`./dev/hls.js` 优秀的视频流处理工具。

`./dev/WARemoteDebug.js` 底层基础库调试工具

可以看出这里都是一些内置好的基础模块，配合webview进行渲染。

接下来看一下`WARemoteDebug Script`标签下面的script标签内容，展开后非常的长。这里呢也是渲染层的重中之重，这里呢就是后面要讲到的`渲染层-编译virtualDOM章节`以及`渲染层-WXSS动态适配章节`。

让我们缕清思绪，继续往下探索。

把目光移动到`<body>`标签中，是不是有点眼熟。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca4267986b5343d1b802e9e4ea974910~tplv-k3u1fbpfcp-watermark.image)

可以看到渲染出来的节点形式，正是上一节我们讲到的`Exparser`编译后的样子。

Webview中这么多的内容与脚本，那么小程序是如何做到快速新建并打开页面的呢？下章节我们来讲解pageFrame。
