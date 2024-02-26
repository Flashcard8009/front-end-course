﻿在上一节课程中，我们讲解了 `src = about:blank` iframe 隔离和同源的 iframe 隔离，后者需要服务端提供空白页或者服务接口才能实现，并且请求本身也会产生性能损耗，如果能够解决前者隔离的 `history` 运行问题，那么可以减少服务接口和网络请求，从而达到隔离效果。本课程会重点讲解如何解决 `src = about:blank` iframe 隔离的运行问题。

> 温馨提示：如果想要解决同源 iframe 隔离中需要网络请求和服务端网关的问题，有没有什么解决方案呢？

## 隔离思路

在 iframe 中可以创建新的全局执行上下文从而和主应用彻底隔离，因此我们可以将 iframe 创建的 `window` 对象作为微应用的全局对象，从而实现应用之间 JS 的彻底隔离，但是在之前的 iframe 隔离设计中还存在以下问题没有解决：

-   `src = about:blank` iframe 的 `history` 无法正常工作，框架的路由功能丢失
-   虽然 DOM 环境天然隔离，却无法使得 iframe 中的 Modal 相对于主应用居中
-   主应用和微应用的 URL 状态没有同步

同源的 iframe 方案虽然能解决 `history` 运行的问题，但是需要主应用额外提供服务接口，对于框架设计而言并不是特别通用（如果业务本身能够支持，那么这种方案是非常不错的选择）。我们可以换个方式思考一下，是否可以将 `src = about:blank` iframe 中的 `history` 使用主应用的 `history` 代替运行，这样带来的好处如下：

-   不同微应用可以拥有各自 iframe 对应的全局上下文执行环境，可以实现 JS 的彻底隔离
-   使用主应用的 `history`，iframe 可以设置成 `src = about:blank`，不会产生运行时错误
-   使用主应用的 `history`，未来可以处理主应用和微应用的历史会话同步问题

为了实现上述思路，可以使用 `Proxy` 对 iframe 的 `window` 对象进行拦截处理，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69730bd61316495ab5c9b511c2c1425d~tplv-k3u1fbpfcp-zoom-1.image)

  


> 温馨提示：本课程只是提供简单的解决思路，真正要实现会话同步还需要考虑主子应用之间的路由冲突问题。
>
> 除此之外，如果要实现 iframe 中的模态框相对于主应用进行居中，可以在设计时将微应用的 `document` 代理成主应用 `document`，并且将需要渲染的微应用内容放在主应用的 DOM 环境中。当然这种方式解除了 iframe 中 DOM 天然隔离的限制，并且可以使得微应用任意修改主应用的 DOM 环境，而且还需要额外处理 DOM 的副作用（例如移除微应用时需要清空相应的 DOM 事件）以及 CSS 隔离问题，如果考虑不够完善，很容易产生意想不到的 Bug。

为了实现该代理功能，接下来需要额外了解一些 JavaScript 的语言特性，从而帮助大家更好的理解方案设计。

## Proxy

`Proxy` 可以对需要访问的对象进行拦截，并可以通过拦截函数对返回值进行修改，这种特性可以使我们在微应用中访问 `window` 对象的属性时，返回定制化的属性值，例如：

``` javascript
const iframe = document.createElement("iframe");
// 设置成 about:blank 后和主应用同源
iframe.src = "about:blank";
// 这里只用于演示 JS 运行，暂时不考虑加载微应用的 DOM，可以隐藏 iframe 处理
iframe.style = "display: none";
document.body.appendChild(iframe);

// iframe.contentWindow： iframe 的 window 对象
iframe.contentWindow.proxy = new Proxy(iframe.contentWindow, {
  get: (target, prop) => {
    console.log("[proxy get 执行] 拦截的 prop: ", prop);
    // 访问微应用的 window.history 时，返回主应用的 history
    if (prop === "history") {
      return window[prop];
    }
    // target 是被代理的 iframe.contentWindow
    // 除了 history，其余都返回 iframe 上 window 的属性值
    return target[prop];
  },

  set: (target, prop, value) => {
    console.log("[proxy set 执行] 拦截的 prop: ", prop);
    target[prop] = value;
  },
});

function execMicroCode() {
  // microCode 可以视为通过请求获取的微应用的 JS 脚本文本（不包括立即执行的匿名函数）
  // 在微应用中执行的 window，就是主应用中执行的 iframe.contentWindow
  // window.proxy 指的是已经设置了代理的 iframe.contentWindow.proxy
  const microCode = `(function(window){

  // 在立即执行的匿名函数中 window.proxy 作为形参被传入
  // 内部使用的 window 本质上是 iframe.contentWindow 的代理对象
  // 任何 window 属性访问和设置都会触发 Proxy 的 get 和 set 拦截行为
  
  // 访问 window.a 触发 Proxy 的 get，本质上读取的是 iframe.contentWindow 的 a 属性值
  // 打印信息 
  // [proxy get 执行] 拦截的 prop:  a
  // [微应用执行] window.a:  undefined
  console.log('[微应用执行] window.a: ', window.a);
  
  // 访问 window.history 触发 Proxy 的 get，本质上读取的是主应用的 history 对象
  // 访问 window.parent 触发 Proxy 的 get，本质上读取的是微应用的 iframe.contentWindow.parent 对象
  // 打印信息 
  // [proxy get 执行] 拦截的 prop:  history
  // [proxy get 执行] 拦截的 prop:  parent
  // [微应用执行] 是否是主应用的 history： true
  console.log('[微应用执行] 是否是主应用的 history：', window.history === window.parent.history);
  
  // 访问 window.history 触发 Proxy 的 get，本质上读取的是主应用的 history 对象
  // 打印信息 
  // [proxy get 执行] 拦截的 prop:  history
  window.history.pushState({}, '', '/micro');
  
  // 设置 window.a 触发 Proxy 的 set，本质上设置的是 iframe.contentWindow 的 a 属性值
  // 打印信息 
  // [proxy set 执行] 拦截的 prop:  a
  window.a = 2;
  
  // 访问 window.a 触发 Proxy 的 get，本质上读取的是 iframe.contentWindow 的 a 属性值
  // 打印信息 
  // [proxy get 执行] 拦截的 prop:  a
  // [微应用执行] window.a:  2
  console.log('[微应用执行] window.a: ', window.a);

})(window.proxy)`;

  const scriptElement =
    iframe.contentWindow.document.createElement("script");
  scriptElement.textContent = microCode;
  // 添加内嵌的 script 元素时会自动触发 JS 的解析和执行
  iframe.contentWindow.document.head.appendChild(scriptElement);
}

// 主应用的 window.a 设置为 1
window.a = 1;

// 执行微应用的代码，此时微应用中的 window 使用了 iframe 的 window 进行代理，不会受到主应用的 window 影响
execMicroCode();

// 主应用的 window 不会受到微应用的影响，输出为 1
// 控制台打印信息 [主应用执行] window.a:  1
console.log('[主应用执行] window.a: ', window.a);
```

