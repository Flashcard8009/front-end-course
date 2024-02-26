﻿在上一节课程中，我们通过封装 NPM 包的方式实现了微前端，该方案需要我们设计时导出 `mount` 和 `unmount` API，从而可以使主应用进行微应用的加载和卸载处理。NPM 包形式的微应用发布后，往往需要主应用升级相应 NPM 版本依赖并进行构建处理。如果想要主应用具备线上动态的微应用管理能力，最简单的方案是动态加载 Script。

## 动态 Script 方案

如果想要主应用具备线上动态的微应用管理能力，最简单的方案是动态加载 Script，大致的示例如下所示：

``` javascript
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <!-- 主导航设计，这里可以根据后端数据动态渲染导航 -->
    <div id="nav">
      <button onclick="handleClick('x')">x 应用</button>
      <button onclick="handleClick('y')">y 应用</button>
    </div>
    <!-- 内容区设计 -->
    <div class="container">
      <!-- 微应用渲染的插槽 -->
      <div id="micro-app-slot"></div>
    </div>

    <!-- 微应用 x：提供 window.xMount 和 window.xUnmount 全局函数-->
    <script defer src="http://xxx/x.js"></script>
    <!-- 微应用 y：提供 window.yMount 和 window.yUnmount 全局函数-->
    <script defer src="http://yyy/y.js"></script>

    <script>
      function handleClick(type) {
        switch (type) {
          case "x":
            // 告诉微应用需要挂载在 #micro-app-slot 元素上
            window.xMount("#micro-app-slot");
            window.yUnmount();
          case "y":
            window.yMount("#micro-app-slot");
            window.xUnmount();
          default:
            break;
        }
      }
    </script>
  </body>
</html>
```

> 温馨提示：该示例中一个时刻只加载一个微应用。

上述只是一个简单的示例，在真正的设计时可以将导航和需要加载的微应用进行动态化，这里给出一个相对完整的示例，大概实现的思路如下所示，主应用 HTML 渲染之后：

-   通过请求获取微应用列表数据，动态进行微应用的预获取和导航创建处理
-   根据导航进行微应用的切换，切换的过程会动态加载和执行 JS 和 CSS 资源
-   微应用需要提供 `mount` 和 `unmount` 全局函数，方便主应用进行加载和卸载处理

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5bbd9a3d3d1b4cb88a2ceaec7fa87a73~tplv-k3u1fbpfcp-zoom-1.image)

> 温馨提示：这里点击导航改变 Hash 来模拟 Vue 或者 React 框架中路由的切换。
  
实现效果如下所示，图中的两个按钮（微应用导航）根据后端数据动态渲染，点击按钮后会请求微应用的静态资源并解析相应的 JS 和 CSS，并渲染微应用的文本信息到插槽中：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2944bff480364061b101d58118bca86f~tplv-k3u1fbpfcp-zoom-1.image)

> 温馨提示：主应用中的文本样式会被切换的微应用样式污染，从一开始的红色变成绿色或蓝色。首次点击按钮加载微应用的静态资源时会命中 `prefetch cache`，从而缩短应用资源的加载时间（不会发送请求给服务端），再次点击按钮请求资源时会命中缓存，状态码是 304。关于缓存会在后续的性能优化课程中详细讲解。

文件的结构目录如下所示：

``` bash
├── public                  # 托管的静态资源目录
│   ├── main/               # 主应用资源目录                        
│   │   └── index.html                                        
│   └── micro/              # 微应用资源目录
│        ├── micro1.css   
│        ├── micro1.js    
│        ├── micro2.css         
│        └── micro2.js      
├── config.js               # 公共配置
├── main-server.js          # 主应用服务
└── micro-server.js         # 微应用服务
```

