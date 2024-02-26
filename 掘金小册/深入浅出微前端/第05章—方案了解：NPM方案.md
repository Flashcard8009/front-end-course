﻿NPM 包是微前端设计方案之一，在设计时需要将微应用打包成独立的 NPM 包，然后在主应用中引入和使用。通过本节课的学习，你可以了解直接开发应用和开发应用 NPM 包的一些细节知识和异同点。当然本文会从基本的模块化和构建工具开始讲解，从而帮助大家更好理解 NPM 包的微前端设计方案。具体的讲解顺序如下：

-   模块化：了解什么是模块化、为什么需要模块化；
-   构建工具：了解为什么需要构建工具；
-   NPM 设计方案：了解 NPM 设计方案的特性；
-   NPM 设计示例：React 和 Vue 微应用的 NPM 聚合示例。

## 模块化

早期的 Web 前端开发相对简单，开发者可以将所有的代码全部写在一个 JavaScript 文件中，也可以将代码拆分成多个 JavaScript 文件进行加载，如下所示：

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
  </head>
  <body>
    <script src="a.js"></script>
    <script src="b.js"></script>
    <script src="c.js"></script>
    <script src="d.js"></script>
  </body>
</html>
```

如果需要依赖三方库，则需要将三方库的代码按顺序进行加载，例如加载 [loadash](https://www.lodashjs.com/) ：

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <script>
      // 未加载 lodash 时，无法获取 _ 变量
      // Uncaught ReferenceError: _ is not defined
      console.log(_); 
    </script>
  
    <script src="https://cdn.bootcdn.net/ajax/libs/lodash.js/4.17.21/lodash.min.js"></script>
    
    <script>
      // 加载 lodash 之后，可以获取 _ 变量
      // ƒ Mc(n,t,r){var e=null==n?X:_e(n,t);return e===X?r:e}
      console.log(_.get); 
    </script>
</body>
</html>
```

除此之外，多个文件一起加载还需要考虑变量名的冲突情况，如下所示：

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <script>
      const _ = 111;
    </script>
    
    <script>
      console.log(_); // 111
    </script>
    
    <script src="https://cdn.bootcdn.net/ajax/libs/lodash.js/4.17.21/lodash.min.js"></script>
    
    <script>
      console.log(_); // 111
      console.log(_.get); // undefined
    </script>
  </body>
</html>
```

> 温馨提示：当声明全局变量 `_` 之后，lodash 库判断如果 `_` 存在则不做任何处理（防冲撞），防止后续的代码需要使用开发者自己声明的 `_` 时出现问题。

如果项目越来越复杂，下述使用方式很容易造成意想不到的问题，类似全局属性冲突的情况也很难进行问题排查：

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <script src="https://cdn.bootcdn.net/ajax/libs/lodash.js/4.17.21/lodash.min.js"></script>
    
    <script>
      // 覆盖三方库的全局变量
      const _ = 111;
    </script>
    
    <script>
      // 这里希望使用 lodash 的 get 方法，但是 _ 被用户覆盖
      // undefined
      console.log(_.get); 
    </script>
  </body>
</html>
```

为了解决上述问题，JavaScript 开始提供一种将程序拆分为可按需导入的单独模块，例如 Node.js 的 CommonJS 和 ES Modules。这里以 ES Modules 为例：

``` bash
├── public                 # 托管的静态资源目录
│   ├── custom_modules/    # 自定义模块
│   │   ├── add.js                         
│   │   └── conflict.js                                        
│   ├── node_modules/      # 三方模块
│   │   └── lodash-es/        
│   └── index.html         # 应用页面
└── app.js                 # 应用服务
```