> 温馨提示：这里立即执行的匿名函数本质上不是为了隔离微应用的执行作用域，因为微应用都运行在 iframe 中，已经天然做到了完全隔离，这里的作用是为了通过传入形参的方式改变内部使用的 `window` 变量。

从下图打印的信息可以可以发现， `history` 被正确的进行了代理，并且主应用和微应用由于使用了不同的全局执行上下文，形成了非常彻底的隔离效果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b64128270d304e23800ab34d34a3e63b~tplv-k3u1fbpfcp-zoom-1.image)

> 温馨提示：从红框部分可以发现，尽管 iframe 设置成了 `src=about:blank`，但是由于微应用使用了主应用的 `history` 执行，使得主应用的 URL 发生了变化。这里只是演示了主子应用共用 `history` 的情况，真正在设计时还需要考虑处理主子应用以及子应用之间在使用 Vue 或者 React 框架时的路由嵌套以及冲突问题。

在 iframe 隔离中我们知道设置 `src=about:blank` 会使得 `history` 无法正常工作，例如修改上述示例：

``` javascript
function execMicroCode() {
  
  const microCode = `(function(window){

      // 省略微应用执行的代码

   // window 指的是 iframe.contentWindow，注意传入的参数不是 winwow.proxy
   })(window)`;
}
```

如果不使用 Proxy 进行代理，那么 `window.history` 访问的是 `iframe` 的 `history`，此时页面会无法正常工作：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f18c1c432524e9c999735931325c9d2~tplv-k3u1fbpfcp-zoom-1.image)

需要注意以下场景无法被 Proxy 的 `set` 拦截，通过 `get` 拦截获取属性值时部分场景无法达到预期的效果：

``` javascript
function execMicroCode() {
    
  // 注意传入了第二个参数 contentWindow，这是没有被代理的 iframe 的 window 对象
  const microCode = `(function(window, contentWindow){

    // 没有触发 Proxy 的 set 函数，由于在立即执行的匿名函数中执行 var，此时不是全局变量
    var a = 2;
    
    // 触发 Proxy 的 get
    // 打印信息 
    // [proxy get 执行] 拦截的 prop:  a
    // [微应用执行] window.a:  undefined
    console.log('[微应用执行] window.a: ', window.a);
    
    // 没有触发 Proxy 的 set 函数，因为 this 指代的是没有被代理的 iframe 的 window 对象
    this.a = 2;
    
    // 打印信息 
    // [proxy get 执行] 拦截的 prop:  parent
    // [微应用执行] this 是否是主应用的 window:  false
    console.log('[微应用执行] this 是否是主应用的 window: ', this === window.parent);
    
    // 打印信息 
    // [微应用执行] this 是否是子应用的 window:  true
    console.log('[微应用执行] this 是否是子应用的 window: ', this === contentWindow);
    
    // 打印信息 
    // [proxy get 执行] 拦截的 prop:  a
    // [微应用执行] window.a:  2
    console.log('[微应用执行] window.a: ', window.a);
    
    // 没有触发 Proxy 的 set 函数，因为属性挂载在没有被代理的 iframe 的 window 对象
    b = 2;
    
    // 打印信息 
    // [proxy get 执行] 拦截的 prop:  b
    // [微应用执行] window.b:  2
    console.log('[微应用执行] window.b: ', window.b);
    
    })(window.proxy, window)`;

  const scriptElement =
    iframe.contentWindow.document.createElement("script");
  scriptElement.textContent = microCode;
  // 添加内嵌的 script 元素时会自动触发 JS 的解析和执行
  iframe.contentWindow.document.head.appendChild(scriptElement);
}

// 省略其余代码

// 控制台打印信息 [主应用执行] window.a:  1
console.log('[主应用执行] window.a: ', window.a);
// 控制台打印信息 [主应用执行] window.b:  undefined
console.log('[主应用执行] window.b: ', window.b);
```

