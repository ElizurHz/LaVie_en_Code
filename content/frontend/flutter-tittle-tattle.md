---
title: 面向前端的 Flutter 与 flutter_web 杂谈
date: 2019-07-19
categories: ["前端", "移动"]
tags: ["flutter", "flutter_web", "Dart"]
---

# TL;DR

> 写在前面：本文写于 flutter_web 技术预览版（也就是 Flutter 1.5）发布后不久，由于 Preview 版本迭代很快如有后续更新，当您看到本文时，本文所提及的内容可能会 out of date。我也会持续关注 flutter 以及 flutter_web 的发展。

flutter 与前端开发到底有什么关系呢？Flutter 其实是移动 App 的框架，它与之前的移动端开发最大的不同就是使用了声明式的 UI，并且 Flutter 的许多概念都是受了 react 的启发的，例如 stateful & stateless component。开发团队曾经说要支持 web 端，而终于在 2019 年 5 月的 Google I/O 上的 1.5 版本发布时，公布了 flutter_web 的第一个技术预览版。现在 Flutter 也已经更新到了 1.8.0 版了。对于前端开发而言，跨平台的技术也是我们需要关注的，毕竟现在的趋势就是这样，说不定我们可能也有被移动端开发者“抢饭碗”的可能性，因为之前的 React Native 不是戏称前端开发要“抢了移动端开发的饭碗”嘛（但是事实上还差得远呢）。对于 React Native 没做到的事，我会很期待 Google 的 Flutter 能不能做到。

# 从 flutter 说起

