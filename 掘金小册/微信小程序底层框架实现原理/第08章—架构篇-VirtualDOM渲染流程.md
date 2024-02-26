
## 前言

前一章节我们看到了`WXSS`文件编译后的源码，没有错`WXML`文件业务要进行编译步骤，这一章节就带大家一步一步刨析`WXML`编译后的源码。

## WXML

第一步，跟`WXSS`章节一样，去寻找用于编译`WXML`文件的可以执行文件`WCC`。

在微信开发者工具控制台中，执行`openVender()`。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ed5a5ccf48b4a4fa12a3f7da7218d31~tplv-k3u1fbpfcp-watermark.image)

可以看到就在`WXCS`上边就是`WCC`可执行文件。我们还是一样的操作。把`WCC`可执行文件拷贝到`pages/logs`目录下面，如下图所示。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f364b945fef946cb922be64059b5ce0a~tplv-k3u1fbpfcp-watermark.image)

`WCC`可执行文件的具体使用命令可以通过`--help`查看到。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48978f26d88f47819a2d2eb70d24ba66~tplv-k3u1fbpfcp-watermark.image)

执行命令```$ ./wcc -d logs.wxml >> wxml.js```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd6084368e7f4d9280af4b16afc58f3b~tplv-k3u1fbpfcp-watermark.image)

可以看到`pages/logs`目录下多了一个`wxml.js`文件，这个就是编译之后的`js`文件，我们来看一下里面的具体结构。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/447cd7a83a484280ad26359304a3df80~tplv-k3u1fbpfcp-watermark.image)

这部分代码不像`WXSS`编译出来的代码可以直接观看，而是进行了压缩混淆。基本上是不可读的。但是我们还是可以一点一点去分析这到底是怎么运行的，我们继续往下看。

可以看出，整体代码结构就是一个函数，函数名称为`$gwx`。它的作用是生成`虚拟dom树`，用于渲染真实节点。内部还有一些边界函数。比如有意思的 _n函数里面包含一个提示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99bfcf8f69d2497aa32962d8f9dc45f0~tplv-k3u1fbpfcp-watermark.image)

英文的意思是`超出Dom限制，请检查是否有任何错误`，DOM数量不可以超过16000个。但是比较之前的版本，这个提示可没有这么友好，它是这个样子的：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d647d6ab2f4242f1a7793aded0f12183~tplv-k3u1fbpfcp-watermark.image)

汉译我就不放在这里了哈哈，有兴趣的自己翻译一下，我们继续。

但是这个时候我们只知道`$gwx`是个函数，但是怎么调用呢？继续去渲染层代码中进行寻找。还是打开渲染层控制台`Elements`中，`control + F` 进行寻找即可，首先我们可以看到编译后的函数在渲染层也有一套。如下图：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a3ce6f9c1fe4fe2898bca175ce3a512~tplv-k3u1fbpfcp-watermark.image)

可以看到编译后的`WXML`文件以`js`的形式插入到了渲染层的`<script>`标签中。对比`WXSS`编译后用`eval`执行还是有区别的。

但是在这个`script`标签中插入了`$gwx`函数之后并没有执行这个函数。我们继续查找，看看`$gwx`函数是在哪里执行的，并且需要传入哪些参数。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a8d71e47db84dcda4291b1f47554663~tplv-k3u1fbpfcp-watermark.image)

在渲染层继续查找就找到了`$gwx`函数执行的地方，我们提取出来看一下。

```javscript
    var decodeName = decodeURI("./pages/index/index.wxml")
    var generateFunc = $gwx(decodeName)
```

这时候就知道了`$gwx`函数的参数正是`WXML`文件的业务路径。如上图是在。这里注意一下，为了后方讲解顺利，我们把`WCC`执行文件放在了`pages/logs`目录下，所以我们这里传入`$gwx`函数的路径是`./pages/logs/logs.wxml`。

我们在渲染层的控制台中执行一下`$gwx`函数并传入参数。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a8d95fd40674a84b292ca17459d42cd~tplv-k3u1fbpfcp-watermark.image)

可以看到执行$gwx函数后的返回值generateFunc也是一个函数。我们再次执行这个函数，看一下返回值。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d30a6b870a53437c852b3e46b7dd43a7~tplv-k3u1fbpfcp-watermark.image)

generateFunc函数的返回值是一颗树状结构，就是该页面wxml对应的js对象形式表示的dom树。

那为什么$gwx函数不直接返回虚拟DOM树呢，而是先返回generateFun？是因为需要注入动态数据。再看一下编译前的WXML文件结构。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfcce1fdf8174a6ca9bc7eab21c1f732~tplv-k3u1fbpfcp-watermark.image)

可以看到内部需要动态接受几个变量

- logs 是一个数组
- log 是个字符串
- index 循环索引

`generateFunc`函数在执行的时候需要动态注入这些数据，我们手动注入一些假数据看一下生成出来的虚拟DOM树与前面没有注入数据的有什么区别。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e02c6396c3d474cb721394e64cc5c0d~tplv-k3u1fbpfcp-watermark.image)

可以看到我们传入了动态数据logs，相对应的虚拟DOM树中也渲染出来了这条动态数据，在`wx-text`标记的children中可以看到，因为logs.wxml内部结构是一个循环，所以前面没有传递参数进去的时候，并没有渲染循环体内部的DOM节点。传进动态数据之后就可以看到循环体内部的DOM节点也被渲染了出来。

渲染出对应的虚拟DOM树之后，之后要干嘛呢？我们还得回到渲染层找一下`generateFunc`的位置，看看它之后干了什么。查找方法参照前面`control + f`方法。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/003c00d6dbbe43a4818173661db53a40~tplv-k3u1fbpfcp-watermark.image)

最开始的两行，是我们去探索`$gwx`函数作用的时候查找到的，并分析了一下，再次看到这段代码的时候我们已经知道了`generateFunc`就是接受动态数据，并生成虚拟DOM树的函数。

这段代码核心的区域就是CustomEvent,自定义事件，自定义事件在讲解Shadow DOM的时候重点提到过。在这里再次出现了，可以看到这里的逻辑如下:

- 如果没有有generateFun那么body标记内部展示 decodeName + "not found"，并输处错误日志
- 如果有，检查window或++global环境中自定义事件CustomEvent是否存在。CE及为CustomEvent缩写
- document.dispatchEvent 触发自定义事件 将generateFunc当作参数传递给底层渲染库，这里后边章节讲到，我也会与这里相互呼应。
- 在触发自定义事件的时候，添加当前时间节点，可以理解为生命周期`pageFrame_generateFunc_ready`。

这样完整的`virtualDOM`渲染流程就完毕了，猜一下后续，肯定有地方需要去接收这个ready状态，那么问题来了，动态数据是怎么传过来的？下个章节我们讲解小程序的通讯系统

