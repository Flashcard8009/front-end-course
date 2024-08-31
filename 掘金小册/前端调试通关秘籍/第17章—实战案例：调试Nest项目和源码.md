﻿Nest 是当下最流行的 Node.js 服务端框架，它建立在 Express 之上，实现了 IOC 的架构模式，并且对很多方案都有集成，比如 websocket、graphql 等。

当然，这节不是讲 Nest 的原理，而是讲如何调试 Nest 的项目，如何调试 Nest 的源码。

## 调试 Nest 项目

Nest 提供了快速创建项目的命令行工具 @nest/cli，首先全局安装它：

```
npm i -g @nestjs/cli
```

然后用 nest new nest-test 快速创建一个 nest 的项目。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a92393403d24398bf443a0cf8ad2497~tplv-k3u1fbpfcp-watermark.image?)

进入项目目录，执行 npm run start 就会启动服务：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3d79777b76d3473f9abeb894f8d6e7d4~tplv-k3u1fbpfcp-watermark.image?)

然后浏览器访问 http://localhost:3000 ，可以看到 Hello World，说明服务启动成功了。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c912a127bfeb4b97afecda4120676b2c~tplv-k3u1fbpfcp-watermark.image?)

然后我们创建一个 node 调试配置，指定 npm 为 runtime：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/282f609ec41644e0aed43601fc49d125~tplv-k3u1fbpfcp-watermark.image?)

这里 console 要设置为 integratedTerminal，这样日志会输出在 terminal，就和我们手动执行 npm run start 是一样的。

不然，日志会输出在 debug console。颜色啥的都不一样：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea50df2adb51414eb50d0ec85dcd4f50~tplv-k3u1fbpfcp-watermark.image?)

点击 Debug 启动：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8548eb619adc4db3bf4814c2d3a49a8d~tplv-k3u1fbpfcp-watermark.image?)

注意，用 deubg 方式跑之前要把之前起的服务关掉，不然端口会被占用。

或者也可以用这样的方式杀死占用哪个端口的进程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2a5aad9ff2241859232c0dcb6499dbd~tplv-k3u1fbpfcp-watermark.image?)

在 controller 打个断点：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0c16dafa5304155a31777b0d533abf8~tplv-k3u1fbpfcp-watermark.image?)

然后浏览器重新访问 http://localhost:3000

这时候代码就会在断点处断住：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/714839bc00f34476be0012174ba7fdcb~tplv-k3u1fbpfcp-watermark.image?)

这样就可以愉快的调试 Nest 项目了。

接下来我们再来调试下 Nest 源码。

## 调试 Nest 源码

其实调用栈里，我们的代码之前的部分，就是 Nest 框架的代码：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/710e83da3ee941fa983c363c96bb17d2~tplv-k3u1fbpfcp-watermark.image?)

但是这明显是编译后的代码：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c59ca63755994d7995355eaadae2d47d~tplv-k3u1fbpfcp-watermark.image?)

而我们是想调试 Nest 的 ts 源码的，这就需要用到 sourcemap 了。

从 npm registry 下载的包是没有 sourcemap 的代码，想要 sourcemap，需要自己 build 源码。

把 Nest 项目下载下来，并安装依赖（加个 --depth=1 是下载单 commit，--single-branch 是下载单个分支，这样速度会快很多）：

```
git clone --depth=1 --single-branch https://github.com/nestjs/nest
```
执行 npm run build，它会用 tsc 编译代码：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d489d0306b648aba18e3cb3a5bb4d84~tplv-k3u1fbpfcp-watermark.image?)

并且在 build 命令执行完之后会自动执行 postbuild：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74e417c52bd849bc86bed2bcf9277dde~tplv-k3u1fbpfcp-watermark.image?)

它做的事情就是把编译后的文件移动到 node_modules/@nestjs 目录下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfedccfc74e3406fae4b16bed4e6c2c4~tplv-k3u1fbpfcp-watermark.image?)

执行 npm run build，你就会在 node_modules/@nestjs 下看到这样的代码：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4d8cee11f2c43bfa2c1ef43309d1f26~tplv-k3u1fbpfcp-watermark.image?)

只包含了 js 和 ts，没有 sourcemap：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79e9d0095df14da5a87825a47863a9d6~tplv-k3u1fbpfcp-watermark.image?)

生成 sourcemap 需要改下 tsc 编译配置，也就是 packages/tsconfig.build.json 文件：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be584d47dc08439989bcb5a168ebeefe~tplv-k3u1fbpfcp-watermark.image?)

设置 sourceMap 为 true 也就是生成 sourcemap，但默认的 sourcemap 里不包含内联的源码，也就是 sourcesContent 部分，需要设置 inlineSources 来包含。

再次执行 npm run build，就会生成带有 sourcemap 的代码：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d988797ac7ea4b028eab5101fe79da36~tplv-k3u1fbpfcp-watermark.image?)