从下图打印的信息可以可以发现， 在立即执行匿名函数中的 `var` 声明没有被拦截，这和微应用独立运行的结果不一致。而 `this` 赋值和未限定标识符变量时，可以正确设置 `window` 属性，但是没有通过 Proxy 代理，因为赋值行为不是发生在代理的 `window` 对象上，而是发生在没有被代理的 `window` 对象上：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64cbfa652ecc404fbc06c5a0c980a50b~tplv-k3u1fbpfcp-zoom-1.image)

如果希望 `this` 也可以受到 Proxy 代理的管控，那么可以重新绑定 `this` 的指向：

``` javascrpt
function execMicroCode() {
    
    // 注意传入了第二个参数 contentWindow，这是没有被代理的 iframe 的 window 对象
    const microCode = `(function(window, contentWindow){
      
      // 打印信息 
      // [proxy set 执行] 拦截的 prop:  a
      this.a = 2;
      
      // 打印信息 
      // [proxy get 执行] 拦截的 prop:  parent
      // this 是否是主应用的 window:  false
      console.log('[微应用执行] this 是否是主应用的 window: ', this === window.parent);
      
      // 打印信息 
      // [微应用执行] this 是否是子应用的 window:  true
      console.log('[微应用执行] this 是否是子应用的 window: ', this === contentWindow);
      
      // 打印信息 
      // [proxy get 执行] 拦截的 prop:  proxy
      // [微应用执行] this 是否是子应用的 window.proxy:  true
      console.log('[微应用执行] this 是否是子应用的 window.proxy: ', this === window.proxy);

    // 将内部的 this 指向 window 的代理对象
    }).bind(window.proxy)(window.proxy, window)`;
  
    const scriptElement =
      iframe.contentWindow.document.createElement("script");
    scriptElement.textContent = microCode;
    // 添加内嵌的 script 元素时会自动触发 JS 的解析和执行
    iframe.contentWindow.document.head.appendChild(scriptElement);
}
```

因此具备了上述的 iframe + Proxy 隔离设计后：

-   可以解决 JS 运行环境的隔离问题，当然除了 `history` 故意不进行隔离
-   微应用在使用 `var` 时需要挂载在全局对象上的能力缺失
-   可以解决微应用之间的全局属性隔离问题，包括使用未限定标识符的变量、`this`
-   使用 `this` 时访问是 iframe 的 `window` 代理对象，可以和主应用的 `this` 隔离
-   可以解决 iframe 隔离中无法进行 `history` 操作和同步的问题

> 温馨提示：可以将 Proxy 理解为微应用隔离的逃生窗口，在上述设计中可以将 `history` 理解为隔离的白名单，在后续的设计中需要精细化考虑子应用的 `history` 操作不能对主应用产生额外的影响。

## Proxy + With

基于上述的 Proxy 示例，我们会发现微应用中的 `var` 在主应用中运行时无法挂载在全局属性上。除此之外，我们单独使用 `history` 而不是 `window.history` 进行访问时也无法进行拦截：

``` javascript
 const microCode = `(function(window){
    // 没有触发 Proxy 的 set 函数，由于在立即执行的匿名函数中执行 var，此时不是全局变量
    var a = 2;
    
    // 触发 Proxy 的 get
    
    // 打印信息 
    // [proxy get 执行] 拦截的 prop:  a
    // [微应用执行] window.a:  undefined
    console.log('[微应用执行] window.a: ', window.a);
    
    // window.history.pushState({}, '', '/micro');
    // 如果不使用 window.history，那么无法进行拦截，此时使用的仍然是子应用的 history，执行会报错
    history.pushState({}, '', '/micro');

})(window.proxy)`;
```

关于 `history` 的问题解决方案有很多，这里可以列举多种实现方式：

-   在执行微应用的代码前修改 `contentWindow.history`，使其指向主应用的 `history`
-   在立即执行的匿名函数中传入第二个形参 `history`，指向 `widnow.proxy.history`
-   对 `contentWindow.history` 进行单独代理，并传入立即执行的匿名函数

而在函数中声明 `var` 局部变量无法将属性挂载在全局对象上，我们希望微应用独立运行和嵌入主应用运行的行为应该保持一致，为了解决该问题可以使用 `with` 配合 `Proxy` 实现。