> 温馨提示：示例源码可以从 micro-framework 的 [demo/dynamic-script](https://github.com/ziyi2/micro-framework/tree/demo/dynamic-script) 分支获取。

主应用 HTML 的实现代码如下所示：

``` html
<!-- public/main/index.html -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
    <style>
      h1 {
        color: red;
      }
    </style>
  </head>

  <body>
    <!-- 主应用的样式会被微应用覆盖 -->
    <h1>Hello，Dynamic Script!</h1>
    <!-- 主导航设计，这里可以根据后端数据动态渲染 -->
    <div id="nav"></div>
    <!-- 内容区设计 -->
    <div class="container">
      <!-- 微应用渲染的插槽 -->
      <div id="micro-app-slot"></div>
    </div>

    <!-- 微应用工具类 -->
    <script type="text/javascript">
      class UtilsManager {
        constructor() {}

        // API 接口管理
        getMicroApps() {
          return window
            .fetch("/microapps", {
              method: "post",
            })
            .then((res) => res.json())
            .catch((err) => {
              console.error(err);
            });
        }

        isSupportPrefetch() {
          const link = document.createElement("link");
          const relList = link?.relList;
          return relList && relList.supports && relList.supports("prefetch");
        }

        // 预请求资源，注意此种情况下不会执行 JS
         // 后续在性能优化的课程中还会讲解，暂时可以忽略
        prefetchStatic(href, as) {
          // prefetch 浏览器支持检测
          if (!this.isSupportPrefetch()) {
            return;
          }
          const $link = document.createElement("link");
          $link.rel = "prefetch";
          $link.as = as;
          $link.href = href;
          document.head.appendChild($link);
        }

        // 请求 & 执行 JS（这里封装的不是很通用，可以考虑更加通用的封装处理）
        loadScript({ script, id }) {
          return new Promise((resolve, reject) => {
            const $script = document.createElement("script");
            $script.src = script;
            $script.setAttribute("micro-script", id);
            $script.onload = resolve;
            $script.onerror = reject;
            document.body.appendChild($script);
          });
        }

        loadStyle({ style, id }) {
          return new Promise((resolve, reject) => {
            const $style = document.createElement("link");
            $style.href = style;
            $style.setAttribute("micro-style", id);
            $style.rel = "stylesheet";
            $style.onload = resolve;
            $style.onerror = reject;
            document.head.appendChild($style);
          });
        }

        // 为什么需要删除 CSS 样式？不删除会有什么后果吗？
        // 为什么没有删除 JS 文件的逻辑呢？
        removeStyle({ id }) {
          const $style = document.querySelector(`[micro-style=${id}]`);
          $style && $style?.parentNode?.removeChild($style);
        }

        hasLoadScript({ id }) {
          const $script = document.querySelector(`[micro-script=${id}]`);
          return !!$script;
        }

        hasLoadStyle({ id }) {
          const $style = document.querySelector(`[micro-style=${id}]`);
          return !!$style;
        }
      }
    </script>

    <!-- 根据路由切换微应用 -->
    <script type="text/javascript">
      // 微应用管理
      class MicroAppManager extends UtilsManager {
        micrpApps = [];

        constructor() {
          super();
          this.init();
        }

        init() {
          this.processMicroApps();
          this.navClickListener();
          this.hashChangeListener();
        }

        processMicroApps() {
          this.getMicroApps().then((res) => {
            this.microApps = res;
            this.prefetchMicroAppStatic();
            this.createMicroAppNav();
          });
        }

        prefetchMicroAppStatic() {
          const prefetchMicroApps = this.microApps?.filter(
            (microapp) => microapp.prefetch
          );
          prefetchMicroApps?.forEach((microApp) => {
            microApp.script && this.prefetchStatic(microApp.script, "script");
            microApp.style && this.prefetchStatic(microApp.style, "style");
          });
        }

        createMicroAppNav(microApps) {
          const fragment = new DocumentFragment();
          this.microApps?.forEach((microApp) => {
            // TODO: APP 数据规范检测 (例如是否有 script、mount、unmount 等）
            const button = document.createElement("button");
            button.textContent = microApp.name;
            button.id = microApp.id;
            fragment.appendChild(button);
          });
          nav.appendChild(fragment);
        }

        navClickListener() {
          const nav = document.getElementById("nav");
          nav.addEventListener("click", (e) => {
            // 并不是只有 button 可以触发导航变更，例如 a 标签也可以，因此这里不直接处理微应用切换，只是改变 Hash 地址
            // 不会触发刷新，类似于框架的 Hash 路由
            window.location.hash = event?.target?.id;
          });
        }

        hashChangeListener() {
          // 监听 Hash 路由的变化，切换微应用
          // 这里设定一个时刻页面上只有一个微应用
          window.addEventListener("hashchange", () => {
            this.microApps?.forEach(async (microApp) => {
              // 匹配需要激活的微应用
              if (microApp.id === window.location.hash.replace("#", "")) {
                console.time(`fetch microapp ${microApp.name} static`);
                // 加载 CSS 样式
                microApp?.style &&
                  !this.hasLoadStyle(microApp) &&
                  (await this.loadStyle(microApp));
                // 加载 Script 标签
                microApp?.script &&
                  !this.hasLoadScript(microApp) &&
                  (await this.loadScript(microApp));
                console.timeEnd(`fetch microapp ${microApp.name} static`);
                window?.[microApp.mount]?.("#micro-app-slot");
                // 如果存在卸载 API 则进行应用卸载处理
              } else {
                this.removeStyle(microApp);
                window?.[microApp.unmount]?.();
              }
            });
          });
        }
      }

      new MicroAppManager();
    </script>
  </body>
</html>
```

> 温馨提示：在微应用切换的执行逻辑中，为什么需要删除 CSS 样文件？那为什么不删除 JS 文件呢？删除 JS 文件会有什么副作用吗？假设删除 `micro1.js`，那么还能获取 `window.micro1_mount` 吗？如果能够获取，浏览器为什么不在删除 JS 的同时进行内存释放处理呢？如果释放，会有什么副作用呢？

  


微应用的设计如下所示（完全可以替换成 React 或者 Vue 框架打包的 JS 脚本和 CSS 资源）：

``` javascript
// micro1.js
// 立即执行的匿名函数可以防止变量 root 产生冲突
(function () {
  let root;

  window.micro1_mount = function (slot) {
    // 以下其实可以是 React 框架或者 Vue 框架生成的 Document 元素，这里只是做一个简单的示例
    root = document.createElement("h1");
    root.textContent = "微应用1";
    // 在微应用插槽上挂载 DOM 元素
    const $slot = document.querySelector(slot);
    $slot?.appendChild(root);
  };

  window.micro1_unmount = function () {
    if (!root) return;
    root.parentNode?.removeChild(root);
  };
})();


// micro1.css
h1 {
  color: green;
}


// micro2.js
// 立即执行的匿名函数可以防止变量 root 产生冲突
(function () {
  let root;

  window.micro2_mount = function (slot) {
    // 以下其实可以是 React 框架或者 Vue 框架生成的 Document 元素，这里只是做一个简单的示例
    root = document.createElement("h1");
    root.textContent = "微应用2";
    const $slot = document.querySelector(slot);
    $slot?.appendChild(root);
  };

  window.micro2_unmount = function () {
    if (!root) return;
    root.parentNode?.removeChild(root);
  };
})();

// micro2.css
h1 {
  color: blue;
}
```

> 温馨提示：如果去除 `micro1.js` 和 `micro2.js` 的立即执行匿名函数，在微应用切换时，会发生什么情况呢？

相应的服务端设计如下所示：

``` javascript
// config.js
import ip from "ip";

export default {
  port: {
    main: 4000,
    micro: 3000,
  },
  // 获取本机的 IP 地址
  host: ip.address(),
};


// main-server.js
import express from "express";
import path from "path";
import morgan from "morgan";
import config from "./config.js";
const app = express();
const { port, host } = config;

// 打印请求日志
app.use(morgan("dev"));

app.use(express.static(path.join("public", "main")));

app.post("/microapps", function (req, res) {
  // 这里可以是管理后台新增菜单后存储到数据库的数据
  // 从而可以通过管理后台动态配置微应用的菜单
  res.json([
    {
      // 应用名称
      name: "micro1",
      // 应用标识
      id: "micro1",
      // 应用脚本（示例给出一个脚本，多个脚本也一样）
      script: `http://${host}:${port.micro}/micro1.js`,
      // 应用样式
      style: `http://${host}:${port.micro}/micro1.css`,
      // 挂载到 window 上的加载函数 window.micro1_mount
      mount: "micro1_mount",
      // 挂载到 window 上的卸载函数 window.micro1_unmount
      unmount: "micro1_unmount",
      // 是否需要预获取
      prefetch: true,
    },
    {
      name: "micro2",
      id: "micro2",
      script: `http://${host}:${port.micro}/micro2.js`,
      style: `http://${host}:${port.micro}/micro2.css`,
      mount: "micro2_mount",
      unmount: "micro2_unmount",
      fetch: true,
    },
  ]);
});

