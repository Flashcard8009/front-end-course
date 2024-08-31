﻿每个前端项目都有 npm scripts，我们会用 npm scripts 来组织编译、打包、lint 等任务。

大家可能经常会跑 npm scripts，但却对这些命令行工具是怎么实现的并不了解。

那如果想了解这些工具的实现原理，应该怎么做呢？

这就是今天的主题：调试 npm scripts。

这些命令行工具的 package.json 里都会有个 bin 字段，来声明有哪些命令：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ac946a9135341dcb8b3f4833ee9abe8~tplv-k3u1fbpfcp-watermark.image?)

npm install 这个包以后，就会放到 node_modules/.bin 目录下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db96c12836cd4515928fc309fd9e4523~tplv-k3u1fbpfcp-watermark.image?)

这样我们就可以通过 node ./node_modules/.bin/xx 来跑不同的工具了。

我们也可以用 npx 来跑，比如 npx xx，它的作用就是执行 node_modules/.bin 下的本地命令，如果没有的话会从 npm 下载然后执行。

当然，最常用的还是放到 npm scripts 里：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a13e78a1329488b98d853860a18800c~tplv-k3u1fbpfcp-watermark.image?)

这样就直接 npm run xxx 跑就行了。

npm scripts 本质上还是用 node 来跑这些 script 代码，所以调试他们和调试其他 node 代码没啥区别。

也就是可以这样跑：

在 .vscode/launch.json 的调试文件里，选择 node 的 launch program：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c72b13cd00148adbe46a4f5881695b0~tplv-k3u1fbpfcp-watermark.image?)

用 node 执行 node_modules/.bin 下的文件，传入参数即可：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1f32ee8885b47f384c23a6092712061~tplv-k3u1fbpfcp-watermark.image?)

其实还有更简单的方式，VSCode Debugger 对 npm scripts 调试的场景做了封装，可以直接选择 npm 类型的调试配置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2074cdd80624e5aba326dc3f2b6f4ee~tplv-k3u1fbpfcp-watermark.image?)

直接指定运行的命令即可：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5140dc65ee8f4f17979f804b59c5c350~tplv-k3u1fbpfcp-watermark.image?)

比如我们就用这个 create-react-app 创建的 react 项目来尝试下 npm scripts 的调试：

先去 node_modules/.bin 下这个文件里打个断点：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a043d37313b847f590bc734df0ca62f8~tplv-k3u1fbpfcp-watermark.image?)
 
然后点击 debug 启动：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be35965e992d4cf1b892d09348fd2f19~tplv-k3u1fbpfcp-watermark.image?)

你会发现会执行 scripts 下的 start 模块：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df14e9c0db814185ba5c13c7461c3026~tplv-k3u1fbpfcp-watermark.image?)

我们再去 start 下打个断点：

代码执行到这里断住：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/394d4dd4c2554591a2e571ea23176c1a~tplv-k3u1fbpfcp-watermark.image?)

这个 config 就是 webpack 的配置：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bcd54d52a4524b8382e73c898da39771~tplv-k3u1fbpfcp-watermark.image?)

再往下走，会发现启动了一个 server：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3d9572a0b7c407586db4125018e265b~tplv-k3u1fbpfcp-watermark.image?)

我们在 server 启动的回调函数里打个断点，看看浏览器是怎么打开的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0159a511a17a4e0ab1c04eb052c20ff3~tplv-k3u1fbpfcp-watermark.image?)

点击 step into 进入这个断点：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a294c078d9c47fb9931870336583374~tplv-k3u1fbpfcp-watermark.image?)

然后单步执行，会走到这样的代码：

依次通过 osascript 来启动这些浏览器，启动失败的话，try catch 里直接忽略了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0d6513e0cce4d3da68c5220328d7a94~tplv-k3u1fbpfcp-watermark.image?)

这些浏览器 hover 上去就可以看到：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/110440a341e94c148fc187e59b4a46c3~tplv-k3u1fbpfcp-watermark.image?)

释放断点，你就会发现浏览器打开了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f345353970dd48a199f8408bb6a0ae33~tplv-k3u1fbpfcp-watermark.image?)

这样，我们不就梳理了一遍 react-scripts start 的流程么？

总结一下就是这样的：

- 根据输入的 start 命令，执行 scripts/start 模块
- 根据配置，创建 webpack 的 Compiler 对象
- 创建 WebpackDevServer
- server 启动之后，启动浏览器打开 url
- 打开 url 的实现就是通过 osascripts 依次尝试那些浏览器

这样调试完一遍，我们就对 npm run start 有了更深入的认识。

而且，调试的方式跑 script 和直接命令行 npm run start 没啥区别。

要说区别，唯一的区别可能就是这个：

默认调试模式下，输出的内容会在 Debug Console 面板显示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0bc33366866c4909ab7d61ea1d9f677d~tplv-k3u1fbpfcp-watermark.image?)

但这个也可以改：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbe36aff315c4fec80e4704a3e5ecaff~tplv-k3u1fbpfcp-watermark.image?)

可以切换成 integratedTerminal，那就会输出在 terminal 了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70d88292908d43958b06240061c80132~tplv-k3u1fbpfcp-watermark.image?)

这样就和平时 npm run start 执行没了任何区别，而且还可以断点调试，它不香么？

我们再来看个例子，比如 vue cli 创建的 vue 项目，在 vue.config.js 里可以改 webpack 配置：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e8bf7e2f931462ca51646dcc9dc6fe3~tplv-k3u1fbpfcp-watermark.image?)

但如果你想知道默认的配置是啥呢？console.log 么？

console.log 打印大对象可不是个好主意，它是这样的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/889c3959499c44cf91990a354b41d007~tplv-k3u1fbpfcp-watermark.image?)

有的同学说用 JSON.stringify，那个更难看，特别长的一串。

如果你会了调试 npm scripts 呢？

你就可以加一个 npm 类型的调试配置：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f08b28861324fc78b92ed1ac42d4bc6~tplv-k3u1fbpfcp-watermark.image?)

然后打个断点，debug 来跑：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1092e937f79a4eaa97c063a5e2b59c62~tplv-k3u1fbpfcp-watermark.image?)

啥配置都能看到，这不香么？

我举的例子只是 webpack 的，但其余的 npm scripts，比如 babel、tsc、eslint、vite 等等都是一样的调试方式。

## 总结

每个项目都有 npm scripts，大多数人只是用它们而不会调试它们，所以就算每天用也不知道这些工具的原理。

这些命令行工具都是在 package.json 中声明一个 bin 字段，然后 install 之后就会放到 node_modules/.bin 下。

可以 node node_modules/.bin/xx 来跑，可以 npx xx 来跑，最常用的还是 npm scripts，通过 npm run xx 来跑。

npm scripts 的调试就是 node 的调试，只不过 VSCode Debugger 做了简化，可以直接创建 npm 类型的调试配置。

把 console 配置为 integratedTerminal 之后，日志会输出到 terminal，和平时直接跑 npm run xx 就没区别了。而且还可以断点看看执行逻辑。

跑 npm scripts 之余，还可以理一下它的实现逻辑，哪里感兴趣断在哪里，这不比直接跑香么？