---
layout: post
title: 简单的webpack loader
feature-img: "assets/img/pexels/webpack.jpg"
thumbnail: "assets/img/pexels/webpack.jpg"
image: "assets/img/pexels/webpack.jpg" #seo tag
tags: [webpack, Typescript]
author-id: OYX
excerpt_separator: <!--more-->
---

工作中遇到这么一个需求：

- 页面上线之后允许通过配置文件修改一些行为；
- 配置文件必须要是 editing-by-notepad friendly 的；

所以配置文件毫无疑问是一个异步加载的 json 文件。

那么问题来了：**如何能够在 Typescript 中正确的引用该配置文件的类型呢？**

<!--more-->

首先想到的是通过 vscode 的[Paste JSON as Code](https://marketplace.visualstudio.com/items?itemName=quicktype.quicktype)插件隔一段时间粘贴一下类型来做到类型的引入，但这有个问题：**不够自动化，不高端！**

俗话说“懒惰使人进步”，我们必须想一个办法能够在开发时引入配置文件的类型来帮助我们写代码，部署时候又能够正确获得我们需要的 json 文件。

说到自动化，我们平时使用的 webpack 就是一个很好的例子，我本人没有用过 gulp 这类工具，不过想来自动化的工作流无非就是`监视文件->处理->输出`这一套，我们可以找一找合适的工具。

### 从何处来，往何处去

现在目标很明确，我们已有的是一个 json 文件，开发时需要获得 json 的类型，那么这个 json 就应该被读取出来然后作为一个普通的 js 对象出现在一个 ts 文件中。每次 json 被修改了，这个转换过程就会自动执行一次。

这么说来我们只要用 node 做一个监视指定文件的工具，通过 fs.readFile 的方式把 json 的内容读出来，然后写成

```typescript
const configJson = {
  /*json.parse出来的配置信息*/
};
export type ConfigJson = typeof configJson;
```

这个样子，最后再把上面这段内容写入指定位置的一个 ts 文件中就 OK 了。

### 慢着，这样真的好吗

可是这种方法总觉得怪怪的，有点别扭。为什么呢？

因为事实上我们写代码期间要的不是 json 文件，而是那个生成的 ts 文件，只有在我们刷新浏览器测试配置文件是否正确工作的时候它才有意义。就是说实际上按照使用顺序来说我们应该是：
`写ts->写代码->用json`，可上面的流程却变成了`写json->生成ts->写代码`。

所以如果能够按照前一种流程走就会更让人舒心。另外以上方案在我们正常的工作流中插入了额外的环节，非常让人心虚。

### 那么还有什么能够帮我们在写代码之后和用代码之前做前一种方案的那种转换呢

很显然这就是 webpack 的角色了嘛。
webpack 有两种扩展机制，一种叫 loader，一种是 plugin，loader 用于处理资源，plugin 用于改变 webpack 的行为。
很显然我们需要的就是一个 loader，一个能帮我们把我们的 ts 文件中那个对象变成一个 json 文件的 loader。

## 开搞

首先创建一个练习文件夹，就叫它`test`吧。

安装`loader-runner`和`webpack`到 devDependencies。

然后在项目根目录下创建一个 loaders 文件夹，以后我们所有的 loader 就都放在这里文件夹里了。

我们知道 webpack 的生命周期是这个样子的

<img src="/assets/img/pexels/webpack-life-time.jpg" />

loader 的加载是在最开始解析 webpack.config.js 的时候就完成了的，所以如果不想每次测试 loader 都`ctrl+c -> 向上箭头 -> enter的话`就需要用我们的 loader-runner 来执行 loader 做测试。

loaders 文件夹建立 loader-runner 的脚本，就叫做 run-loaders.js 好了。内容如下：

```js
const fs = require("fs");
const path = require("path");
const { runLoaders } = require("loader-runner");

runLoaders(
  {
    resource: "./src/runtime.config.ts",

    loaders: [
      {
        loader: path.resolve(__dirname, "./runtime-configs-transformer"),
        options: {
          name: "demo.[ext]"
        }
      }
    ],

    context: {
      emitFile: () => {}
    },
    readResource: fs.readFile.bind(fs)
  },
  (err, result) => (err ? console.error(err) : console.log(result))
);
```

很简单，就是 resource 中指明要被处理的文件，然后在 loaders 中如同 webpack 配置文件那样依次写下要使用的 loader 就可以了。

我们将要写的 loader 的名字叫做`runtime-configs-transformer`，它的功能就如上述所说将一个运行时配置文件做转换。

这时候我们的想法是这样的：

- 我们需要使用 ts 文件来做开发
- ts 文件需要被转换，那么我们可以使用 typescript 来编译这个文件，将其转换为普通 js 文件
- 转换之后的 js 文件类似于这样

```js
const runtimeConfig = {
  /*配置内容*/
};
```

- 如果能够将这段内容作为字符串的代码 eval 出来就可以获得声明的对象了
- 最后只需要将其写入目标文件 runtime-config.js

很多时候写 loader 为我们需要动用 ast 来将代码分析为语法树，可是我们这次的需求并没有那么复杂。

查看文档没有发现 typescript 的 compiler 该如何调用，后来就想到既然我们的配置文件格式是固定的，那我干脆直接把他当作一个文本文件来处理岂不是更方便，于是就有了如下代码:

```js
// content是被处理的文本内容
const exportIndex = content.indexOf("export");
const equalIndex = content.lastIndexOf("=");
const result = JSON.stringify(
  eval(content.substring(equalIndex + 1, exportIndex))
);
```

好嘛，先 eval 再 JSON.stringify 出来就是标准的 json 字符串了，最后再写入目标位置

```js
fs.writeFileSync(
  path.resolve(__dirname, "../public/runtime-config.json"),
  result
);
```

就成功了。

这次先解决工作中的问题，下次有需要再来学习ast吧。