// 启动 Node 服务
app.listen(port.main, host);
console.log(`server start at http://${host}:${port.main}/`);

// micro-server.js
import express from "express";
import morgan from "morgan";
import path from 'path';
import config from "./config.js";
const app = express();
const { port, host } = config;

// 打印请求日志
app.use(morgan("dev"));
app.use(express.static(path.join("public", "micro")));

// 启动 Node 服务
app.listen(port.micro, host);
console.log(`server start at http://${host}:${port.main}/`);
```

同时启动主应用和微应用的服务，并通过访问主应用的 HTML 实现上述动图中的切换效果。动态 Script 的方案相对于 NPM 方案而言，具备如下优势：

-   主应用在线上运行时可以动态增加、删除和更新（升级或回滚）需要上架的微应用
-   微应用可以进行构建时性能优化，包括代码分割和静态资源分离处理
-   不需要额外对微应用进行库构建配置去适配 NPM 包的模块化加载方式

当然，动态 Script 方案和 NPM 包方案一样，会存在如下问题：

-   主应用和各个微应用的全局变量会产生属性冲突
-   主应用和各个微应用的 CSS 样式会产生冲突

## 小结

  


本课程简单讲解了动态 Script 的方案的示例和优缺点，接下来可以在动态 Script 方案的基础上做少许改动，实现一个简单的 Web Components 方案，下个课程我们来看下 Web Components 的方案设计。