> 温馨提示：一般情况下不需要考虑 `var` 声明的问题，例如在 Vue 或者 React 框架中使用的 `var` 都是在模块中声明的变量，不会挂载在全局属性上。除此之外，在写代码时大概率也不会采用上述的设计风格（声明了 `var` 以后不直接访问，而是通过 `window` 进行访问）。本课程讨论该问题单纯是为了讲述该问题的解决方案。

`with` 设计的本意用于缩短查找作用域链，并且可以减少变量的使用长度，例如：

``` javascript
var person = {
    name: "ziyi",
    address: {
      country: "China",
      city: "Hangzhou",
    },
  };

  with (person) {
    console.log("name: ", name); // ziyi
    with (address) {
      console.log("country: ", country); // China
      console.log("city: ", city); // Hangzhou
    }
  }
```

在上述示例中，通过 `with` 语句将 `person` 对象的所有属性和方法添加到了当前作用域链的顶端（作用域链的最底下是全局对象 `window`），因此在 `with (person)` 内部需要访问 `person.name` 时可以直接使用 `name`，本质上在查找 `name` 的过程中先判断 `name in person`，如果返回的是 `true`，则直接获取 `person.name` 的值。有了 `with` 的能力后，可以使用 Proxy 的 `has` 对 `in` 操作进行拦截，从而对 `var` 进行拦截处理：

``` javascript
const person = {}

with(person) {
  // 此时声明的 var 在全局作用域下，会添加属性到 window 上
  // 注意块级作用域对 var 没有任何作用
  var a = 2;
  console.log('person a: ', person.a); // undefined
  console.log(window.a); // 2
}

const person1 = new Proxy({}, {
  set(target, prop, value) {
    console.log('触发 set: ', prop);
    target[prop] = value
  },
    
  // has 可以拦截的操作类型：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/has#%E6%8B%A6%E6%88%AA
  // has 会对 with(proxy) { (foo); }  中的 foo 访问进行拦截 
  // has 返回为 true 时告诉 with 语句，with(proxy) 的 proxy 对象一定存在 foo 属性（实际上可能不存在 foo 属性）
  // 一旦 foo in proxy 为 true，那么 foo 就不会沿着原型链进行向上查找，从而切断了 with 中变量对象的原型链查找能力
  has(target, prop) {
    console.log('触发 has: ', prop);
    return true
  }
})

with(person1) {
  // 实现对 var 的拦截操作，此时的 var 被拦截，无法到达全局对象
  // 触发 has:  a
  // 触发 set:  a
  var a = 2;
}
```


了解了上述能力后，我们在 Proxy 示例的基础上进行如下代码的尝试：

``` javascript
iframe.contentWindow.proxy = new Proxy(iframe.contentWindow, {
        get: (target, prop) => {
          console.log("[proxy get 执行] 拦截的 prop: ", prop);

          if (prop === "history") {
            return window[prop];
          }

          // 访问 window.window 时返回 proxy
          // 访问 proxy.history 时 history 可以被拦截处理
          if(prop === 'window') {
            return iframe.contentWindow.proxy;
          }

          // target 就是被代理的 iframe.contentWindow
          // 除了 history，其余都返回 iframe 上 window 的属性值
          return target[prop];
        },

        set: (target, prop, value) => {
          console.log("[proxy set 执行] 拦截的 prop: ", prop);
          target[prop] = value;
        },

        
        // 拦截 in 操作符
        // has 可以拦截的操作类型：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/has#%E6%8B%A6%E6%88%AA
        // 返回为 true 时告诉 with 语句，一定存在属性，禁止向上进行原型链查找
        has: (target, prop) => true
      });



const microCode = `(function(window){
    with(window) {
         // 打印信息
         // [proxy get 执行] 拦截的 prop:  Symbol(Symbol.unscopables)
         // [proxy set 执行] 拦截的 prop:  a
         var a = 2;
            
         // 打印信息   
         // [proxy get 执行] 拦截的 prop:  Symbol(Symbol.unscopables)
         // [proxy get 执行] 拦截的 prop:  console
         // [proxy get 执行] 拦截的 prop:  Symbol(Symbol.unscopables)
         // [proxy get 执行] 拦截的 prop:  window
         // [proxy get 执行] 拦截的 prop:  a
         // with a:  2
         console.log('with a: ', window.a); // 注意访问 window.a 在 with 下是访问了 window.window.a
         
         // 打印信息
         // [proxy get 执行] 拦截的 prop:  Symbol(Symbol.unscopables)
         // [proxy get 执行] 拦截的 prop:  history
         history.pushState({}, '', '/micro1');
         
         // 打印信息
         // [proxy get 执行] 拦截的 prop:  window   
         // [proxy get 执行] 拦截的 prop:  history
         window.history.pushState({}, '', '/micro2'); // 在 with 下是访问了 window.window.history
    }

}).bind(window.proxy)(window.proxy)`;
```

> 温馨提示：感兴趣的同学可以额外了解一下 `Symbol.unscopables` 的作用。

具备了上述 iframe + Proxy + With 的隔离设计后：

