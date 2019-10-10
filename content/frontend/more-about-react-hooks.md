---
title: More About React Hooks
date: 2019-10-10
categories: ["前端"]
tags: ["React"]
---

# 写在前面

这篇文章是关于我前一篇关于 React Hooks 的文章（[React Hooks 能给我们带来什么 | La Vie en Code - 编码人生](https://elizurhz.cn/frontend/how-hooks-helps/)）的延伸。如果说前一篇文章是基础，告诉大家怎么使用基本的 hooks 的话，那这篇文章则是一些踩坑和更多 hooks 的用法。

# 不当的使用可能导致无限循环

当你写下 `useState` 那一行后，你可能会在某处去重设这个 state。有一个误区就是在函数式组件的函数体直接调用 set 的方法。虽然这样确实可以运行得到（它会在每次 render 的时候调用到），但是要知道 state 改变了之后组件会 re-render 的。这样就会又一次调用 set 的方法而造成无限循环:

> Too many re-renders. React limits the number of renders to prevent an infinite loop.

设置成不变的基础类型也会导致这个问题。

所以如果你想重设 state，请用 `useEffect` 或者通过事件触发。但是在使用 `useEffect` 的时候也切记不要造成无限循环。如果你懂得怎样使用 `componentWillReceiveProps`、`componentDidUpdate` 或者 `getDerivedStateFromProps` 这些生命周期更钩子的话，那么也应该知道怎样在特定的时候执行特定的操作，例如某个 prop 变化时：要用条件语句来判断，否则会在不必要的情况下执行了你编写的语句。`useEffect` 也一样。最糟糕的情况就是在 `useEffect` 中多次地、重复地重设 state，处理不当的话也会导致 re-render 的无限循环。当然除了用条件语句来判断之外，还有 dependencies，也就是 `useEffect` 的第二个参数可以用。

另外 `useSelector` 也可能导致这种问题，除了使用 `react-redux` 提供的 `shallowEqual` 之外，还可能出现各种复杂的情况。所以处理这种问题，关键是要了解 React 在什么情况下会 re-render，functional component 和 class component 在处理渲染上又有什么区别。

# useMemo & useCallback

Hooks 都只存在于 Functional Component 之中，而 Functional Component 在 re-render 时会把整个函数执行一遍，而不是像 Class Component 一样只执行生命周期钩子而不会单独执行类方法。并且由于和 React 渲染相关的函数/计算都要写在 Functional Component 的函数体中，所以这些函数不免会再执行一遍。当我们遇到比较耗性能的计算时，运算函数的重复执行会占用大量的资源。这时候就需要 useMemo 了。

useMemo 会保存一个带记忆功能的值。它会在 render 的时候，根据设定的依赖来执行。

从 `@types/react` 的包中可以找到 useMemo 的函数签名：

```TypeScript
function useMemo<T>(factory: () => T, deps: DependencyList | undefined): T;
```

它的第一个参数是一个函数，也就是在 hook 被执行的时候，会执行这个函数。但需要注意的是这个函数必须要有一个返回值，因为我们的目的就是要得到一个值啊。然后第二个参数是依赖，是一个数组，它的意思是只有数组里的变量发生变化时才会执行这个 hook，这一点和[之前我写过的 `useEffect`](https://elizurhz.cn/frontend/how-hooks-helps/) 是一样的。

然后，<i>`useCallback(fn, deps)` 实际上就相当于 `useMemo(() => fn, deps)`</i>：

```TypeScript
function useCallback<T extends (...args: any[]) => any>(callback: T, deps: DependencyList): T;
```

# useRef

`useRef` 其实可以把它视为 React 中 `ref` 的 hooks 版本。它也有“记忆”的功能，但是和 useMemo 不一样的是，它会创建一个包含 `.current` 的 JavaScript Object，并且每次 render 都会返回<i>相同</i>的 Object。我们可以在 Functional Component 中用它替代 ref 的功能，也可以[用于记忆其他的东西](https://reactjs.org/docs/hooks-faq.html#is-there-something-like-instance-variables)。

函数签名如下：

```TypeScript
function useRef<T>(initialValue: T): MutableRefObject<T>;

interface MutableRefObject<T> {
  current: T;
}
```

用法参考 [React 官方文档](https://reactjs.org/docs/hooks-reference.html#useref)：

```jsx
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` points to the mounted text input element
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```

## 存储组件内部的私有变量

官方的例子是用于获取原生 DOM 节点，而 useRef 其实并没有被限制必需用于 DOM 节点上。

```jsx
function TestComponent() {
  const count = useRef(0);
  useEffect(() => {
    count.current = 1;
  }, []);
  return <div>{count.current}</div>;
}
```

如上面的例子所示，我们可以将它设成一个值，他会返回这样一个对象：

```JavaScript
{ current: 1 }
```

通过直接给 `current` 赋值，即可修改它。并且它不会触发 re-render。这其实就很类似 class component 中的类的私有变量了。

# React.forwardRef

这东西在业务开发里基本是用不着的，在组件设计可能会用到。我们知道 Class Component 中 ref 的使用方法如下：

```jsx
class MyInput extends React.Component {
  render() {
    return <input ref={ref => (this.inputElem = ref)} />;
  }
}
```

但是在某些需求里，我们可能会需要在这个组件外部创建 ref，并以此来管理它的 focus 状态、动画等。

使用方法如下：

```jsx
const MyInput = React.forwardRef((props, ref) => (
  <input ref={(ref) => this.inputElem = ref} />
))

const ref = React.createRef()
<MyInput ref={ref} />
```

它和 `React.createRef` 是需要一起使用的。
