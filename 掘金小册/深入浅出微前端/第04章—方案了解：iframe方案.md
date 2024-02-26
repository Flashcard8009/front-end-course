﻿iframe 是常用的微前端设计方案之一，可能很多同学觉得 iframe 没什么可讲的，它已经是非常成熟的一种技术手段。本文的侧重点不是讲解 iframe 微前端方案实现，而是讲解它背后的浏览器知识，从而帮助大家更好地理解 iframe 和后续的微前端课程。本小节的讲解顺序如下：

-   浏览器多进程架构：了解浏览器多进程架构设计；
-   浏览器沙箱隔离：了解浏览器的沙箱隔离设计；
-   浏览器站点隔离：了解浏览器中 iframe 的沙箱隔离策略；
-   iframe 设计方案：基于浏览器知识，讲解 iframe 设计方案的优缺点；

## 浏览器多进程架构

浏览器是一个多进程（Multi Process）的设计架构，通常在打开浏览器标签页访问 Web 应用时，多个浏览器标签页之间互相不会受到彼此的影响，例如某个标签页所在的应用崩溃，其他的标签页应用仍然可以正常运行，这和浏览器的多进程架构息息有关。

以 Chrome 浏览器为例，在运行时会常驻 Browser 主进程，而打开新标签页时会动态创建对应的 Renderer 进程，两者的关系如下所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/145bfbb9fd2341b8b5f7a769b2a488e7~tplv-k3u1fbpfcp-watermark.image?)

-   **Browser 主进程**：主要负责处理网络资源请求、用户的输入输出 UI 事件、地址栏 URL 管理、书签管理、回退与前进按钮、文件访问、Cookie 数据存储等。Browser 进程是一个常驻的主进程，它也被称为代理进程，会派生进程并监督它们的活动情况。除此之外，Browser 进程会对派生的进程进行沙箱隔离，具备沙箱策略引擎服务。Browser 进程通过内部的 I/O 线程与其他进程通信，通信的方式是 [IPC](https://www.chromium.org/developers/design-documents/inter-process-communication/) & [Mojo](https://chromium.googlesource.com/chromium/src/+/HEAD/mojo/README.md)。
-   **Renderer 进程**：主要负责标签页和 iframe 所在 Web 应用的 UI 渲染和 JavaScript 执行。Renderer 进程由 Browser 主进程派生，每次手动新开标签页时，Browser 进程会创建一个新的 Renderer 进程。

> 温馨提示：新开的标签页和 Renderer 进程并不一定是 1: 1 的关系，例如，多个新开的空白标签页为了节省资源，有可能被合并成一个 Renderer 进程。

上图只是一个简单的多进程架构示意，事实上 Chrome 浏览器包括 Browser 进程、网络进程、数据存储进程、插件进程、Renderer 进程和 GPU 进程等。除此之外，Chrome 浏览器会根据当前设备的性能和存储空间来动态设置部分进程是否启用，例如低配 Andriod 手机的设备资源相对紧张时，部分进程（存储进程、网络进程、设备进程等）会被合并到 Browser 主进程，完整的多进程架构如下所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6448133e9f694689bced2f4ec1fc104a~tplv-k3u1fbpfcp-watermark.image?)

如果想要查看 Chrome 浏览器的进程运行情况，可以通过右上角的**设置 / 更多工具 / 任务管理器**打开：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f5d3b31540e43d9a7ab03444f7d4e68~tplv-k3u1fbpfcp-zoom-1.image)

打开以后可以在任务管理器的列表中查看进程情况：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5f80c36ebc44b49af6e01436c091f15~tplv-k3u1fbpfcp-zoom-1.image)

## 浏览器沙箱隔离

由于 Web 应用运行在 Renderer 进程中，浏览器为了提升安全性，需要通过常驻的 Browser 主进程对 Renderer 进程进行沙箱隔离设计，从而实现 Web 应用进行隔离和管控，如下所示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b196411ee9004595af2afad774707b15~tplv-k3u1fbpfcp-watermark.image?)

> 温馨提示：从 Chrome 浏览器开发商的角度出发，需要将非浏览器自身开发的 Web 应用设定为三方不可信应用，防止 Web 页面可以通过 Chrome 浏览器进入用户的操作系统执行危险操作。

