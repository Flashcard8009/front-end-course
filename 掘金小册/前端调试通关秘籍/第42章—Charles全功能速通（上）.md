﻿Charles 做为主流的代理工具，有很多强大好用的功能，这两节我们就快速过一遍：

没安装 charles 的话可以在[官网](https://www.charlesproxy.com/download/)下载：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/efcd0a5442e54a428a8f373e4684bf62~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1950&h=1212&s=491249&e=png&b=fefefe)
## charles 代理的两种配置方式

网页默认是走系统代理。

可以把 Charles 也设置为系统代理，点击 Proxy > macOS Proxy 即可：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96f0bf72bd07432e9bcfb649a354c337~tplv-k3u1fbpfcp-watermark.image?)

这样就能抓到网页的数据包了。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e357e3105c34eca8d192985eb7388bd~tplv-k3u1fbpfcp-watermark.image?)

或者不把 charles 设置为系统代理，而是 SwitchyOmega 单独指定网页的代理服务为 charles：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3bb9f6a5fbab49c7b4373d606e135d0b~tplv-k3u1fbpfcp-watermark.image?)

这里的代理服务的端口可以在 Proxy > proxy Settings 里看到：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31152db5ac7b4dee823f5af14ae7c2ae~tplv-k3u1fbpfcp-watermark.image?)

当然，更好的方式是 auto switch，也就是根据条件自动切换。

也就是我们在选项里配置的情景模式：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6a4ba15a66541919569c94804255018~tplv-k3u1fbpfcp-watermark.image?)

这两种还是很容易理解的：

- 默认都是走系统代理，所以把 charles 配置为系统代理服务器就可以抓包了
- 网页也可以通过 SwitchyOmega 单独指定代理服务器，这时候请求会直接走 charles

此外，如果 charles 抓不到 localhost 的包，那就把 charles 设置为代理，然后http://localhost:8080 换成 http://local.charles:8080 就好了。

## no caching

可以让 http 请求禁用缓存，在 Tools > No Caching 里开启：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff92116504414a6399eb1444c884bd21~tplv-k3u1fbpfcp-watermark.image?)

当开启之后，你会发现 header 会带上 cache-control: no-cache：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4afa964a06043909fd8350bb43ad3af~tplv-k3u1fbpfcp-watermark.image?)

no-cache 的意思是会用本地的缓存，但每次都协商。

而不开启的时候，header 里没有这个：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ba236f335e74e88a6dbf296229c01ff~tplv-k3u1fbpfcp-watermark.image?)

这就是禁用缓存的实现原理。

其实浏览器的强制刷新也是这么实现的，你可以强刷一下，抓包看看 header。

除了全部请求禁用缓存外，还可以单独指定某些 url 禁用缓存：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0ff8a5be3f54dd6be25eaae4deeacba~tplv-k3u1fbpfcp-watermark.image?)

## block cookies

这个也很容易理解，就是禁用 cookie 和 set-cookie 的 header：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/755921ab77a64fca98d80e1ffade570d~tplv-k3u1fbpfcp-watermark.image?)

在 Chrome DevTools 的 Network 里开启这两个 header 的展示：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f815f0593bd40859dd03dfda4f29aa4~tplv-k3u1fbpfcp-watermark.image?)

在开启 block cookies 之前，是有请求携带或设置 cookie 的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f858b768d19434c931bfeac700f8de7~tplv-k3u1fbpfcp-watermark.image?)

开启 block cookies 之后，你会发现所有的 set-cookie 都没有了：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35e28cf1d5834c55bb0680ce81b1070a~tplv-k3u1fbpfcp-watermark.image?)

但请求还是会带上 cookie，这是因为还没到 charles 那层。

同样的请求，到 charles 看下，就会发现没有 cookie 了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/420b3ccd1f6440c1b7c1b356f6f4a358~tplv-k3u1fbpfcp-watermark.image?)

关闭 block cookies 之后又有了：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/123261719502498ca445ea782ffa5cb9~tplv-k3u1fbpfcp-watermark.image?)

同样，它也可以单独 block 某些 url：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc9ae2ce85984059b8c944fc017d3f2c~tplv-k3u1fbpfcp-watermark.image?)

## map remote

charles 可以把请求转发给另一个 url，用 Tools > map remote 的功能：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7f3fc3d1c5f4b9882ddba1b940b9e19~tplv-k3u1fbpfcp-watermark.image?)

