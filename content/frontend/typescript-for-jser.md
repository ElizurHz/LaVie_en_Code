---
title: React + TypeScript 从云玩家到入门
date: 2019-03-24
categories: ["前端"]
tags: ["typescript"]
---

# React + TypeScript 从云玩家到入门

TypeScript 对我们开发最大的帮助就是类型检查，所以玩转 TypeScript 其实就是在玩转类型。而 TypeScript 本身和 Java, C# 等面向对象的语言有非常多类似的地方，有相应经验的读者能够较快地入门。本文旨在于帮助无面向对象开发经验的 JavaScript 使用者能最快速地上手并使用 TypeScript 编写 React 应用。由于篇幅和定位所限，本文将不会涉及到较为复杂的组件设计模式。

## TypeScript 基础

这部分主要介绍入门 TypeScript 必须掌握的，以及 React 开发中经常会用到的一些语法规范。

### 基础类型

- number: 数字
- string: 字符串
- boolean: 布尔值
- Array: 数组。在使用数组类型的时候必须标记数组中的数据类型，如 `number[]` 或者 `Array<string>`
- Tuple: 元组。`let x: [string, number];`
- any: 任意类型
- void: 空，一般用于函数的返回值
- never: 永远不会出现的类型，一般用于函数的返回值，例如抛出错误或者永远不结束的死循环。

### 可选类型

标记为可选类型的属性并非是必须的。

```typescript
interface IObj {
  str?: string;
  num: number;
}

// tslint 不报错
let obj1: IObj = {
  num: 123
};

// tslint 报错
let obj2: IObj = {
  str: "asdf"
};
```

![optional-error](/images/typescript-for-jser/optional-error.png)

### Readonly

一般在 Interface 和 Class 中会使用。设置为 `readonly` 的变量、属性，一旦定义后就无法直接修改。如果在代码中有修改的操作，则 tslint 会报错。

![readonly-error](/images/typescript-for-jser/readonly-error.png)

### 函数

```typescript
function add(x: number, y: number): number {
  return x + y;
}

let myAdd: (x: number, y: number) => number = function(
  x: number,
  y: number
): number {
  return x + y;
};
```

函数的类型限制主要在参数和返回值的位置需要定义。如果在等号左侧已经定义了类型，那么右侧可以不需要写类型，如：

```typescript
let myAdd: (baseValue: number, increment: number) => number = function(x, y) {
  return x + y;
};
```

这个在 TypeScript 中被称作 “contextual typing”。

## interface

> One of TypeScript’s core principles is that type-checking focuses on the shape that values have. This is sometimes called “duck typing” or “structural subtyping”.

Interface 一般用于描述一个较为复杂的限制，例如在一个对象中限制某些属性的类型。有时被称为“鸭子类型”。

> "当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。"
> 在鸭子类型中，关注的不是对象的类型本身，而是它是如何使用的。

```typescript
interface ITest {
  a: number;
  b: string;
}

const test: ITest = {
  a: 1,
  b: "123"
};
```

- Interface 和 class 的区别:

  - Interface 和 class 在 TypeScript 中都可以去描述一个限制，与 PHP、Java 中的 interface、class 一样，TypeScript 中的 interface 只做声明不做实现。
  - 如果把上面的 interface 声明换成 class，tslint 不会报错，代码可以正常执行，但是二者编译出来的 JavaScript 不同。

```typescript
// 使用 interface 编译后的代码
var test = {
  a: 1,
  b: "123"
};

// 使用 class 编译后的代码
var ITest = /** @class */ (function() {
  function ITest() {}
  return ITest;
})();
var test = {
  a: 1,
  b: "123"
};
```

- class 可以使用 implements 关键字来实现 interface。

### 类型别名

TypeScript 中允许使用 type 关键字来声明类型别名。

```typescript
type State = {
  a: string;
  b: Array<number>;
};
```

