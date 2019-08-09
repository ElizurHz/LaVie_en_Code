---
title: 用 React 16.3 的 Context API 实现类似 ant design 的 Material Design 版 Radio Group 组件
date: 2019-07-22
categories: ["前端"]
tags: ["React"]
---

# Radio 组件是什么样的

以官方文档的例子：

```jsx
import React from "react";
import { Radio } from "antd";

class App extends React.Component {
  state = {
    value: 1
  };

  onChange = e => {
    console.log("radio checked", e.target.value);
    this.setState({
      value: e.target.value
    });
  };

  render() {
    return (
      <Radio.Group onChange={this.onChange} value={this.state.value}>
        <Radio value={1}>A</Radio>
        <Radio value={2}>B</Radio>
        <Radio value={3}>C</Radio>
        <Radio value={4}>D</Radio>
      </Radio.Group>
    );
  }
}

ReactDOM.render(<App />, mountNode);
```

除了这种方式，还有使用 options 这个 prop 传递配置文件进行渲染的：

```jsx
import React from "react";
import { Radio } from "antd";

const plainOptions = ["Apple", "Pear", "Orange"];
const options = [
  { label: "Apple", value: "Apple" },
  { label: "Pear", value: "Pear" },
  { label: "Orange", value: "Orange" }
];
const optionsWithDisabled = [
  { label: "Apple", value: "Apple" },
  { label: "Pear", value: "Pear" },
  { label: "Orange", value: "Orange", disabled: false }
];

class App extends React.Component {
  state = {
    value1: "Apple",
    value2: "Apple",
    value3: "Apple"
  };

  onChange1 = e => {
    console.log("radio1 checked", e.target.value);
    this.setState({
      value1: e.target.value
    });
  };

  onChange2 = e => {
    console.log("radio2 checked", e.target.value);
    this.setState({
      value2: e.target.value
    });
  };

  onChange3 = e => {
    console.log("radio3 checked", e.target.value);
    this.setState({
      value3: e.target.value
    });
  };

  render() {
    return (
      <div>
        <Radio.Group
          options={plainOptions}
          onChange={this.onChange1}
          value={this.state.value1}
        />
        <Radio.Group
          options={options}
          onChange={this.onChange2}
          value={this.state.value2}
        />
        <Radio.Group
          options={optionsWithDisabled}
          onChange={this.onChange3}
          value={this.state.value3}
        />
      </div>
    );
  }
}

ReactDOM.render(<App />, mountNode);
```

# ant design 怎样实现的