> 温馨提示：示例源码可以从 micro-framework 的 [demo/npm-esmodule](https://github.com/ziyi2/micro-framework/tree/demo/npm-esmodule) 分支获取。

浏览器中的 JavaScript 由于受到了沙箱限制无法直接访问本地的文件（例如 `file://` 路径文件），因此在浏览器中使用模块化进行开发，无法像 Node 应用那样直接在 JavaScript 中通过 `require` 加载模块，只能通过 HTTP 请求的形式获取，因此在上述示例中，需要使用 Node.js 设计 `app.js` 启动 Web 服务：

```
// app.js
import ip from 'ip';
import express from 'express';
import path from "path";
import { fileURLToPath } from "url";
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const app = express();
const host = ip.address();
console.log("host: ", host);

// 所有托管的静态资源都放在 public 目录下
app.use(express.static(__dirname + "/public"));

// 启动 Node 服务
app.listen(4444, host);
console.log(`server start at http://${host}:4444/`);
```

安装依赖并启动 Web 服务，如下所示：

``` bash
# 安装依赖
npm i
# 进入 public 目录
cd public
# 安装浏览器 HTTP 请求需要的 public/node_modules 模块
npm i
# 回退到根目录
cd ..
# 启动服务
npm start
```

`public` 目录下的静态资源都可以被 HTTP 请求访问，如下所示：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/781aa60189d14a82b4307d4c2710dadc~tplv-k3u1fbpfcp-watermark.image?)

`public/node_modules` 目录是 NPM 安装三方模块后自动生成的模块目录，其中的 `lodash-es` 是一个符合 ES Modules 规范的工具库。`public/custom_modules` 目录是开发者自定义的 ES Modules 目录，其中的`add.js` 和 `conflict.js` 设计很简单，如下所示：

``` javascript
// add.js
export function add(a, b) {
  return a + b;
}

// conflict.js
const _ = 111;
```

`public` 目录下的 `index.html` 是在浏览器中启动的 HTML 页面，如下所示：

``` html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>

  <body>
    <!-- import-maps 主要用于映射 HTTP 请求路径别名，类似于 Webpack 中的 alias 和 TypeScript 中的 paths 配置 -->
    <!-- import-maps:  https://github.com/WICG/import-maps -->
    <script type="importmap">
      {
        "imports": {
          "custom_modules/": "/custom_modules/",
          "lodash/": "/node_modules/lodash-es/"
        }
      }
    </script>

    <script type="module">
      // 自定义模块 - add 函数
      import { add } from "custom_modules/add.js";
      // 自定义模块 - 防冲撞测试
      import "custom_modules/conflict.js";
      // 三方模块 - 按需引入
      import isNull from "lodash/isNull.js";

      console.log(isNull);
      console.log(add(1, 2));

      // 在 confilict.js 中声明的 const _ = 111; 不会作用在当前脚本中
      // Uncaught ReferenceError: _ is not defined
      console.log(_);
    </script>
  </body>
</html>
```

> 温馨提示：`import-maps` 存在[浏览器兼容性](https://caniuse.com/?search=import%20maps)问题，可以找相应的 [polyfill](https://github.com/guybedford/es-module-shims) 进行兼容性处理。

在脚本中引入的 `custom_module` 目录下的 `add.js`、`conflict.js` 以及 `node_modules` 目录下的三方模块 lodash，都会通过 HTTP 请求的方式进行加载，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa2aeba9051a4ea0b6375fa936d47cab~tplv-k3u1fbpfcp-zoom-1.image)

  


在模块化的示例中可以发现，设计 JavaScript 时只需要通过 `import` 的方式进行按需加载，例如需要依赖 `add.js`，则只需要通过 `import` 的方式加载 `add.js`，而且不需要考虑加载顺序的问题。

  


除此之外，在 `conflict.js` 中声明的 `_` 变量只能在当前模块范围内生效，不会影响到全局作用域，而 lodash 也可以通过模块化的方式进行按需加载，从而可以避免在全局 `window` 对象挂载 `_` 所需要考虑的防冲撞问题。

  


通过模块化的方式引入 lodash 的 `isNull` 函数，还可以天然实现三方库的按需引入，从而减少 HTTP 的请求体积。因此，在浏览器中使用 ES Modules:

-   不需要构建工具进行打包处理；
-   天然按需引入，并且不需要考虑模块的加载顺序；
-   模块化作用域，不需要考虑变量名冲突问题。

> 温馨提示：你知道 [Vite](https://vitejs.dev/guide/) 为什么可以很快了吗？

## 构建工具

应用的开发态可以直接在浏览器中使用 ES Modules 规范，但是生产态需要生成浏览器兼容性更好的 ES5 脚本，为此需要通过构建工具将多个 ES Modules 打包成兼容浏览器的 ES5 脚本。如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54f3d4272ca34a38822df96a00f0976d~tplv-k3u1fbpfcp-zoom-1.image)

> 温馨提示：Webpack 可以对 ES Modules 和 CommonJS 进行模块化组装，是一种常用的打包工具。除此之外， Webpack 可以借助 Babel 转译工具实现代码的 ES6+ 到 ES5 的转译工作，最终可以打包出兼容浏览器的代码。

当然，除了应用开发需要使用构建工具，在业务组件的开发中也需要使用构建工具，需要注意的是两者的构建是有差异的，**应用的构建需要生成 HTML 文件并打包 JS、CSS 以及图片等静态资源，业务组件的构建更多的是打包成应用需要通过模块化方式引入使用的** **[JavaScript 库](https://webpack.docschina.org/guides/author-libraries/)**。

业务组件的设计是一种通用的库包建设，当开发完一个版本之后，通常会发布成 NPM 包。**应用在构建时为了提升构建速度，同时也为了简化构建配置，通常在使用** **[babel-loader](https://github.com/babel/babel-loader#babel-loader-is-slow)** **（转译工具）进行转译时** **，** **会屏蔽** **`node_modules`** **目录下的 NPM 包**，因此需要将发布的 NPM 组件转译成大多数浏览器能够兼容的 ES5 标准，如下所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e72ee094995c452da5f16c8456f8ad56~tplv-k3u1fbpfcp-watermark.image?)

> 温馨提示：从 [JavaScript 标准版本的兼容性](https://kangax.github.io/compat-table/es6/) 可以发现，想要兼容大部分浏览器，需要将 ES6 或更高标准的 ECMAScript 转换成 [ES5](https://caniuse.com/?search=ES5) 标准，而如果要支持 IE9 及以下版本的浏览器，还需要使用 [polyfill](https://en.wikipedia.org/wiki/Polyfill_(programming)) (例如 [core-js](https://github.com/zloirock/core-js)) 来扩展浏览器中缺失的 API（例如 ES3 标准中缺失 `Array.prototype.forEach`）。如果对上图中的 ECMAScript 标准不了解，可以自行搜索和查看 ES2015 ~ ES2022（ES6 ~ ES13）、ESNext 等。

不同应用的开发态环境可能不同，例如应用 A 采用 ES6 & TypeScript 进行开发，而应用 B 则采用 ES13 进行开发，两个应用最终都需要通过构建工具生成浏览器能够兼容的 ES5 标准。

假设业务组件的开发环境使用 ES13 & TypeScript ，如果不进行组件本身的 ES5 标准构建，而是直接在应用中引入源码使用，那么应用 A 和应用 B 各自需要额外考虑组件的构建配置项，例如需要使应用 A 支持更高的 ES13 标准，需要使应用 B 支持 TypeScript，才能将加载的组件对应的源码在应用层面构建成 ES5 标准。如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a316fe112e54ceda7e6b94235914abd~tplv-k3u1fbpfcp-zoom-1.image)

> 温馨提示：注意直接输出 ES13 源代码和 NPM 发布的 ES Modules 模块有一定的差异，前者所有的代码都是 ES13 标准，后者只是导入导出的 `import` 和 `export` 符合 ES Modules 规范，其余的语法仍然是浏览器或 Node.js 能够兼容的 ES5 标准。这样前者一定需要在应用层被转译成 ES5 标准，而后者只是模块化标准需要被转译和打包处理，而内部代码不需要编译，具体可以查看 `pkg.module`。除此之外，这里需要额外了解发布 ES Modules 模块的好处是什么。

从上图可以看出，为了使应用层面不用关心业务组件设计的开发态环境，建议将业务组件的源代码构建成 ES5 标准，这样应用 A 和应用 B 就不需要考虑组件设计的开发态环境所带来的构建配置问题，只需要关注应用自身的开发态构建配置即可。

## NPM 设计方案

NPM 的微前端设计方案如下所示，微应用 A / B / C 可以采用不同的技术栈，但是构建时需要像发布业务组件一样输出 ES5 & 模块化标准的 JavaScript 库（尽管开发的是应用，但是构建的不是应用程序，不需要额外生成 HTML），从而使主应用安装各自的依赖时，可以通过模块化的方式引入微应用。主应用不需要关心微应用的技术栈，不需要关心微应用的构建配置项：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/23122eedd814465bbfa95fe5e0eadf71~tplv-k3u1fbpfcp-zoom-1.image)

  


业务组件的开发往往是和框架息息相关，组件发布以后的应用环境通常和组件使用同一个框架，例如 React 组件发布以后会被 React 技术栈的项目进行使用，此时业务组件发布时不需要构建 React 框架代码，只需要通过 [peerDependencies](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#peerdependencies) 来指定兼容的 React 版本。

而微应用 A / B / C 理论上不会依赖主应用的技术栈，主应用可以是 React、Vue、Angular 甚至是 jQuery 技术栈，因此大部分情况下需要将框架代码完全构建到 NPM 包中，当然如果知晓主应用的技术框架并且可以兼容，那么也可以像业务组件那样构建时排除框架代码，从而依赖主应用的框架运行。

了解了 NPM 设计方案的原理之后，会发现它的特点如下：

-   微应用可以使用不同的技术栈进行开发；
-   微应用不需要进行静态资源托管，只需要发布到 NPM 仓库即可；
-   移植性和复用性好，可以便捷地嵌在不同的主应用中；
-   微应用和主应用共享浏览器的 Renderer 进程、浏览上下文和内存数据。

使用该方案需要关注以下一些注意事项：

-   如何处理主应用和各个微应用的全局变量、CSS 样式和存储数据的冲突问题；
-   微应用的构建需要做额外的配置，构建的不是应用程序而是 [JavaScript 库](https://webpack.docschina.org/guides/author-libraries/)；
-   由于微应用构建的是库包，因此不需要[代码分割](https://webpack.docschina.org/guides/code-splitting/)和静态资源分离（例如图片资源、CSS 资源需要内联在 JS 中）；
-   微应用发布后，主应用需要重新安装依赖并重新构建部署。

从上述特性可以发现，NPM 设计仅仅适合集成一些小型微应用，如果微应用的资源过大，势必要对微应用的构建进行资源优化处理，例如多入口应用的三方库去重、弱网环境下 chunk 大小分离控制、懒加载、预加载等。

除此之外，主应用集成的微应用过多，也会导致构建多份重复的框架带来构建体积过大的问题，还需要考虑主子应用的共用框架抽离问题。

## NPM 的设计示例

设计示例采用 [Monorepo](https://monorepo.tools/#understanding-monorepos) 结构，项目的目录结构如下所示：

``` bash
├── packages                                                                                                 
│   ├── main-app/      # 主应用                
│   ├── react-app/     # React 微应用
│   └── vue-app/       # Vue 微应用
└── lerna.json         # Lerna 配置
```

> 温馨提示：示例源码可以从 micro-framework 的 [demo/npm-micro](https://github.com/ziyi2/micro-framework/tree/demo/npm-micro) 分支获取。

  


在主应用中通过路由的方式切换微应用，代码如下所示：

``` javascript
// main-app/index.js
import React from "react";
import ReactDOM from "react-dom/client";
import "./index.css";
import App from "./App";
import reportWebVitals from "./reportWebVitals";
import { createBrowserRouter, RouterProvider } from "react-router-dom";
import ReactApp from "./React";
import VueApp from "./Vue";

const router = createBrowserRouter([
  {
    path: "/",
    element: <App />,
    children: [
      {
        path: "react",
        element: <ReactApp />,
      },
      {
        path: "vue",
        element: <VueApp />,
      },
    ],
  },
]);

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<RouterProvider router={router} />);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();


