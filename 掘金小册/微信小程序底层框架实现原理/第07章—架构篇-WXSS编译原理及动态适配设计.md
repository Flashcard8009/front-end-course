## 前言

本想这一章节只单独把rpx动态适配拿出来讲解的，现在发现WXSS这里的逻辑比较独立，并且不多，也就顺带在这里一起讲解了。

本章节内容分解：

- WXSS语法解析
- WXSS编译原理
- WXSS动态适配设计

## WXSS

我们先来看一下官方给予的概念

`WXSS (WeiXin Style Sheets)`是一套样式语言，用于描述` WXML `的组件样式。
`WXSS` 用来决定 `WXML` 的组件应该怎么显示。

7.架构篇-WXSS编译原理及动态适配设计6 修改标题  设置为试读 本章节已完成删除这两句话听起来就是那么的耳熟，这不就是`html`与`css`之间的关系嘛，这样的描述非常合情合理。

为了适应广大的前端开发者，`WXSS` 具有 `CSS`的大部分特性。同时为了更适合开发微信小程序，`WXSS` 对 `CSS` 进行了扩充以及修改。通俗的可以理解成基于CSS改了点东西，又加了点东西。

这里我们就先与`CSS`进行一个比对，看看保留了哪些大部分特性。

## 写法

1. 支持内联样式概念。
2. 支持全局样式概念，定义在 `app.wxss` 中的样式为全局样式，作用于每一个页面。在 `pages`目录下的 `*.wxss` 文件中定义的样式为局部样式，只作用在对应的页面，并会覆盖 `app.wxss` 中相同的选择器。这些点与`css`高度相似。
3. 不支持嵌套写法。

## 语法

常用样式类型与`css`保持一致。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e81b1d34a845423da2be8fe2fdaa1eda~tplv-k3u1fbpfcp-watermark.image)

盒子模型

![盒子模型.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e792c4e60c5472394ef75d0aec22172~tplv-k3u1fbpfcp-watermark.image)


函数及其他

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a76dd66ba114a56bd10e5fbe56e71d8~tplv-k3u1fbpfcp-watermark.image)

基本语法方便也与`css`高度相似，并且上述所有标签都支持微信小程序特有的尺寸单位`rpx`。比如下图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0072b21ec474cc1a292fe3dec2af217~tplv-k3u1fbpfcp-watermark.image)

## 选择器

目前支持的选择器有：

- `.class`	 样例 ：`.intro`	选择所有拥有 `class="intro"` 的组件
- `#id	`        样例 ： `#firstname`	选择拥有 `id="firstname"` 的组件
- `element`     样例 ： `view`	选择所有 `view` 组件
- `element, element`	样例 ： `view, checkbox`	选择所有文档的 `view 组件和所有的 checkbox 组件`
- `::after` 样例 ： `view::after`	在 `view` 组件后边插入内容
- `::before` 样例 ： `view::before`	在 `view` 组件前边插入内容

官方给予的是上面这些，但善于发现的人察觉到了一些，`:nth-child`系列，`:first-child`, `:end-child`等等都是支持的，可能后续小程序版本迭代更新了，文档并没有更新。

看到如上那些，是不是`WXSS`给予你了一些熟悉的感觉，非常熟悉。

## 拓展

与 `CSS` 相比，`WXSS` 扩展的特性有：`尺寸单位`和`样式导入`两个方面，我们最为熟悉的就是`尺寸单位rpx`。

### rpx

`rpx （responsive pixel）`直译为：`响应像素`。写过小程序的都知道这个单位，可以自动适配所有大小的屏幕，而不必使用一些第三方插件进行响应式布局。

通常我们开发移动端Web的时候，总会遇到像素点适配问题，其中就存在布局问题，曾经我们为了做一些响应式的布局，引入`REM`，`VW`等，或者工程化之后使用`px2remvw`,这样的自动转化插件。而小程序的适配方案则为`rpx`。我们继续分析。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb5e9d4aba19438db062cf22932ba2be~tplv-k3u1fbpfcp-watermark.image)

上方放置了一个小程序转换完成的`px`对比`rpx`的表格。这样明显不够直观，接下来我们将找寻源码，刨析一下`rpx`具体的实现原理。但是在这之前，我们需要看一下`WXSS`是怎么编译并且执行的。

## 编译

`WXSS`并不可以直接执行在`webview`层进行渲染，而是通过了一层编译。我们接下来就带大家编译一个`WXSS`看一下。

## WCSC

编译的工具名字叫`WCSC`，这个编译的过程是在微信开发者工具端执行的，那么这个编译工具在哪呢，我们来找一下。在微信开发者工具的控制台界面，输入`help()`命令可见如所示界面。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6caa2f288d7e4f49b387dd8414cf911c~tplv-k3u1fbpfcp-watermark.image)