类型别名和 interface 通常具有同样的功能，但是类型别名不能被 `extends` 或者 `implements`，也不能 `extends` 或者 `implements` 其他类型。并且类型别名在编译器中显示的是 Object 字面量类型，而 interface 显示是 interface。

> Because [an ideal property of software is being open to extension](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle), you should always use an interface over a type alias if possible.
> 【软件中的对象应该对于扩展是开放的，但是对于修改是封闭的】

### 泛型

```typescript
function identity(arg: any): any {
  return arg;
}
```

当我们在写这样一个函数的时候，参数和返回值都可以是任意类型，但实际上这样我们就丢失了参数和返回值的类型信息（实际上输入和输出应该是相同的类型）。
这时我们可以用一个类型变量 T：

```typescript
function identity<T>(arg: T): T {
  return arg;
}
```

这样保证了参数和返回值的类型都是 T。
我们可以通过如下的方式使用这个泛型函数：

```typescript
let output = identity<string>("myString"); // type of output will be 'string'
```

### 映射类型

映射类型包括 Partial 和 Readonly，它们可以在原有 interface 的基础上分别映射出所有属性都是可选类型，和所有属性都是只读的新的 type。这两种映射类型实现如下：

```typescript
type Readonly<T> = { readonly [P in keyof T]: T[P] };
type Partial<T> = { [P in keyof T]?: T[P] };
```

## 在 React 中使用 TypeScript

### 创建一个 React + TypeScript 的项目

1. create-react-app
   通过 create-react-app 来搭建，我们只需要在命令中加入额外参数即可：
   `create-react-app my-app --scripts-version=react-scripts-ts`
   搭建出来的项目结构与 JavaScript 版本基本相同，区别在于根目录有`tsconfig.json`、`tslint.json` 等 TypeScript 相关的文件。`tsconfig.json` 用于告诉 `ts-loader` 如何编译 .ts 文件，而 `tslint.json` 则是 tslint (TypeScript 语法检查器) 的配置。
2. 手动安装
   手动安装需要通过 npm 或者 yarn 安装 `typescript` 和 `ts-loader` 等 TypeScript 相关的 package，同时需要手动配置 `tsconfig.json`。

### DefinitelyTyped - \*.d.ts

