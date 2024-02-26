## 前言

接上一章节，这一章节中我们将对逻辑层的底层基础库WAService.js进行一个分析，看看里面都有哪些模块。

本章内容分解：

- 逻辑层基础库源码解析

## WAService.js注入

我们依然是重温一下在哪里看到过逻辑层基础库的注入。和渲染层基础库是一样的，在最开始章节找寻小程序逻辑层代码中我们找寻到了逻辑层基础库的注入：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b50c9d79b61438e95815554102216db~tplv-k3u1fbpfcp-watermark.image)

## 模块分析

首先我们进行WAService.js文件的解析。在我们2.17.3版本的基础库包中，逻辑层基础库代码大概有11万行。

我们依然使用传统的折叠法，对代码进行折叠，然后进行分析。

我们从上往下进行分析，第一个模块是core-js模块。

### core-js模块

第一个模块与渲染层基础库中是一样的，core-js模块。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a38ab2e10aac47b1ab7196f9733dd6b8~tplv-k3u1fbpfcp-watermark.image)

### Foundation

Foundation是基础模块。我们来看一下声明的核心位置，在文件的4283行左右。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b18cadb91e154b35a70f0b871a5871c0~tplv-k3u1fbpfcp-watermark.image)

与渲染层一致

### WeixinJSBridge

WeixinJSBridge通讯模块，在4536行左右。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e10a73e7f0614d289879554b5eb97e0d~tplv-k3u1fbpfcp-watermark.image)

可以看到`WeixinJSBridge`通讯模块中包含了我们在通讯系统章节讲解的通讯api，包含有`on`、`publish`、`invoke`、`subscribe`、`invokeCallbackHandler`、`subscribeHandler`。 

通讯系统的具体实现在通讯系统章节有讲解，这里只是对Native注入通讯api的封装，便于内部调用。

这里与渲染层基础库也是一致的。

### NativeBuffer模块

`NativeBuffer`模块的定义在文件4924行左右。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/309026f6b6fc4876b602e2f73081a622~tplv-k3u1fbpfcp-watermark.image)

javascript语言自身只有字符串数据类型，没有二进制数据类型。
但在处理像TCP流或文件流时，必须使用到二进制数据。因此在微信小程序中，定义了一个`NativeBuffer`模块，该模块用来创建一个专门存放二进制数据的缓存区。

### 日志打印模块

日志打印模块在文件5028行左右。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee1086c6aee149e0904723ab0716fe36~tplv-k3u1fbpfcp-watermark.image)

日志打印模块包含`wxNativeConsole`、`wxConsole`、`wxPerfConsole`等，与WAService.js相比少了一个`__webviewConsole__`，

### WeixinWorker模块

worker模块在文件5195行左右。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f76b01976514f5c9ecab8949eb0f7b3~tplv-k3u1fbpfcp-watermark.image)

内部包含创建worker、结束当前workder、发送数据请求、监听回调等方法。

### JSContext模块

JSContext在文件5328行左右。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5593b2401e4c4519a35d26fd9d2bb616~tplv-k3u1fbpfcp-watermark.image)

JsContext 是js代码执行的上下文对象，相当于一个 webview 中的 window 对象。在同一个 VM 中，你可以传递不同的 Context。

### __appServiceEngine__模块

最后很大部分模块都是此模块，`__appServiceEngine__`，提供了`App`、`Page`、`Component`、`Behavior`、`getApp`、`getCurrentPages`等框架的基本对外接口。在文件109273行左右。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f7a6763d7864a2ab436cc33f672b80b~tplv-k3u1fbpfcp-watermark.image)

声明位置：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92efe0525e0c4ae6a94b9e5ec2f2d037~tplv-k3u1fbpfcp-watermark.image)

路由的跳转，优雅的`triggerLifeTime`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb90a26702cc40f1954aa8e4cb3f551f~tplv-k3u1fbpfcp-watermark.image)

### Exparser模块

Exparser组件系统提供了框架底层的能力，如实例化组件，数据变化监听，View 层与逻辑层的交互等。在文件100703行左右。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f1052799a07405d83ec286961487c59~tplv-k3u1fbpfcp-watermark.image)

### virtualDOM模块

`__virtualDOM__`连接着`__appServiceEngine__`和`exparser`，如对开发者传入Page方法的对象进行格式化再传入exparser的对应方法处理。在文件104361行左右

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d15af7ce088e44d8a1703a12b09e56e5~tplv-k3u1fbpfcp-watermark.image)

## 总结

这一章节分析了逻辑层基础库WAService的模块分析。















