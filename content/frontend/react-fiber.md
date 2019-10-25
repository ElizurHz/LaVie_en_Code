---
title: 重新了解 React - React Fiber
date: 2019-10-12
categories: ["前端"]
tags: ["React"]
---

# TL;DR

我第一次使用 React 大概是在 2017 年吧，那个时候大概是 React v15 的时代。我学 React 的方法确实很普通，从例子入手，让代码能够跑起来而已。接着就是看各种实战和开源的 sample。然后我在任职的第一家公司，从不知道怎么定位的大学刚毕业的新人，正式转岗到前端。那个时候在做 Vue 的项目（我硕士的时候在学校就有在做 Vue 的项目了），也有幸碰到了几位有点经验的前端，稍微学了学一个完整的项目是怎样的构架。接着就是跳槽，背了一大堆应付面试的东西，包括 React 的一些东西，结果发现实际面试的效果却不理想。由于一直在小公司，一直追求功能的发布上线，而不是原理和性能，因此我根本没有什么机会去学习、运用这些知识。就这样，我到了网龙，在网龙前前后后也做了几个项目。大公司里，同事的水平确实不太一样，大家也会更多关注这些方面的东西。特别是最近做了新的项目，我算是项目从 0 开始搭建的主力了。我们在项目中用了最新的技术，包括 Hooks 这种社区中还没有大规模投入生产的技术，我也发现我在技术上其实存在很多的不足。在工作的过程中，一次次地看文档、一次次地与同事交流，能一次次地学到新东西，也发现其实很有必要把这些东西总结起来。

# 还在谈 React 的 diff 算法和虚拟 DOM？你可能已经 Out 了！

2018 年初和年中我经历过一些面试，React 的 diff 算法和虚拟 DOM 其实是遇到得比较多的 React 的原理的面试题，当然这些知识从一些专门讲面试的文章里可以看到。其实很多人都会为了准备面试而去背这些原理，包括我。不过说实话，里面有些东西确实开发中有用，例如列表渲染时 key 的使用，以及找出为什么列表渲染时数据不会变化的问题。但是真的要认真讨论起来，其实有一些概念并不完全正确，比如“虚拟 DOM”：
React 中有 reconciler，它是负责运行 diff 算法的。它仅仅相当于一个运算器，最终的渲染需要靠一个渲染环境来完成，例如浏览器的 DOM 环境。我们把它称作“虚拟 DOM”不过是因为这个名词使用很广泛而已，而它实际上是个 Tree 而已。React Native 也用它，在 React Native 中我们可能不应该把它称作为“虚拟 DOM”，因为 React Native 中并没有 DOM，而是会渲染成原生控件。

# 从“diff 算法”说起