可能很多前端更关注的是 flutter_web 到底是什么东西，这边可以先说，它其实和 Flutter 是同一个东西，只是运用了不同的编译和渲染方式而已。所以为了了解 flutter_web，我们需要先了解 Flutter。下面就说下基本的原理（算是从[官网的 Technical Overview](https://flutter.dev/docs/resources/technical-overview)上总结出来的，我个人认为是对于前端开发者来说易理解的）：

## Everything's a Widget

这里就先不说 Widget 具体是什么以及底层实现了，我们可以把它当做是和 React.Component 一样的东西，但是它所涉及的会比 React.Component 更多，例如样式、布局等等，这是与 JavaScript 的开发最大的不同（与 SwiftUI 其实是一样的）。

## Building Widget

我们写出来的 App 最后会被编译成一个 Widget Tree。

> A widget’s build function should be free of side effects. Whenever it is asked to build, the widget should return a new tree of widgets regardless of what the widget previously returned.

这个 Widget Tree，我们其实可以把它理解成是 React 的虚拟 DOM 一样的东西。如果需要 build 的时候，也就是说 Widget Tree 有变更的时候，会返回一棵新的 Widget Tree，并且无副作用。这个生成新 Tree 的算法，大致相当于 React 的 diff 算法。在构建 Widget 的时候，我们需要一个 key 的 property，这个就是用于比较 Widget Tree 的。

## Handling user interaction

这部分其实就是关于 Stateful Widget 的。这部分用起来其实和 React 的 state 区别不大，前端理解起来也不难。

# Dart

提到 Flutter，就不得不说到 Dart 这个语言。我们在做 JavaScript 开发时肯定会遇到很多痛点，例如静态类型检查。而我们确实有相应的解决方法，比如 flow 和 TypeScript，但是不是所有开发者都会用它们，迁移起来也有一定的难度，特别是 TypeScript，对于没有 Java/C++ 开发经验的人可能会有点棘手，并且上手一段时间后能可能能跑起来，但是写出来的代码仍然有漏洞，比如滥用 `any`、类型约束丢失等等。
而我会把 Dart 看做和 TypeScript 类似的语言，对于 JavaScript 开发者，或是 Java 开发者，都很好上手。但是 Dart 确实有一些很强大的功能，这也是 Flutter 高性能的基础。

## JIT(Just-in-Time)

JIT 编译是 Flutter 在开发中能够实现 hot reload 的基础。它会使用 DartVM 来进行 JIT 编译。

## AOT(Ahead-of-Time)

AOT 编译是在生产环境编译时，直接将我们的代码编译成 ARM Code，这样用 Flutter 写的 app 就不像用其他框架写的 app 那样需要在运行时编译，可以极大提高运行的速度，这也是 Flutter 的主要亮点之一。

## Compile to JavaScript

Dart 可以通过 dart2js 编译成 JavaScript 代码，这是 flutter_web 的基础。

## 异步编程

和 JavaScript 一样，Dart 中有内置一些异步编程的 library。

1. async/await，和 ES7 的 async/await 是类似的，可以参考 [Flutter 官方教程视频](https://www.youtube.com/watch?v=SmTCmDMi4BY)
2. Future，和 Promise 是类似的，可以参考 [Flutter 官方教程视频](https://www.youtube.com/watch?v=OTS-ap9_aXc)
3. Stream，提供了流的发布与订阅，是 Dart 异步编程的核心之一，可以参考 [Flutter 官方教程视频](https://www.youtube.com/watch?v=nQBpOIHE4eE)

# 从 Flutter 到 flutter_web

上面已经简要说明了一下 Flutter，那么 flutter_web 又是什么呢，它和 Flutter 有什么关系呢？我们可以通过两张 layer cake 图来看下吧。

![Flutter's Layer Cake](/images/flutter-tittle-tattle/flutter-layer-cake.png)

![flutter_web's Layer Cake](/images/flutter-tittle-tattle/flutter_web-layer-cake.png)

从上面两张图我们可以看到，Framework 层其实二者是完全一样的！不同之处仅仅在于渲染引擎。在 Flutter 中，绘图引擎使用的是 skia，而在 flutter_web 中，元素都是绘制在 canvas 上的！

我们可以结合实际的例子来看看，例如 NYTIMES 做的 [KENKEN](https://www.nytimes.com/games/prototype/kenken#/) 小游戏。

如果使用 Chrome Devtools 查看的话，我们可以看出它的 DOM 结构是这样的：

![](/images/flutter-tittle-tattle/kenken-1.png)

我们认知的 Web App 结构完全不一样。然后我在 [Reddit](https://www.reddit.com/r/FlutterDev/comments/blvrou/flutter_for_web_preview_goes_public/) 上找到了一位 flutter_web 的开发者的留言：

![](/images/flutter-tittle-tattle/kenken-2.png)

大意是：`<flt-scene-host>` 是展示层，包括 DOM 和 Canvas 等；`<flt-glass-pane>` 是用于监听事件的层；`<flt-ruler-host>` 是用于测量 text 的层。

这种渲染方式就与 Flutter 在移动端的渲染方式完全一样了。在移动端，Flutter 靠 skia 在原生的 Canvas 上直接绘制元素/控件，这样不仅渲染速度快，而且在不同平台上可以显示出同样的样式，不会因为各平台原生控件的不同而导致样式的不同。目前 Flutter 提供了 Cupertino 和 Material 两种风格可供选择，分别代表 iOS 与 Android 的两种设计风格。

换句话说，Flutter 团队是完整地将移动端的渲染方式搬运到了 web 上，以使用 canvas 为主，而不是像 React 那样还是会渲染成真实 DOM。

## 实际体验

要把项目从 Flutter 迁移到 flutter_web 不是一件容易的事。首先 flutter_web 不使用 Cupertino 和 Material，所以 Widget 需要更改；其次它们的依赖库不一样，项目依赖也需要更改。我尝试过把 Flutter 应用迁移到 flutter_web，但是仍然没法处理好这两个问题，所以暂时放弃了。

我更期待的其实是像 Taro 那样，能够在一个项目中，根据配置和命令的不同，能够编译成不同的版本。这样的好处就是，我们可以直接编译出 Native App 的 H5 版本，这对于一些分享功能是非常方便的，因为分享出来的页面是需要 H5 的。看来 flutter_web 还有很长的路要走，我也很期待它发布正式版的时候会是什么样子。
