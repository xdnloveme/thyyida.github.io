---
layout:     post
title:      "从剖析Vue-cli源码出发完整的React业务脚手架实践（二）"
subtitle:   "项目的构建及服务（create）"
date:       2020-05-07
author:     "Leon"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 前端开发
    - JavaScript
    - cli
    - React
---



> 上一篇文章我们介绍了如何搭建项目的架构和脚手架的基础模式，这一章节我们继续上次的业务：项目的构建以及服务，着重从如何构建项目文件目录的流程来剖析。

## 写在前面

这是一篇长期持续更新的React脚手架实践，吸取Vue Cli的脚手架经验，通过我们习惯的**插件-预设**的思想去构造我们的React业务脚手架，这可能不是最好的脚手架的开发实践，但是一定是**最完整的脚手架开发实践教程**。

文章导航：
1. [从剖析Vue-cli源码出发完整的React业务脚手架实践（一）——脚手架架构基础搭建](https://juejin.im/post/6844904131266625549)
2. [从剖析Vue-cli源码出发完整的React业务脚手架实践（二）——项目的构建及服务（create）](https://juejin.im/post/6844904149507637256)

## 引言

上一篇文章，我们写到了`create.js`业务，上述命令是终端命令`create`的实现入口，主要处理一些在create中的终端交互，具体的`Creator`我们利用工厂模式，另开一个**构造项目的类**来实现具体的构造业务。

我们整理的Cli构造的整体架构如图：

![creator flow](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/7/171eb16b54dc6139~tplv-t2oaga2asx-image.image)

我们需要四个类：

1. **Creator（创建项目主逻辑）**
2. **Generator（构造器，负责构造项目文件架构）**
3.  **Interface（接口，对外暴露的构造器接口，负责扩展构造器逻辑，执行逻辑反射给构造器）**
4. **PackageManager（包管理器，负责执行npm命令，比如npm install）**

通过`Generator + Interface` 实现项目文件目录的架构构造，`PackageManager`负责**安装依赖等包管理操作**，整体业务逻辑通过`Creator`类实现。下面，我们就开始慢慢实现整体流程模块的功能。

## 项目构建的Cli模块开发

### 一、区分开发环境和生产环境

开发一个软件，首当其冲的就是如何区分开发环境和生产环境？因为我们这个脚手架特殊，所以在区分各个环境中我们也需要采用一些比较特殊的形式。

#### 通过设置环境变量env区分环境？

我想大多数人想到的第一个办法就是这个吧？我们可以通过设置process.env环境变量的值，比如**设置一个`CAT_SMOKER_DEBUG`为true**就表示在开发环境，通过`cat-smoker create -d`的参数`-d`表示指令在开发环境下执行。

问题？随之而来的就是这种模式的弊端：

**冗余**：首先不像我们一般项目可以预置一些指令，因为脚手架是创建指令的，而你如果仅仅为了一个开发环境，就多几个指令，想必也不太好吧？

**麻烦且难以维护**：你难道想在各个包模块通过下列代码来判断很多情景？

```javascript
if (process.env.CAT_SMOKER_DEBUG) {
  // ....
} else {//}
```

那你要多写多少代码啊？而且代码也难以维护，所以我们换个思路

#### 通过特定路径区分开发环境

这个是从vue-cli中参考来的，我们通过设置路径为`packages/test`路径作为我们测试脚手架用例的方式来设置开发环境

这种做法的好处：

1. 不用维护很多冗余的代码，通过自动判断上下文来决定是否是开发模式。
2. 加载本地模块的时候，可以直接`require`获取项目的公用`module`**(前提是用`yarn workspacce`)**本地安装，不需要远程拉取了。

这里我们选择第二种方法，方便开发，找到我们的`cli/bin/react-build-cli.js`文件

```javascript
// enter debug mode when creating test repo
if (
  slash(process.cwd()).indexOf('/packages/test') > 0 && (
    fs.existsSync(path.resolve(process.cwd(), '../@cat-smoker')) ||
    fs.existsSync(path.resolve(process.cwd(), '../../@cat-smoker'))
  )
) {
  process.env.CAT_SMOKER_DEBUG_MODE = true
}

```

`slash`的作用是格式化不同平台的路径地址。

### 二、Cli创建项目的流程图

我们先明确Cli创建项目的create业务逻辑流程：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/7/171eb17fd17ea8b8~tplv-t2oaga2asx-image.image)

在上篇的文章结尾，我们通过一段代码引入这篇文章：

```javascript
const creator = new Creator(projectName, destDir);
creator.create(options)
```

所以我们一步步来，大体的项目目录如上图所示，通过预设**解析的Plugins（业务插件）**，通过**版本控制**，**git仓库初始化**，初始化项目结构，在项目树的**package.json依赖树里面注入插件提供的dep**，通过**Generator**构造器执行插件内部的逻辑，**生成所有的依赖**，从而构造出了项目结构。

我们通过上述逻辑，一步步来进行编写

### 三、Creator创建器

根据上述逻辑的Creator构造程序：

```javascript
module.exports = class Creator {
  constructor (projectName, context) {
    // 项目名称
    this.projectName = projectName;
    // 上下文
    this.context = context;
  }
}
```

#### 预设与版本控制

```javascript
module.exports = class Creator {
  constructor (projectName, context) {
    // 项目名称
    this.projectName = projectName;
    // 上下文
    this.context = context;
  }
  	
  // 主要的create逻辑
  async create (cliOptions, preset = null) {
    const name = this.projectName;
    const context = this.context;
    // 清除窗口打印信息
    clearConsole();
    
    // 这里预留检测版本控制逻辑
    const latestMinor = await getVersions();
    // 初始化package.json的基础格式
    const pkg = {
      name,
      version: '0.1.0',
      private: true,
      devDependencies: {},
    };
    
    // 如果预设为null，则取默认预设的配置（这里预留预设的配置逻辑）
    if (!preset) {
      preset = defaults.presets['default'];
    }
    
    // 接下来处理插件的注入逻辑
    // 确保@cat-smoker的cli-service脚手架的服务肯定可以注入
    // 这里的preset结构也预留一下，后续继续开发
    preset.plugins['@cat-smoker/cli-service'] = Object.assign({
      projectName: name
    }, preset)
    
    // 把刚刚初始化的各个插件抽取出来，注入到pkg的结构里面去
    // 同时过滤出@cat-smoker开头的依赖，添加版本号，同时为后续解析插件做准备
    const deps = Object.keys(preset.plugins);
    deps.forEach(dep => {
      pkg.devDependencies[dep] =
        preset.plugins[dep].version || (/^@cat-smoker/.test(dep) ? `~${latestMinor}` : `latest`);
    });
  }
}
```

上述代码把预设初始化了，还有注入了基本的`@cat-smoker/cli-service`脚手架服务，同时产生一个`pkg`代表的是package.json的json内容值。

其中预留了很多代码空间，准备后续继续开发，现在**我们先专注于基础架构的逻辑**。

接下去直接把`pkg`的内容输出到`package.json`就可以了：

```javascript
// write package.json
await writeFileTree(context, {
  'package.json': JSON.stringify(pkg, null, 2),
});
```

`writeFileTree`这个方法是写入文件树，代码如下:

```javascript
const fs = require('fs-extra');
const path = require('path');

module.exports = async function writeFileTree (dir, files) {
  Object.keys(files).forEach(name => {
    let fileName = name;
    if (fileName === 'gitignore') {
      fileName = '.gitignore';
    }
    const filePath = path.join(dir, fileName);
    fs.ensureDirSync(path.dirname(filePath));
    fs.writeFileSync(filePath, files[fileName]);
  });
};
```

上述代码通过`dir`目标路径和文件列表对应写入内容。

#### Git仓库控制

```javascript
// check if git repository
const shouldInitGit = this.shouldInitGit(cliOptions);
if (shouldInitGit) {
  logWithSpinner(`🗃`, `初始化git仓库...`);
  await this.run('git init');
}
```

shouldInitGit逻辑判断是否需要初始化git仓库：

```javascript
// from vue cli tools function
shouldInitGit (cliOptions) {
  // hasGit 判断是否有git安装
  if (!hasGit()) {
    return false;
  }
  // --git 指令的配置
  if (cliOptions.forceGit) {
    return true;
  }
  // --no-git
  if (cliOptions.git === false || cliOptions.git === 'false') {
    return false;
  }
  // default: true unless already in a git repo
  return !hasProjectGit(this.context);
}
```

我们看到上面有个`this.run('git init');`执行了git初始化的命令，对于这个`run`函数我们可以通过之前的`@cat-smoker/cli-shared-utils`的`execa`执行逻辑来达到目的：

```javascript
run (command, args) {
  if (!args) {
    // 格式化 options 参数
    [command, ...args] = command.split(/\s+/);
  }
  return execa(command, args, { cwd: this.context });
}
```

初始化完`Git`和`package.json`，接下去可以执行安装插件（比如@cat-smoker/cli-service）的依赖。

#### 安装初始插件依赖

此时我们的package.json依赖只有一个`@cat-smoker/cli-service`，那么接下去就是安装这个依赖了，同时执行依赖内部的函数。

现在随之而来就有一个问题：**安装依赖一定要从远程拉取吗？**

答案当然是否定的，因为远程拉取的话，不方便我们调试，试想一下，你开发了`cli-service`，难道还要先`npm publish`发布到npm再来安装依赖？

当然是不合理的，所以这里我们要区分一下开发环境和生产环境：

```javascript
stopSpinner();

log(`⚙\u{fe0f}  安装脚手架插件, 这可能会花费一点时间...`);
log();

// 包管理实例
const pm = new PackageManager(context, { pkg });

// 区分开发环境
if (process.env.CAT_SMOKER_DEBUG_MODE) {
  // debug mode
  console.log(`${chalk.blueBright('[Cat-smoker]: ')}${chalk.yellowBright('开启本地调试模式!\n')}`);
  await require('../utils/setDevSetup.js')(context);
} else {
  await pm.install();
}
```

上面逻辑要注意的几点：

1. 如果处于debug模式下的话，表示开启本地调试模式：`require('../utils/setDevSetup.js')(context)`**这个执行表示把`cli-service`的script指令`link`到本地的代码了**，同时更改权限，方便本地开发，具体[linkBin代码见此](https://github.com/xdnloveme/cat-smoker/blob/master/packages/%40cat-smoker/cli/lib/utils/setDevSetup.js)，因为**yarn workspace的原因，有些依赖被提到根目录了，可以直接通过Module里面的require()获取到本地的模块，不需要远程拉取了**

2. PM：这是**PackageManager**包提供的实例，这个pm实例指向你项目的目录下，里面内置的指令表示不同的包指令，下面我们说一下这个**PackageManager**。

#### PackageManager包管理

```javascript
const { executeCommand } = require('../utils/executeCommand');
// 不同的包管理工具（npm,yarn,pnpm等，这边先只提供npm）
const PACKAGE_MANAGER_CONFIG = {
  npm: {
    install: ['install', '--loglevel', 'error'],
    add: ['install', '--loglevel', 'error'],
    upgrade: ['update', '--loglevel', 'error'],
    remove: ['uninstall', '--loglevel', 'error'],
  },
};

module.exports = class PackageManager {
  constructor (context, { pkg }) {
    this.context = context;
    this.pkg = pkg;
    // 先只支持npm
    this.bin = 'npm';
  }
  // 执行命令
  async runCommand (command, args) {
    return await executeCommand(
      this.bin,
      [...PACKAGE_MANAGER_CONFIG[this.bin][command], ...(args || [])],
      this.context
    );
  }
	// 安装依赖（npm install）
  async install () {
    return await this.runCommand('install');
  }
};

----------------------------------------
// executeCommand.js 预留代码后续开发会用到
const { execa } = require('@cat-smoker/cli-shared-utils')
const debug = require('debug')('cat-smoker-cli:execute')
exports.executeCommand = function executeCommand (command, args, cwd) {
  debug(`command: `, command)
  debug(`args: `, args)

  return new Promise((resolve, reject) => {
    const child = execa(command, args, {
      cwd,
      // child_process 的options配置项
      stdio: ['inherit', 'inherit', 'inherit']
    })

    child.on('close', code => {
      if (code !== 0) {
        reject(`command failed: ${command} ${args.join(' ')}`)
        return
      }
      // 命令执行完成
      resolve()
    })
  })
}
```

上述代码主要目的为了管理和整合`npm yarn`等不同工具的包管理，预留一些准备开发的代码，现在阶段只提供npm的相关命令逻辑。而且这里不是全部的，这里比较**关键的点就是`executeCommand`这个执行逻辑的实现**。

`execa`这个是基于[child_process](https://nodejs.org/api/child_process.html)实现的命令执行器，具体参数可以查看文档，**这一块内容可能涉及到和主线程的通信，这一块后续会继续展开**，这里先暂时搁置一下。

#### 解析插件

上述我们添加了一些插件或者预设必备的业务插件（比如工具插件`cli-service`），那么接下去就是解析插件了，我们定义一下插件的基础架构：

**generator**文件夹是插件的基础架构入口，文件夹的`index.js`作为插件的入口，所以我们把这个generator作为入口，只要通过解析插件引入这个generator下的`index.js`就可以达到执行插件的目的了：

```javascript
// 解析plugins
const plugins = await this.resolvePlugins(preset.plugins);

---------------------------------------------
async resolvePlugins (rawPlugins) {
  // 确保@cat-smoker/cli-service 插件是被正确添加进去的
  rawPlugins = sortObject(rawPlugins, ['@cat-smoker/cli-service'], true);
  // 插件列表数据缓存
  const plugins = [];
  // 遍历插件列表
  for (const id of Object.keys(rawPlugins)) {
    
    // ** 重要： 因为刚刚上面说过generator/index.js作为插件的入口，所以加载入口模块
    // ** 重要： 通过之前说过的loadModule 加载模块，去加载插件的构造程序
    const apply = loadModule(`${id}/generator`, this.context) || (() => {});
    // 某些配置，后续会说
    const options = rawPlugins[id] || {};
    
    // 在插件列表中保存起来，返回
    plugins.push({ id, apply, options });
  }
  return plugins;
}
```

这里的`resolvePlugins`执行完，返回的是一长串plugin列表，**里面保存着这个plugin的id，apply执行程序和配置options**，接下去要怎么做呢？当然就是通过构造器**Generator**去构造项目了。

#### Generator

接上面的逻辑，我们声明一个构造器实例，传入相关数据，可以进行项目的create构造了：

```javascript
const gen = new Generator(this.context, {
  // 项目名称
  name: this.projectName,
  // package.json包
  pkg: pkg,
  // 插件
  plugins,
  // packagemanager包管理实例
  pm,
});

log(`🚀  开始执行项目构造程序...`)
gen.generate(); // 构造
```

先声明构造逻辑：

```javascript
module.exports = class Generator {
  constructor (context, { pkg, plugins, pm }) {
    this.context = context;
    this.pkg = pkg;
    this.plugins = plugins;
    this.pm = pm;
    this.depSources = {};
  }
  // 解析plugins
  async initPlugins () {
    // 把刚刚解析完的plugins列表进行遍历，执行里面的apply函数
    for (const plugin of this.plugins) {
      const { id, apply, options } = plugin;
      // 声明一个接口，给apply调用
      const api = new Interface(id, this, options);
      await apply(api, options);
    }
  }
  
  generate () {
    // 初始化构造器的时候，初始化插件
    this.initPlugins();
    // 这里通过writeFileTree构造,先搁置一下，后续说明
  }
}
```

上述有个关键的逻辑：

```javascript
const api = new Interface(id, this, options);
await apply(api, options);
```

这里可以看到我们声明了一个**Interface**接口，**来抛出一些原构造器的方法，提供外部的plugins使用，任其apply调用此实例**，接下去我们看Interface。

#### Interface

根据上述的构造函数：

```javascript
module.exports = class Interface {
  constructor (id, generator, pluginOptions) {
		// 插件的id名称
    this.id = id;
    // 构造器实例
    this.generator = generator;
    // 插件设置
    this.pluginOptions = pluginOptions;
  }
}
```

**我们对外要抛出的有非常多的方法，这里我们写一些通用的api方法**

**比如`extendPackage`：扩展项目包的各种依赖，这个方法非常通用，使用频率也很高**，所以我们以此为例子展开讲解，当然未来肯定不止这些方法，这边简写了：

注意：**extendPackage**参考了`vue-cli`的**extendPackage**方法，两者类似可以直接参考。

```javascript
// 判断是否是一般对象？
const isPlainObject = function (obj) {
  if (typeof obj !== 'object' || obj === null) return false;

  let proto = obj;
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto);
  }

  return Object.getPrototypeOf(obj) === proto;
};

// 类数组合并拷贝
const mergeArrayWithDedupe = (a, b) => Array.from(new Set([...a, ...b]));


/**
 * 扩展package
 * @param {Object|Function} fields package字段名
 * @param {Object} options 设置options
*/
async extendPackage (fields, options = {}) {
  // 扩展方法的一些配置
  const extendOptions = { 
    merge: true, // 是否合并
    warnIncompatibleVersions: true, // 是否显示版本不一致的警告（逻辑后续补上）
  };
	
  if (typeof options === 'boolean') {
    // 如果是boolean类型，则直接改变 “版本不一致警告”的配置
    extendOptions.warnIncompatibleVersions = !options;
  } else {
    // 合并配置
    Object.assign(extendOptions, options);
  }
	
  // pkg内容，由generator实例提供
  const pkg = this.generator.pkg; 
  // 准备合并的字段，可能是function，那就执行
  const toMerge = typeof fields === 'function' ? fields(pkg) : fields;
  // 遍历扩展的package.json字段
  for (const key in toMerge) {
    const value = toMerge[key];
    // 是否在原pkg包存在？
    const existing = pkg[key];
    
    // 判断对象是否是一般对象，并且是依赖dep对象？
    if (isPlainObject(value) && (key === 'dependencies' || key === 'devDependencies')) {
      // 如果是dependencies（依赖）对象，直接合并merge
      // mergeDeps 业务逻辑直接copy抄的vue-cli的代码：
      // 可以直接查看 https://github.com/vuejs/vue-cli/blob/5cb988cb273d9bc1bbdddd4b7c71ab1c4e3d6e57/packages/%40vue/cli/lib/util/mergeDeps.js
      pkg[key] = mergeDeps(
        this.id,
        existing || {},
        value,
        this.generator.depSources,
        extendOptions
      );
    } else if (!extendOptions.merge || !(key in pkg)) {
      // 不合并的情景
      pkg[key] = value;
    } else if (Array.isArray(value) && Array.isArray(existing)) {
      // 数组合并的情景
      pkg[key] = mergeArrayWithDedupe(existing, value);
    } else if (isObject(value) && isObject(existing)) {
      // 通过loadsh提供的深拷贝直接覆盖原有的值
      pkg[key] = deepmerge(existing, value, { arrayMerge: mergeArrayWithDedupe });
    } else {
      // 新值
      pkg[key] = value;
    }
  }
}
```

大概逻辑如上，可以继续优化，基本是参考了`vue-cli`的源码，不得不说，vue-cli还是非常完善，一些基础的工具函数都可以直接复用，这里我们可以扩展包的依赖了，接下去就可以直接在项目中直接使用。

### 三、如何桥接插件api和业务？

上述我们完成了**Interface**接口的构建，我们可以使用`extendPackage`来扩展我们插件的package.json包了，是不是蠢蠢欲试了？

下面我们通过基础的`cli-service`来扩展一下我们包：(在`cli-service/generator/index.js`):

```javascript
module.exports = api => {
  api.extendPackage({
    scripts: {
      start: 'cat-smoker-cli-service serve',
      build: 'cat-smoker-cli-service build',
    },
    dependencies: {
      react: '^16.13.0',
      'react-dom': '^16.13.0',
    },
    browserslist: ['> 1%', 'last 2 versions'],
  });

};
```

我们在上述的代码添加了`cat-smoker-cli-service`的服务指令，这个具体后续再说，还有我们基础的`react`的依赖包:`react: '^16.13.0' `,`'react-dom': '^16.13.0'`。

没错！就是这样，大功告成，**api**就是之前的**Interface**实例，可以使用内部的方法，而在**generator目录下是约定的，默认插件的构造逻辑在此完成！**

插件api和业务的桥接就完成了，基础架构完成了，接下去就是完善各个方法，包括业务的扩展，那么问题来了？我们怎么拉取项目的模板？请看下一章节：

### 四、如何声明项目模板结构？

#### download-git-repo

首先我们肯定会想到，如果我们可以通过**download-git-repo**拉取了远程模板，直接类似`git clone`的模式，直接拉取具体仓库的模板。

```javascript
const download = require('download-git-repo')
download(repository, destination, options, callback)
```

当然弊端和优势也非常明显：优势：简单快捷，直接拉取，弊端：不够灵活，要改变模板内容，不具有可靠的扩展性。

#### 脚手架内部render

我们脚手架采用的是这一方式：通过**EJS模板引擎**，通过不同的判断条件，如下，渲染不同的js模板：

```javascript
<%_ if (条件) { _%>
// 这里输入js语句
<%_ } else { _%>
// 其他js语句
<%_ } _%>
```

这个条件语句可以通过process.env注入，或者直接通过ejs模板引擎注入，这里我留存一定的逻辑空间。

接下去我们**约定每个插件下generator/template**为我们的模板入口，通过约定目录结构，读取模板文件夹树，最后merge合并文件夹树，进行输出。

这里我们把逻辑留存，代码简化，我们固定一个模板入口：

```javascript
const templatePath = '../../../cli-service/generator/template';
```

模板目录如下：

```javascript
├── gitignore
├── public
│   ├── favicon.ico 
│   ├── index.html
├── src
│   ├── assets
│   ├── components
│   ├── layout
│   ├── page
│   ├── App.js
│   ├── index.js
│   ├── index.css
```

generate逻辑：

```javascript
const renderFile = function (name) {
  // 如果是二进制流文件（比如favicon.ico）
  if (isBinaryFileSync(name)) {
    return fs.readFileSync(name); // 返回流
  }
	
  // 读取文件内容
  let template = fs.readFileSync(name, 'utf-8');
  // 举例子
  if (/.js/g.test(name)) {
    // 如果是js文件，则调取ejs的方法，直接通过引擎渲染ejs的模板
    // 这里举个例子，仅供参考
    template = ejs.render(template)
  }

  return template;
};

-------------------------------
async generate () {
  // 上述逻辑
  this.initPlugins();
  // 模板基础入口，这里作为样例，实际逻辑没有这么简单
  const baseDir = path.resolve(__dirname, templatePath);
  // 利用globby读取基础入口下所有模板的文件树
  const _files = await globby(['**'], { cwd: baseDir });
  
  // 利用reduce读取文件树的内容，renderFile方法见上
  const filesContentTree = _files.reduce((content, sourcePath) => {
    content[sourcePath] = renderFile(path.resolve(baseDir, sourcePath));
    return content;
  }, {});
	
  // 在文件内容树添加package.json
  filesContentTree['package.json'] = JSON.stringify(this.pkg, null, 2) + '\n'
	// 直接写入文件树
  await writeFileTree(this.context, filesContentTree);
}
```

按照上述的逻辑，我们就可以实现了模板的渲染，当然实际的逻辑还是比较复杂的，后续再展开。

安装完模板的逻辑，在我们的目录下就形成了对应的项目架构，如下图GIF所示：


![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/7/171eb30b6b4edaf2~tplv-t2oaga2asx-image.image)

可以发现，通过`cat-smoker create testdemo`产生的文件目录就是我们预期的项目架构，至此流程完成。

## 主要逻辑代码

下面，我们把主要的逻辑代码一同整理上来。

#### Creator.js

```javascript
const { clearConsole } = require('../utils/clearConsole');
const sortObject = require('../utils/sortObject');
const {
  hasGit,
  hasProjectGit,
  log,
  logWithSpinner,
  stopSpinner,
  execa,
  chalk,
  loadModule,
} = require('@cat-smoker/cli-shared-utils');
const { defaults } = require('../options');
const writeFileTree = require('../utils/writeFileTree');
const Generator = require('./Generator');
const getVersions = require('../utils/getVersions');
const PackageManager = require('./PackageManager');

module.exports = class Creator {
  constructor (projectName, context) {
    this.projectName = projectName;
    this.context = context;
  }

  async create (cliOptions, preset = null) {
    const name = this.projectName;
    const context = this.context;

    clearConsole();

    const latestMinor = await getVersions();

    const pkg = {
      name,
      version: '0.1.0',
      private: true,
      devDependencies: {},
    };

    if (!preset) {
      preset = defaults.presets['default'];
    }

    preset.plugins['@cat-smoker/cli-service'] = Object.assign({
      projectName: name
    }, preset)

    const deps = Object.keys(preset.plugins);
    deps.forEach(dep => {
      pkg.devDependencies[dep] =
        preset.plugins[dep].version || (/^@cat-smoker/.test(dep) ? `~${latestMinor}` : `latest`);
    });

    const pm = new PackageManager(context, { pkg });

    // write package.json
    await writeFileTree(context, {
      'package.json': JSON.stringify(pkg, null, 2),
    });

    // check if git repository
    const shouldInitGit = this.shouldInitGit(cliOptions);
    if (shouldInitGit) {
      logWithSpinner(`🗃`, `初始化git仓库...`);
      await this.run('git init');
    }

    stopSpinner();

    log(`⚙\u{fe0f}  安装脚手架插件, 这可能会花费一点时间...`);
    log();

    if (process.env.CAT_SMOKER_DEBUG_MODE) {
      // debug mode
      console.log(`${chalk.blueBright('[Cat-smoker]: ')}${chalk.yellowBright('开启本地调试模式!\n')}`);
      await require('../utils/setDevSetup.js')(context);
    } else {
      await pm.install();
    }

    const plugins = await this.resolvePlugins(preset.plugins);
    
    const gen = new Generator(this.context, {
      name: this.projectName,
      pkg: pkg,
      plugins,
      pm,
    });

    log(`🚀  开始执行项目构造程序...`)
    gen.generate();

    await pm.install();
  }

  // { id: options } => [{ id, apply, options }]
  async resolvePlugins (rawPlugins) {
    // ensure cli-service is invoked first and sort
    rawPlugins = sortObject(rawPlugins, ['@cat-smoker/cli-service'], true);
    const plugins = [];
    for (const id of Object.keys(rawPlugins)) {
      const apply = loadModule(`${id}/generator`, this.context) || (() => {});
      const options = rawPlugins[id] || {};
      plugins.push({ id, apply, options });
    }
    return plugins;
  }

  run (command, args) {
    if (!args) {
      [command, ...args] = command.split(/\s+/);
    }
    return execa(command, args, { cwd: this.context });
  }

  // from vue cli tools function
  shouldInitGit (cliOptions) {
    if (!hasGit()) {
      return false;
    }
    // --git
    if (cliOptions.forceGit) {
      return true;
    }
    // --no-git
    if (cliOptions.git === false || cliOptions.git === 'false') {
      return false;
    }
    // default: true unless already in a git repo
    return !hasProjectGit(this.context);
  }
};
```

#### Generator.js

```javascript
const fs = require('fs');
const globby = require('globby');
const writeFileTree = require('../utils/writeFileTree');
const Interface = require('../generator/Interface');
const { isBinaryFileSync } = require('isbinaryfile');
const path = require('path');
const templatePath = '../../../cli-service/generator/template';

const renderFile = function (name) {
  // 如果是二进制流文件（比如favicon.ico）
  if (isBinaryFileSync(name)) {
    return fs.readFileSync(name); // 返回流
  }

  const template = fs.readFileSync(name, 'utf-8');

  return template;
};

module.exports = class Generator {
  constructor (context, { pkg, plugins, pm }) {
    this.context = context;
    this.pkg = pkg;
    this.plugins = plugins;
    this.pm = pm;
    this.depSources = {};
  }

  async initPlugins () {
    for (const plugin of this.plugins) {
      const { id, apply, options } = plugin;
      const api = new Interface(id, this, options);
      await apply(api, options);
    }
  }

  async generate () {
    this.initPlugins();
    const baseDir = path.resolve(__dirname, templatePath);
    const _files = await globby(['**'], { cwd: baseDir });
    const filesContentTree = _files.reduce((content, sourcePath) => {
      content[sourcePath] = renderFile(path.resolve(baseDir, sourcePath));
      return content;
    }, {});

    filesContentTree['package.json'] = JSON.stringify(this.pkg, null, 2) + '\n'

    await writeFileTree(this.context, filesContentTree);
  }
};
```

#### Interface.js

```javascript
const deepmerge = require('deepmerge');
const mergeDeps = require('../utils/mergeDeps');

const isPlainObject = function (obj) {
  if (typeof obj !== 'object' || obj === null) return false;

  let proto = obj;
  while (Object.getPrototypeOf(proto) !== null) {
    proto = Object.getPrototypeOf(proto);
  }

  return Object.getPrototypeOf(obj) === proto;
};

const mergeArrayWithDedupe = (a, b) => Array.from(new Set([...a, ...b]));

module.exports = class Interface {
  constructor (id, generator, pluginOptions) {
    this.id = id;
    this.generator = generator;
    this.pluginOptions = pluginOptions;
  }

  /**
   * 扩展package
   * @param {Object|Function} fields package字段名
   * @param {Object} options 设置options
   */
  async extendPackage (fields, options = {}) {
    const extendOptions = {
      prune: false,
      merge: true,
      warnIncompatibleVersions: true,
    };
    
    if (typeof options === 'boolean') {
      extendOptions.warnIncompatibleVersions = !options;
    } else {
      Object.assign(extendOptions, options);
    }

    const pkg = this.generator.pkg;
    const toMerge = typeof fields === 'function' ? fields(pkg) : fields;
    for (const key in toMerge) {
      const value = toMerge[key];
      const existing = pkg[key];
      if (isPlainObject(value) && (key === 'dependencies' || key === 'devDependencies')) {
        // use special version resolution merge
        pkg[key] = mergeDeps(
          this.id,
          existing || {},
          value,
          this.generator.depSources,
          extendOptions
        );
      } else if (!extendOptions.merge || !(key in pkg)) {
        pkg[key] = value;
      } else if (Array.isArray(value) && Array.isArray(existing)) {
        pkg[key] = mergeArrayWithDedupe(existing, value);
      } else if (isObject(value) && isObject(existing)) {
        pkg[key] = deepmerge(existing, value, { arrayMerge: mergeArrayWithDedupe });
      } else {
        pkg[key] = value;
      }
    }
  }
};
```

#### PackageManager.js

```javascript
const { executeCommand } = require('../utils/executeCommand');

const PACKAGE_MANAGER_CONFIG = {
  npm: {
    install: ['install', '--loglevel', 'error'],
    add: ['install', '--loglevel', 'error'],
    upgrade: ['update', '--loglevel', 'error'],
    remove: ['uninstall', '--loglevel', 'error'],
  },
};

module.exports = class PackageManager {
  constructor (context, { pkg }) {
    this.context = context;
    this.pkg = pkg;
    // 先只支持npm
    this.bin = 'npm';
  }
  
  async runCommand (command, args) {
    return await executeCommand(
      this.bin,
      [...PACKAGE_MANAGER_CONFIG[this.bin][command], ...(args || [])],
      this.context
    );
  }

  async install () {
    return await this.runCommand('install');
  }
};
```

主要代码仓库[戳这里](https://github.com/xdnloveme/cat-smoker/tree/master/packages/%40cat-smoker/cli/lib/generator)，可以对照看，且持续更新。

## 总结

此文章解决了`create`项目架构的业务流程，主要实现还是偏向于简单，具体的场景会比较复杂，后续代码会持续更新和完善，此文章仅供参考和提供思路，并不是最终代码。

另外，此项目的GitHub[仓库地址戳这里](https://github.com/xdnloveme/cat-smoker)，整体进度会比文章进度快一点，因为文章是边写边去构建`cat-smoker`这个项目的，所以大家不要着急，这个是个长期的工程。

我们会基于这个项目，向**大家一点点剖析`vue-cli`的源码**，介绍给大家`vue-cli`脚手架的架构和一些插件设计思路。**摒弃CRA(create-react-app)脚手架，加入的plugin插件也不是一些配置插件了，而是我们的业务结构插件**。

此项目为了的是解决**多业务场景下**，**CRA**满足不了React用户的一些需求，而做的基于vue-cli模式的业务脚手架，开源出来的部分可能有点缺点，也希望大家一起改进。

## 下一章节的展望

上一章节：[从剖析Vue-cli源码出发完整的React业务脚手架实践（一）——脚手架架构基础搭建](https://juejin.im/post/6844904131266625549)

下一章节：**从剖析Vue-cli源码出发完整的React业务脚手架实践（三）——脚手架的(cli-service)黑箱服务**，会着重说明以下几个问题：

1. **如何定义和扩展config.js，合并到整体配置？**
2. **如何解决黑箱下的webpack配置和各类扩展？**
3. **参与编写脚手架黑箱服务的指令**
4. **本地开发需要注意的事项等**

**等一系列问题**，文章会尽快推出，大家敬请期待，谢谢关注 Thanks♪(･ω･)ﾉ。


