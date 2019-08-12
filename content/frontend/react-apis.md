---
title: React 中一些容易被忽视的 API
date: 2019-07-22
categories: ["前端"]
tags: ["React"]
---

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