-   可以解决 JS 运行环境的隔离问题，当然除了 `history` 故意不进行隔离
-   微应用在使用 `var` 时可以挂载在全局对象上
-   可以解决微应用之间的全局属性隔离问题，包括使用未限定标识符的变量、`this`
-   使用 `this` 时访问是 iframe 的 `window` 代理对象，可以和主应用的 `this` 隔离
-   可以解决 iframe 隔离中无法进行 `history` 操作和同步的问题

> 温馨提示：使用 `with` 可能会产生性能损耗，例如在 `with(window)` 下访问 `window` 时事实上是访问了 `window.window`，导致拦截的频率变高。如果不考虑全局作用域的 `var` 兼容问题，那么完全可以去掉 `with` 从而减少性能损耗。在 qiankun 中为了使内部的微应用执行可以快速访问 `history`、`location` 等，专门进行了作用域内的局部声明，从而防止作用域链查找带来的性能损耗。

## 修正 `this` 指向

使用 Proxy 加载微应用时，由于内部使用的是代理后的 `window` 对象，会产生如下问题：

``` javascript
const microCode = `(function(window){
    with(window) {
       // Uncaught TypeError: Illegal invocation
       // 此时 `this` 指向了 proxy，而不是 iframe 的 window 对象
       window.alert(1);
       // Uncaught TypeError: Illegal invocation
       window.addEventListener('load', (e) => {});
    }

}).bind(window.proxy)(window.proxy)`;
```

> 温馨提示：阅读 ["Illegal invocation" errors in JavaScript](https://mtsknn.fi/blog/illegal-invocations-in-js/) 了解更多详情信息。

可能上述示例需要一定的理解能力，我们可以换一个示例：

``` javascript
// 正常运行
alert(1);
// 正常运行
window.alert(1);

const obj = {
  // 将 window.alert 赋值给 obj.alert
  alert
}

// Uncaught TypeError: Illegal invocation
// 因为内部的 alert 执行的 this 指向了 obj
// 此时所需要的执行上下文是 window，否则会报错 Illegal invocation
obj.alert(1);

// 正常运行，因为修正了 this 的指向
obj.alert.bind(window)(1);
```

为了修正 `this` 指向，需要将 `proxy` 中的原生函数调用指向原始的 `window` 对象，但是在这个识别过程中还需要考虑其他情况，例如有些函数已经进行了 `bind`，那么不应该进行再次的绑定操作，此时可以通过函数名来识别，例如：

``` javascript
// MDN: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/name#%E7%BB%91%E5%AE%9A%E5%87%BD%E6%95%B0%E7%9A%84%E5%90%8D%E7%A7%B0
function foo() {};
// 绑定后生成的函数的 name 属性会以 bound 开头
foo.bind({}).name; // "bound foo"
```

除此之外，一些构造函数不需要进行 `bind` 操作，因为 `bind` 生成的函数会失去原有函数的属性和 `prototype`：

``` javascript
// 也可以是 function Person {} 构造函数
class Person {
    constructor(name) {
      this.name = name;
    }
    getName() {
      return this.name;
    }
}

// 添加一个属性
Person.age = 1;

console.log("Person.prototype: ", Person.prototype); // {constructor: ƒ, getName: ƒ}
console.log("Person.age: ", Person.age); // 1

// 假设不小心进行了 bind 操作
// bind 的 this 指向 Person，bind 的参数为 window，简单理解为 Person.bind(window)
// 返回的是一个改变 this 指向 window 的新的 Person 构造函数
const BoundPerson = Function.prototype.bind.call(Person, window);

// ECMA 2022：https://262.ecma-international.org/13.0/#sec-function.prototype.bind (无法跳转可以搜索 Function.prototype.bind)
// Function objects created using Function.prototype.bind are exotic objects. They also do not have a "prototype" property.
// 从打印信息可以发现 bind 之后的函数失去了原有函数的 prototype 和属性
console.log("BoundPerson.prototype: ", BoundPerson.prototype); // undefined
console.log("BoundPerson.age: ", BoundPerson.age); // undefined

// window.name、this.name 都可以获取
var name = "global ziyi";

// MDN 构造函数使用的绑定函数：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind#%E4%BD%9C%E4%B8%BA%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0%E4%BD%BF%E7%94%A8%E7%9A%84%E7%BB%91%E5%AE%9A%E5%87%BD%E6%95%B0
// 绑定函数自动适应于使用 new 操作符去构造一个由目标函数创建的新实例。当一个绑定函数是用来构建一个值的，原来提供的 this 就会被忽略。不过提供的参数列表仍然会插入到构造函数调用时的参数列表之前。
const person = new BoundPerson("ziyi");
// 构造函数绑定的 this 指向被忽略，仍然指向新实例，因此打印的不是 global ziyi
console.log("person.name: ", person.getName()); // ziyi
```

为了修复上述问题，可以在 iframe 的 `window` 被拦截时对 `prop` 进行判断，通过 `bind` 对 `window.alert` 、`window.addEventListener`、`window.atob` 等进行 `this` 修正：

``` javascript
// 1. 修正 window.alert、window.addEventListener 等 API 的 this 指向，需要识别出这些函数
// 2. 过滤掉已经做了 bind 的函数
// 3. 过滤掉构造函数，例如原生的 Object、Array 以及用户自己创建的构造函数等

function isFunction(value) {
   return typeof value === "function";
}

function isBoundedFunction(fn) {
   // 被绑定的函数本身没有 prototype
   return fn.name.indexOf("bound ") === 0 && !fn.hasOwnProperty("prototype")
}

// 是否是构造函数（这个实现比较复杂，这里可以简单参考 qiankun 实现）
function isConstructable() {
  // 可以过滤 Object、Array 等
  return (
    // 过滤掉箭头函数、 async 函数等。这些函数没有 prototype
    fn.prototype &&
    // 通常情况下构造函数的 prototype.constructor 指向本身
    fn.prototype.constructor === fn &&
    // 通常情况下构造函数和类都会存在 prototype.constructor，因此长度至少大于 1
    // 需要注意普通函数中也会存在 prototype.constructor，
    // 因此如果 prototype 有自定义属性或者方法，那么判定为类或者构造函数，因此这里的判断是大于 1
    // 注意不要使用 Object.keys 进行判断，Object.keys 无法获取 Object.defineProperty 定义的属性
    Object.getOwnPropertyNames(fn.prototype).length > 1
  );
  // TODO: 没有 constructor 的构造函数识别、class 识别、function Person() {} 识别等
  // 例如 function Person {};  Person.prototype = {}; 此时没有 prototype.constructor 属性
}


// 最后可以对 window 的属性进行修正，以下函数执行在 Proxy 的 get 函数中
function getTargetValue(target, prop) {
    const value = target[prop];
    // 过滤出 window.alert、window.addEventListener 等 API 
    if(isFunction(value) && !isBoundedFunction(value) && !isConstructable(value)) {
        // 修正 value 的 this 指向为 target 
        const boundValue = Function.prototype.bind.call(value, target);
        // 重新恢复 value 在 bound 之前的属性和原型（bind 之后会丢失）
        for (const key in value) {
          boundValue[key] = value[key];
        }
        // 如果原来的函数存在 prototype 属性，而 bound 之后丢失了，那么重新设置回来
        if(value.hasOwnProperty("prototype") && !boundValue.hasOwnProperty("prototype")) {
            boundValue.prototype = value.prototye;
        }
        return boundValue;
    }
    return value;
}    
```

> 温馨提示：实际在微应用使用的情况千变万化，上述设计并不能覆盖所有场景，如果想了解社区框架的完善程度，可以查看 qiankun 框架中的 `getTargetValue` 处理。


## 隔离方案设计

了解了上述 Proxy 和 with 的作用后，我们可以重新修改 iframe 隔离中的空白页隔离方案，具体方案如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09b0b67875ae4dc89f6e5530eb1d30a5~tplv-k3u1fbpfcp-zoom-1.image)

  