// main-app/React.js
// React 微应用
const { mount, unmount } = require("react-micro-app");
import React, { useEffect } from "react";

const containerId = 'react-app';

function ReactApp() {
  useEffect(() => {
    mount(containerId);
    return () => {
      unmount();
    };
  }, []);
  return <div id={containerId}></div>;
}

export default React.memo(ReactApp);



// main-app/Vue.js
// Vue 微应用
import React, { useEffect } from "react";
const { mount, unmount } = require('vue-micro-app')

const containerId = 'vue-app';

function VueApp() {
  useEffect(() => {
    mount(containerId);
    return () => {
      unmount();
    };
  }, []);
  return <div id={containerId} style={{ textAlign: "center" }}></div>;
}

export default VueApp;

```

  


进入 `main-app` 后执行 `npm start` 启动主应用，可以根据左侧导航切换 React 和 Vue 微应用，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b17ad4109e7246d68424085252effdf0~tplv-k3u1fbpfcp-zoom-1.image)

微应用需要发布成模块规范的 NPM 包，这里为了方便，采用了 [Lerna](https://github.com/lerna/lerna) 工具进行 NPM 包之间的软 Link 操作，主要的思路是将入口文件构建成 ES5 标准的 CommonJS 规范（在 Webpack 中除了可以构建 CommonJS 规范，也可以构建 UMD 和 ES Modules 规范，具体可以查看 `output.library.type` 配置项），通过主应用的引入方式可以发现示例中输出了 CommonJS 规范。

> 温馨提示：在 Lerna 中执行`lerna bootstrap`，可以让本地的 NPM 包之间快速形成软链接，从而不需要发布 NPM 仓库，当然如果不使用 Lerna 工具，也可以手动通过 `npm link` 进行处理。

除此之外，如果不想额外引入微应用的 CSS 样式，那么可以将样式内联到 JS 中，当然其它的一些静态资源也需要内联到 JS 中，通过 Webpack 可以进行很好的处理。在 Vue 微应用中采用 [Vue CLI](https://cli.vuejs.org/zh/guide/build-targets.html) 快速生成了 Vue 3.x 项目，只需要简单做如下改动即可：

``` javascript
// package.json
{
  // NPM 包名
  "name": "vue-micro-app",
  // NPM 包默认的入口文件
  "main": "dist/vue-app.common.js",
    
  "scripts": {
    // Vue CLI 构建目标：https://cli.vuejs.org/zh/guide/build-targets.html
    // target 设置为 lib，可以输出 commonjs 和 umd 模块文件
    // --inline-vue 可以将 Vue 框架代码构建到模块文件中
    // src/main.js 为构建的入口文件
    "build": "vue-cli-service build --target lib --name vue-app --inline-vue src/main.js",
  },
}
  
