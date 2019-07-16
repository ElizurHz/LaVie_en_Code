---
title: React Hooks 对于我而言意味着什么
date: 2019-07-08
categories: ["前端"]
tags: ["React", "react-redux", "JavaScript"]
draft: true
---

# TL;DR

很多 React 的新技术，我们大多数人也只是听说而已，实际上投入生产的机会非常少。首先很多旧项目不会去升级 React 的版本，很多团队因为不精通技术而且只需要专注于做业务，所以也没有这样的成本来升级 React 的版本。而我刚好遇到了好的机会，让我有足够的技术自由度，我也趁着一个重构的机会，顺便把负责组件全部用 hooks 重构了。这次重构是在 2019 年 6 月下旬开始的，并且也要感谢 react-redux 在 6 月上旬的时候发布了 v7.1.0，Hooks API 正式投入生产了，让我能够大展身手。

# Before Hooks

# After Hooks

# Tech Spec

## useEffect

Class Component 对于我们来说有足够的功能新，因为我们可以通过自定义生命周期钩子来实现很多复杂的功能，这是在 React v16.8 前 Functional Component 无法实现的。这也是之前我喜欢用 Class Component 的原因。而 useEffect 这个近乎“万能”的 hook 可以实现几乎所有的生命周期钩子，这对于 Functional Component 非常关键，也是我重构组件最依赖的一个 hook。useEffect 的类型签名如下:

```TypeScript
function useEffect(effect: EffectCallback, deps?: DependencyList): void;

type EffectCallback = () => (void | (() => void | undefined));
type DependencyList = ReadonlyArray<any>;
```

可以看到 `useEffect` 有两个参数，第一个参数是一个 callback，里面写的是我们的逻辑。而第二个参数不要小看它，虽然它是可选参数，但是如何玩转 `useEffect` 完全就靠它了！它是“依赖”，React 官网也有不小的篇幅来描述怎样使用它。简而言之，这个“依赖”，就是**当它改变的时候才会执行里面的 callback**。先说下不传这个参数的情况，那就是每次 re-render 都会调用这个 hook。而我们在实际开发中往往不会希望我们所有的 `useEffect` 的 callback 在每次渲染都被调用一次。所以“依赖”的作用就在于此，例如：

```TypeScript
useEffect(() => {
  console.log(props.name)
}, [props.name])
```

这个 hook 只会在 `props.name` 发生变化的时候把 `props.name` 在控制台打印出来。这个相当于 Class Component 中的：

```TypeScript
public componentDidUpdate(prevProps) {
  if (this.props.name !== prevProps.name) {
    console.log(this.props.name)
  }
}
```

或者是即将被废弃的 `componentWillReceiveProps`。

而如果我们修改一下这个依赖呢？这其实挺纠结的，因为如果你在你的项目中使用了 eslint，那么这里它会提示一个 `exhaustive-deps` 的 warning。而如果你开了 Prettier 的 eslint 错误自动修复。。。那不好意思，要么你只能不写，要么会被强制改成里面所有用到的依赖，如 state、props 或者其他变量等。起初我也陷入了深思，到底是不允许这么写，还是 eslint 有毛病呢？但是实际上，我们应该先抛弃 eslint，先在这将它 disable 了，因为 `useEffect` 能够做到的远比我们想象的多。

上面既然我们实现了 `componentDidUpdate`，那其他的钩子呢？有两个很关键的钩子 `componentDidMount` 和 `componentWillUnmount` 我们还不知道怎么实现呢。而我之前提到它是“近乎‘万能’”的，是的，它就是可以实现这些生命周期钩子的效果。

为了实现 `componentDidMount`，我们只需要在依赖中传入一个空数组 `[]`。

而为了实现 `componentWillUnmount`，我们只需要在 `useEffect` 中 return 一个 callback，那个 callback 便是在 `componentWillUnmount` 时执行的。上面提到的

```TypeScript
type EffectCallback = () => (void | (() => void | undefined));
```

是有写返回值类型的，而它也不是白写的，正是用在这种场景。

---

依赖很万能，但是不要乱用，因为乱写依赖会导致你的组件渲染的行为异常，甚至会导致 Infinite Loop (`useEffect` 的 callback 触发了某个变量的变化导致了 re-render, 而 re-render 又触发了 useEffect 并运行了 callback)。因为 re-render 大概是我们在 React 开发中最经常需要处理的一件事了，所以用好 `useEffect` 是用好 React hooks 的关键。

## Hooks in react-redux

draft：

1. 使用 Hooks 重写了 connect
2. 提供了 Hooks 来替代 connect，可以在 Functional Component 中使用
3. useSelector 和 shallowEqual

## React.memo

`React.memo` 其实并不是 Hooks 的内容，但是由于我们只能在 Functional Component 中使用 Hooks，Class Component 中能用的生命周期钩子也都不能用，所以之前的
`React.PureComponent` 和 `shouldComponentUpdate()` 也都无法使用。那么对于应用优化，我们常用的手段在 Functional Component 中都无法使用了。`React.memo` 的出现就很有效地解决了这个问题。它能实现与 `React.PureComponent` 和 `shouldComponentUpdate()` 相同的功能，就是防止父组件 re-render 导致的子组件的不必要的 re-render。

对于“不必要的 re-render”，举个例子来说，就是父组件使用 `useSelector` 订阅了 redux store 的某个属性，但是它与子组件无关。如果这个属性有变化，那么按照 React 的 diff 算法，它的子组件都会 re-render 一遍。这也不能说 React 本身有问题，因为 diff 算法已经足够好，能够高效地解决 React 渲染计算的问题。那么我们其实是不希望与该属性无关的子组件去 re-render 的，因为对我们来说它其实一点变化都没有。所以 `React.memo`、`React.PureComponent` 和 `shouldComponentUpdate()` 就是用于解决这种问题的。

它在 react-redux 中也挺重要的，因为官方说 `useSelector` 不会阻止子组件的不必要的 re-render，而官方也推荐使用 `React.memo` 进行优化。

## 单元测试

Jest + Enzyme 是 React 中常用的单元测试库，但是 Enzyme 官方文档中提到：`With React 16 and above, instance() returns null for stateless functional components.`。对于 Hooks 而言（使用了 Hooks 的组件必然是 functional components），虽然官方文档也说明了[对 Hooks 的支持](https://github.com/airbnb/enzyme#react-hooks-support)：目前只在 `shallow()` 里提供有限的支持，但是想要照搬 class component 中用的测试套路显然是不行的。React 官方则推荐使用 [react-testing-library](https://github.com/testing-library/react-testing-library) 来测试。但是这些目前只能测试 React 官方的 Hooks，对于 react-redux 的 `useDispatch` 和 `useSelector`，目前我还没有找到测试的方法，也没有找到相关的教程或者文档，甚至 react-redux 官方也没有。所以只能稍微研究或者等待一下了。
