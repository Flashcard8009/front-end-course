﻿很多同学不知道为什么要用 debugger 来调试，console.log 不行么？

还有，会用 debugger 了，还是有很多代码看不懂，如何调试复杂源码呢？

这篇文章就来讲一下为什么要用这些调试工具：

## console.log vs Debugger

相信绝大多数同学使用 console.log 调试的，把想看的变量值打印在控制台。

这样能满足需求，但是遇到对象的打印就不行了。

比如我想看 webpack 源码里的 compilation 对象的值，我打印了一下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e46f250b62e44aefa82525cb61577b39~tplv-k3u1fbpfcp-watermark.image?)

但你会发现对象的值也是对象的时候不会展开，而是打印一个 [Object] [Array] 这种字符串。

更致命的是打印的太长会超过缓冲区的大小，terminal 里会显示不全：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02b36f283e3946b4a0a22eee0f07adc0~tplv-k3u1fbpfcp-watermark.image?)

而你用 debugger 来跑，在这里打个断点来看就没这些问题了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66399947ea6b45289c8d77b6d4568cc5~tplv-k3u1fbpfcp-watermark.image?)

有的同学可能会说，那打印一个简单的值的时候用 console.log 还是很方便呀。

比如这样：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef9425410b784a7dbecbe63a73b76f6e~tplv-k3u1fbpfcp-watermark.image?)

真的么？

那还不如用 logpoint：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/984309e3bd42450899107ce25924e251~tplv-k3u1fbpfcp-watermark.image?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42d87e8ab5614b9093274400819cb9f8~tplv-k3u1fbpfcp-watermark.image?)

代码执行到这里就会打印：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08edc583fe6744649ec4a1ac92b1c0f0~tplv-k3u1fbpfcp-watermark.image?)

而且没有污染代码，用 console.log 的话调试完之后这个 console 不也得删掉么？

但是 logpoint 不用，它就是个断点的设置，不在代码里。

当然，最重要的是 Debugger 调试是可以看到调用栈和作用域的！

首先是调用栈，它就是代码的执行路线。

比如这个 App 的函数组件，你可以看到渲染这个函数组件会经历 workLoop、beginWork、renderWithHooks 这些流程：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72588b94a4c54cb9b2fb6cf9d032d66d~tplv-k3u1fbpfcp-watermark.image?)

你可以点开调用栈的每一帧看下都执行了啥逻辑，用到啥数据。比如可以看到这个函数组件的 fiber 节点：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/316982c49ddf4ef28c45ffa672fdb449~tplv-k3u1fbpfcp-watermark.image?)

再就是作用域，点击每一个栈帧就可以看到每个函数的作用域中的变量：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2809ab2ad48e489aa49ccf7c92730921~tplv-k3u1fbpfcp-watermark.image?)

用 Debugger 可以看到代码的执行路径，每一步的作用域信息。而你用 console.log 呢？

只能看到那个变量值而已。

拿到的信息量的差距不是一点半点，调试时间长了，别人会对代码的运行流程越来越清晰，而你用 console.log 呢？还是老样子，因为你看不到代码执行路径。

所以，不管是调试库的源码还是业务代码，不管是调试 Node.js 还是网页，都推荐用 Debugger 打断点，别再用 console.log 了，就算想打印日志，也可以用 LogPoint。

而且在排查问题的时候，用 Debugger 的话可以加一个异常断点，代码跑到抛异常的地方就会断住：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/312a53cf4d2746c68e5eae18e57c2271~tplv-k3u1fbpfcp-watermark.image?)

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8916e0b5bcae42ed99bc72a74ad1dc5c~tplv-k3u1fbpfcp-watermark.image?)

可以看到调用栈来理清出错前都走了哪些代码，可以通过作用域来看到每一个变量的值。

有了这些东西，排查错误不就很轻松了么！

而你用 console.log 呢？

啥也没，只能自己猜。