并且 sourcemap 是内联了源码的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea66020f83e34f03ae1325b6a94d05a1~tplv-k3u1fbpfcp-watermark.image?)

然后我们跑一下 Nest 的项目，直接跑 samples 目录下的项目即可，这是 Nest 内置的一些案例项目：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/903e5b83481846d883771b742d7d6160~tplv-k3u1fbpfcp-watermark.image?)

进入 cats-app 项目，安装依赖：

```
npm install
```

把根目录 node_modules 下生成的代码覆盖下 cats-app 项目的 node_modules 下的代码：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a247da392ce04a8bb5337cdc163faff6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1408&h=816&s=366457&e=png&b=fcfbfb)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48106b06653a4e39985d6a087bbbf586~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1420&h=722&s=355032&e=png&b=f5f3f2)

创建这样一个调试配置：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32f351652496424b9e235c2b48da9402~tplv-k3u1fbpfcp-watermark.image?)

```json
{
    "name": "Launch via NPM",
    "request": "launch",
    "runtimeArgs": [
        "run-script",
        "start"
    ],
    "runtimeExecutable": "npm",
    "console": "integratedTerminal",
    "cwd": "${workspaceFolder}/sample/01-cats-app/",
    "skipFiles": [
        "<node_internals>/**"
    ],
    "type": "node"
}
```
主要是要指定 cwd 为那个项目的目录，也就是在那个目录下执行 npm run start。

我们在 cats 的 controller 里打个断点：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5515626c54e42cd9a53847ac4989c56~tplv-k3u1fbpfcp-watermark.image?)

然后浏览器访问 http://localhost:3000/cats ，代码就会在断点处断住：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2de4930689a94ad3b27f6d5e5fc17148~tplv-k3u1fbpfcp-watermark.image?)

但是现在调用栈中的依然不是源码：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9218d22deaee451181286cef87bf07cf~tplv-k3u1fbpfcp-watermark.image?)

这是为什么呢？

这是因为这样一个调试配置 resolveSourceMapLocations：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb0ac6d362764a3796af7c983fc3fc90~tplv-k3u1fbpfcp-watermark.image?)

它的默认值是排除掉了 node_modules 目录的，也就是不会查找 node_modules 下的 sourcemap。

去掉那条配置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42de1380058e46f8910513c4e3ab1b9a~tplv-k3u1fbpfcp-watermark.image?)

再跑一下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6244c35bf8ad4609b3445dd824f9efb9~tplv-k3u1fbpfcp-watermark.image?)

这时候 sourcemap 就生效了，可以看到调用栈中显示的就是 Nest 的 ts 源码。

这样就达到了调试 Nest 源码的目的。

只不过现在 nest 的 ts 源码是只读的：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfc2d82ff4b54eddaad652f260bee667~tplv-k3u1fbpfcp-watermark.image?)

这是因为 sourcemap 到的路径不对，我们可以再去改下配置，修改生成的 sourcemap 的路径：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2bf4fc112384492bdf08e9a7adee632~tplv-k3u1fbpfcp-watermark.image?)

那为什么 sourcemap 到的路径不对呢？

看下 map 文件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d965020271dc4d9da225e0d3aee0bc3d~tplv-k3u1fbpfcp-watermark.image?)

sourcemap 到的路径是 sourceRoot + sources 文件名。

我们需要设置下 sourcesRoot。

nest 的这几个目录下都有单独的 tsconfig.build.json，需要分别设置：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58da205a7e8d4df794cedc89184e89f7~tplv-k3u1fbpfcp-watermark.image?)

比如我设置了 core 包下的 tsconfig.build.json 的 sourceRoot 为 core 包的绝对路径：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd21efc75cda4f3b9685de7b634e23cd~tplv-k3u1fbpfcp-watermark.image?)

再次 build，生成的 map 文件就是这样的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0373e62cdc7c48bdbd099288c16f619b~tplv-k3u1fbpfcp-watermark.image?)

重新跑下调试：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ce213abb14741b7931ba9b40a1203c0~tplv-k3u1fbpfcp-watermark.image?)

现在的路径就对了，内容不再只读，而且点击可以直接跳到源码文件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9e0a80e214d4784a548edaa5503c9cf~tplv-k3u1fbpfcp-watermark.image?)

至此，我们就可以愉快的调试 Nest 源码了。

## 总结

Nest 是流行的 node 服务端框架，这节我们实战了下 Nest 项目的调试和源码的调试。

调试 Nest 项目就是用 npm 的方式启动调试配置，指定下 console 为 terminal 就可以了。

调试 Nest 源码的话，需要把 Nest 源码下载下来，build 出一份带有 sourcemap 版本的代码，同时还要设置 resolveSourcemapLocations 去掉排除 node_modules 的配置，然后再调试，就可以直接调试 Nest 的 ts 源码了。