比如把 https://juejin.cn 转发到 https://www.baidu.com

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/264b32a748d54c4e82cb37b1bca609c0~tplv-k3u1fbpfcp-watermark.image?)

除了在 Tools > map remote 里开启之外，也可以右键 url 来开启：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21b8b7dd853c49ecbfa416119e5fe006~tplv-k3u1fbpfcp-watermark.image?)

好处是请求信息不用自己填一遍了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ee1ab2ad3dd458d9793e6731f4ec208~tplv-k3u1fbpfcp-watermark.image?)

之后也可以在 Tools > map remote 里管理：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c6a8ca103404b54bb33136fe3ae0996~tplv-k3u1fbpfcp-watermark.image?)

## map local

可以把请求转发给另一个 url，也同样可以读取本地的文件内容作为响应，也就是 map local。

在 Tools > map local 里，或者右键请求都可以设置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e0f7081b16143ebba844db477c6403e~tplv-k3u1fbpfcp-watermark.image?)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f64a1f3ef1e443ea0bfe12aacaaf0f4~tplv-k3u1fbpfcp-watermark.image?)

比如把这个 json 文件的请求换成本地的一个 svg 文件：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e7e58e4944142a2be3dc144d3786fd1~tplv-k3u1fbpfcp-watermark.image?)

map 之前是这样：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0bd0cf777fc34f398abfcf2338277596~tplv-k3u1fbpfcp-watermark.image?)

map 之后就是这样了：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67eb4d28a5a44510b37e4213dfbc5aae~tplv-k3u1fbpfcp-watermark.image?)

**map local 功能在移动端开发的时候还是挺常用的，比如把请求的 js、css 等换成本地刚打包出来的，就可以在线上环境调试本地代码。**

## Mirror

mirror 可以把响应内容保存在本地：

在 Tools > Mirror 里开启 mirror 功能：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0462dbf2f4fb4bb290c9986339c52b27~tplv-k3u1fbpfcp-watermark.image?)

指定保存的位置，之后再访问 url，就会把响应内容保存下来：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1025c5dcd6c14c049857e32459e5ccfe~tplv-k3u1fbpfcp-watermark.image?)

当然，我们没必要全部保存，也可以开启只 mirror 选择的 url，然后指定 url：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cbd4a9f56074b3d8cdac27668ea6fb4~tplv-k3u1fbpfcp-watermark.image?)

之后在保存的目录下就可以看到该请求的响应数据的文件：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31597e1737c24615891b01d1e02a4e20~tplv-k3u1fbpfcp-watermark.image?)

但是这个内容是不能修改的，每次刷新都会覆盖，这也是 mirror 镜像的含义。

但是可以配合 map local 使用，比如把这个响应保存下来，然后再复制到别的地方修改之后再 map local。

## rewrite

除了转发请求外，自然也支持对请求做修改，也就是 rewrite 功能。

在 Tools > rewrite 里开启：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/086cc940e898434684b8bb54a88bcbd7~tplv-k3u1fbpfcp-watermark.image?)

支持修改的包括 header、host、path、url、query，response status、body，也就是所有的内容都可以改：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/faa58f60ebbc41f8b16860d3d053cb2d~tplv-k3u1fbpfcp-watermark.image?)

比如我添加一个 query param：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c9be14a8a8b43a2bc4f853b7d71b4f7~tplv-k3u1fbpfcp-watermark.image?)

效果如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00cbfa3748e4421e84ca4ec7ed4d68d2~tplv-k3u1fbpfcp-watermark.image?)

这里操作的时候，如果希望 charles 一直在最上面，可以在 charles > preferences 里开启 always on top：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7b47b9420364d5eb9173829d19c911d~tplv-k3u1fbpfcp-watermark.image?)

它就一直在上面了：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f910940cdc146e4aa3b1220669104aa~tplv-k3u1fbpfcp-watermark.image?)

## block list 

bloack list 可以禁止一些 url 的请求：

在 Tools > block list 里：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6e4bed142e84226b80243a635c562ba~tplv-k3u1fbpfcp-watermark.image?)

处理方式可以是 断开连接或者返回 403。

当然，也可以右键请求来把它加入到 block list：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8aa9c93a17b8402db2e1227913832eeb~tplv-k3u1fbpfcp-watermark.image?)