// vue.config.js
const { defineConfig } = require("@vue/cli-service");
module.exports = defineConfig({
  transpileDependencies: true,
  // 内联 CSS 样式处理
  css: { extract: false }
});

// src/main.js 
// 对外提供 mount 和 unmount 接口，用于加载和卸载 vue 微应用
import { createApp } from "vue";
import App from "./App.vue";
let app;

export function mount(containerId) {
  console.log("vue app mount");
  app = createApp(App);
  app.mount(`#${containerId}`);
}

export function unmount() {
  console.log("vue app unmount: ", app);
  app && app.unmount();
}
```

  


执行 `npm run build`构建之后，默认会生成如下文件，此时可以通过 `require('vue-micro-app')` 引入 `dist/vue-app.common.js`中的 `mount` 和 `unmount` 方法：

```
 DONE  Compiled successfully in 3021ms                                                                     21:09:03

  File                       Size                                      Gzipped

  dist/vue-app.umd.min.js    85.44 KiB                                 35.97 KiB
  dist/vue-app.umd.js        424.09 KiB                                111.78 KiB
  dist/vue-app.common.js     423.69 KiB                                111.65 KiB

  Images and other types of assets omitted.
  Build at: 2023-01-05T13:09:03.077Z - Hash: 1af212ed2f5ac61f4283f8246edc6311fdcda0a55a38a10c - Time: NaNms
