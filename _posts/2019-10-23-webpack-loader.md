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

很显然这就是 webpack 的角色了嘛。webpack 有两种扩展机制，一种叫 loader，一种是 plugin，loader 用于处理资源，plugin 用于改变 webpack 的行为，很显然我们需要的就是一个 loader，一个能帮我们把我们的 ts 文件中那个对象变成一个 json 文件的 loader。

## 开搞

首先创建一个练习文件夹，就叫它`test`吧。

安装`loader-runner`和`webpack`到 devDependencies。

然后在 test 目录下创建