其实 diff 这个概念并不是很严格地能代指我们现在的场景。[Wikipedia](https://en.wikipedia.org/wiki/Diff) 上 diff 的定义是：

> In computing, the diff utility is a data comparison tool that calculates and displays the differences between two files.

而 [React 官方文档](https://reactjs.org/docs/reconciliation.html)中提到的 `“diffing” algorithm` 其实是来自于[这篇论文](https://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf)，是计算**从一棵树转变成另一棵树需要的最少操作次数**的 state of the art 的算法，算法复杂度是 O(n³)。React 团队对它做了改进，实现了启发式的 O(n) 的算法，**这个算法叫做 [Reconciliation](https://reactjs.org/docs/reconciliation.html)**。`Reconciliation` 这个名词很少在中文社区被提到，我也是看了 React Fiber 的相关文章才知道它的，然而它可是天天被面试官问到的东西啊！不知道是大家英文不好还是怎么的，在二次传播的过程中竟然把这东西给忽略掉了。这样的二次传播其实是不友好的，这也是为什么都推荐去阅读英文文档。而 Reconciliation 具体的策略是什么，想必大家都应该了然于胸了：

1. 在比较中遇到不同的的 DOM Element 或者 Component 会直接重建该子树
2. 相同 type 的元素则不会，但是会判断是否重渲染。官方文档说的是 "React updates the props of the underlying component instance to match the new element, and calls componentWillReceiveProps() and componentWillUpdate() on the underlying instance."，但是这个文档应该是很早写的，v16.3 以后应该有所变化了，首先我们有用于优化的 PureComponent、shouldComponentUpdate、React.memo 等方式，其次 componentWillReceiveProps() and componentWillUpdate() 即将 deprecated 了
3. 遇到同类但不同 key 的元素会直接重建该子树，反之则不会

“diff 算法”在 v16.8 中当然还是有用的，但是现在的 React 渲染没有这么简单了！如果继续使用“diff 算法”，那在渲染大量元素的时候会出现严重的“掉帧”现象。而 React 团队在 v16 发布的时候就已经对渲染算法进行了重构。所以现在你还只停留在“diff 算法”中的话，你就 out 了！

# React Fiber

在 React 中，有一个 Reconciler，它就是负责运行 reconciliation 算法的。在 React v15 以及更早的版本，React 使用的是 Stack Reconciler，而 v16 开始使用的则是 Fiber Reconciler，也就是我们所说的“React Fiber”。

## Why React Fiber

简单总结下就是：Stack Reconciler （React v16 以前）在执行渲染的时候，会占用浏览器的主线程，直到渲染完成，这样任何的操作都会被阻塞；而 Fiber Reconciler （React v16 以后）则可以拆分渲染任务，每隔一段时间可以去确定是否有更高优先级的任务（例如用户输入），并优先执行它们。这个最直观的体现就是渲染一棵很大的组件树的性能大大提高了。

## What is fiber

官方文档的介绍：

> Its main goals are:

> - Ability to split interruptible work in chunks.
> - Ability to prioritize, rebase and reuse work in progress.
> - Ability to yield back and forth between parents and children to support layout in React.
> - Ability to return multiple elements from render().
> - Better support for error boundaries.

### requestIdleCallback & requestAnimationFrame

[React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture) 这篇文章说到，fiber 是基于这两个浏览器 API 来优化的。

MDN 上这两个 API 的定义如下：

> window.requestIdleCallback() 会在浏览器空闲时期依次调用函数，这就可以让开发者在主事件循环中执行后台或低优先级的任务，而且不会对像动画和用户交互这样延迟敏感的事件产生影响。

> window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行。

也就是说，requestAnimationFrame 可以用于执行高优先级的任务，而 requestIdleCallback 可以用于执行低优先级的任务。那么二者到底是怎么执行的呢？

![Life of a frame (main thread edition)](https://cdn-images-1.medium.com/max/2600/1*ad-k5hYKQnRQJF8tv8BIqg.png)

浏览器中，每一帧里需要做的事如上图所示。其中 也就是说，requestAnimationFrame 的 callback 会在 rAF 阶段执行。假设我们想要我们动画的 FPS 保持在 60，那么我们需要保证在 1 帧（1000/60 ms）里能够做完这些事。而如果 1 帧里做完了这些事并且有空闲时间，那么就可以执行 requestIdleCallback 的 callback。另外关于 60FPS，通常我们认为 60 是个理想的数值，FPS 小于 20 会感受到明显的卡顿，而大于 60 需要额外的资源但是人眼无法感受到明显的变化。

## Scheduling

关于调度：

1. 没必要实时更新所有的 UI
2. 不同类型的更新有不同的优先级，例如动画的优先级会高于 store 的更新
3. React 的实现方式是 "pull" 的，它会帮助我我们决定事务的优先级。它不像某些库的 "push" 方式，在有新的 data 的时候就执行计算。

而 Fiber Reconciler 中有个 Scheduler，它可以用于调度任务，高优先级的任务会被先执行，而 diff 属于低优先级的任务，会在高优先级的任务执行完成后再执行。

## 渲染过程

在执行渲染的时候，Fiber Reconciler 会根据虚拟 DOM 生成一棵 Fiber Tree。这个阶段是可以被打断的。在生成 Fiber Tree 的时候，每生成一个节点，都会把控制权交还给主线程，看是否有更高优先级的任务，如果没有则继续构建，否则会打断 Fiber Tree 的构建。如果在这个阶段被打断，那么 Fiber Reconciler 会重新生成新的 Fiber Tree。这个阶段主要是在 React 组件的 render/reconciliation 阶段，对应生命周期钩子就是渲染或者重渲染前那些。而 Fiber Tree 上的每个节点中，如果有 side effect，就会进行标记，在这个阶段中会生成一个 effect list。

在经过这个阶段之后，会进入到 commit 的阶段，此时会对需要更新的节点进行批量更新，如更新 DOM 树、调用组件生命周期函数以及更新 ref 等内部状态。该阶段不能被打断，所以尽可能不要在 componentDidMount、componentDidUpdate 和 componentWillUnmount 中做很耗资源的操作。

## 参考

[React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)

[React Fiber 原理介绍 - 前端大宝剑 - SegmentFault 思否](https://segmentfault.com/a/1190000018250127)

[完全理解 React Fiber | 黯羽轻扬](http://www.ayqy.net/blog/dive-into-react-fiber/#articleHeader3)