从 ant design 的 [github](https://github.com/ant-design/ant-design/tree/master/components/radio) 上，可以看到（这里我们只关心 radio group）这个模块里面有 radio.tsx 和 group.tsx 组件。

## index.tsx

首先在 index.tsx 中，实现的是模块的 export，这里通过 Radio 的 static property 把 Group 组件引入，这样就可以实现 `<Radio.Group>` 的写法。同时由于 TypeScript 的类型检查，我们需要再 Radio 的类里面写上 static property。

```TypeScript
/** index.tsx */
import Radio from './radio';
import Group from './group';

Radio.Group = Group;
export { Group };
export default Radio;

/** radio.tsx */
static Group: typeof RadioGroup;
```

## group.tsx 与 radio.tsx

首先这里使用了 [config-provider](https://github.com/ant-design/ant-design/blob/master/components/config-provider/index.tsx)，这个组件的作用是使用 `@ant-design/create-react-context` 注入了一些配置在 context 中。在这里我们不需要关注它的实现方式，只需要知道它使用了 React 的 context。

接着我们看 group.tsx 中 [render](https://github.com/ant-design/ant-design/blob/28323b785cb8d42a4af7d2ab551c2e3141bd0545/components/radio/group.tsx#L98) 的部分。

使用 options 的部分就不多说了，根据数组渲染 Radio 组件即可。关键是另一种渲染方式，我们知道我们需要在 Radio.Group 的 props 传入这个 radio group 的 value 和 onChange 事件，而 `<Radio.Group>` 在这里只是一个包裹的组件，在 Radio.Group 中我们需要将 Radio 的 onChange 和 props 的 onChange 关联起来。在使用 options 的方式中，我们可以直接把 onChange 通过 props 传给 Radio，但是 Radio 作为 Radio.Group 的 children 时我们却不能这样传。

ant design 的做法是在 group.tsx 中用了 [getChildContext](https://github.com/ant-design/ant-design/blob/28323b785cb8d42a4af7d2ab551c2e3141bd0545/components/radio/group.tsx#L68) 将父组件的 onChange 设置到 context 中：

```TypeScript
getChildContext() {
  return {
    radioGroup: {
      onChange: this.onRadioChange,
      value: this.state.value,
      disabled: this.props.disabled,
      name: this.props.name,
    },
  };
}
```

同时在 radio.tsx 中，在 Radio 的 [onChange](https://github.com/ant-design/ant-design/blob/28323b785cb8d42a4af7d2ab551c2e3141bd0545/components/radio/radio.tsx#L40) 中调用 context 的 onChange：

```TypeScript
 onChange = (e: RadioChangeEvent) => {
  if (this.props.onChange) {
    this.props.onChange(e);
  }

  if (this.context.radioGroup && this.context.radioGroup.onChange) {
    this.context.radioGroup.onChange(e);
  }
};
```

`getChildContext` 是 legacy 的 Context API 了，React 官方也推荐使用最新的 [Context API](https://reactjs.org/docs/context.html)。

# 我的实现方式

## TL;DR

首先为什么会有这种需求呢，是因为正在做的项目的设计风格是 antd + Material Design 的结合体，我们使用了 antd，但是有很多 Material Design 的动画效果无法简单地实现，所以需要一部分 Material Design 的组件。而 material-ui 的学习成本对我们团队来说有点高，恰好 Material Design 官方有提供 web 组件与封装好的 [React 组件](https://github.com/material-components/material-components-web-react)，引入单个组件以适应某些控件的设计需求比较不容易增加包大小。同时，这些 React 组件其实并不易用，做一层封装之后会让它能够和 ant design 的组件具有相同的用法，同样也更适合为业务组件定制样式、统一管理样式等。

## 具体实现

```TypeScript
// context.ts
import React from 'react'

export const RadioGroupContext = React.createContext({
  groupValue: '',
  onChange: (value: string) => {}
})
```

这里使用了新的 Context API 创建了 context。

```tsx
// radio.tsx
import React from "react";
import Radio, { NativeRadioControl } from "@material/react-radio";
import { RadioGroupContext } from "./context";
import "@material/react-radio/dist/radio.css";

export default class MyRadio extends React.Component<RadioProps> {
  public static Group: typeof RadioGroup;

  public render(): JSX.Element {
    const { disabled, nativeProps, value, children } = this.props;
    return (
      <RadioGroupContext.Consumer>
        {({
          groupValue,
          onChange
        }: {
          groupValue: string;
          onChange: (value: string) => void;
        }) => (
          <div>
            <Radio>
              <NativeRadioControl
                {...nativeProps}
                checked={value === groupValue}
                value={value}
                disabled={disabled}
                onChange={(e: React.ChangeEvent<HTMLInputElement>) => {
                  onChange(e.target.value);
                }}
              />
            </Radio>
            {children}
          </div>
        )}
      </RadioGroupContext.Consumer>
    );
  }
}
```

```tsx
// group.tsx
import React from "react";
import { RadioGroupContext } from "./context";

xport default class MyRadioGroup extends React.Component<PortalRadioGroupProps, PortalRadioGroupState> {
  public constructor(props: PortalRadioGroupProps) {
    super(props)
    this.state = {
      value: props.value || ''
    }
  }

  private onChange = (value: string) => {
    const { onChange } = this.props
    this.setState({
      value
    })
    onChange && onChange(value)
  }

  public render(): JSX.Element {
    const { children } = this.props
    return (
      <RadioGroupContext.Provider
        value={{
          groupValue: this.state.value,
          onChange: this.onChange
        }}
      >
        {children}
      </RadioGroupContext.Provider>
    )
  }
}
```

新 Context API 的使用方式很简单，在父组件用 Provider 传入 value，这样所有的子组件就可以接收到 value 了。而在使用的地方需要用到 Consumer，要注意的是这个组件是以 render callback 的方式实现的，所以我们也需要写成 render callback 的形式。callback 的参数是上述的 value，而返回值则是我们需要的组件，在这里我们就可以使用从 Provider 传入的 context 了。

而它使用的方式也很简单，和 antd 基本相同。

```tsx
<Radio.Group onChange={onChange} value={value}>
  <Radio key="1" value="1">
    111
  </Radio>
  <Radio key="2" value="2">
    222
  </Radio>
</Radio.Group>
```

这个组件唯一的问题是无法在 antd 的表单中使用。antd Form 其实实现了双向绑定，劫持了 value（或者自定义属性）以及 onChange，但是这个组件点击的时候无法触发 onChange，可能与 Material Design 组件的[内部实现](https://github.com/material-components/material-components-web-react/blob/master/packages/checkbox/index.tsx)有关。
