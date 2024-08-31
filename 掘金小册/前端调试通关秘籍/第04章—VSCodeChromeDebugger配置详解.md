﻿上节我们学会了如何用 VSCode Debugger 调试网页的 JS，其实它还有很多有用的配置项，这节我们一起来过一遍：

首先，调试配置文件不用自己创建，可以直接点击 Debug 窗口的 `create a launch.json file` 快速创建：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00039d6dbc2b41c3bcad8a9d3c7942bc~tplv-k3u1fbpfcp-watermark.image?)

## launch/attach

创建 Chrome Debug 配置有两种方式：launch 和 attach：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9eb056709f19465b99ba6461a3793f76~tplv-k3u1fbpfcp-watermark.image?)
 
它们只是 request 的配置不同：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d448792e02224ebab00592c1c157d5c9~tplv-k3u1fbpfcp-watermark.image?)

我们知道，调试就是把浏览器跑起来，访问目标网页，这时候会有一个 ws 的调试服务，我们用 frontend 的 ws 客户端连接上这个 ws 服务，就可以进行调试了。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab654aeeb1b44368a4ebb11a47aabcf1~tplv-k3u1fbpfcp-watermark.image?)

VSCode 的 Debugger 会多一层适配器协议的转换，但是原理差不多。

launch 的意思是把 url 对应的网页跑起来，指定调试端口，然后 frontend 自动 attach 到这个端口。

但如果你已经有一个在调试模式跑的浏览器了，那直接连接上就行，这时候就直接 attach。

比如我们手动把 Chrome 跑起来，指定调试端口 remote-debugging-port 为 9222，指定用户数据保存目录 user-data-dir 为你自己创建一个目录。

在命令行执行下面的命令：
```
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222 --user-data-dir=你自己创建的某个目录
```

Chrome 跑起来之后，你可以打开几个网页，比如百度、掘金，然后你访问 localhost:9222/json，这时候你就会发现所有的 ws 服务的地址了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34b33f777f6f440aad6d178d27059841~tplv-k3u1fbpfcp-watermark.image?)

为什么每个页面有单独的 ws 服务呢？

这个很正常呀，每个页面的调试都是独立的，自然就需要单独的 ws 服务。

然后你创建一个 attach 的 Chrome Debug 配置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ccee7b4e00f34a6caca16b29e90e3cd6~tplv-k3u1fbpfcp-watermark.image?)

点击启动，就会看到 VSCode Debugger 和每一个页面的 ws 调试服务建立起了链接：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8746bd0c22a84b72b8ff35b9e09c6e5d~tplv-k3u1fbpfcp-watermark.image?)

比如访问之前的 React 项目，就可以进行调试了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/941051639fa242499cf4be34d486c21e~tplv-k3u1fbpfcp-watermark.image?)

可以多个页面一起调试，每个页面都有独立的调试上下文。

## userDataDir

不知道你有没有注意到刚才手动启动 Chrome 的时候，除了指定调试端口 remote-debugging-port 外，还指定了用户数据目录 user-data-dir。

为什么要指定这个呢？

user data dir 是保存用户数据的地方，比如你的浏览记录、cookies、插件、书签、网站的数据等等，在 macOS 下是保存在这个位置：

```
~/Library/Application\ Support/Google/Chrome
```

