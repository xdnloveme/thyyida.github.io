---
layout:     post
title:      "从剖析Vue-cli源码出发完整的React业务脚手架实践（一）"
subtitle:   "脚手架架构基础搭建"
date:       2020-04-19
author:     "Leon"
header-img: "img/post-bg-js-module.jpg"
tags:
    - 前端开发
    - JavaScript
    - React
---



> 随着公司业务线增加了以后，基础脚手架已经满足不了需求，于是开始着手业务线的脚手架开发，我基于vue cli源码和自己的业务实践，吸取vue-cli插件模式的开发优势和业务结合，做一套关于React的项目脚手架。

## 写在前面

这是一篇长期持续更新的React脚手架实践，为的是吸取Vue Cli的脚手架经验，通过我们习惯的**插件-预设**的思想去构造我们的React业务脚手架，这可能不是最好的脚手架的开发实践，但是一定是**最完整的脚手架开发实践**。

全套实践我们将通过现有的vue cli源码一一解说的方式进行，一方面是为了熟悉成熟脚手架的代码实现，另一方面是为了完善自己的代码和实践的理解，让大家在自己开发脚手架或者学习的过程中，能有更深刻的认识。

文章导航：
1. [从剖析Vue-cli源码出发完整的React业务脚手架实践（一）——脚手架架构基础搭建](https://juejin.im/post/6844904131266625549)
2. [从剖析Vue-cli源码出发完整的React业务脚手架实践（二）——项目的构建及服务（create）](https://juejin.im/post/6844904149507637256)

## 脚手架架构

首先我们先给脚手架取个酷炫的名字吧，当时在做的时候突然看到一个图很酷，如下，是一个猫在抽烟

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/19/1718e783a03841ba~tplv-t2oaga2asx-image.image)CAT-SMOKER-FLOW" alt="cat" style="zoom:54%;" />

然后当时就爆出来杨超越的拍的照片有香烟的的新闻，当时解释到说是她家的猫在吸烟，当时就觉得就这个吧，感觉很酷炫！然后就取名叫做**cat-smoker**吧！

名字取完了，前期准备得整理一下架构思路和流程图，我自己整理出基于vue-cli的脚手架思维逻辑导图如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/19/1718e795dc7deb17~tplv-t2oaga2asx-image.image)


我们一开始的主体架构是两方面，一个是`cli`还有一个是`cli-service`，一个**主要在插件的基础上提供构造项目的能力**，**另外一个是为构造的项目提供基础服务（serve，build等**），这个也是基本vue cli的基础架构，CRA太过基础化，很多配置不够我们平时用的，所以照搬了vue cli的模式套在React的项目脚手架下进行架设。

这个架构的好处是对于自己公司内部业务的定制化开发有着比较好的**针对性和扩展性**，说实话`react-script`我是用不太习惯，我们可以通过本地磁盘写入**预设**来保存我们平时一些共通的业务设置，通过类似`vue-cli`的选择模式（select）来组合我们的项目架构，通过**外部插件api来扩展我们的架构**。

`cli-service`为我们提供了黑箱操作，我们可以在内部进行**webpack设置**，**优化（optimization）配置，打包（第三方，过滤，cdn设置等业务）配置，runtime，项目路由，webpackChain链，项目模板，业务架构等配置**等都集中从黑箱抛出一个**cs.config.js**来配置这一切，可以实现**规范化**，这就是基础化的cat-smoker架构。

## 脚手架包管理方案（lerna + yarn）

