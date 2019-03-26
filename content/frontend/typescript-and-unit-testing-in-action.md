---
title: TypeScript + 单元测试：从零开始的经验之谈
date: 2019-03-26
categories: ["前端"]
tags: ["TypeScript", "单元测试", "React"]
---

## TL;DR

公司和部门内部之前有在推广 TypeScript，但是由于种种原因很难投入使用，但这次我参与了一个从零开始的项目，没有技术栈限制，领导也说希望用 TypeScript 试一试。作为开发团队中唯一认真学过 TypeScript 的人（在部门进行过内部分享，写过内部博客），我们的经验其实都是很缺乏的。

另外，在这个项目中，除了 TypeScript 之外，单元测试也是我们团队首次尝试的。

同时，我的性格和开发风格也比较偏向“探索者”，所以领导也布置给我一些预研、新尝试，所以自然而然便成了一个 leader 的角色。

## 项目构架

我们使用 create-react-app 搭建项目，没有 `eject`。

而对于文件的组织，我们采用了按模块划分的方式，即各个模块集成自己的 store, reducers, actions, 展示组件, 容器组件等，并设置一个入口文件将它们都暴露给外部，外部需要引用的时候直接引用这个文件即可。

主目录结构大致如下：

```javascript
├── docs // 文档
├── e2e // E2E 测试
├── src
│   ├── App.tsx // 主页面
│   ├── index.tsx // 主入口
│   ├── common.d.ts // 全局通用类型
│   ├── components // 通用组件
│   ├── i18n // 国际化
│   ├── @types // 如果某个依赖包没有 TypeScript 支持，则在这里编写 d.ts 文件
│   ├── modules // 业务模块
│   │   ├── moduleA
│   │   ├── moduleB
│   │   ...
│   ├── styles // 全局 scss 样式文件
│   ├── services // 与服务端请求相关的封装
│   ├── constants // 通用常量
│   └── utils // 通用工具函数
```

模块内部结构大致如下：

```javascript
├── index.ts // 主入口，用于导出外部可能引用到的所有该模块的内容
├── [MODULE_NAME].d.ts // 该组件下的通用类型
├── actions // 该模块的 redux action
├── common // 该模块内的通用组件（一般是与模块业务比较相关的）
│   ├── commonComponentA // 组件文件夹，包含组件、样式、单元测试
│   │   ├── index.tsx
│   │   ├── index.test.tsx
│   │   └── index.module.scss
│   ├── componentB
│   ...
├── components // 模块内的展示组件
│   ├── componentA
│   │   ├── __test__ // 单元测试
│   │   │   ├── __snapshots__ // 快照测试，如果有快照测试，Jest 会自动创建它
│   │   │   ├── index.test.js
│   │   │   └── someComponent.test.js
│   │   ├── index.ts // 主入口，用于导出外部可能引用到的所有组件
│   │   ├── componentA.tsx // 展示组件
│   │   ├── componentAContainer.tsx // 容器组件
│   │   ├── someComponent.tsx // 其他组件
│   │   └── index.module.scss
│   ...
├── reducers // 该模块内的 reducer
│   ├── index.ts // 主入口
│   ├── featureA.ts // 各子功能的 reducers
│   ...
├── sagas // redux-saga
│   ├── index.ts // 主入口
│   ...
```

## TypeScript

在使用 TypeScript 时，遇到的问题也是唯一可能的问题就是类型不匹配。刚开始的时候，大家对 React 的各种类型都非常不熟悉，所以不知道设定成什么好。事实上这应该也是所有的开发者入门遇到的最大问题。解决这个问题的方法，一是去稍微读一下 react 的 `index.d.ts`，知道它大概有什么类型；二是参考一下成熟的 TypeScript 项目。Ant Design 可能是最合适的项目参考了。怎么写类型，React 里有什么常用的类型，以及怎么去继承复用类型，这些问题都可以在 Ant Design 上找到答案。

这里总结一些常用的 React 类型（特殊未注明的都需要 import）：

- 使用 children 时用 `React.ReactNode`
- inline CSS 使用 `React.CSSProperties`
- input 的相关事件使用 `React.ChangeEvent` (泛型为 `ChangeEvent<HTMLInputElement>`)
- 鼠标相关的事件使用 `React.MouseEvent` (泛型为 `MouseEvent<HTMLButtonElement>`)
- `render()` 函数的返回值类型是 `JSX.Element`，不需要 import。

另外 `react-router` 和 `react-intl` 分别有各自的类型 `RouteComponentProps` 和 `InjectedIntlProps`，需要 import。

---

使用 TypeScript 对于刚入门的新手来说绝对是很浪费时间的，因为你写同样一个组件，相比 JavaScript 来说可能会花上两倍的时间。但是我们团队差不多在开发的第二周开始就能熟练开发了，所以 TypeScript 需要的仅仅是一些前期的学习成本，到后面它并不会成为风险或者负担。

