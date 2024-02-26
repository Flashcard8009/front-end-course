﻿在之前的课程中，我们重点讲解了 iframe、NPM、动态 Script 以及 Web Components 方案的微应用集成方式，本课程以 iframe 方案为例，重点讲解 iframe 中的 Cookie 数据在跨域和跨站中的表现情况，从而指导大家更加深入的了解微前端 iframe 中的 Cookie 处理。

> 温馨提示：本课程不会讲解如何实现微应用的 SSO 免登和 CSRF 防御等处理，只提供解决上述问题所需要的最基础的浏览器和服务端知识。感兴趣的同学需要自行了解实现主子应用免登的方案。

在**方案了解：iframe 方案**中，我们重点讲解了跨站和跨域的区别，在使用 iframe 进行主子应用集成时，很可能存在如下几种情况，接下来我们重点观察这些情况在浏览器中的 Cookie 表现：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4423bdeaedcf4fa5905df2f3ddef8f7c~tplv-k3u1fbpfcp-watermark.image?)

> 温馨提示：在微前端方案的设计中，如果没有纯三方应用的集成，大多数 iframe 应用可以设计成和主应用同域的业务场景，从而简化设计方案。出于安全考虑，浏览器中的 LocalStorage 在跨域情况下无法共享，但是面试官可能会给你挖个坑，询问 Cookie 在跨域情况下是否可以共享？阅读本课程可以很好的解答该问题。

iframe 测试 Cookie 的目录结构如下所示：

``` bash
├── same-origin/                 # 主子应用同域
├── same-site/                   # 主子应用跨域同站
├── cross-site/                  # 主子应用跨站
├── public/                      # 前端静态页面
├── config/                      # 服务端配置：端口、IP 等
└── package.json    
```

> 温馨提示：示例源码可以从 micro-framework 的 [demo/iframe-cookie](https://github.com/ziyi2/micro-framework/tree/demo/iframe-cookie) 分支获取。

`public` 目录主要存放 Web 前端的静态页面，如下所示：
 
``` html
<!-- 主应用 HTML：main.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>主应用</title>
  </head>
  <body>
    <h1>主应用</h1>
    <button onclick="handleConsoleCookie()">主应用 - 打印 cookie</button>
    <button onclick="handleLoadIframe()">加载 iframe 微应用</button>
    <br />
    <br />
  </body>

  <script>
    function handleConsoleCookie() {
      console.log("[主应用 - 打印 cookie]: ", document.cookie);
    }

    function handleLoadIframe() {
      const iframe = document.createElement('iframe');
      // ejs 模版 micro 变量填充: 指向微应用的打开地址
      iframe.src = "<%= micro %>";
      document.body.appendChild(iframe);
    }
  </script>
</html>


<!-- 微应用 HTML：micro.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>iframe 微应用</title>
  </head>
  <body>
    <h1>iframe 微应用</h1>
    <button onclick="handleConsoleCookie()">微应用 - 打印 cookie</button>
  </body>

  <script>
    function handleConsoleCookie() {
      console.log("[微应用 - 打印 cookie] micro-app cookie: ", document.cookie);
    }
  </script>
</html>
```

`config.js` 是服务端所有示例的公共配置项，如下所示：

``` javascript
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

  


`package.json` 中设置了各个示例启动服务的脚本：

``` json
  "scripts": {
    // 启动同域服务
    "same-origin": "node same-origin/main-server.js",
    // 启动跨域同站的两个服务(注意 & 和 && 的区别)
    "same-site": "node same-site/main-server.js & node same-site/micro-server.js",
    // 启动跨站的两个服务
    "cross-site": "node cross-site/main-server.js & node cross-site/micro-server.js"
  },
```

## 主子应用同域

首先设计一个同域的微前端方案，查看同域情况下主应用和 iframe 微应用的 Cookie 表现，服务端设计如下所示：

``` javascript
// same-origin/main-server.js
import path from "path";
// express：https://github.com/expressjs/express
import express from "express";
// ejs 中文网站: https://ejs.bootcss.com/#promo
// ejs express 示例: https://github.com/expressjs/express/blob/master/examples/ejs/index.js
import ejs from "ejs";

import config from "../config.js";
const { port, host, __dirname } = config;
const app = express();

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "public"));
app.set("view engine", "html");

app.get("/", function (req, res) {
  // 设置主应用的 cookie 标识
  res.cookie("main-app", "true");
  // 使用 ejs 模版引擎填充主应用 public/main.html 中的 micro 变量，并将其渲染到浏览器
  res.render("main", {
    // micro 指向微应用的打开地址
    micro: `http://${host}:${port.main}/micro`,
  });
});

