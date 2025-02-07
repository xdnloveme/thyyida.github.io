---
layout:     post
title:      "从Vite工具看ESM模块化开发探索（一）"
subtitle:   "ESM模块化探索"
date:       2020-07-21
author:     "Leon"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 前端开发
    - JavaScript
    - ESM
---

> 对于公司内部现有的Webpack打包流程认识后，发现由于WebGL，threejs，pixijs等包的本身性能、体量瓶颈，导致我们代码进行rebuild或者HotReplacement热更新的时候会导致速度非常慢的情况

场景大，业务复杂，代码多导致现有开发模式有以下业务痛点：

1. 打包编译速度过慢，开发效率低，有时编译甚至要8分钟，热更新最慢每次也要几分钟
2. 热更新不太贴近业务需求，每次都要reload页面，效率低

## 探索起点

我最初有这个想法是看到尤雨溪尤大在Vue3.0计划直播中用到的vite打包工具，当时介绍就说这个打包工具编译速度极快，用的ESModule的规范，当时就产生了浓厚的兴趣

当时就想到我们公司内部的代码，在webpack加持下，那是一个“重”，每次跑项目的时候，我的16G内存的MBP就扛不住了，风扇很响，烫的可以煮鸡蛋，编译时间也是非常头疼。

随后在后续的开发中，我也一直持续在关注尤大仓库的vite代码更新，发现他更新得比较频繁，可以看出尤大对于这个项目还是有一定的投入的。

后续关注中我又了解了snowpack，esbuild等半成品的工具，了解其中的原理后，觉得现在是合适的机会去推广ESModule规范下的实践，在对于前期探索阶段，到底能不能给我们开发带来质的提升，还有待勘察。

## 技术原点

对于ESModule的原理起点，可以从`script`标签的`type="module"`说起：

```html
<script type="module">
	import foo from '../foo.js';
  import bar from '../bar.js'
  
  foo(bar);
  //... to do
</script>
```

现在随着浏览器的发展和技术规范的推进，当代大部分浏览器都已经支持在`type="module"`的`script`标签下直接执行解析`import`语句，在直接执行到这一步的时候，浏览器会**自动根据目录路径去请求路径下的资源**，只要触发了**请求**，我们就可以“拦截”了，把请求的assets资源截取进行处理，返回给浏览器执行。

依照上述的技术理论，只要有一个`index.js`就可以一步一步import不停地向下收集依赖，直至最后一个依赖，依次执行，前几年的打包工具（类似webpack）不就做了这个事吗？当然他做的整合更加复杂和详细，但是现在浏览器都已经支持了，这一步就可以交给浏览器做了，我们要做的，就是对依赖进行解析。

## 技术调研

对于前期的技术调研，我看到阿里的一篇文章[Webpack 打包太慢？来试试 Bundleless 吧！](https://mp.weixin.qq.com/s/Wr9d6yrNWjrmP8_Sxbmzfw)，当时看到这里时，觉得跟我们的业务场景的痛点非常相似：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737134b823a5991~tplv-t2oaga2asx-image.image)

我们内部也是依赖底层支撑，整合多方业务插件的业务架构，由于WebGL，Threejs，pixjs过于臃肿的包而导致整体热更新体验不好，而sourceMap的输出更加加重了架构的负担，首次的build开发体验差劲。

对于当下ESM推广程度的调研，发现存在一些技术问题：

### 1.模块导入语法不支持

因为ESM规范下的所有开发体验都是强依赖于浏览器的，所以对于模块的依赖收集，我们也是依靠浏览器直接解析下列语句而做的：

```javascript
import foo from '../foo.js'
```

foo文件会根据上下文去寻找路径文件，然后直接发起请求，但是问题来了，我们原先不都是这么写的吗？

```javascript
import React from 'react';
```

他会按照预期，找当前目录下的`react.js`吗？答案当然是否定的，原来的打包器会根据require（commonJs规范）去进行映射，引入`node_modules`下对应的模块。

但是现在会报错，浏览器会报以下错误：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737133d46c3f7e9~tplv-t2oaga2asx-image.image)

考虑到浏览器的安全性，是不支持这样直接找模块引入的，而要指定具体的路径，解决办法有吗？当然有

#### 解决模块化引入的问题

我们在触发`import`语句之前，可以通过转译成AST，重写当前import节点路径后，映射到我们服务器的静态资源目录（vite就是这么做的）或者直接映射到node_modules：

```javascript
// 转译前
import React from 'react'

// 转译后
import React from '__Module__react';

// 路径解析的时候替换 __Module__
import React from '../node_modules/react/entry.js'
```

解析成`__Module__react`后可以替换关键字，映射到真实node_module模块，通过`package.json`的`main`字段（寻找主入口，此处为例）进行`lookupFiles`的操作，找到打包后的真实入口，然后直接根据模块路径引入。

当然，记得每个模块引入的时候进行缓存。

### 2. 很多模块不支持ESModule导出