Chrome 浏览器在进行沙箱设计时，会尽可能的复用现有操作系统的沙箱技术，例如以 Windows 操作系统的沙箱架构为例，**所有的沙箱都会在进程粒度进行控制**，所有的进程都通过 IPC 进行通信。在 Windows 沙箱的架构中，存在一个 Broker 进程和多个 Target 进程， Broker 进程主要用于派生 Target 进程、管理 Target 进程的沙箱策略、代理 Target 进程执行策略允许的操作，而所有的 Target 进程会在运行时受到沙箱策略的管控：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50b9752ee97d455a9977576969c410e3~tplv-k3u1fbpfcp-watermark.image?)

在 Chrome 浏览器的多进程架构中，Browser 进程对应 Broker 进程，可以理解为浏览器沙箱策略的总控制器， Renderer 进程对应沙箱化的 Target 进程，它主要运行不受信任的三方 Web 应用，因此，在 Renderer 进程中的一些系统操作需要经过 IPC 通知 Browser 进程进行代理操作，例如网络访问、文件访问（磁盘）、用户输入输出的访问（设备）等。

> 温馨提示：Chrome 浏览器的插件进程是否已经沙箱化？

## 浏览器站点隔离

在 Chrome 浏览器中沙箱隔离以 Renderer 进程为单位，而在旧版的浏览器中会存在多个 Web 应用共享同一个 Renderer 进程的情况，此时浏览器会依靠[同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)来限制两个不同源的文档进行交互，帮助隔离恶意文档来减少安全风险。

Chrome 浏览器未启动站点隔离之前，标签页应用和内部的 iframe 应用会处于同一个 Renderer 进程，Web 应用有可能发现安全漏洞并绕过同源策略的限制，访问同一个进程中的其他 Web 应用，因此可能产生如下安全风险：

-   获取跨站点 Web 应用的 Cookie 和 HTML 5 存储数据；
-   获取跨站点 Web 应用的 HTML、XML 和 JSON 数据；
-   获取浏览器保存的密码数据；
-   共享跨站点 Web 应用的授权权限，例如地理位置；
-   绕过 [X-Frame-Options](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-Frame-Options) 加载 iframe 应用（例如百度的页面被 iframe 嵌套）；
-   获取跨站点 Web 应用的 DOM 元素。

在 Chrome 67 版本之后，为了防御多个**跨站的 Web 应用**处于同一个 Renderer 进程而可能产生的安全风险，浏览器会给来自不同站点的 Web 应用分配不同的 Renderer 进程。例如当前标签页应用中包含了多个不同站点的 iframe 应用，那么浏览器会为各自分配不同的 Renderer 进程，从而可以基于沙箱策略进行应用的进程隔离，确保攻击者难以绕过安全漏洞直接访问跨站 Web 应用：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87d1934a63fc48cfb19c80dcafc128b6~tplv-k3u1fbpfcp-watermark.image?)

> 温馨提示：Chrome 为标签页分配 Renderer 进程的策略和 iframe 中的站点隔离策略是有差异的，例如用户自己新开标签页时，不管是否已经存在同站的应用都会创建新的 Renderer 进程。用户通过`window.open` 跳转新标签页时，浏览器会判断当前应用和跳转后的应用是否属于同一个站点，如果属于同一个站点则会复用当前应用所在的 Renderer 进程。

  


需要注意跨站和跨域是有区别的，使用跨站而不是跨域来独立 Renderer 进程是为了兼容现有浏览器的能力，**例如同站应用通过修改** **`document.domain`** **进行通信**，如果采用域名隔离，那么会导致处于不同 Renderer 进程的应用无法实现上述能力。这里额外了解一下同源和同站的区别，如下所示：

-   同源：协议（protocol）、主机名（host）和端口（port）相同，则为同源；
-   同站：有效顶级域名（Effective Top-Level-Domain，eTLD）和二级域名相同，则为同站。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/586bf4c5f34d4bf9ae7992e544cf538d~tplv-k3u1fbpfcp-watermark.image?)

