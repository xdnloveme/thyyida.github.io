---
layout:     post
title:      "bundleless热更新原理探索"
subtitle:   "ESM模块化探索"
date:       2020-11-19
author:     "Leon"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 前端开发
    - JavaScript
    - bundleless
---

> 原理是bundleless + react-refresh + websocket 通信实现热更新

## 前言

最近实践了bundleless，在热更新上栽了很多跟头，这次更新一下最近热更新原理探索的实践，我们先来梳理一下Bundleless + react-refresh实现热更新的时序图：

![时序图](![alt text](../img/image.png))

上面时序图需要注意的点有几点：

1. 需要向客户端HTML注入一些client.js的Websocket客户端代码，来响应热更新的文件系统，类似于webpack在配置entry入口，注入需要的热更新代码

2. 在React热更新提供方的react-refresh，需要在**整个组件树**之前注入注册器代码同时注册组件树，下面是援引官方的说明：

> Then you need to create a new JS entry point which *must run before any code in your app*, including `react-dom` (!) This is important; if it runs after `react-dom`, nothing will work. That entry point should do something like this:

   关于react-refresh的介绍和详细的用法，在[https://github.com/facebook/react/issues/16604](https://github.com/facebook/react/issues/16604)这篇文章中有详细的叙述，可以提供参阅。

3. 在接受改变组件树的同时，需要主动accept，接受变化的消息，在webpack中通过

   ```javascript
   module.hot.accept();
   ```

   来实现模块的接收，同时告诉页面去渲染，在bundleless中也是类似，通过**import动态加载模块并执行accept方法（存在import.meta 元数据中）**来接收新的模块，援引官方的叙述：

> Once you hook this up, you have one last problem. Your bundler doesn't know that you're handling the updates, so it probably reloads the page anyway. You need to tell it not to. This is again bundler-specific, but the approach I suggest is to check whether *all of the exports are React components*, and in that case, "accept" the update. In webpack it could look like something:

   ```javascript
   // ...ALL MODULE CODE...
   
   const myExports = module.exports; 
   // Note: I think with ES6 exports you might also have to look at .__proto__, at least in webpack
   
   if (isReactRefreshBoundary(myExports)) {
     module.hot.accept(); // Depends on your bundler
     enqueueUpdate();
   }
   ```

在上述的[https://github.com/facebook/react/issues/16604](https://github.com/facebook/react/issues/16604) 有详细介绍。

下面详细说一下我在应用bundless + react-refresh中遇到的坑和几个问题。

## react-refresh注册组件必须在最前面

之前遇到没有注册上去的情况，原因有好几种

### 没有放在最前面

这段代码：

```javascript
import RefreshRuntime from "/@react-refresh" // 重要
RefreshRuntime.injectIntoGlobalHook(window)
window.$RefreshReg$ = () => {}
window.$RefreshSig$ = () => (type) => type
```

全局一些runtime的依赖注入，来保证react-refresh提供的注册组件更新能力可以有效实现，这个一定要在最头部，最好放在head标签内实现，在整个App加载之前执行。

其中比较重要的是`RefreshRuntime`，这个变量实际引入的路径是

```javascript
react-refresh/cjs/react-refresh-runtime.development.js
```

一定要保证引入的这个文件路径的正确，错误会导致无法运行。

如果没有全局指定生产环境变量

```javascript
window.process = { env: { NODE_ENV: "development" }}
```

也会导致无法注册的情况，我当时就是引入错误，一直都是prod环境的，所以导致无效。

## 每个react文件都会被react-refresh/babel改写

因为要配合react-refresh对每个组件进行注册，以后续热更新作为依据，所以每个文件先需要通过babel的插件：react-refresh/babel去转译一下：

```javascript
const result = require('@babel/core').transformSync(code, {
  plugins: [require('react-refresh/babel')],
  ast: false,
  sourceMaps: true,
  sourceFileName: path
});
```

转译出来的代码会带有类似这样的逻辑：

```javascript
// ...
function Test() {
  return /* @__PURE__ */React.createElement("div", null, "test ff33 ajd19s3479");
}

_c = Test;

var _c;

$RefreshReg$(_c, "Test");
// ...
```

这样就相当于注册了这个组件，为后续更新依赖做预置工作。

另外，文件也需要注入下列代码：

```javascript
const header = `
	// 这些逻辑都是向react-refresh注册当前react文件的逻辑，通过文件path正确注册

	import RefreshRuntime from "../node_modules/react-refresh/cjs/react-refresh-runtime.development.js";

	import { createContext } from '/public/HotModuleReplacementClient.js';
	import.meta.hot = createContext('${path}');

	let prevRefreshReg;
	let prevRefreshSig;

	if (import.meta.hot) {
		prevRefreshReg = window.$RefreshReg$;
		prevRefreshSig = window.$RefreshSig$;
		window.$RefreshReg$ = (type, id) => {
		RefreshRuntime.register(type, ${JSON.stringify(path)} + " " + id)
		};
		window.$RefreshSig$ = RefreshRuntime.createSignatureFunctionForTransform;
	}`.replace(/[\n]+/gm, '');

	const footer = `
	if (import.meta.hot) {
		window.$RefreshReg$ = prevRefreshReg;
		window.$RefreshSig$ = prevRefreshSig;
		import.meta.hot.accept();  	// 这是模块接收后的accept
		RefreshRuntime.performReactRefresh(); 	// 这是执行组件树的更新的逻辑
	}`
```

至此，react-refresh可以协同作用了

## accept的作用（只看了vite）？

你经常会看到一些文章说snowpack，vite类似的热更新的实现是通过`import.meta.hot`实现的，糊里糊涂讲了一通，其实没有讲到重点。

其实accept只是**作为一个桥梁**，连接了**模块接收**的作用，accept里面后只是做了一个模块副本数据的存储，负责下一次引入后模块的更新，vite中是这么解释的[https://github.com/vitejs/vite](https://github.com/vitejs/vite)：

> Note that Vite's HMR does not actually swap the originally imported module: if an accepting module re-exports imports from a dep, then it is responsible for updating those re-exports (and these exports must be using `let`). In addition, importers up the chain from the accepting module will not be notified of the change.
>
> This simplified HMR implementation is sufficient for most dev use cases, while allowing us to skip the expensive work of generating proxy modules.

而真正实现热更新功能的，只是websocket和react-refresh，其实只需要这两者就能简单实现热更新了，react-refresh注册组件，ws通知前端主动import，动态加载js执行，react-refresh负责通知组件树进行组件渲染，如下所示：

```javascript
const client = new WebSocket('ws://localhost:8000');
const moduleMap = new Map();

client.addEventListener('open', function (event) {
    client.send('start');
});

client.addEventListener('message', function (payload) {
	console.log('received', payload);
	updateModule(payload.data)
})

async function updateModule (id) {
  // 动态import引入
	await import(`${id}?r=${new Date().getTime()}`);
}

export const createContext = function (id) {
	return {
    // 外部调用accept触发
		accept() {
			// todo
      // moduleMap 存储副本
		}
	}
}

```

在这里有个重要的问题BUG需要注意一下，如果你的代码里面有类似的逻辑：

```javascript
document.appendChild(node)
```

会导致一个很奇葩的BUG：因为js是import动态引入的，所以会导致`document.appendChild(node)`再次执行，重复执行会导致插入两次节点。

## css热更新？

还有一点也要重点关注，如何实现的CSS热更新，其实都模块化以后，CSS也要向模块化靠拢。

bundleless的解决方案是，把CSS当做**普通文件处理，读取内容后，通过转成js代码执行，插入到CSSStyleSheet**：

```javascript
const fs = require('fs');

module.exports = function (url, context) {
	const cssContent = fs.readFileSync(`${context}${url}`).toString();

	const testStyle = `
		var addSheets = function (content) {
			var style = new CSSStyleSheet();
            style.replaceSync(content);
            document.adoptedStyleSheets = [...document.adoptedStyleSheets, style];
		}
		addSheets(cssCode);
	`
	const content = `var cssCode = \`${cssContent.toString()}\`; ${testStyle}`;

	return content;
}

```

[CSSStyleSheet](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet)是一个比较特殊的存在，vite里面就靠CSSStyleSheet的css表来存储css内容，通过一个Map文件和hash值来映射每个模块和css文件的关系，整个页面的渲染都依赖于这个CSS表，通过更新Map数据内的css内容来反馈到组件模块。更详细的可以参考[MDN](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet)。

## 后续

这是第一版的bundleless热更新总结，有些纰漏 ，希望后续可以一起补充完全热更新的细节和坑。