```

React 微应用采用 [Create React App](https://create-react-app.dev/docs/getting-started) 快速生成项目，由于需要构建成类似于库包的模块化规范，需要更改 Webpack 的配置，这里采用 `npm run eject` 的方式暴露 Webpack 配置，需要做如下简单的改动：

``` javascript
// package.json
{
  name: "react-micro-app",
  main: "build/main.js",
}

// config/webpack.config.js
module.exports = function(webpackEnv) {
  // ...

  // common function to get style loaders
  const getStyleLoaders = (cssOptions, preProcessor) => {
    const loaders = [
      // 注释掉抽离 CSS 样式的插件功能
      // isEnvDevelopment && require.resolve("style-loader"),
      // isEnvProduction && {
      //   loader: MiniCssExtractPlugin.loader,
      //   // css is located in `static/css`, use '../../' to locate index.html folder
      //   // in production `paths.publicUrlOrPath` can be a relative path
      //   options: paths.publicUrlOrPath.startsWith(".")
      //     ? { publicPath: "../../" }
      //     : {},
      // },
      
      require.resolve("style-loader"),
      {
        loader: require.resolve("css-loader"),
        options: cssOptions,
      },
      // ...
    ].filter(Boolean);
    // ...
    return loaders;
  };
  
  return {
    output: {
      // ...
      // 老版本 Webpack 可以使用 libraryTarget 生成 CommonJS 规范
      // libraryTarget: "commonjs",
      library: {
        type: 'commonjs'
      }
    },

    module: {
      rules: [
        {
          oneOf: [
            // TODO: Merge this config once `image/avif` is in the mime-db
            // https://github.com/jshttp/mime-db
            {
              test: [/.avif$/],
              mimetype: "image/avif",
              // 内联处理
              // https://webpack.js.org/guides/asset-modules/#inlining-assets
              type: 'asset/inline',
            },
            // "url" loader works like "file" loader except that it embeds assets
            // smaller than specified limit in bytes as data URLs to avoid requests.
            // A missing `test` is equivalent to a match.
            {
              test: [/.bmp$/, /.gif$/, /.jpe?g$/, /.png$/],
              // 内联处理
              type: 'asset/inline',
            },
            {
              test: /.svg$/,
              // 内联处理
              type: 'asset/inline',

              // 注释
              
              // use: [
              //   {
              //     loader: require.resolve("@svgr/webpack"),
              //     options: {
              //       prettier: false,
              //       svgo: false,
              //       svgoConfig: {
              //         plugins: [{ removeViewBox: false }],
              //       },
              //       titleProp: true,
              //       ref: true,
              //     },
              //   },
              //   {
              //     loader: require.resolve("file-loader"),
              //     options: {
              //        name: "static/media/[name].[hash].[ext]",
              //     },
              //   },
              // ],
              
              issuer: {
                and: [/.(ts|tsx|js|jsx|md|mdx)$/],
              },
            },
          ]
        }
      ].filter(Boolean),
    },

    plugins: [
      
      // ...
      
      // 构建单个 JS 脚本
      new webpack.optimize.LimitChunkCountPlugin({
        maxChunks: 1,
      }),
    ].filter(Boolean),
  }
}