从上图可以看出，eTLD + 1 代表有效顶级域名 + 二级域名。需要注意，有效顶级域名和顶级域名是不一样的概念，例如 `github.io` 是一个有效顶级域名，如果将 `.io` 视为有效顶级域名，那么 `https://ziyi2.github.io` 和 `https://xxholly32.github.io` 将被浏览器视为同站，但显然它们是两个不同的开发者创建的博客站点。有效顶级域名有一个维护列表，具体可以查看 [publicsuffix/list](https://publicsuffix.org/list/public_suffix_list.dat)。举个例子：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d8e6a8ec79f40169dcdf2e43358af3e~tplv-k3u1fbpfcp-watermark.image?)

> 温馨提示：什么是有方案同站？


关于站点隔离可以通过启动 Node 并聚合 iframe 应用进行验证，目录结构如下：

``` bash
├── views    
│   └── iframe.html                        
│   └── main.html         
├── main-server.js           # main 应用服务
├── micro-server.js          # iframe 应用服务
├── config.js                # 端口，host 等配置
└── package.json    
```

> 温馨提示：示例源码可以从 micro-framework 的 [demo/iframe-isolate](https://github.com/ziyi2/micro-framework/tree/demo/iframe-isolate) 分支获取。

启动 Node 服务渲染 `main` 和 内部的 `iframe` 应用：

``` javascript
 // main-server.js
import path from 'path';
// https://github.com/expressjs/express
import express from 'express';
// ejs 中文网站: https://ejs.bootcss.com/#promo
// ejs express 示例: https://github.com/expressjs/express/blob/master/examples/ejs/index.js
import ejs from "ejs";
import config from './config.js';
const { port, host, __dirname } = config;

const app = express();

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "views"));
app.set("view engine", "html");

// 浏览器访问 http://${host}:${port.main}/ 时会渲染 views/main.html 
app.get("/", function (req, res) {
  // 使用 ejs 模版引擎填充主应用 views/main.html 中的 iframeUrl 变量，并将其渲染到浏览器
  res.render("main", {
    // 填充 iframe 应用的地址，只有端口不同，iframe 应用和 main 应用跨域但是同站
    iframeUrl: `http://${host}:${port.micro}`
  });
});

// 启动 Node 服务
app.listen(port.main, host);
console.log(`server start at http://${host}:${port.main}/`);


// micro-server.js
import path from 'path';
import express from 'express';
import ejs from "ejs";
import config from './config.js';
const { port, host, __dirname } = config;

const app = express();

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "views"));
app.set("view engine", "html");

app.get("/", function (req, res) {
  res.render("iframe");
});

// 启动 Node 服务
app.listen(port.micro, host);
console.log(`server start at http://${host}:${port.micro}/`);


// config.js
// https://github.com/indutny/node-ip
import ip from 'ip';
import path from "path";
import { fileURLToPath } from "url";
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

export default {
  port: {
    main: 4000,
    micro: 3000,
  },

  // 获取本机的 IP 地址
  host: ip.address(),

  __dirname
};
```

  


在 `main` 对应的 HTML 中使用 iframe 聚合同站应用：

``` html
<!-- main.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>站点隔离测试</title>
</head>
<body>
    <h1>main 应用</h1>
    <button onclick="javascript:window.open('<%= iframeUrl %>')">在新的标签页打开 iframe 应用</button>
    <br>
    <!-- 同站应用：iframe.html -->
    <iframe src="<%= iframeUrl %>"></iframe>
    <!-- 跨站应用: https://juejin.cn/ -->
    <iframe src="https://juejin.cn"></iframe>
</body>
</html>

<!-- iframe.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>同站的 iframe 应用</title>
  </head>
  <body>
    <h1>同站的 iframe 应用</h1>
  </body>
</html>
```

使用 `npm run main-start` 和 `npm run micro-start` 同时启动 main 和 iframe 应用的服务后，在浏览器中打开 main 应用，通过任务管理器查看各自的进程，可以发现**跨站的掘金应用启动了一个新的进程：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9af3c45876a45fa8ba28ba9f5c57c96~tplv-k3u1fbpfcp-zoom-1.image)

为了查看同站 iframe 所在应用的进程，可以点击按钮使用 `window.open` 跳转新的标签页，此时可以发现 iframe 应用和 main 应用处于同一个 Renderer 进程中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56ef90f2c9fb40c68a923dddaf5b67f0~tplv-k3u1fbpfcp-zoom-1.image)

浏览器的站点隔离功能是可以关闭的，通过 `chrome://flags` 进入：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1267ed5ecd744d696fb8325b45d015d~tplv-k3u1fbpfcp-zoom-1.image)

