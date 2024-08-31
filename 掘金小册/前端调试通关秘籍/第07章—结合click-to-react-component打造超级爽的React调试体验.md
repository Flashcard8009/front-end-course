﻿如果说业务开发中最重要的能力，那定位代码的能力肯定是其中之一。

业务项目一般代码都很多，你拿到一个需求之后，可能改起来不难，但是要定位在哪里改比较难。

特别是接手别人写的代码的时候。

大家都是怎么在不熟悉的项目里定位的代码呢？

很多都学都是搜文案，搜 className。

这样没问题，但如果你用了 styled-component 之类的方案之后，className 都是动态生成的：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2fb18a54ba554a6b8fd0353f1ee38195~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1050&h=778&s=179621&e=png&b=fefefe)

而且不少项目都做了国际化，你搜文案会搜到资源包里，而不是组件代码里：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb62bde803084b8184f17d534431fe97~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=960&h=364&s=113890&e=png&b=1f1f1f)

当然，你可以进一步根据国际化的 key 来搜索源码的对应组件。

但这样总归比较麻烦，而且还不一定能搜到准确的位置。

那有什么好的办法可以快速定位代码么？

有，就是 click-to-react-component。

我们创建个项目：
```
npx create-vite
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7401111e3694e369fd30764cd2c4045~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=936&h=362&s=71234&e=png&b=fefefe)

改下 main.tsx：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b0173e548684acd9848b56300dbc3a3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1066&h=448&s=85571&e=png&b=1f1f1f)

安装 antd，我们随便写几个页面：
```
npm install
npm install --save antd
```
App.tsx：

```javascript
import React from 'react';
import { ColorPicker, Space } from 'antd';
import Aaa from './Aaa';

const Demo = () => (
  <div>
    <Space>
      <Space direction="vertical">
        <ColorPicker defaultValue="#1677ff" size="small" />
        <ColorPicker defaultValue="#1677ff" />
        <ColorPicker defaultValue="#1677ff" size="large" />
      </Space>
      <Space direction="vertical">
        <ColorPicker defaultValue="#1677ff" size="small" showText />
        <ColorPicker defaultValue="#1677ff" showText />
        <ColorPicker defaultValue="#1677ff" size="large" showText />
      </Space>
    </Space>
    <Aaa></Aaa>
  </div>
);

export default Demo;
```

Aaa.tsx：

```javascript
import React, { useState } from 'react';
import { Slider, Switch } from 'antd';
import Bbb from './Bbb';

const Aaa: React.FC = () => {
  const [disabled, setDisabled] = useState(false);

  const onChange = (checked: boolean) => {
    setDisabled(checked);
  };

  return (
    <>
        <div>
            <Slider defaultValue={30} disabled={disabled} />
            <Slider range defaultValue={[20, 50]} disabled={disabled} />
            Disabled: <Switch size="small" checked={disabled} onChange={onChange} />
        </div> 
        <Bbb></Bbb>
    </>
  );
};

export default Aaa;
```
Bbb.tsx：

```javascript
import React from 'react';
import { Card, Space } from 'antd';

const Bbb: React.FC = () => (
  <Space direction="vertical" size={16}>
    <Card title="Default size card" extra={<a href="#">More</a>} style={{ width: 300 }}>
      <p>Card content</p>
      <p>Card content</p>
      <p>Card content</p>
    </Card>
    <Card size="small" title="Small size card" extra={<a href="#">More</a>} style={{ width: 300 }}>
      <p>Card content</p>
      <p>Card content</p>
      <p>Card content</p>
    </Card>
  </Space>
);

export default Bbb;
```
这些都是从 antd 官网复制的 demo 代码。

不用管具体的代码内容，我们只需要看下怎么定位代码。

把开发服务跑起来：

```
npm run dev
```

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35d73f10a8d84e0e9f1fc37cca6f4544~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=864&h=298&s=39718&e=png&b=191919)

渲染出来是这样的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e19d1bccd7941d1a7dcdf5bfbf41dd1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1566&h=1438&s=126412&e=png&b=ffffff)

如果我们想定位下面卡片的代码，就可以通过搜索文案或者 className：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbeeb37e241d4e39aedaa45571526bca~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=774&h=432&s=53716&e=png&b=1e1e1e)

但复杂项目就不行了。

这时候可以引入 click-to-react-component：

```
npm install --save-dev click-to-react-component
```
在 main.tsx 引入下：

```javascript
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.tsx'
// @ts-ignore
import { ClickToComponent } from 'click-to-react-component';

ReactDOM.createRoot(document.getElementById('root')!).render(
    <>
        <ClickToComponent />
        <App />
    </>
)