但是花这些时间去学习 TypeScript，带来的好处是不言而喻的。首先我们开发下来的体验就是：极少 Bug。Bug 大部分都是在开发时被解决的，因为有了 TypeScript 的约束，我们一旦出现类型错误等等，在开发时就可以被及时发现并修复，所以部署上去之后能发现的 Bug 就少了很多了。当然对于减少 Bug 贡献更多的其实是单元测试，这个后面会说。

## 单元测试

单元测试框架我们使用的是 jest + enzyme。React 的单元测试相关的文章很多，但是都是关于“框架怎么用”，很少有人总结“一个项目里到底该怎么组织测试”。最近看到了一篇 [React 单元测试策略及落地 #一篇就够系列](https://juejin.im/post/5bd640abf265da0ad63c125e) 写得非常好，我们的单元测试也基本是按照这个来组织的。

参考上面的文章，我们的单元测试的组织方式是：

- connect 组件不测
- 展示组件需要测试
  - UI 用快照测试(Jest)
  - 分支渲染必测
  - 事件必测
- 工具函数需要测试
- reducers 需要测试
- sagas 需要测试

我们的单元测试全部用 JavaScript 编写，这点也参考了 Ant Design。我认为单元测试没必要去真正完整地把那些 props mock 出来，因为我们是在验证代码逻辑的正确性，应该更多关注与代码本身，而不是类型有没有正确。如果在使用时传入了错误的类型，那么编译是不会通过的，这是 TypeScript 的编译器帮我们做的，再写关于类型错误的测试用例其实是画蛇添足。另外，在项目变得复杂之后，往往会出现一些很难 mock 的类型，强行去 mock 它们只会浪费我们写测试的时间。

### 组件

我们的组件分为业务组件和通用组件。通用组件的测试就比较简单了，针对各个 props 进行测试即可，目的就是覆盖所有的分支逻辑和事件，同时用快照测试测 UI。而业务组件经常涉及到一些很复杂的 props，更有可能出现各种高阶组件，所以我们团队约定，在业务组件中不进行任何的高阶组件装饰。

在测试组件的时候，必然会使用到 enzyme。在 enzyme 的使用上，我们确实碰到了很多问题，最大的问题还是在没经验的时候很难把目的转化成代码。例如我需要找一个“props 中有 `type="primary"` 的 Button 组件”，或者“触发 Button 子组件的 onClick 方法”。当然这也和官方文档写的不够（有的可能找不到）有关，并且很多文章也不会涉及到这种深入的内容。

### reducers

reducers 测试比较简单，首先需要模拟出我们需要的 store，然后触发不同的 actions 去调用 reducers 的不同处理逻辑。具体测试方法官方文档都有说明，这里不再赘述。

### sagas

sagas 的测试在官方文档也有专门页面说明。首先 saga 一定是异步的，而我们单元测试并不关注异步的结果，只需要测试相应的 saga 是否被正确调用。redux-saga 中有 `runSaga()` 的 API 可以帮助我们完成单元测试。

### 测试规范

在项目中，单元测试的规范化也是非常重要的。首先我们需要保证用例能实现你的测试目的，其次还需要有可读性。所以我对于单元测试质量的理解大概如下：

- 测试描述需写精确，以便他人阅读，也方便说明该组、某个测试在测什么
  - `describe` 的第一个参数表示这一组测试在做什么
  - `it` 的第一个参数表示这个测试在做什么
- 测试的 expect 需要符合测试目的(即 `it` 的描述)
  - 例如给某个 input 赋初始 value，想判断它是否显示正确的 value，此时不能简单判断某个 DOM 存在
- 如果需要测试是否渲染成功，请使用 Jest 的快照测试，不要去判断某个 DOM 或者某个子组件是否存在
  - 我们无法从某个 DOM 节点存在推导出组件是正常渲染的

---

单元测试其实也是很耗时间的，特别是在不熟练的时候，可能在写测试时花的时间比开发花的时间还多。但是单元测试能够直接带来的好处就是极大减少了 Bug 数量。QA 往往是以业务场景来进行黑盒测试，他们不了解我们代码的写法，很多情况就是 QA 测试能通，但是实际上蕴藏了很多能产生 bug 的地方。在编写单元测试的过程中，我们也可以和 QA 一样去构思不同的业务场景，编写不同的测试用例，在这个过程中我们也会发现代码可能对于某些场景、逻辑的处理有缺失，可以依此对代码进行修正和补充。

## 总结

TypeScript 和单元测试刚入门的时候会觉得这些东西没用而且浪费时间，但是真正会用它们了之后其实是“真香”，很推荐开发者们都投入这个时间成本，因为它们后期带来的回报是完全可以弥补这个成本，甚至还有赚的。