因为我们有两个架构模块都是独立的，但是同时又是相互依赖的，整个项目又是一个**仓库**，解决这种场景下的仓库设计模式我们可以采用**Monorepo仓库的模式**，这个模式可以很好的管理我们**多个包之间依赖的关系**，不同于我们常用的一个项目一个**仓库（是multiple repo模式）**，在**Monorepo**这种模式下，各个模块包需要单独的管理（单独一个repo仓库），[Lerna](https://github.com/lerna/lerna)的出现就是为了解决这个问题。比如你项目单独需要引用`cli-serive`，若是Multi模式下，你需要好几个仓库来管理`cli`和`cli-service`，**Monorepo**模式下这一点就比较容易解决。

至于Monorepo模式并不是万能的，和Multi模式下的区别和优缺点这里不再展开了，要不然篇幅太多了，大家可以看下面的链接，也可以自己搜索相关知识：

1. [monorepo 新浪潮 | introduce lerna](https://github.com/pigcan/blog/issues/3)
2. [Monorepos: Please don’t!](https://medium.com/@mattklein123/monorepos-please-dont-e9a279be011b)

至于脚手架为什么采用这种模式，这是通过CRA，vue-cli等库的实践的，基于前人的经验和我们这种架构下多模块**依赖关系紧密和分包开发**的特点，我们也采用这种模式进行包管理。因为yarn的workspace模式，可以利用lerna+yarn的`workspace`特性进行工作区域的包管理，这里因为考虑到一些小伙伴也没有装yarn，在本人实践以后，发现yarn还是非常需要的，yarn workspace+ lerna的模式开发起来非常顺手！

### Yarn Workspace + Lerna

如果还没有安装`yarn`小伙伴，推荐看一下官网安装一下：[Yarn](https://yarn.bootcss.com/)的`workspace`:[yarn workspace](https://yarn.bootcss.com/blog/2017/08/02/introducing-workspaces/)，通过yarn的一些指令可以映射到lerna的指令中，在我们的`lerna.json`文件夹中声明`useWorkspaces`

```javascript
{
  "useWorkspaces": true,
   "npmClient": "yarn",
  "packages": [],
  "version": "0.0.8-alpha.0"
}
```

接下去可以愉快地使用`Yarn`，

1. **安装依赖**

```bash
$ yarn install # 等价于 lerna bootstrap --npm-client yarn --use-workspaces
```

2. **清理包依赖**

```bash
$ lerna clean # 清理所有的node_modules
$ yarn workspaces run clean # 执行所有package的clean操作
```

3. **增加（删除）依赖**

```bash
$ yarn workspace packageB add(remove) packageA # 向packageB增加packageA依赖
$ yarn workspaces add(remove) lodash # 所有package增加依赖
$ yarn add(remove) -W -D loadash # root增加依赖
```

发布和测试等其他命令，大家可以看[基于lerna和yarn workspace的monorepo工作流](https://zhuanlan.zhihu.com/p/71385053)，这里不多赘述，接下来可以愉快地搭建了

### 开始搭建

先进行全局安装：

```bash
npm install --global lerna
```

然后进到我们初始化的项目（`cd cat-smoker`）内进行init

```bash
lerna init
```

初始化后的项目结构如下：

```javascript
├── cat-smoker/
├── packages/
│   ├── package-child1/ // 不同的包
│   ├── package-child2/
├── package.json
├── lerna.json // 管理的json文件
```

创建属于我们项目的package包，看我们根目录的`lerna.json`文件，为了统一包的名称和管理，在`packages`字段内填入`packages/@cat-smoker/cli*`表示匹配packages下有`@cat-smoker/cli-`开头的包。

```javascript
{
  "useWorkspaces": true,
  "packages": [
    "packages/@cat-smoker/cli*"
  ],
  "version": "0.0.1"
}

```

然后我们通过`lerna create <package>`来创建属于我们的包，从上述的路径我们进行build：

```bash
lerna create cli 0
```

0表示匹配上述packages位置的数组下标0，我们可以看到`@cat-smoker/`文件夹下多了一个文件`cli`，这是我们上述流程图的cli主体包。ps：注意在lerna管理的包内别用`npm install`会容易出错，尽量用`lerna add --scope`来添加依赖，或者`yarn workspace XX add XX `。

同理添加我们的服务包和公共包：`lerna create cli-service 0`,`lerna create cli-shared-utils`。项目结构如下：

```javascript
├── cat-smoker/
├── packages/
│   ├── @cat-smoker/ // 不同的包
│   │   └── cli/ // 主体包
│   │   └── cli-service/ // 服务包
│   │   └── cli-shared-utils/ // 包之间公用的工具类包
├── package.json
├── lerna.json // 管理的json文件
```

搭完这个基础架构，可以看到每个包都有自己独立的package.json，注意里面一个字段：

```json
{
  // ...
  "bin": {
    "cat-smoker": "bin/react-build-cli.js"
  }
  // ...
}
```

这是项目的主程序bin入口，也就是命令`cat-smoker`shell入口，我们需要定义个指令`cat-smoker`像`vue`那样的command命令，在这个入口的文件里头部写上：

```javascript
#!/usr/bin/env node

// 代码
```

这段代码是什么意思？表示**一个shell命令执行这个脚本的时候，语言解释器是node，node路径是沿着文中的值进行寻找**

当你使用`npm link`的时候，**可以把这个命令链接到全局**，好处就在于link了以后我们**可以本地开发实时更改，方便了我们开**发，然后通过`cat-smoker`进行逻辑操作，确定了入口，我们可以写具体命令了，本地调试的时候记得`npm link `或`yarn link`

## 技术栈准备

nodejs的出现为前端工程脚手架提供了可能性，为我们操作Terminal终端和操作IO读取提供了支持。上述bin文件的入口提供了操作shell的可能，那么就需要一个终端解析器，首先是node下的Terminal操作需要下列的库：


首先我们像上面一样在`package.json`内指定我们的**终端入口**：

```json
{
  // ...
  "bin": {
    "cat-smoker": "bin/react-build-cli.js"
  }
  // ...
}
```

在`bin/react-build-cli.js`目录下的文件声明一个`#!/usr/bin/env node`头，然后就可以开始我们愉快的终端命令之旅了：

### 终端操作（commander）

[commander](https://www.npmjs.com/package/commander)：这是完整的nodejs下的终端解决方案，比如你的执行命令`vue create`就是这个库来解析的，同理，我们可以在这里定义一些命令,比如举个例子，我们想开发`cat-smoker create`这个命令：

```javascript
#!/usr/bin/env node

const program = require('commander');
// 声明版本
program.version(require('../package').version);

program
  .version(`@cat-smoker/cli ${require('../package').version}`)
  // 就是平时cat-smoker --help显示的，声明一下
  .usage('<command> [options]')

program
  .command('create <app-name>') // 定义指令create
  .description('create a new react project') // 指令的描述信息
	.option('-g, --git [message]', 'Force git initialization with initial commit message', true) // 参数配置 -g 表示cat-smoker create -g
	.action((name, cmd) => {
  	// cmd 是命令行参数配置项， name是 vue create <name> 的参数
  	// 这里对create命令做出反应
  	// 逻辑见下
	})

// 记得解析命令行参数（process.argv）
program.parse(process.argv);
```

上面就是`cat-smoker create`命令的入口，我们可以在`action`里面执行`create`逻辑，我们需要`minimist`这个库来解析我们的命令行参数，相当于是一个格式化参数的过程。

此外，`vue-cli`提供了一个`cleanArgs`的函数，我们直接拿来用，这个函数的作用是把cmd的`options`选项依次提取出来，作为一个`options`对象来保存参数的值，比如 `-g`就会在抛出的options对象里面声明一个`{ git: true }`依次类推。

这样格式化的好处是把**命令行=>对象**的数据映射关系了直观表示了出来，方便后续代码调用命令行参数。

```javascript
const minimist = require('minimist');
// ... 省略上述代码
// .action
program.action((name, cmd) => {
  // 解析命令行参数
  if (minimist(process.argv.slice(3))._.length > 1) {
    log('你的命令行参数超过1个，不是正确的命令，请参照帮助.');
  }
  // 这里进行参数的格式化成对象
  const options = cleanArgs(cmd)
  // 这里调取lib的create方法，是我们create方法的主体逻辑
  require('../lib/create')(name, options);
})

// 这一步是把 - 分隔符变成驼峰式
function camelize (str) {
  return str.replace(/-(\w)/g, (_, c) => c ? c.toUpperCase() : '')
}

// 解析配置。抛出options，为后续开发提供配置对象
function cleanArgs (cmd) {
  const args = {}
  cmd.options.forEach(o => {
    const key = camelize(o.long.replace(/^--/, ''))
    if (typeof cmd[key] !== 'function' && typeof cmd[key] !== 'undefined') {
      args[key] = cmd[key]
    }
  })
  return args
}
```

可以看出上面的终端操作，最后的逻辑走向了`lib/create`下的js文件，所以我们在对应目录下创建它进行create命令的逻辑编写。同理，你可以声明类似`add`,`delete`等命令，结构如上，这个对应vue-cli的源码，[传送门](https://github.com/vuejs/vue-cli/blob/master/packages/@vue/cli/bin/vue.js)，还有一些细节逻辑，比如检查node版本等，这里不展开，感兴趣的可以直接看源码。

### 终端命令逻辑业务（ cat-smoker-cli）

我们先来捋一捋具体逻辑的业务，参照`vue-cli`的模式我自己整理的大体的**cat-smoker-cli**黑箱业务架构如下：

![command-flow](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/19/1718e7a48ace45fd~tplv-t2oaga2asx-image.image)

我们需要在主业务线（create.js）中，通过**Creator和PackageManage**分别实现**项目结构构造**和**项目依赖包管理**两种功能。

在Creator中有一个关键的**Generator**构造器，这个构造器是构造我们项目的主要逻辑，内部抛出一个**Interface接口，在vue-cli中我们称作”GeneratorAPI“**，这里提供了一切插件可用的方法进行业务扩展。

### 确定一些基础工具函数

我们先创建一个`cli-shared-utils`包：`lerna create cli-shared-utils 0 `。

package.json文件默认入口地址：

```json
{
  // ...
  main: 'index.js'
}
```

index.js

先确定一些必会用到的库，可能不完整，后续会补充：

1. **chalk**：终端样式库，可以提供酷炫的颜色等显示效果，让你的log看起来很酷。
2. **execa**：执行终端命令必备库，类似开启子任务`child_process`的增强版。
3. **semver**：版本语义化解析库，工具类涉及到各种版本的管理，比如node版本，package版本，所以肯定需要这个库。

还有一些我们自己写的工具类，举例如下：

1. **log**：控制台日志输出函数，可以控制不同的类型的日志打印，简化如下，有error，warn，info，success等类型的函数

```javascript
const logInfo = console.log;
const logError = console.error;
const logWarn = console.warn;

const prefix = 'cat-smoker-';

module.exports.info = msg => logInfo(`${chalk.hex('#3333').bgBlue(`${prefix}INFO `)} ${msg}`);

module.exports.error = msg => logError(`${chalk.hex('#3333').bgRed(`${prefix}ERROR `)} ${msg}`);

module.exports.warn = msg => logWarn(`${chalk.hex('#3333').bgYellow(`${prefix}WARN `)} ${msg}`);

module.exports.success = msg =>
  logInfo(`${chalk.hex('#3333').bgGreen(`${prefix}SUCCESS `)} ${msg}`);

module.exports.log = (msg = '') => {
  console.log(msg);
};

// 清除终端内容
module.exports.clearConsole = title => {
  // 根据isTTY 判断是否位于终端上下文
  // 这个照搬的vue-cli的判断，虽然我觉得这个没啥意义判断
  // 存在这个判断的意义就是 区分一般的终端环境和其他文件环境
  if (process.stdout.isTTY) {
    const blank = '\n'.repeat(process.stdout.rows);
    console.log(blank);
    readline.cursorTo(process.stdout, 0, 0);
    readline.clearScreenDown(process.stdout);
    if (title) {
      console.log(title);
    }
  }
};
```

2. **module**模块加载函数，直接照搬的vue-cli模块文件，这个模块比较重要，见下列解析

```javascript
const Module = require('module')
const path = require('path')
const semver = require('semver')
// 参数解释：request 是请求的相对路径，context是上下文绝对路径

// 兼容不同node版本的模块加载方式，返回require函数
const createRequire = Module.createRequire || Module.createRequireFromPath || function (filename) {
  const mod = new Module(filename, null)
  mod.filename = filename
  mod.paths = Module._nodeModulePaths(path.dirname(filename))

  mod._compile(`module.exports = require;`, filename)

  return mod.exports
}
function resolveFallback (request, options) {
  // polyfill 下列解析文件resolve的兼容写法
  // ...
}

const resolve = semver.satisfies(process.version, '>=10.0.0')
  ? require.resolve
  : resolveFallback

// 返回加载模块的路径
exports.resolveModule = function (request, context) {
  let resolvedPath
  try {
    try {
      // 从指定的上下文绝对路径开始解析，返回一个require.resolve函数解析的路径
      resolvedPath = createRequire(path.resolve(context, 'package.json')).resolve(request)
    } catch (e) {
      // 出错则调用resolve方法
      resolvedPath = resolve(request, { paths: [context] })
    }
  } catch (e) {
    console.log(e);
  }
  return resolvedPath
}
// 加载模块逻辑
exports.loadModule = function (request, context, force = false) {
  try {
    // 返回目标文件上下文的require依赖
    return createRequire(path.resolve(context, 'package.json'))(request)
  } catch (e) {
    const resolvedPath = exports.resolveModule(request, context)
    if (resolvedPath) {
      // force表示删除缓存，重新加载一遍
      if (force) {
        clearRequireCache(resolvedPath)
      }
      return require(resolvedPath)
    }
  }
}
// 清除模块缓存
exports.clearModule = function (request, context) {
  const resolvedPath = exports.resolveModule(request, context)
  if (resolvedPath) {
    clearRequireCache(resolvedPath)
  }
}

// 我们知道commonjs规范里面的require是有缓存的，这个为了清除require缓存
function clearRequireCache (id, map = new Map()) {
  // ... 见源码：https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-shared-utils/lib/module.js
}

```

3. **env.js**： 环境变量和环境函数，比如包含git检测，返回git仓库信息等函数
4. **schema.js**：schema数据结构的映射，我们推荐用[@hapi/joi](https://www.npmjs.com/package/@hapi/joi)，因为我们的脚手架函数肯定需要抛出一个options配置去提供用户修改，那么如何管理options的数据类型和数据映射等，这里就需要schema。
5. **Spinner.js**：就是你平时看的终端里面转圈圈那个东西。。。

暂时需要的包就是这些，需要的话后续会继续补充，同时我们在index里面抛出，供外部使用：

```javascript
['env', 'logger', 'spinner', 'module', 'schema'].forEach(key => {
  // 直接赋值exports
  Object.assign(exports, require(`./lib/${key}`));
});

exports.chalk = require('chalk');
exports.execa = require('execa');
exports.semver = require('semver');
```

声明完一些必须要的工具函数后，激动人心的来了：进行具体业务的逻辑编写了！

### Create.js

我们声明一个create.js：

```javascript
const create = async (projectName, options) => {
  // projectName是项目名
  // options是
}

module.exports = function (...args) {
  try {
    return create(...args);
  } catch (e) {
    console.log(e);
  }
};
```

声明了项目名和上下文后，我们进行create逻辑的编写,先把create逻辑基础业务流程图如下，非完整版，仅供参考：
![create-flow](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/19/1718e7abe38c192b~tplv-t2oaga2asx-image.image)

我们先忽略判断项目目录是否符合规范，这个无所谓，主要先判断项目名是否符合规范，我们引用`validate-npm-package-name`库，这个库来判断你的项目是否符合npm的包规范？解决是否可以发布，是否和别人重复等问题。

```javascript
const { log } = require('@cat-smoker/cli-shared-utils');
const validateNpmPackageName = require('validate-npm-package-name');

const isValidate = validateNpmPackageName(projectName);

// 如果不符合规范
if (!isValidate.validForNewPackages) {
  // 这里输出信息提示出错
  log(`Invalid project name: "${projectName}", causes of this:`, 'red');
  if (isValidate.errors && isValidate.errors.length > 0) {
    log(`reason: ${isValidate.errors.join('')}`, 'red');
  }
  if (isValidate.warnings && isValidate.warnings.length > 0) {
    log(`reason: ${isValidate.warnings.join('')}`, 'yellow');
  }
  process.exit(1);
}
```

ps:**这里我们注意一点：在`@cat-smoker/cli`里面我们引入了`@cat-smoker/cli-shared-utils`的包，也就是说，我们的内部包之间有相互依赖的关系**。

那么在**我们`npm link`后**，可以支持本地的包进行本地开发了，这时候我们**不要**通过npm install 安装我们的依赖，因为有lerna管理包的实践，所以我们直接：

```bash
$ lerna add @cat-smoker/cli-shared-utils --scope=@cat-smoker/cli
# 或者
$ yarn workspace @cat-smoker/cli add @cat-smoker/cli-shared-utils@xx.xx
#（最后这里一定要写上版本号，要不然会加载远程的依赖）
```

通过这个命令，通过`scope`来确定cli可以单独依赖cli-shared-utils，不会污染别的包。

接下去判断是否有重名的文件夹

```javascript
// 这里的fs我们用的fs-extra库，这个库的方法相对于原生nodejs的比较好用
const fs = require('fs-extra');

// 这里获取到当前工作目录（current working destination），就是执行指令的目录
const cwd = options.cwd || process.cwd();
// 这个是目标项目的文件夹路径，直接解析绝对路径
const destDir = path.resolve(cwd, projectName);
// 判断项目是否有重名的文件夹
const isProjectExist = fs.existsSync(destDir);
```

如果有重名的文件夹，**这时候是不是我们要让用户来选择**？这时候，一个非常重要的库**inquirer**就是我们的需要的，这是[git仓库](https://www.npmjs.com/package/inquirer)，这个是干什么用的呢？

在我们使用vue-cli的时候，是不是经常有选项跳出来让你选择？比如选择框，确认，如下图所示，就是这个库可以做到的。

![inquirer](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/19/1718e7b15f8b1205~tplv-t2oaga2asx-image.image)

继续我们的逻辑：

```javascript
// 如果文件夹存在
if (isProjectExist) {
  const { ok } = await inquirer.prompt([
    {
      name: 'ok',
      type: 'confirm',
      message: `Warning: 您的项目${chalk.cyan(projectName)}已存在，您的操作会导致当前项目${projectName}被覆盖，确定？`,
    },
  ]);

  if (!ok) {
    return;
  }
	
  // 如果用户选择了确定，直接遍历删除对象目标地址所有的文件
  console.log(`\nWill removing ${chalk.cyan(destDir)}...`);
  await fs.remove(destDir);
}
```

至此，看上述的流程图，我们可以看到，基础业务解决了，可以进去Creator的逻辑了：

```javascript
const creator = new Creator(projectName, destDir);
creator.create(options)
```

这个具体的create业务的逻辑，我们放到下一章节进行说明。

## 基础逻辑代码

整体逻辑代码如下：

**bin/react-build-cli.js**:

```javascript
#!/usr/bin/env node

const program = require('commander');
const minimist = require('minimist');
const { log } = require('@cat-smoker/cli-shared-utils');

program.version(require('../package').version);

program
  .version(`@cat-smoker/cli ${require('../package').version}`)
  .usage('<command> [options]')

program
  .command('create <app-name>')
  .description('create a new react project')
  .option('-g, --git [message]', 'Force git initialization with initial commit message', true)
  .option('-n, --no-git', 'Skip git initialization')
  .action((name, cmd) => {
    if (minimist(process.argv.slice(3))._.length > 1) {
      log(
        'Info: You provided more than one argument. The first one will be used as the appName, check it by cat-smoker --help.'
      );
    }

    const options = cleanArgs(cmd)
    if (process.argv.includes('-g') || process.argv.includes('--git')) {
      options.forceGit = true
    }
    
    require('../lib/create')(name, options);
  });
```

**create.js**

```javascript
const path = require('path');
const fs = require('fs-extra');
const inquirer = require('inquirer');
const validateNpmPackageName = require('validate-npm-package-name');
const { log, chalk } = require('@cat-smoker/cli-shared-utils');
const Creator = require('./generator/Creator');

const create = async (projectName, options) => {
  const cwd = options.cwd || process.cwd();
  const destDir = path.resolve(cwd, projectName);

  const isValidate = validateNpmPackageName(projectName);

  // 判断项目名是否符合规范
  if (!isValidate.validForNewPackages) {
    log(`Invalid project name: "${projectName}", causes of this:`, 'red');
    if (isValidate.errors && isValidate.errors.length > 0) {
      log(`reason: ${isValidate.errors.join('')}`, 'red');
    }
    if (isValidate.warnings && isValidate.warnings.length > 0) {
      log(`reason: ${isValidate.warnings.join('')}`, 'yellow');
    }
    process.exit(1);
  }

  // 判断项目是否有重名的文件夹
  const isProjectExist = fs.existsSync(destDir);

  if (isProjectExist) {
    const { ok } = await inquirer.prompt([
      {
        name: 'ok',
        type: 'confirm',
        message: `Warning: 您的项目${chalk.cyan(
          projectName
        )}已存在，您的操作会导致当前项目${projectName}被覆盖，确定？`,
      },
    ]);

    if (!ok) {
      return;
    }

    console.log(`\nWill removing ${chalk.cyan(destDir)}...`);
    await fs.remove(destDir);
  }
  const creator = new Creator(projectName, destDir);
  creator.create(options)
};

module.exports = function (...args) {
  try {
    return create(...args);
  } catch (e) {
    console.log(e);
  }
};

```

## 总结

此项目的GitHub[仓库地址戳这里](https://github.com/xdnloveme/cat-smoker)，整体进度会比文章进度快一点，因为文章是边写边去构建`cat-smoker`这个项目的，所以大家不要着急，这个是个长期的工程。

我们会基于这个项目，向**大家一点点剖析`vue-cli`的源码**，介绍给大家`vue-cli`脚手架的架构和一些插件设计思路，重要：**当然我们这个项目不是一个简单的vue-cli拷贝项目，而是在借鉴vue-cli的架构的基础上，摒弃CRA(create-react-app)脚手架，加入的plugin插件也不是一些配置插件了，而是我们的业务结构插件**

此项目为了的是解决**多业务场景下**，**CRA**满足不了React用户的一些需求，而做的基于vue-cli模式的业务脚手架，开源出来的部分可能有点缺点，也希望大家一起改进。

## 下一章节的展望

下一章节：[从剖析Vue-cli源码出发完整的React业务脚手架实践（二）——项目的构建及服务（create）](https://juejin.im/post/6844904149507637256)会着重说明以下几个问题：

1. **如何声明模板（ejs）项目结构？**
2. **create项目的完整流程？**
3. **如何桥接插件api和业务？**
4. **如何区分debug开发环境和正式环境？**

**等等一系列问题**，文章会尽快推出，大家敬请期待，谢谢关注 Thanks♪(･ω･)ﾉ。