// src/index.js
// 对外提供 mount 和 unmount 接口，用于加载和卸载 react 微应用
import React from "react";
import ReactDOM from "react-dom/client";
import "./index.css";
import App from "./App";

let root;

export function mount(containerId) {
  console.log("react app mount");
  root = ReactDOM.createRoot(document.getElementById(containerId));
  root.render(
    <React.StrictMode>
      <App />
    </React.StrictMode>
  );
}

export function unmount() {
  console.log("react app unmount: ", root);
  root && root.unmount();
}
```

> 温馨提示：使用应用程序的 Webpack 配置进行库构建改造会相对麻烦，这里也可以直接使用一些成熟的业务组件脚手架做处理，仅需要将 React 框架的代码构建到输出目标中即可。

使用 React 脚手架制作 NPM 微应用的 Webpack 配置相对比较麻烦，本质思路是需要将所有的资源都构建成单个 JS 脚本的方式进行引入（不要抽离通用的 JS 文件，因为不是直接在浏览器中运行），此时执行构建会生成 CommonJS 规范的 `build/main.js` 目标文件：

```
Creating an optimized production build...
Compiled successfully.

File sizes after gzip:

  50.28 kB  build/main.js
```

> 温馨提示：通常在开发态时，为了使得主应用能够查看微应用 NPM 包的集成效果，在修改微应用后需要对微应用进行构建才能在主应用中查看微应用的修改效果。有没有方法可以做到主应用引入方式不变的情况下，在开发态引入的是微应用的源码（修改微应用的代码后不需要构建，直接在主应用中会热更新变更），而在生产态引入的是微应用构建后发布的 NPM 包，这种配置在单个业务组件的开发中尤为重要。

## 小结

本节课首先讲解了模块化的知识，旨在帮助大家了解为什么在 Web 前端中需要使用模块化标准进行开发（如果对各种模块化标准不清楚的同学，需要额外了解各个模块化标准的差异），其次讲解了基于模块化开发需要使用的构建工具，目的是为了生成能够兼容大多数浏览器的 ES5 标准代码（主要是针对 Webpack 进行讲解，当然也可以了解其它构建工具的特性）。

  


在构建工具中，还讲解了应用程序和库的构建差异（在后续的工程设计课程中还会重点讲解应用和库构建的构建工具选型），以及为什么需要进行库构建。接着讲解了 NPM 微前端的设计方案以及它的特性，并基于设计方案给出了一个简单的实现示例。真正在设计的过程中还需要考虑一些注意事项，例如如何处理主应用和各个微应用的全局变量属性冲突、CSS 样式冲突和存储数据冲突等问题。

  


除此之外，如果要设计一个跨框架的组件，这也是非常好的一种实现思路，例如想要实现一个 React 组件并让它在 Vue 中使用，可以实现一种库 API 的调用方式。当然也可以在组件的外层进行 Web Components 或者 iframe 的封装，从而实现技术无关的能力。

  


在下一节课中，我们会讲解基于 Web Components 的微前端方案。