禁用站点隔离后，可以发现主应用和两个 iframe 应用处于同一个 Renderer 进程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d99e7cbeb1884217b88b7a11bcb91ed4~tplv-k3u1fbpfcp-zoom-1.image)

## 浏览上下文

每一个 iframe 都有自己的[浏览上下文](https://www.w3.org/html/wg/spec/browsers.html#browsing-context)，不同的浏览上下文包含了各自的 Document 对象以及 History 对象，通常情况下 Document 对象和 Window 对象存在 1:1 的映射关系，如下所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce810f4e807e43d1bacc68b4bd0d612f~tplv-k3u1fbpfcp-watermark.image?)

在上述示例中，如果主应用是在空白的标签页打开，那么主应用是一个顶级浏览上下文，顶级浏览器上下文既不是嵌套的浏览上下文，自身也没有父浏览上下文，通过访问 `window.top` 可以获取当前浏览上下文的顶级浏览上下文 `window` 对象，通过访问 `window.parent` 可以获取父浏览上下文的 `window` 对象。

例如想要知道当前应用是否在 iframe 中打开，可以简单通过如下代码进行判断：

``` javascript
// 如果自己嵌自己会发生什么情况呢？
// 是否可以使用 if(window.parent !== widnow) {} 代替
if(window.top !== window) {}
```

> 温馨提示：如果希望判断微应用是否被其他应用进行嵌入，也可以使用 [location.ancestorOrigins](https://developer.mozilla.org/zh-CN/docs/Web/API/Location/ancestorOrigins) 来进行判断。

## iframe 设计方案

在微前端中 iframe 方案需要一个主应用，包含导航和内容区的设计，通过切换导航来控制内容区微应用 A / B / C 的加载和卸载，如下所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ae48f1330354c7bbbdf9821373b2f3a~tplv-k3u1fbpfcp-watermark.image?)

在之前的课程中讲解了导航切换的设计方案可以是前端框架路由、服务端路由和自己设计的切换逻辑，在 iframe 的方案中，导航设计可以是前端框架路由来控制不同微应用所在 iframe 的显示和隐藏，也可以通过自己设计切换逻辑来动态加载 iframe。

不论使用哪一种切换方式，在首次加载 iframe 应用时，都会因为服务端请求而导致内容区带来短暂的白屏效果。当然，相比普通 MPA 应用，通过服务端路由的方式来处理，最大的好处是每次切换微应用都不需要刷新主应用。除此之外，iframe 应用的特点主要包括：

-   站点隔离和浏览上下文隔离，可以使微应用在运行时天然隔离，适合集成三方应用；
-   移植性和复用性好，可以便捷地嵌在不同的主应用中。

当然在使用 iframe 应用时，会产生如下一些问题：

-   主应用刷新时， iframe 无法保持 URL 状态（会重新加载 `src` 对应的初始 URL）；
-   主应用和 iframe 处于不同的浏览上下文，无法使 iframe 中的模态框相对于主应用居中；
-   主应用和 iframe 微应用的数据状态同步问题：持久化数据和通信。

> 温馨提示：对于非后台管理系统而言，使用 iframe 还需要考虑 SEO、移动端兼容性、加载性能等问题。

海康威视安防的管理后台系统采用了 iframe 作为微前端解决方案，很好利用了 iframe 的优点，从而实现了不同定制产品的组装功能。当然在 iframe 的设计方案中，**首要解决的是主应用和微应用的免登问题**，该问题可以通过主应用和微应用共享 Cookie 进行处理，在后续的课程中会讲解在 iframe 中的 Cookie 携带情况。

## 小结

本小节首先讲解了和 iframe 息息相关的浏览器原理知识，包括多进程架构、沙箱隔离、站点隔离以及浏览上下文等，基于原理知识讲解了 iframe 设计方案的优缺点。在下一小节中会讲解基于 NPM 包的微前端方案。