```
可能有类型的报错，我们直接 @ts-ignore 忽略好了。

然后打开页面试一下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ea76f3023d443c7865e94b3be89ac7e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1732&h=1502&s=1105588&e=gif&f=94&b=fefefe)

可以看到，现在按住 option + 单击，就会直接打开它的对应的组件的源码。

如果按住 option + 右键单击，可以看到它的所有父级组件，然后选择一个组件打开：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/997df95e386e4cad902a7553702a74f7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1114&h=680&s=94966&e=png&b=fdfdfd)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ea205a0a32e452faae136ba207866d6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2224&h=1434&s=1586194&e=gif&f=127&b=fefefe)

这样在页面上看到了啥东西就可以直接打开它的组件代码来改，特别高效。

当然，我们的 demo 比较简单，来看个真实项目里的使用效果：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68d68648c94c4178a7a786ea4071e1e4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2494&h=1648&s=4847416&e=gif&f=162&b=fdfcfc)

比如我想改这个登录弹窗的表单，就可以直接定位到对应组件的 Input。

对于大项目的维护来说真的超级方便。

而且如果把 click-to-react-component 和组件调试结合，调试体验会非常爽。

比如我们要调试这个页面，理清这个输入框的内容是哪里来的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b28277d191f1499592679df67943c675~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1540&h=1306&s=205214&e=png&b=fcfcfc)

我们先在项目里引入 click-to-react-component

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e626b4145584cafb692ee1e6d86bb25~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1232&h=828&s=180924&e=png&b=1f1f1f)

用它来定位这个输入框的源码在哪：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/972cfbbb2ff44abd8c0187a32e7a2bf1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2552&h=1762&s=1189631&e=gif&f=41&b=fdfcfc)

添加调试配置后启动 debug：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8cc29977c3f3432fa7e7032cea348482~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2170&h=702&s=178418&e=png&b=1d1d1d)

然后在刚定位到的组件 return 的地方打个断点：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9253705cd8cc44618be9b68c9f990cb2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2148&h=1470&s=579776&e=png&b=1d1d1d)

发现这个 value 是从父组件通过 props 传进来的。

所以要继续去父组件里打断点。

但是 React 组件渲染的时候调用栈里都是 react 源码，找不到父组件在哪。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71f81ac9d8a64eeca29bb19335eba2b3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1446&h=970&s=295654&e=png&b=1b1b1b)

那怎么办呢？

这时候可以借助 click-to-react-component 展示所有父组件的能力：

按住 option + 右键，点击父组件：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c885fa92d934ef79381d9e3f545a894~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2278&h=1474&s=1203026&e=gif&f=44&b=fcfcfc)

这样就定位到了父组件里渲染这个子组件的源码位置：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd04a9daeea749b1bc578e63e023c1d4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1980&h=1620&s=648178&e=png&b=1d1d1d)

这时候发现这个值依然是通过 props 传进来的。

所以我们用同样的方式继续往上找：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/331e7ab81aed403eaccb848429ebf549~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2300&h=1674&s=1402828&e=gif&f=47&b=fcfcfc)

定位到了一个自定义 hooks，参数都是这里返回的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b8e9ada372a40faacbc1cab7c178a9c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1708&h=1056&s=318166&e=gif&f=18&b=1d1d1d)

在这里打个断点调试下：
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1eedbe28a4c4099b16a97a6690320ab~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1814&h=998&s=1315018&e=gif&f=50&b=1c1c1c)

不断 step into 进入函数内部，最终，我们找到了这个参数的来源：

在 localStorage 里读取了某个 key，key 的名字也调试出来了。

这样我们就达成了调试目标。

整个过程我们根本不需要去理清业务代码的逻辑。

想想看，你是接手这个业务项目的新人，虽然你还没看这些业务代码，但你可以快速定位界面上显示的值的来源，定位到代码在哪里改。

是不是就很高效！

## 总结

对于业务代码来说，快速定位源码是很重要的。

因为改动可能很简单，但是项目大了定位在哪里改就比较麻烦了。

我们也可以通过搜索文案、className 的方式，但对于用了 styled-component、做了国际化的项目来说，这种方式也不行。

所以更推荐用 click-to-react-component 来快速定位源码。

只要在页面上 option + 单击，或者 option + 右键单击然后选一个组件，就可以直接打开对应组件源码的行列。

结合 vscode 断点调试，可以快速定位某个值的来源：

- option + 点击页面上想调试的元素，定位到 VSCode 里的源码
- 打断点看一下值的来源
- 如果是来自父组件，那就用 option + 右键查找父组件，直接定位到源码
- 在定位到的父组件源码里打断点
- 不断往上找，直到找到产生这个值的地方，断点调试

这样，就算你不懂这段业务逻辑，也能快速梳理清楚整个流程，并知道在哪里改代码。
