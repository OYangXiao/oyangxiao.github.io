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

想到了可以使用webpack的loader来完成这样的工作，因为loader可以指定要处理什么样名字的文件，那么我或许可以指定处理到我的目标文件的时候将其转换并输出到JSON文件。

## 开搞（然而这一节方法是错的，最后会有修正）

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
- 待转换的文件是这样的

```typescript
const runtimeConfig = {
  /*配置内容*/
};
export type RuntimeConfig = typeof runtimeConfig
```

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
const equalIndex = content.indexOf("=");
const exportIndex = content.lastIndexOf("export");
const result = JSON.stringify(
  eval(content.substring(equalIndex + 1, exportIndex))
);
```

OK，先 eval 再 JSON.stringify 出来就是标准的 json 字符串了，最后再写入目标位置

```js
fs.writeFileSync(
  path.resolve(__dirname, "../public/runtime-config.json"),
  result
);
```

就成功了。

## 然而很快就打脸了

在vue.config.js中配置这样一条规则
```js
module.exports = {
  chainWebpack: config => {
    config.module
    .rule('my-config')
    .test(function(filename){return filename === path.resolve(__dirname, '/src/runtime.config.ts')})
    .use('./loaders/runtime-configs-transformer')
    .loader('./loaders/runtime-configs-transformer')
    .end()
  }
}
```
却发现没有生效，what happened？

### 原来是被tree shake掉了

webpack从某一版本中加入了tree shake的功能，就是将不使用到的文件不打包。

所以我们的config文件仅仅导出了type，却没有导出有效的代码，也没有其他地方引用它，所以被tree shake掉了。

最后没办法，想了一个比较tricky的方式，就是直接在rule的test这里写逻辑代码
```js
module.exports = {
  chainWebpack: config => {
    config.module
    .rule('my-config')
    .test(function(filename){
      if(filename === path.resolve(__dirname, /*将test的文件名改成某个只引用一次的文件，比如pages中的入口文件*/)}){
        // 直接用fs读出要处理的文件内容
        // 读出来的东西需要转换为string内容做处理
        const code = fs.readFileSync(path.resolve(__dirname, './src/runtime.config.ts')).toString()
        // 将前面loader的处理过程直接拿过来
      }
    })
    .end()
  }
}
```
最后就只能先这么样了，以后再来研究如何用webpack执行一些自动化的工作，或者我还是需要引入gulp？

