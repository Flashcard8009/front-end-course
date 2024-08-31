﻿作为前端开发，我们每天都会用 Chrome DevTools 调试 Chrome 的网页，但其实它还可以远程调试安卓手机的网页。

那 Chrome DevTools 如何远程调试安卓网页呢？它的实现原理是什么？

今天我们就来了解一下：

## 远程调试安卓网页

用数据线把安卓手机和电脑连接起来，在手机设置里打开 USB 调试：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ce501a5aa4e4e3793b26e98b36c1d4c~tplv-k3u1fbpfcp-watermark.image?)

然后在 chrome 打开 chrome://inspect 页面，勾选 Discover USB devices（默认是勾选的）：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15bf80ec62174050ba88e3a3d008ade2~tplv-k3u1fbpfcp-watermark.image?)

这时候下面就会出现一个提示：请在设备上接受 debugging 会话

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37d503b980be460c9ae6077f7c4f196c~tplv-k3u1fbpfcp-watermark.image?)

在手机上点击确定，就会建立调试会话：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0408c13b4464f228db6e0c23e53fba0~tplv-k3u1fbpfcp-watermark.image?)

下面就会列出所有可以调试的网页：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/838a4d272dc24df199d6dacd485f0758~tplv-k3u1fbpfcp-watermark.image?)

浏览器里的网页，或者 APP 调试包的 webview 的网页都会列出来。

点击 inspect 就可以调试了：

可以审查元素：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72f479134b20453c9d1290d8fcd083e9~tplv-k3u1fbpfcp-watermark.image?)

可以打断点：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a72b42495ea467891bf73a556dbd57f~tplv-k3u1fbpfcp-watermark.image?)

也可以用 Performance 分析性能：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/836cc392be604b628846d694a905be9a~tplv-k3u1fbpfcp-watermark.image?)

各种调试 PC 网页的功能基本都支持。
这样就可以愉快的调试安卓的移动端网页了。

不过这个过程你可能会遇到这样的问题，打开的窗口是空白的或者是 404:

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8d946a5856642c0b61043d28b6a121f~tplv-k3u1fbpfcp-watermark.image?)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e08c3f89379f40fda680097e713a0ada~tplv-k3u1fbpfcp-watermark.image?)

这是因为调试的目标可能是任意 chrome 版本，那么 Chrome DevTools 自然也要用相应的版本才行，所以就需要动态下载。

而动态下载的 devtools 网页是在 google 域名下的，需要科学上网才行。

科学上网之后，就可以正常的下载 Chrome DevTools 来做调试，也就不会白屏或 404 了。

但也不是每次都要科学上网，一个调试目标只需要下载一次 Chrome DevTools 的代码，之后就可以一直用了。

我们了解了 Chrome DevTools 怎么调试安卓的网页，那它的原理是什么呢？

## Chrome DevTools 的原理

Chrome DevTools 被设计成了和 Chrome 分离的架构，两者之间通过 WebSocket 通信，设计了专门的通信协议： Chrome DevTools Protocol。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8ada18bdeec4c7fad820047618e6e93~tplv-k3u1fbpfcp-watermark.image?)

这样只要实现对接 CDP 协议的 ws 服务端，就可以用 Chrome DevTools 来调试，所以 Chrome DevTools 可以用来调试浏览器的网页、调试 Node.js，调试 Electron 等。

那自然也就可以远程调试安卓手机的网页了，只要开启了 USB 调试，那手机和电脑就可以做网络通信，从而实现基于 CDP 的调试。

这个 CDP 的调试协议是 json 格式的，如果想看它传输了什么也是可以的：

在 Chrome DevTools 的设置里开启 Protocol Monitor：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c901b71ab7a45f6864b797d886153ae~tplv-k3u1fbpfcp-watermark.image?)

然后在 more tools 里打开 Protocol Monitor 面板：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65bf3e82eba745e0a9496bce235d0d47~tplv-k3u1fbpfcp-watermark.image?)

然后你就可以在 Protocol Monitor 面板里看到所有的 CDP 协议的数据交互了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebb31341c2d741c7b4a57ea5505c5be7~tplv-k3u1fbpfcp-watermark.image?)

这就是调试的实现原理。

那如何知道哪些界面是网页，哪些是原生绘制的呢？

Android 下可以在设置里打开显示布局边界：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8fd61cd891a44bd8b450808d53ddb10~tplv-k3u1fbpfcp-watermark.image?)

如果是原生的组件，就会显示边框：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26b5a36bc1564c588592f43ab8e6914e~tplv-k3u1fbpfcp-watermark.image?)

而没有边框的，就是网页。比如钉钉的这个界面：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e88111a1609846f8b63304bc97090aa2~tplv-k3u1fbpfcp-watermark.image?)

## 总结

Chrome DevTools 和 Chrome 是分离的架构，两者通过 WebSocket 通信，通信协议是 Chrome DevTools Protocol，可以在 Protocol Monitor 里看到 CDP 的数据交互。

因为这样的实现原理，Chrome DevTools 可以调试很多目标，其中就包括 USB 设备。

打开 USB 调试之后，在 chrome://inspect 页面就可以看到可调试的网页了，点击对应的网页就可以调试。

要注意的是调试的目标浏览器要和用的 Chrome DevTools 版本一一对应才行，所以第一次调试会先下载 Chrome DevTools，这需要访问 google 的域名，如果没有科学上网，会有白屏和 404 的问题。

理解了调试的原理，Chrome DevTools 调试安卓网页的流程，就可以愉快的远程调试安卓手机的网页了。