// 微应用和主应用同域，只是服务路由不同
app.get("/micro", function (req, res) {
  // 设置 iframe 微应用的 cookie 标识，顺便观察能否覆盖主应用的 cookie 标识
  res.cookie("micro-app", "true").cookie("main-app", "false");
  // 渲染 views/micro.html
  res.render("micro");
});

// 启动 Node 服务
app.listen(port.main, host);
console.log(`server start at http://${host}:${port.main}/`);
```

  


使用 `npm run same-origin` 启动服务后，可以操作按钮来打印 Cookie 信息，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c11e90a24e14b53a88f238774013736~tplv-k3u1fbpfcp-zoom-1.image)

从上图可以发现，一开始只加载主应用的 HTML 时打印 `main-app=true`，动态加载 iframe 微应用后由于微应用将 `main-app` 设置为 `false`，发现主应用同名属性的 Cookie 值被覆盖。因此在同域的方案下，主子应用可以共享 Cookie 数据，但是存在同名属性值被覆盖的风险。

> 温馨提示：主子应用同域可以通过共享 Cookie 来解决登录态的问题，是非常常用的一种微前端方案。

## 主子应用跨域同站

在 iframe 方案中，我们已经讲解了同站的条件，两个应用的 eTLD + 1 相同则是同站。这里可以设计两个应用地址分别为 `http://ziyi.example.com` 和 `http://example.com`，其中 `com` 是一个有效顶级域名，因此 eTLD + 1 相同。为了模拟上述域名地址，可以使用 [iHosts](https://github.com/toolinbox/iHosts) 进行域名映射，主要实现如下功能：

-   在主应用中使用 iHosts 进行域名映射，将 `example.com` 映射到本地 IP 地址
-   在子应用中使用 iHosts 进行域名映射，将 `ziyi.example.com` 映射到本地 IP 地址

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b78ca0bfa3c2436b863de85d387de3b3~tplv-k3u1fbpfcp-zoom-1.image)

> 温馨提示：iHosts 工具本质上是更改了本地系统的 hosts 文件，从而起到域名和 IP 的匹配映射。这里可以额外了解本地 hosts 匹配和 DNS 服务解析的差异。

在 iHosts 中同一个 IP 地址可以指向不同的域名，配置如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96483581c4514829972ccc4f48e96670~tplv-k3u1fbpfcp-zoom-1.image)

服务代码如下所示：

``` javascript
// same-site/main-server.js
// 主应用服务代码
import path from "path";
import express from "express";
import ejs from "ejs";

import config from "../config.js";
const { port, host, __dirname } = config;
const app = express();

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "public"));
app.set("view engine", "html");

// 使用 iHosts 配置 example.com 指向本机的 ip 地址
// 主应用访问地址：http://example.com:4000
app.get("/", function (req, res) {
  res.cookie("main-app", "true");
  // 为了使同站的微应用可以共享主应用的 Cookie
  // 在设置 Cookie 时，可以使用 Domain 和 Path 来标记作用域
  // 默认不指定的情况下，Domain 属性为当前应用的 host，由于 ziyi.example.com 和 example.com 不同
  // 因此默认情况下两者不能共享 Cookie，可以通过设置 Domain 使其为子域设置 Cookie，例如 Domain=example.com，则 Cookie 也包含在子域 ziyi.example.com 中
  res.cookie("main-app-share", "true", { domain: "example.com" });
  res.render("main", {
    // 使用 iHosts 配置 ziyi.example.com 指向本机的 ip 地址
    // 子应用访问地址： http://ziyi.example.com:3000
    micro: `http://ziyi.example.com:${port.micro}` 
  });
});

// 启动 Node 服务
app.listen(port.main, host);
console.log(`main app start at http://${host}:${port.main}/`);

// same-site/micro-server.js
// 微应用服务代码
import path from "path";
import express from "express";
import ejs from "ejs";
import config from "../config.js";

const { host, port, __dirname } = config;
const app = express();

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "public"));
app.set("view engine", "html");

// 浏览器访问 http://${host}:${port.micro}/ 时会渲染 public/micro.html
// 端口不同，主应用和微应用跨域但是不跨站
app.get("/", function (req, res) {
  // 设置 iframe 微应用的 cookie 标识，顺便观察能否覆盖主应用的 cookie 标识
  res.cookie("micro-app", "true").cookie("main-app", "false");
  res.render("micro");
});