比如你打开 Default/Bookmarks 看一下，是不是都是你保存的书签？
```
open ~/Library/Application\ Support/Google/Chrome/Default/Bookmarks
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc9f337a6af24d23a48fb734050dc402~tplv-k3u1fbpfcp-watermark.image?)

你还可以删掉 Deault/Cookies，之后再访问之前登陆过的网站试一下，是不是都需要登录了？

这就是用户数据目录的作用。

那为什么启动 Chrome 要手动指定这个呢？都用默认的不行么？

用户数据目录有个特点，就是只能被一个 Chrome 实例所访问，如果你之前启动了 Chrome 用了这个默认的 user data dir，那就不能再启动一个 Chrome 实例用它了。

如果用户数据目录已经跑了一个 Chrome 实例，再跑一个候会报这样的错误：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7bf00421bea44de9774c429698685f6~tplv-k3u1fbpfcp-watermark.image?)

所以我们用调试模式启动 Chrome 的时候，需要单独指定一下 user data dir 的位置。或者你也把之前的 Chrome 实例关掉，这样才能用默认的。

launch 的配置项里也有 userDataDir 的配置：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cb3e64b2f2e496d83531b587af42b2b~tplv-k3u1fbpfcp-watermark.image?)

默认是 true，代表创建一个临时目录来保存用户数据。

你也可以设置为 false，使用默认 user data dir 启动 chrome。

这样的好处就是登录状态、历史记录啥的都有：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8174d69349814d39b02884754834b856~tplv-k3u1fbpfcp-watermark.image?)

把 userDataDir 设置为 true 就每次都需要登录了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71087ca612904d0f97cf6c2186747de3~tplv-k3u1fbpfcp-watermark.image?)

你也可以指定一个自定义的路径，这样用户数据就会保存在那个目录下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cc4e9bcf67e47b7a1a079a4d7419bf4~tplv-k3u1fbpfcp-watermark.image?)

更重要的是，你安装的 React DevTools、Vue DevTools 插件都是在默认用户数据目录的，要是用临时数据目录跑调试，那这些不都没了？

比如你 userDataDir 设置为 true 的时候，React DevTools 插件是没有的，需要再安装：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a0ff3234335421fbf1de1634c5a6966~tplv-k3u1fbpfcp-watermark.image?)

userDataDir 设置为 false 的时候，安装过的插件都可以直接用：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6679f7e4cb414288a137a9246c4defc8~tplv-k3u1fbpfcp-watermark.image?)

但是除了调试用之外，平时也会用到 Chrome 呀，同一个 user data dir 只能跑一个 Chrome 实例的话，那不就冲突了？

这个问题可以用下面的配置解决：

## runtimeExecutable

调试网页的 JS，需要先把 Chrome 跑起来，默认跑的是 Google Chrome，其实它还有另外一个版本 Canary：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34dcb4210fcc43a99ab7f1bf1827cb4b~tplv-k3u1fbpfcp-watermark.image?)

这是给开发者用的每日构建版，能够快速体验新特性，但是不稳定。

而 Google Chrome 是给普通用户用的，比较稳定。

这俩是独立的，相互之间没影响，可以都用同一个 user data dir 来启动。

你可以在[官网](https://www.google.com/intl/zh-CN/chrome/canary/)把 canary 下载下来。

然后指定 runtimeExecutable 为 canary，使用默认的用户数据目录启动：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff15afb548584879a545373721cb5df6~tplv-k3u1fbpfcp-watermark.image?)


![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66cbf73a76864dadb521d9b6a059d081~tplv-k3u1fbpfcp-watermark.image?)

这样你就可以调试用 canary，平时用 chrome 了，两者都有各自的默认数据目录。

（注意，一定要先安装了 canary，才能指定 canary 跑）

当然，runtimeExecutable 还可以指定用别的浏览器跑：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f5594c11a60748a5bcbb55e1f0c7136c~tplv-k3u1fbpfcp-watermark.image?)

可以是 stable，也就是稳定的 Google Chrome，或者 canary，也就是每日构建版的 Google Chrome Canary，还可以是 custom，然后用 CHROME_PATH 环境变量指定浏览器的地址。

不过常用的还是 Chrome 和 Canary。

### runtimeArgs

启动 Chrome 的时候，可以指定启动参数，比如每次打开网页都默认调起 Chrome DevTools，就可以加一个 --auto-open-devtools-for-tabs 的启动参数：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21f888286ffe45398cfec5a23ae8a7ea~tplv-k3u1fbpfcp-watermark.image?)

效果就是这样的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4502d4fa52041a7bf4c477ca850de76~tplv-k3u1fbpfcp-watermark.image?)

想要无痕模式启动，也就是不加载插件，没有登录状态，就可以加一个 --incognito 的启动参数：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3497609cca8c4648bec7e01d68020894~tplv-k3u1fbpfcp-watermark.image?)

调试用的浏览器就会以无痕模式启动了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69c9e34e897c46fbb1414e399769c25d~tplv-k3u1fbpfcp-watermark.image?)

其实我们设置的 userDataDir 就是指定了 --user-data-dir 的启动参数。

## sourceMapPathOverrides

代码是经过编译打包然后在浏览器运行的，比如这样：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02702be2b689431695c92ae00107463a~tplv-k3u1fbpfcp-watermark.image?)

但我们却可以直接调试源码，这是通过 sourcemap 做到的。

调试工具都支持 sourcemap，并且是默认开启的：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca7f08796cca40c7925648f12aa2a746~tplv-k3u1fbpfcp-watermark.image?)

当然也可以关掉:

Chrome DevTools 里这么关（command + shift + p）：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c234eab021e943c5a5b7c8e1b68292d9~tplv-k3u1fbpfcp-watermark.image?)

VSCode Debugger 这么关：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ccd5e07eaa4420484ebbae36660cd7c~tplv-k3u1fbpfcp-watermark.image?)

这样调试的就是编译后的代码了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11c46b323caf45f2b1151d545f3f93f3~tplv-k3u1fbpfcp-watermark.image?)

在开启 sourcemap 的情况下，用 Chrome DevTools 可以看到，源文件的路径是 /static/js/bundle.js

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80e2fc692c6e4f97bf2cf26e917daa79~tplv-k3u1fbpfcp-watermark.image?)

被 sourcemap 到了 /Users/guang/code/test-react-debug/src/index.js

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0fd237cbe82944d6a6db4b4b44f953fd~tplv-k3u1fbpfcp-watermark.image?)

而在 VSCode 里，这个路径是有对应的文件的，所以就会打开对应文件的编辑器，这样就可以边调试边修改代码。

但有的时候，sourcemap 到的文件路径在本地里找不到，这时候代码就只读了，因为没有地方保存：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf78d4af25f7459c956d5164cc6b2b2c~tplv-k3u1fbpfcp-watermark.image?)

这种情况就需要对 sourcemap 到的路径再做一次映射：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/841aabf5f3754dd8a8cfe8bc225aa393~tplv-k3u1fbpfcp-watermark.image?)

通过 sourceMapPathOverrides 这个配置项。

默认有这么三个配置：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af692dc83a8242fa83629058048eae5c~tplv-k3u1fbpfcp-watermark.image?)

分别是把 meteor、webpack 开头的 path 映射到了本地的目录下。

其中 ?:* 代表匹配任意字符，但不映射，而 * 是用于匹配字符并映射的。

比如最后一个 webpack://?:\*/\* 到 ${workspaceFolder}/\* 的映射，就是把 webpack:// 开头，后面接任意字符 + / 然后是任意字符的路径映射到了本地的项目目录。（workspaceFolder 是一个内置变量，代表项目根目录）

把调试的文件 sourcemap 到的路径映射到本地的文件，这样调试的代码就不再只读了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/695d9b4a95ad4ed38c49a9165252d40a~tplv-k3u1fbpfcp-watermark.image?)

## file

除了启动开发服务器然后连上 url 调试之外，也可以直接指定某个文件，VSCode Debugger 会启动静态服务器提供服务：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/439210ed856d4579a0da178b6bcbe846~tplv-k3u1fbpfcp-watermark.image?)

index.html 的内容如下：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/995bc01c5dd844b486516443c2b601cf~tplv-k3u1fbpfcp-watermark.image?)

打了个断点，然后启动调试：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee44c176af7743559ff9f4d03feb7d5c~tplv-k3u1fbpfcp-watermark.image?)

这样就可以直接调试静态网页了。

同样，要修改调试的内容需要把 url 映射到本地文件才行，所以有这样一个 pathMapping 的配置：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/105d794394d040bda13061eb02ef067b~tplv-k3u1fbpfcp-watermark.image?)

webRoot 其实就相当于把 / 的 url 映射到了 ${workspaceFoder}/。
 
这些配置倒是很少用，一般我们还是启动 dev server，再调试某个 url 更多一些。

## 总结

这节我们过了一遍 VSCode Chrome Debugger 的配置：

- launch：调试模式启动浏览器，访问某个 url，然后连上进行调试
- attach：连接某个已经在调试模式启动的 url 进行调试
- userDataDir： user data dir 是保存用户数据的地方，比如浏览历史、cookie 等，一个数据目录只能跑一个 chrome，所以默认会创建临时用户数据目录，想用默认的目录可以把这个配置设为 false
- runtimeExecutable：切换调试用的浏览器，可以是 stable、canary 或者自定义的
- runtimeArgs：启动浏览器的时候传递的启动参数
- sourceMapPathOverrides：对 sourcemap 到的文件路径做一次映射，映射到 VSCode workspace 下的文件，这样调试的文件就可以修改了
- file：可以直接指定某个文件，然后启动调试

这些配置后面调试的时候经常会用到，要理解它们的含义。