实现效果如下所示，图中的两个按钮（微应用导航）根据后端数据动态渲染，点击按钮后会跨域请求微应用的 JS 静态资源并创建空白的 iframe 进行隔离执行：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3384d75f71f146e98fa4489297d49fdd~tplv-k3u1fbpfcp-zoom-1.image)

文件的结构目录如下所示：

``` bash
├── public                  # 托管的静态资源目录
│   ├── main/               # 主应用资源目录                        
│   │   └── index.html                                        
│   └── micro/              # 微应用资源目录
│        ├── micro1.js        
│        └── micro2.js      
├── config.js               # 公共配置
├── main-server.js          # 主应用服务
└── micro-server.js         # 微应用服务
```

> 温馨提示：示例源码可以从 micro-framework 的 [demo/iframe-proxy-sandbox](https://github.com/ziyi2/micro-framework/tree/demo/iframe-proxy-sandbox) 分支获取。

对前端微应用进行隔离测试：

``` javascript
// public/micro/micro1.js
this.a = 'a';
console.log('微应用1 a: ', a);

var b = 'b';
console.log('微应用1 b: ', window.b);

window.c = 'c';
console.log('微应用1 c: ', window.c);

let root = document.createElement("button");
root.textContent = "微应用 1 更改 history 为 micro1";
document.body.appendChild(root);

// 尽管 iframe 设置了 src="about:blank"，但是由于 history 被代理成主应用的 history，因此不会出错
root.onclick = () => {
  history.pushState({}, '', '/micro1');
}

// public/micro/micro2.js
this.a = 1;
console.log("微应用2 a: ", a);

var b = 2;
console.log("微应用2 b: ", window.b);

window.c = 3;
console.log("微应用2 c: ", window.c);

let root = document.createElement("button");
root.textContent = "微应用 2 更改 history 为 micro2";
document.body.appendChild(root);

// 尽管 iframe 设置了 src="about:blank"，但是由于 history 被代理成主应用的 history，因此不会出错
root.onclick = () => {
  history.pushState({}, "", "/micro2");
};
```

在原有空白 iframe 隔离的基础上对 `MicroAppSandbox` 类进行修改，具体如下所示：

``` javascript
<!-- public/main/index.html -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Main App Document</title>
  </head>

  <body>
    <h1>Hello，Sandbox Script!</h1>

    <!-- 主应用导航 -->
    <div id="nav"></div>

    <!-- 主应用内容区 -->
    <div id="container"></div>

    <script type="text/javascript">
      // 隔离类
      class IframeSandbox {
        // 沙箱配置信息
        options = null;
        // iframe 实例
        iframe = null;
        // iframe 的 Window 实例
        iframeWindow = null;
        // 是否执行过 JS
        execScriptFlag = false;

        constructor(options) {
          this.options = options;
          // 创建 iframe 时浏览器会创建新的全局执行上下文，用于隔离主应用的全局执行上下文
          this.iframe = this.createIframe();
          this.iframeWindow = this.iframe.contentWindow;
          this.proxyIframeWindow();
        }

        createIframe() {
          const { rootElm, id, url } = this.options;
          const iframe = window.document.createElement("iframe");
          const attrs = {
            src: "about:blank",
            "app-id": id,
            "app-src": url,
            style: "border:none;width:100%;height:100%;",
          };
          Object.keys(attrs).forEach((name) => {
            iframe.setAttribute(name, attrs[name]);
          });
          rootElm?.appendChild(iframe);
          return iframe;
        }

        isBoundedFunction(fn) {
          return (
            // 被绑定的函数本身没有 prototype
            fn.name.indexOf("bound ") === 0 && !fn.hasOwnProperty("prototype")
          );
        }

        isConstructable(fn) {
          // 可以识别 Object、Array 等原生构造函数，也可以识别用户自己创建的构造函数
          return (
            fn.prototype &&
            // 通常情况下构造函数和类的 prototype.constructor 指向本身
            fn.prototype.constructor === fn &&
            // 通常情况下构造函数和类都会存在 prototype.constructor，因此长度至少大于 1
            // 需要注意普通函数中也会存在 prototype.constructor，
            // 因此如果 prototype 有自定义属性或者方法，那么可以判定为类或者构造函数，因此这里的判断是大于 1
            // 注意不要使用 Object.keys 进行判断，Object.keys 无法获取 Object.defineProperty 定义的属性
            Object.getOwnPropertyNames(fn.prototype).length > 1
          );
          // TODO: 没有 constructor 的构造函数识别、class 识别、function Person() {} 识别等
          // 例如 function Person {};  Person.prototype = {}; 此时没有 prototype.constructor 属性
        }

        // 修复 window.alert、window.addEventListener 等报错 Illegal invocation 的问题
        // window.alert 内部的 this 不是指向 iframe 的 window，而是指向被代理后的 proxy，因此在调用 alert 等原生函数会报错 Illegal invocation
        // 因此这里需要重新将这些原生 native api 的 this 修正为 iframe 的 window
        getTargetValue(target, prop) {
          const value = target[prop];

          // 过滤出 window.alert、window.addEventListener 等 API
          if (
            typeof value === "function" &&
            !this.isBoundedFunction(value) &&
            !this.isConstructable(value)
          ) {

            console.log('修正 this: ', prop);

            // 修正 value 的 this 指向为 target
            const boundValue = Function.prototype.bind.call(value, target);
            // 重新恢复 value 在 bound 之前的属性和原型（bind 之后会丢失）
            for (const key in value) {
              boundValue[key] = value[key];
            }
            // 如果原来的函数存在 prototype 属性，而 bound 之后丢失了，那么重新设置回来
            if (
              value.hasOwnProperty("prototype") &&
              !boundValue.hasOwnProperty("prototype")
            ) {
              boundValue.prototype = value.prototye;
            }
            return boundValue;
          }
          return value;
        }

        proxyIframeWindow() {
          this.iframeWindow.proxy = new Proxy(this.iframeWindow, {
            get: (target, prop) => {
              // console.log("get target prop: ", prop);

              // TODO: 这里只是课程演示，主要用于解决 src:about:blank 下的 history 同域问题，并没有真正设计主子应用的路由冲突问题，后续的课程会进行该设计
              // 思考：为了防止 URL 冲突问题，是否也可以形成设计规范，比如主应用采用 History 路由，子应用采用 Hash 路由，从而确保主子应用的路由不会产生冲突的问题
              if (prop === "history" || prop === "location") {
                return window[prop];
              }

              if (prop === "window" || prop === "self") {
                return this.iframeWindow.proxy;
              }

              return this.getTargetValue(target, prop);
            },

            set: (target, prop, value) => {
  
              target[prop] = value;
              return true;
            },

            has: (target, prop) => true,
          });
        }

        execScript() {
          const scriptElement =
            this.iframeWindow.document.createElement("script");
          scriptElement.textContent = `
              (function(window) {
                with(window) {
                  ${this.options.scriptText}
                }
              }).bind(window.proxy)(window.proxy);
              `;
          this.iframeWindow.document.head.appendChild(scriptElement);
        }

        // 激活
        async active() {
          this.iframe.style.display = "block";
          // 如果已经通过 Script 加载并执行过 JS，则无需重新加载处理
          if (this.execScriptFlag) return;
          this.execScript();
          this.execScriptFlag = true;
        }

        // 失活
        // INFO: JS 加载以后无法通过移除 Script 标签去除执行状态
        // INFO: 因此这里不是指代失活 JS，如果是真正想要失活 JS，需要销毁 iframe 后重新加载 Script
        inactive() {
          this.iframe.style.display = "none";
        }

        // 销毁沙箱
        destroy() {
          this.options = null;
          this.exec = false;
          if (this.iframe) {
            this.iframe.parentNode?.removeChild(this.iframe);
          }
          this.iframe = null;
        }
      }

      // 微应用管理
      class MicroAppManager {
        // 缓存微应用的脚本文本（这里假设只有一个执行脚本）
        scriptText = "";
        // 隔离实例
        sandbox = null;
        // 微应用挂载的根节点
        rootElm = null;

        constructor(rootElm, app) {
          this.rootElm = rootElm;
          this.app = app;
        }

        // 获取 JS 文本（微应用服务需要支持跨域请求）
        async fetchScript(src) {
          try {
            const res = await window.fetch(src);
            return await res.text();
          } catch (err) {
            console.error(err);
          }
        }

        // 激活
        async active() {
          // 缓存资源处理
          if (!this.scriptText) {
            this.scriptText = await this.fetchScript(this.app.script);
          }

          // 如果没有创建沙箱，则实时创建
          // 需要注意只给激活的微应用创建 iframe 沙箱，因为创建 iframe 会产生内存损耗
          if (!this.sandbox) {
            this.sandbox = new IframeSandbox({
              rootElm: this.rootElm,
              scriptText: this.scriptText,
              url: this.app.script,
              id: this.app.id,
            });
          }

          this.sandbox.active();
        }

        // 失活
        inactive() {
          this.sandbox?.inactive();
        }
      }

      // 微前端管理
      class MicroManager {
        // 微应用实例映射表
        appsMap = new Map();
        // 微应用挂载的根节点信息
        rootElm = null;

        constructor(rootElm, apps) {
          this.rootElm = rootElm;
          this.setAppMaps(apps);
        }

        setAppMaps(apps) {
          apps.forEach((app) => {
            this.appsMap.set(app.id, new MicroAppManager(this.rootElm, app));
          });
        }

        // TODO: prefetch 微应用
        prefetchApps() {}

        // 激活微应用
        activeApp(id) {
          const current = this.appsMap.get(id);
          current && current.active();
        }

        // 失活微应用
        inactiveApp(id) {
          const current = this.appsMap.get(id);
          current && current.inactive();
        }
      }

      // 主应用管理
      class MainApp {
        microApps = [];
        microManager = null;

        constructor() {
          this.init();
        }

        async init() {
          this.microApps = await this.fetchMicroApps();
          this.createNav();
          this.navClickListener();
          this.hashChangeListener();
          // 创建微前端管理实例
          this.microManager = new MicroManager(
            document.getElementById("container"),
            this.microApps
          );
        }

        // 从主应用服务器获请求微应用列表信息
        async fetchMicroApps() {
          try {
            const res = await window.fetch("/microapps", {
              method: "post",
            });
            return await res.json();
          } catch (err) {
            console.error(err);
          }
        }

        // 根据微应用列表创建主导航
        createNav(microApps) {
          const fragment = new DocumentFragment();
          this.microApps?.forEach((microApp) => {
            // TODO: APP 数据规范检测 (例如是否有 script）
            const button = document.createElement("button");
            button.textContent = microApp.name;
            button.id = microApp.id;
            fragment.appendChild(button);
          });
          nav.appendChild(fragment);
        }

        // 导航点击的监听事件
        navClickListener() {
          const nav = document.getElementById("nav");
          nav.addEventListener("click", (e) => {
            // 并不是只有 button 可以触发导航变更，例如 a 标签也可以，因此这里不直接处理微应用切换，只是改变 Hash 地址
            // 不会触发刷新，类似于框架的 Hash 路由
            window.location.hash = event?.target?.id;
          });
        }

        // hash 路由变化的监听事件
        hashChangeListener() {
          // 监听 Hash 路由的变化，切换微应用（这里设定一个时刻只能切换一个微应用）
          window.addEventListener("hashchange", () => {
            this.microApps?.forEach(async ({ id }) => {
              id === window.location.hash.replace("#", "")
                ? this.microManager.activeApp(id)
                : this.microManager.inactiveApp(id);
            });
          });
        }
      }

      new MainApp();
    </script>
  </body>
</html>
```

可以发现在空白 iframe 隔离的基础上增强了能力设计：

-   解决了 iframe `src="about:blank"` 时的 `history` 报错问题
-   后续还可以增强子应用和主应用的 URL 同步问题
-   DOM 环境天然隔离，但是无法处理 Modal 相对于主应用的居中问题

## 小结

本课程主要讲解了如何利用 iframe + Proxy 实现 JS 隔离，该隔离在 iframe 隔离的基础上解决了 `history` 的报错问题，从而可以兼容 Vue 或者 React 框架的隔离执行。当然，由于微应用和主应用共用 `history`，还可以额外去实现主子应用的历史会话同步问题。由于 Proxy 存在浏览器兼容性，在下一节课程中我们会简单讲解快照隔离方案。