比如当设置 block 的 url 的时候：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88212fcc464343dab80a53b250db96ed~tplv-k3u1fbpfcp-watermark.image?)

效果是这样的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae23c441098d4ea1abac74a62213769a~tplv-k3u1fbpfcp-watermark.image?)

当你需要模拟请求失败的时候，可以用这个功能。

## allow list

allow list 和 block list 正好反过来，这个是只允许列表中的 url 的请求：

比如当设置了 allow list 的时候：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5009a70cb7564aa8a2282ced90d22ba3~tplv-k3u1fbpfcp-watermark.image?)

所有不在列表中的请求都会失败，可以通过 status-code:0 的过滤器过滤出来：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3019673aa374abc920bb256a8bb3157~tplv-k3u1fbpfcp-watermark.image?)

## ssl proxying

https 的代理是很常用的，这个我们上节讲过，需要安装证书到系统。

默认抓取到的 https 的数据是加密过的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fc4b6db9f21438daa1f2afe5aa8de6f~tplv-k3u1fbpfcp-watermark.image?)

因为 https 会用证书里的公钥加密请求，用私钥加密响应数据：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b517aad07b9640f7ade178defcb36c93~tplv-k3u1fbpfcp-watermark.image?)

想要抓取明文数据，需要把证书换成 charles 的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c7ce0a2e429453b9f0dd75d851064ae~tplv-k3u1fbpfcp-watermark.image?)

首先，在 Proxy >  SSL Proxying Settings 里开启这个功能：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d48a731bda8b4722936125588994f3d7~tplv-k3u1fbpfcp-watermark.image?)

然后点击 help > SSL Proxying > Install Charles Root Certificate，把 charles 证书安装到系统的钥匙串中并信任：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5569ddeb00cd4f79ae6eef2fbaa71aff~tplv-k3u1fbpfcp-watermark.image?)

这样证书就换成了 charles 的并被标记为信任：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59df492f3a094315a3ef1e2ae5cd8d4e~tplv-k3u1fbpfcp-watermark.image?)

也就能抓到明文数据了：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf8120c972994be4b008e32a0105f03c~tplv-k3u1fbpfcp-watermark.image?)

## breakpoints

断点同样是很常用的功能，这个上节也讲过。

在 Proxy > Enable breakpoints 开启这个功能：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0700d1ca64064c198d80767af58193e6~tplv-k3u1fbpfcp-watermark.image?)

然后在 Proxy > Breakpoint Settings 里配置：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c778e8432704d31ae55561e80135d87~tplv-k3u1fbpfcp-watermark.image?)

可以单独开启请求/响应的断点：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b36d59564f1040f1ab791f97a1263786~tplv-k3u1fbpfcp-watermark.image?)

或者右键请求来设置：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9f4287b4470444bbcd9119943c1bf85~tplv-k3u1fbpfcp-watermark.image?)

再发送请求的时候或响应的时候就会断住：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/97d757064a414b1193e3637db20e7c71~tplv-k3u1fbpfcp-watermark.image?)

可以修改各种内容：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de71599275de46bab99950b26f1dcaf7~tplv-k3u1fbpfcp-watermark.image?)

然后 execute 即可。

## 总结

这节我们过了 charles 一些常用的功能：

首先设置代理可以用 SwitchyOmega 单独指定网页的代理服务器为 charles，也可以直接把 charles 设置为系统代理。

然后我们学习了常用的 charles 代理功能：

- no caching： 禁用缓存
- block cookies：禁用 cookie
- map remote：转发请求到另一个 url
- map local：用本地的文件作为该请求的响应内容
- mirror：把响应保存在本地文件
- rewrite： 重写请求或响应的各种数据
- block list：禁止一些 url 的请求
- allow list：只允许一些 url 的请求
- SSL Proxying：代理 https 请求，用 charles 自己的证书，需要安装 charles 证书到系统并信任
- breakpoints：断点修改请求/响应

这些功能中有重合的部分，比如可以通过 rewrite 实现 no caching 的功能，也就是修改 cache-control 的 header，比如 rewrite 的功能和 breakpoint 的功能很类似，但不同的场景下直接用对应的功能会更方便。

这些是比较常用的 charles 代理功能了，下节我们继续学习其余的功能。