> 如果help()函数执行后无效果或者抛错，请检查控制台下方位置是否为top选项卡。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26a56cf3c49f49ea879d730e5ab0fd53~tplv-k3u1fbpfcp-watermark.image?)

可以看到这里有一些命令。我们继续在控制台执行第八条`openVendor()`命令。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ed5a5ccf48b4a4fa12a3f7da7218d31~tplv-k3u1fbpfcp-watermark.image)

这时候弹出了一个名为`WeappVendor`的文件夹。在我截图的这个顺序里，可以看到最后一个文件名称正是我们要寻找的`WCSC`。文件种类是可执行文件。`WXSS`正是用这个工具来编译的。

大家记得这个寻找`WeappVendor`文件夹的方法，后续还要用到好几次，当然后续我还是会一步一步带大家寻找，留一个印象即可。

我们找到了`WCSC`编译工具后，把这个工具复制到项目的`pages/index`目录下，与`index.wxss`同目录。像下图一样即可，项目就是最开始文章中创建的新项目。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/164db8cbd68e47afb78d81599239d7b9~tplv-k3u1fbpfcp-watermark.image)

在执行编译前，我们先看一下`index.wxss`内部结构是怎样的。

看完内部结构后，把终端目录打开到`pages/index`目录中。执行：

```shell
$ ./wcsc -js index.wxss >> wxss.js
```

这时候可以看到目录中多了一个`wxss.js`文件

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b3e118e19784d7898415d9528a23cc9~tplv-k3u1fbpfcp-watermark.image)

`wxss.js`文件就是`WXSS`文件编译后的文件，`index.wxss`文件会先通过`WCSC`可执行程序文件编译成`js`文件。并不是直接编译成`css`文件。

那么我们直接看一下内部代码是怎样的呢。

这里我拆成了三部分来看，三部分加一起就是完整的代码。第一部分：`设备信息`。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a869ebcf8aed435cb1a054b210faafed~tplv-k3u1fbpfcp-watermark.image)

在这里我备注写的细节一些，因为备注配合代码一起看，更方便去理解。

这个部分用于获取一套基本设备信息，包含`设备高度`、`设备宽度`、`物理像素与CSS像素比例`、`设备方向`。

我们来看下一部分：`转化rpx`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acd0f17a6e964dc7b2722862058aaeb6~tplv-k3u1fbpfcp-watermark.image)

这里就是`rpx转化`的方法了，`rpx转化`的具体算法就是中间那两句，并且做了一个精度收拢的优化。把那两句单独提取出来再看一下，平常在开发中自己写一个这样的方法也是一种不错的选择。

```javascript
number = number / BASE_DEVICE_WIDTH * (newDeviceWidth || deviceWidth);
number = Math.floor(number + eps);
```

以后再看到`rpx`适配屏幕就可以直接想到这段源码。再来看第三部分。

![carbon (1).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/411a61d797ce465c8bdd126ab4bbe6fc~tplv-k3u1fbpfcp-watermark.image)

这一段代码比较长，看到方法名称我们就可以猜到这个函数是干嘛用的了`setCssToHead`。

首先看到最下方执行`setCssToHead`方法时候的传入参数。隐约可以看出来是我们在`index.wxss`之中写入的样式。但是仔细一看，格式不太一样了，变成了结构化数据，方便遍历处理，并且处理后便于`makeup`组装。还哪里不一样了呢，可以看到其中在`index.wxss`中写`rpx`单位的属性都变成了区间的样子`[0, 128]`、`[0, 20]`。其他单位并没有转换。这样的话就可以方便的识别哪里写了`rpx`单位，然后执行第二部分的`transformRPX`方法即可。

`makeup`组装之后，创建`<style>`标记，插入到`<head>`中。

那我们来看一下经由这样转化完成并且组装完后，具体插入到了渲染层的哪里呢？

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59ff32579c1b4b6b92c562e4a2ab5562~tplv-k3u1fbpfcp-watermark.image)

果然是插入到了渲染层中，可以看到文件中用`rpx`写的单位都转化成了`px`，后方`wxcs_styles_xx`则为转化之前的属性值。

我们看完了`js`代码后，有个疑问，在前面章节，解析渲染层基础模块的时候，并没有看到类似的样式`js`文件注入，对吧。那是怎么注入的呢? 我们来寻找看一下。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7133062a9b514d169a52cdc6803f35e2~tplv-k3u1fbpfcp-watermark.image)

在渲染层的一个不起眼的`<script>`标签中,有很长的一串字符串，并且用`eval`方法执行。如果你仔细看的话，还是可以勉强分辨出，这个字符串正是我们前面编译出来的`js`转换成的。

这样就可以得知，编译后的代码是通过`eval`方法注入执行的。这样的话完成了`WXSS`的一整套流程。