这次调研中，发现强如React，也没有支持ESModule导出，他们给出的答复是：[The React team is working on ESM support](https://github.com/facebook/react/issues/11503)，他们在dist打包中并没有支持ESModule的导出，这也导致了你如果用ESM规范去引入React势必会报错，那么现有解决方案可以解决吗？答案是肯定的

#### Rollup处理ESModule导出

rollup也崇尚ES模块：

> #### Why are ES modules better than CommonJS Modules?
>
> ES modules are an official standard and the clear path forward for JavaScript code structure, whereas CommonJS modules are an idiosyncratic legacy format that served as a stopgap solution before ES modules had been proposed. ES modules allow static analysis that helps with optimizations like tree-shaking and scope-hoisting, and provide advanced features like circular references and live bindings.

可以说更好的ES标准生态，更能明确未来的发展大方向，我们可以通过rollup在服务器启动时进行预优化，可以很好地优化到每个不支持ESM的模块。

### 3. 依赖过多，请求数过多导致实际开发很慢

还是通过rollup，可以再预启动阶段，打包一些文件，有条件的话可以缓存（service-worker），优化下一次请求依赖，减少浏览器因为import语句过多而产生的请求数过多的问题。

### 4. Css,less等样式和其它资源如何解决？

对于以下写法

```javascript
import './index.module.less'
```

其实浏览器是无法解析的，这一点我们也可以借鉴原来打包器的经验，通过import一个js模块的模式引入，通过post-css，less等处理预编译的能力，学习类似`css-loader, style-loader`类似的方式，进行style标签的嵌入样式，此处也可以处理一些hash值和前缀等

### 5.jsx,typescript?

我发现了一个非常好用的基于ESModule规范下的工具esbuild，可以支持构建和转译代码，并且速度极快（go语言并发高），以native代码直接构建，构建速度也是极快的。在加载的时候，就可以通过转译器进行翻译。

### 6.开发环境和生产环境？

我们运用浏览器支持ESModule规范的原理，当然可以享受非常快速的编译和开发，但是随之而来的问题就是，你开发环境的这些配置肯定无法应用到生产环境？毕竟你不能要求所有人都用ESModule支持的浏览器。还有一些语法是需要polyfill来解决兼容问题的。

rollup可以支持这个操作，同时要保证在开发环境的一些rollup打包和生产环境表现是一致的，另外要注意一些babel-runtime的操作，babel应该还没有完全支持类似的打包解析，可能请求路径或者资源和实际会有出入。

### 7. Todo-热更新？

## 技术可探索亮点

### 1. services-worker

对于一些不常变动的资源，因为都是请求，我们甚至可以在SW层面做一些缓存，在对于一些依赖的收集后，可以给出一个配置字段，来配置一些需要长时间本地缓存的打包文件，直接通过SW返回，加快传输速度，提高开发效率

### 2. HTTP2

对于HTTP2提供的Header 压缩和二进制传输对于我们场景内的**大模型数据传输慢**的问题可以有效解决？这个问题待验证，但是不失为一种思路，可以进行探索，而多路复用对于我们场景内大量模型的载入和资源的加载也肯定有相当棒的优势，这个为技术亮点，可以进行探索发掘。

## 落地实践成果预测

### 启动体验

我自己跑了几个demo，效果还是明显的，首先是webpack层面开发环境的：

+ webpack

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737135ae5f0c171~tplv-t2oaga2asx-image.image)

+ vite
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/173713602b44f580~tplv-t2oaga2asx-image.image)

可以看到，首次打包的效率在demo里面表现还是比较明显的，webpack的明显慢许多，vite已经做到了无感知的开发体验。

### 热更新体验

+ webpack

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/173713657ce030de~tplv-t2oaga2asx-image.image)

+ vite

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/17371369823df6fc~tplv-t2oaga2asx-image.image)

### 打包体验

在最终的生产环境打包方面，两者相差无几，webpack甚至还快一点：

+ Webpack

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737136da1407100~tplv-t2oaga2asx-image.image)

+ Vite

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/173713969e540688~tplv-t2oaga2asx-image.image)

### SourceMap

其中比较具有诱惑的点就是SourceMap的功能，用ESM的话会自带这个功能，因为每个请求的地址都是映射到真实的文件，就不需要一些臃肿的SourceMap了，可以直接在Source里面搜索源文件，可以说是非常香了。图中因为vite会做一层对应js文件再静态资源目录的转译，本质上是不变的。

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737141572d1b3ef~tplv-t2oaga2asx-image.image)

## 其他

因为公司级别的项目较复杂，涉及牵扯面比较多，对于打包的迁移成本有点大，这里我们依照阿里内部的落地实践来看，效果还是明显的：

+ webpack

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737141f91fb542d~tplv-t2oaga2asx-image.image)

+ vite(为例)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/21/1737142325982e10~tplv-t2oaga2asx-image.image)

  从阿里实践来看，在启动单个 bundle 时，Webpack 需要 10s 左右的时间，用 Vite （ESModule规范下）只需要 1s 左右，提升 10 倍，热更新更是到达了毫秒级的体验，

## 未来

探索的目的就是为了明确未来的方向，可以说ESModule的规范化和推广度在稳固上升的场景下，利用这些特性使用在开发中是大家所需要的，从尤大每隔几天就提交一次修改记录可以看到他对这个项目也非常上心。

可以说规范化的推进有助于前端的持续发展，此次的探索成果可以说是很有用，想象一下，你开发的时候，不需要再过多等待编译和打包，而是”毫秒级“的体验，加快你的开发效率。从前，你编译一次2分钟，调试一次3分钟，一天下来，浪费在这些无用的地方的时间可能就有小时级别的，那是低效率的表现。

未来的ESM打包本质就是把webpack低效率的依赖收集解析的工作交给浏览器去执行，加快编译转换，提高效率

下一期会从简单demo完成一个打包器，把其中涉及到的知识点巩固一遍。