## Performance

前面说 Debugger 调试可以看到一条代码的执行路径，但是代码的执行路径往往比较曲折。

比如那个 React 会对每个 fiber 节点做处理，每个节点都会调用 beginWork。处理完之后又会处理下一个节点，再次调用 beginWork：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a78b0013128241cdbad5c765d7334920~tplv-k3u1fbpfcp-watermark.image?)

就像你走了一条小路，然后回到大路之后又走了另一条小路，用 Debugger 只能看到当前这条小路的执行路径，看不到其他小路的路径：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c99e014477e4331b15aba9c58d3da24~tplv-k3u1fbpfcp-watermark.image?)

这时候就可以结合 Performance 工具了，用 Performance 工具看到代码执行的全貌，然后用 Debugger 来深入每一条代码执行路径的细节。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/919cad7201574741a798522e4c6eeceb~tplv-k3u1fbpfcp-watermark.image?)

## SourceMap

sourcemap 很重要，因为我们执行过的都是编译打包后的代码，基本是不可读的，调试这种代码也没啥意义，而 sourcemap 可以让我们直接调试最初的源码。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f172c75c3bea4bfb81e4a8018b1cde19~tplv-k3u1fbpfcp-watermark.image?)

比如 vue，关联了 sourcemap 之后，我们能直接调试 ts 源码：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1eb7c86fa08c44218e74ae5888d0550d~tplv-k3u1fbpfcp-watermark.image?)

nest.js 也是：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78694dcfbb184003a114c523962565ea~tplv-k3u1fbpfcp-watermark.image?)

不用 sourcemap 的话，想搞懂源码，但你调试的是编译后的代码，怎么读懂呢？

## 读懂一行

前面说的 Debugger、Performance、SourceMap 只是调试代码的工具，那会了调试工具，依然读不懂代码怎么办呢？

我觉得这是不可能的。

为什么这么说呢？

就拿 react 源码来说：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/85996569a6984f82bac6d26d529ba78b~tplv-k3u1fbpfcp-watermark.image?)

switch case 能读懂吧。三目运算符能读懂吧。函数调用能读懂吧。

每一行代码都能读懂，而全部的代码不就是由这一行行代码组成的么？

加上我们可以单步执行来知道代码执行路径。

为啥每行代码都能读懂，连起来就读不懂了呢？

那应该是代码太多了，而你花的时间不够而已。

先要读懂一行，一个函数，读懂一个小功能的实现流程，慢慢积累，之后了解的越来越多之后，你能读懂的代码就会越多。

## 总结

这篇文章讲了为什么要用调试工具，如何读懂复杂代码。

console.log 的弊端太多了，大对象打印不全，会超过 terminal 缓冲区，对象属性不能展开等等，不建议大家用。就算要打印也可以用 LogPoint。

用 Debugger 可以看到调用栈，也就是代码的执行路径，每个栈帧的作用域，可以知道代码从开始运行到现在都经历了什么，而 console.log 只能知道某个变量的值。

此外，报错的时候也可以通过异常断点来梳理代码执行路径来排查报错原因。

但 Debugger 只能看到一条执行路径，可以用 Performance 录制代码执行的全流程，然后再结合 Debugger 来深入其中一条路径的执行细节。

此外，只有调试最初的源码才有意义，不然调试编译后的代码会少很多信息。可以通过 SourceMap 来关联到源码，不管是 Vue、React 的源码还是 Nest.js、Babel 等的源码。

会了调试之后，就能调试各种代码了，不存在看不懂的源码，因为每一行代码都是基础的语法，都是能看懂的，如果看不懂，只可能是代码太多了，你需要更多的耐心去读一行行代码、一个个函数、理清一个个功能的实现，慢慢积累就好了。

掌握基于 Debugger、Performance、SourceMap 等调试代码之后，各种网页和 Node.js 代码都能调试，各种源码都能读懂！