// 启动 Node 服务
app.listen(port.micro, host);
console.log(`server start at http://${host}:${port.micro}/`);
```

  


使用 `npm run same-site` 可以同时启动主应用和子应用的服务，操作按钮来打印 Cookie 信息，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca535046efa14406b780377c2ca96d72~tplv-k3u1fbpfcp-zoom-1.image)

  


从上图可以发现，在 host 不同的情况下，主应用的 `main-app` 字段不会被子应用进行覆盖，主应用和子应用无法共享 Cookie。但是如果主应用和子应用同站，那么可以通过设置 Domain 使得两个应用可以共享彼此的 Cookie，例如上图中的 `main-app-share` 字段，通过设置 Domain 为 `example.com` 后可以被子应用进行共享。因此在同站的某些特殊情况下，Cookie 是可以在不同的应用中产生共享的。

> 温馨提示：如果两个应用同站，那么在哪些情况下尽管设置了 Domain 仍然无法共享 Cookie 呢？

## 主子应用跨站

跨站的实现方式也非常简单，只需要更改原有的 iHosts 配置即可，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acf7b596f5f64a63974f6b2c2971e659~tplv-k3u1fbpfcp-zoom-1.image)

-   在主应用中使用 iHosts 进行域名映射，将 `ziyi.com` 映射到本地 IP 地址
-   在子应用中使用 iHosts 进行域名映射，将 `ziyi.example.com` 映射到本地 IP 地址

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6eb0be6e94844cbbd6eca4654653aa7~tplv-k3u1fbpfcp-zoom-1.image)

此时主子应用因为 eTLD + 1 不同形成跨站（`com` 是一个有效顶级域名），由于 iHosts 映射的 IP 地址不变，因此不需要将将主子应用的 Node 服务断开，重新在浏览器中通过 `ziyi.com:4000` 进行访问，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/621f4153e6f24919bb4613fdf9c13f83~tplv-k3u1fbpfcp-zoom-1.image)

从上述浏览器的测试信息可以看出：

-   在主应用中设置的 `main-app-share` Cookie 值失效，因为主应用不在 `example.com` 下
-   在子应用中设置的 `micro-app` 和 `main-app` 的 Cookie 值失效

从打印的信息可以发现子应用在跨站的情况下无法设置 Cookie，重点可以查看一下 iframe 微应用 Cookie 设置失效的原因：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8a1958991f247b9a617fec6a498b2c0~tplv-k3u1fbpfcp-zoom-1.image)

  


从警告信息可以看出，如果主应用和内部的 iframe 子应用（非顶级浏览上下文）跨站，那么 iframe 应用的 Cookie 设置会被阻止。查看 [SameSite cookies](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Set-Cookie/SameSite) 发现 Chrome 浏览器 80 版本将 SameSite 的默认值设置为 Lax，用于解决 iframe 应用产生 CSRF 跨站请求伪造的安全问题（防止跨站携带 Cookie）。此时解决方案是将 SameSite 设置为 None 并使用 Secure 属性（必须使用 HTTPS 协议）。为了实现 HTTPS 协议，这里采用 [Ngrok](https://github.com/bubenshchykov/ngrok#readme) 进行反向代理，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7ee35133966435f814561cc8e6636be~tplv-k3u1fbpfcp-zoom-1.image)

-   在主应用中使用 Ngrok 进行反向代理，建立 HTTPS 连接
-   在子应用中使用 Ngrok 进行反向代理，建立 HTTPS 连接，形成主子应用跨站

> 温馨提示：通过查看 [publicsuffix/list](https://publicsuffix.org/list/public_suffix_list.dat) 可以发现 ngrok-free.app 是顶级有效域名，因此上述主应用和子应用跨站。
>
> **警告 ⚠️**：这里为了建立 HTTPS 连接使用了 Ngrok 进行反向代理，但是这种方式切忌在公司内网尝试，本质上会将本地的网络暴露到公网进行内网穿透，存在安全风险。如果你是在公司内部网络，那么可以尝试下一个课程 Web Components 持久化示例中的 Nginx 配置方案建立 HTTPS 连接。课程中使用 Ngrok 是为了提供大家一种创建 HTTPS 连接的简单示例。

  


为了实现上述功能，可以借助 Node 的 Ngrok 客户端建立反向代理，修改服务代码：

``` javascript
// cross-site/main-server.js
// 主应用服务代码
import path from "path";
import express from "express";
import ngrok from "ngrok";
import ejs from "ejs";

import config from "../config.js";
const { port, host, authtoken, __dirname } = config;
const app = express();