[GitHub - DefinitelyTyped/DefinitelyTyped: The repository for high quality TypeScript type definitions.](https://github.com/DefinitelyTyped/DefinitelyTyped)

TypeScript 的模块发布的时候都是打包成 JavaScript 来发布的，但是这样会丢失定义好的数据类型。 .d.ts 可以约定 type, class, interface, function, 变量以及常量等的行为，相当于是一个 package 或者 library 的“说明书”。

在 React 中，我们常常会需要安装一些如 `@types/react-router-dom` 这种以 `@types` 作为开头的 package。在 JS 中我们直接 `npm install react-router-dom` 即可，但是在 TypeScript 中，如果没有 DefinitelyTyped，将会报如下的错误：
![@types-error](/images/typescript-for-jser/types-error.png)

当然并不是所有的 package 都需要一个对应的 `@types` ，不少 package 自身已经有 .d.ts 文件了，例如 antd，所以我们不需要额外的 `@types` 。

### 无状态组件

在 JavaScript 中，我们可以按如下的方式写一个 Stateless Functional Component ：

```javascript
import React from "react";
const Button = ({ onClick, children }) => (
  <button onClick={handleClick}>{children}</button>
);
```

但是在 TypeScript 中，我们需要加入类型，所以具体怎么加呢？我们可以通过 `@types/react` 中的 `index.d.ts`来进行分析：

> 注：以下代码都去除了大段的注释，若需要查看官方的注释请至 `npm install` 后下载的 `./node_modules/@types/react` 中查看。

```typescript
type SFC<P = {}> = StatelessComponent<P>;
interface StatelessComponent<P = {}> {
  (props: P & { children?: ReactNode }, context?: any): ReactElement<
    any
  > | null;
  propTypes?: ValidationMap<P>;
  contextTypes?: ValidationMap<any>;
  defaultProps?: Partial<P>;
  displayName?: string;
}
```

从这段代码中可以看出，props 是 P 和 一个对象的联合类型，P 指的是我们通常传入的 props，而对象中的 children 就是 React 中的 children，是可选类型，type 是 ReactNode，是 React 中的类型。另外在 React 的 `index.d.ts`中，对于事件有专门的类型定义 `MouseEvent`（同时也有其他 Event 的类型定义）：

```typescript
interface MouseEvent<T = Element> extends SyntheticEvent<T> {
  altKey: boolean;
  button: number;
  buttons: number;
  clientX: number;
  clientY: number;
  ctrlKey: boolean;
  getModifierState(key: string): boolean;
  metaKey: boolean;
  nativeEvent: NativeMouseEvent;
  pageX: number;
  pageY: number;
  relatedTarget: EventTarget;
  screenX: number;
  screenY: number;
  shiftKey: boolean;
}
```

所以针对上面的 JavaScript 代码，我们可以按如下的方式写出相应的 TypeScript 代码：

```typescript
import React, { MouseEvent, ReactNode } from "react";
interface IProps {
  onClick(e: MouseEvent<HTMLElement>): void;
  children?: ReactNode;
}
const Button = ({ onClick: handleClick, children }: IProps) => (
  <button onClick={handleClick}>{children}</button>
);
```

其中 `HTMLElement` 被定义在`@types/react` 中的`global.d.ts`

```typescript
interface Element {}
interface HTMLElement extends Element {}
```

### 有状态组件

无状态组件其实是一个函数，只需要按照函数的写法来实现即可。而对于有状态组件，这边有一个计数器的例子：

```typescript
import React, { Component } from "react";

import Button from "./Button"; // 这里的 Button 是上面 SFC 部分的 Button 组件

const initialState = { clicksCount: 0 };
type State = Readonly<typeof initialState>;

class ButtonCounter extends Component<object, State> {
  readonly state: State = initialState;

  public render() {
    const { clicksCount } = this.state;
    return (
      <>
        <Button onClick={this.handleIncrement}>Increment</Button>
        <Button onClick={this.handleDecrement}>Decrement</Button>
        You've clicked me {clicksCount} times!
      </>
    );
  }

  private handleIncrement = () => this.setState(incrementClicksCount);
  private handleDecrement = () => this.setState(decrementClicksCount);
}

const incrementClicksCount = (prevState: State) => ({
  clicksCount: prevState.clicksCount + 1
});
const decrementClicksCount = (prevState: State) => ({
  clicksCount: prevState.clicksCount - 1
});
```

若要读懂这段代码，可以从`@types/react` 中的 `index.d.ts`说起：

> 注：以下代码都去除了大段的注释，若需要查看官方的注释请至 `npm install` 后下载的 `./node_modules/@types/react` 中查看。

```typescript
interface Component<P = {}, S = {}, SS = any>
  extends ComponentLifecycle<P, S, SS> {}

class PureComponent<P = {}, S = {}, SS = any> extends Component<P, S, SS> {}

class Component<P, S> {
  constructor(props: Readonly<P>);
  setState<K extends keyof S>(
    state:
      | ((prevState: Readonly<S>, props: Readonly<P>) => Pick<S, K> | S | null)
      | (Pick<S, K> | S | null),
    callback?: () => void
  ): void;

  forceUpdate(callBack?: () => void): void;
  render(): ReactNode;
  readonly props: Readonly<{ children?: ReactNode }> & Readonly<P>;
  state: Readonly<S>;
  context: any;
  refs: {
    [key: string]: ReactInstance;
  };
}
```

我们常用的 Component 类就是在这里被定义的，其中 P 代表 props，S 代表 state。在这里我们可以发现，props 和 state 都使用了 readonly 或者是使用了 Mapped types 中的 Readonly。
而 Component 的 Interface 继承了 ComponentLifecycle 的 Interface，源码如下：

```typescript
interface <P, S, SS = any> extends NewLifecycle<P, S, SS>, DeprecatedLifecycle<P, S> {
	componentDidMount?(): void;
	shouldComponentUpdate?(nextProps: Readonly<P>, nextState: Readonly<S>, nextContext: any): boolean;
	componentWillUnmount?(): void;
	componentDidCatch?(error: Error, errorInfo: ErrorInfo): void;
}
```

ComponentLifecycle 的 Interface 定义了一些常用的生命周期函数的类型如 componentDidMount、shouldComponentUpdate、componentWillUnmount 和 componentDidCatch。ComponentLifecycle 同样继承了 NewLifecycle 和 DeprecatedLifecycle 两个接口，代码分别如下：

```typescript
interface NewLifecycle<P, S, SS> {
  getSnapshotBeforeUpdate?(
    prevProps: Readonly<P>,
    prevState: Readonly<S>
  ): SS | null;
  componentDidUpdate?(
    prevProps: Readonly<P>,
    prevState: Readonly<S>,
    snapshot?: SS
  ): void;
}

interface DeprecatedLifecycle<P, S> {
  componentWillMount?(): void;
  UNSAFE_componentWillMount?(): void;
  componentWillReceiveProps?(nextProps: Readonly<P>, nextContext: any): void;
  UNSAFE_componentWillReceiveProps?(
    nextProps: Readonly<P>,
    nextContext: any
  ): void;
  componentWillUpdate?(
    nextProps: Readonly<P>,
    nextState: Readonly<S>,
    nextContext: any
  ): void;
  UNSAFE_componentWillUpdate?(
    nextProps: Readonly<P>,
    nextState: Readonly<S>,
    nextContext: any
  ): void;
}
```

NewLifecycle 是 React 16.3 新增的生命周期钩子，而 DeprecatedLifecycle 则是 React 16.3 之前常用的生命周期钩子，即将在未来版本被废弃。

所以通过以上分析，我们可以知道，React 的 Class 继承于 `class Component<P, S>`，而 `class Component<P, S>` 是泛型 `interface Component<P = {}, S = {}, SS = any>` 的实现。其中 P 是 props 的类型，S 是 state 的类型，它们都是 object 类型。

**因此，TypeScript 的有状态组件大致可以按以下步骤来编写或者阅读：**

- 设置 initialState，并根据 initialState 来设定 State 类型或者 State 的 interface：

```typescript
const initialState = { clicksCount: 0 };
type State = Readonly<typeof initialState>;
```

- 使用 type 或者 interface 设定 props 类型
- `class Example extends Component<P, S>` 这里需要指定 props 和 state 的类型 P 和 S
- `readonly state: State = initialState;` 这里建议将 state 设置为 readonly 类型，防止 state 被直接修改
- 生命周期方法可以使用 public 来修饰（tslint 要求 class 内部方法必须加入 public 或者 private 来修饰）
- 内部方法可以使用 private 来修饰以保护其安全性

## 小 Tips

1. 从上面的 d.ts 文件的分析可以得知， `setState()` 的具体逻辑是可以被提取到组件外部的，这样做的优点是可以将操作 state 的方法剥离出来维护，不需要了解渲染逻辑。
2. 在使用 mock 数据调试组件的时候，我们可以多用 any 类型来节约 React 组件调试时间成本，直至数据结构完全确定后，再编写完整的数据验证流程
3. 如果你想使用一个 package 但是遇到了找不到 @types 相关 package 的报错，可以到 Microsoft 的 [TypeScript Types Search](https://microsoft.github.io/TypeSearch/) 查询需要 `npm install` 什么 package

## Reference

### TypeScript 官方文档

- [英文版](https://www.typescriptlang.org/)
- [中文版](https://www.tslang.cn/)

### Ultimate React Component Patterns with Typescript 2.8

- [英文原版](https://levelup.gitconnected.com/ultimate-react-component-patterns-with-typescript-2-8-82990c516935)
- [蚂蚁金服技术团队翻译版](https://juejin.im/post/5b07caf16fb9a07aa83f2977)
