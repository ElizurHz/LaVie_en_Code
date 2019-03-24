---
title: 在 React 中使用防抖动
date: 2019-03-24
categories: ["React", "Javascript"]
---

# 在 React 中使用防抖动

## 什么是防抖动

防抖动其实就是保证在事件发生后的固定时间内，如果再触发该事件，则重新计算延时，直到这段延时内没有再次触发事件，则执行用户自定的函数。

更进一步说，防抖动分为立即执行和非立即执行，立即执行的运作方式有所不同，它是会先执行用户自定的函数，若在一段延时内未再触发该事件，则触发该事件才能再次执行函数；在该延时内触发的事件不执行函数，且重新计算延时。

### 基础版

关于防抖动的详情和具体实现，网上很多文章应该都介绍过了，这里不赘述，推荐一篇看过的应该是最好的文章：[函数防抖和节流 - 掘金](https://juejin.im/post/5b651dc15188251aa30c8669) 。不过这种代码实现比较“传统”，使用的是 ES5 和闭包。

```JavaScript
function debounce(func, wait) {
  var timeout;

  return function () {
    var context = this;
    var args = arguments;

    if (timeout) clearTimeout(timeout);

    timeout = setTimeout(function(){
      func.apply(context, args)
    }, wait);
  }
}
```

如果是按照上面的 ES5 + 闭包的形式编写 debounce 函数，那么使用方法如下：

```JavaScript
function print(value) {
  console.log(value)
}

debounce(print, 100)('123')
```

这也是 [lodash](https://github.com/lodash/lodash/blob/4ea8c2ec249be046a0f4ae32539d652194caf74f/debounce.js) 和 [underscore](https://github.com/jashkenas/underscore/blob/d5fe0fd4060f13b40608cb9d92eda6d857e8752c/underscore.js#L887) 中 debounce 的实现与使用方式。

### 进阶版

当然，我们也可以使用 Promise 来实现防抖动，参考：[理解函数防抖 Debounce - 掘金](https://juejin.im/post/5bdb155b5188257f62136ce8) 。

```JavaScript
function debounce(method, wait, immediate) {
  let timeout
  let result
  let debounced = function(...args) {
    return new Promise(resolve => {
      let context = this
      if (timeout) {
        clearTimeout(timeout)
      }
      if (immediate) {
        let callNow = !timeout
        timeout = setTimeout(() => {
          timeout = null
        }, wait)
        if (callNow) {
          result = method.apply(context, args)
          resolve(result)
        }
      } else {
        timeout = setTimeout(() => {
          result = method.apply(context, args)
          resolve(result)
        }, wait)
      }
    })
  }

  debounced.cancel = function() {
    clearTimeout(timeout)
    timeout = null
  }

  return debounced
}
```

使用方法：

```JavaScript
function print(value) {
  return value
}

let debouncedFn = debounce(print, 1000, false)

debouncedFn('wtf').then(val => {
  console.log(val)
})
```

## React 中的防抖动

在 React 中，我们经常会需要在虚拟 DOM 上添加事件，比如最常用的 button 的 onClick 以及 input 的 onChange。

```JavaScript
class Test extends React.Component {
  onInputChange = (e) => {
    console.log(e.target.value)
  }

  render() {
    return (
      <div>
        <input onChange={this.onInputChange} />
      </div>
    )
  }
}
```

假如有如上所示的一个组件，如果我们想实现 debounce 触发 onChange 的话，我们需要这么写（假设以上面的 Promise 版本作为组件中的 debounce 函数）：

```JavaScript
class Test extends React.Component {
  constructor(props) {
    super(props)
    this.debounceInputChange = debounce(this.onInputChange, 666, false)
  }

  onInputChange = (e) => {
    console.log(e.target.value)
  }

  render() {
    return (
      <div>
        <input onChange={this.debounceInputChange} />
      </div>
    )
  }
}
```

或者：

```JavaScript
class Test extends React.Component {
  onInputChange = debounce((e) => {
    console.log(e.target.value)
  }, 666, false)

  render() {
    return (
      <div>
        <input onChange={this.onInputChange} />
      </div>
    )
  }
}
```

不可以写成这样：

```JavaScript
class Test extends React.Component {
  onInputChange = (e) => {
    debounce(this.requestAPI(), 666, false)
  }

  requestAPI = () => {
    // some API request
  }

  render() {
    return (
      <div>
        <input onChange={this.onInputChange} />
      </div>
    )
  }
}
```

如果需要在输入的时候防抖动请求服务端数据，这样写的结果就是仍然每次输入都会触发 input 的 onChange 事件，并且每次都会向服务端发出请求，只是每个请求会在 666 ms 的延时之后依次执行。

## 事件对象

React 中的事件都是[合成事件（SyntheticEvent）](https://reactjs.org/docs/events.html)，而合成事件有个特性是 [Event Pooling](https://reactjs.org/docs/events.html#event-pooling)。

简单说就是在使用 debounce 函数之后 event 对象的所有属性都会变成 null。这时只要在代码前面加入 `e.persist()` 即可移除合成事件，这样在 debounce 包装之后，函数仍然可以获取到 event 对象。

## PS

在我使用百度地图的 localSearch 时，我将 setState 放在了它的 JSONP Callback 中，这样 debounce 也无法解决返回顺序不一致的问题。同时如果进行快速输入，触发和数据返回的时间可能也不太一样。所以当遇到有服务端请求的并且需要使用 debounce 来对触发事件进行防抖动时，最好加入一个判断，当服务端返回值和请求值能对应上时才 setState。