// 内网穿透（主应用反向代理）
// ngrok: https://github.com/bubenshchykov/ngrok#readme
// authtoken 需要进行免费注册获取：https://github.com/bubenshchykov/ngrok#auth-token
// 本课程通过 ngrok 全局 CLI 设置了 authtoken，因此不需要传递 authtoken 参数
// https://ngrok.com/docs/getting-started/#step-3-connect-your-agent-to-your-ngrok-account
// ngrok config add-authtoken TOKEN
const main = await ngrok.connect({
  proto: "http",
  // authtoken,
  addr: `http://${host}:${port.main}`,
  // 使用 https 协议
  bind_tls: true,
});

console.log("main app ngrok url: ", main);

// 内网穿透（微应用反向代理）
const micro = await ngrok.connect({
  proto: "http",
  // authtoken,
  addr: `http://${host}:${port.micro}`,
  bind_tls: true,
});

console.log("micro app ngrok url: ", micro);

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "public"));
app.set("view engine", "html");

app.get("/", function (req, res) {
  res.cookie("main-app", "true");
  res.render("main", {
    micro,
  });
});

// 启动 Node 服务
app.listen(port.main, host);
console.log(`server start at http://${host}:${port.main}/`);

// cross-site/micro-server.js
// 微应用服务代码
import path from "path";
import express from "express";
import ejs from "ejs";
import config from "../config.js";

const { host, port, __dirname } = config;
const app = express();

app.engine(".html", ejs.__express);
app.set("views", path.join(__dirname, "public"));
app.set("view engine", "html");

app.get("/", function (req, res) {
  // 增加 SameSite 和 Secure 属性，从而使浏览器支持 iframe 子应用的跨站携带 Cookie
  // 注意 Secure 需要 HTTPS 协议的支持
  const cookieOptions = { sameSite: "none", secure: true };
  res
    .cookie("micro-app", "true", cookieOptions)
    // 设置 iframe 微应用的 cookie 标识，顺便观察能否覆盖主应用的 cookie 标识
    .cookie("main-app", "false", cookieOptions);
  res.render("micro");
});

// 启动 Node 服务
app.listen(port.micro, host);
console.log(`server start at http://${host}:${port.micro}/`);
```

使用 `npm run cross-site` 可以同时启动主应用和子应用的服务，如下所示：

``` bash
# 执行
npm run cross-site

# 打印
> micro-framework@1.0.0 cross-site
> node cross-site/main-server.js & node cross-site/micro-server.js

server start at http://30.120.112.54:3000/
main app ngrok url:  https://3c88-42-120-103-150.ngrok-free.app
micro app ngrok url:  https://64f6-42-120-103-150.ngrok-free.app
server start at http://30.120.112.54:4000/
```

默认情况下打开 Ngrok 代理地址会产生警告信息，这些信息会提示你切忌使用敏感数据：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/285e15b0a5d94d369dfe96cfc8ccebd1~tplv-k3u1fbpfcp-zoom-1.image)

  


为了去除警告页从而直接访问本地应用，需要在请求时加上请求头 `ngrok-skip-browser-warning`，可以使用 Chrome 的 [ Header Editor ](https://chrome.google.com/webstore/detail/header-editor/eningockdidmgiojffjmkdblpjocbhgh?hl=zh-CN)插件进行设置，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/daa465e4f5274532b475a6190b7e9c73~tplv-k3u1fbpfcp-zoom-1.image)

  


打开应用后，可以进行 iframe 子应用跨站携带 Cookie 测试，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/949572954dd44e1abd5451c93529fe62~tplv-k3u1fbpfcp-zoom-1.image)

通过该示例可以得出两个结论：

-   采用 iframe 进行微前端设计时，Chrome 浏览器中主子应用跨站的情况下，默认 iframe 子应用无法携带 Cookie，需要使用 HTTPS 协议和并设置服务端 Cookie 属性 SameSite 和 Secure 才行
-   跨站的情况下，主子应用无法进行 Cookie 共享

## 小结

在 iframe 微前端方案的示例测试中，我们可以发现：

-   主子应用同域：可以携带和共享 Cookie，存在同名属性值被微应用覆盖的风险
-   主子应用跨域同站：默认主子应用无法共享 Cookie，可以通过设置 Domain 使得主子应用进行 Cookie 共享
-   主子应用跨站：子应用默认无法携带 Cookie（防止 CSRF 攻击），需要使用 HTTPS 协议并设置服务端 Cookie 的 SameSite 和 Secure 设置才行，并且子应用无法和主应用形成 Cookie 共享

了解上述几种 Cookie 的携带情况可以帮助我们更好的进行微前端业务方案的设计，Cookie 是 SSO 免登设计中非常重要的凭证数据。