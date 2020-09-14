---
title: 面向前端的 Node.js（一） - 前端需要了解的 Node.js 基础与实践
date: 2020-08-12
categories: ["后端", "前端"]
tags: ["Node.js"]
draft: true
---

## 系列前言

在学习极客时间上的 Node.js 课程之后我决定自己整理一下学到的知识。

在 Node.js 推出并越来越流行的今天，全栈工程师这种名词也开始流行起来，自然会有很多同学为了拓展自己的知识面和更好地找工作去学 Node.js。其实大多数这样做的人并不会期望着自己学完了之后变成一个“后端工程师”的，毕竟很多后端的知识和实践经验并不是前端开发者们一朝一夕就能拥有的。那么前端同学怎样入门 Node.js 最好呢？那就要从 Node.js 的运用来说起。因为大型项目的后台大多数是由 Java 等语言编写的，这有历史遗留问题，也有技术选型问题，这里不赘述了，总体来说用传统后端的项目还是远超 Node.js 的。而 Node.js 对于前端来说最常用的场景其实就是接驳前后端的一个中间层，也就是 Backend for Frontend。通过它我们能获得什么呢？一是可以自由拼装后端数据，二是可以做 SSR 来优化首屏加载速度和 SEO。

另外，这篇文章放在“前端”的栏目里主要是因为它是面向前端的，当然它对后端人员也有帮助，能够使后端工程师更好地理解这种构架，这样能使后端工程师与前端工程师的协作更加顺利。

## 为何要多做一层？

首先我们要清楚一点，前后端通信一般是使用 HTTP 协议，而后端服务间的数据传输往往会使用 RPC。而 RPC 往往会使用到二进制协议而不是 HTTP 中常用的文本协议。关于 RPC vs HTTP 和二进制协议 vs 文本协议，可以自行查询资料。总体来说 RPC + 二进制协议的效率会更高，因为我们的 BFF 层是可以和众多后端服务一起部署的，完全可以把它视作众多后端服务之一而融入进后端服务的 DevOps 中，可以算作微服务中的一个服务。至于这个构架怎样做最好，这大概率是构架师和后端的事情，前端同学基本不会参与这方面的决策。而前端同学最需要知道的就是，如果给了我们一个这样的任务，我们该怎么做。

另外关于这个 BFF 层的必要性，我个人从构架的角度认为，如果要做它，必定要有它的优越性才行，包括但不限于性能、DevOps、开发成本。例如后端如果已经提供了 HTTP 接口，那么除非是做 SSR 优化，BFF 层并没有存在的必要。至于前端常讲的自由组装数据，我觉得在非极限的情况下都是没有必要的。如果确定有某种业务场景无法满足性能需求，那么应当前后端与构架师一同商讨改进方案，不管是 BFF、传输协议还是接口数据格式等等，而不是加一个 Node.js 服务用服务器的运算能力去处理这么简单的事情。

## Node.js 基础

关于 Node.js 的基础，其实对于前端来说并不需要讲太多，我们在浏览器端应用开发中所用到的 JavaScript 语法、特性等等都是一样的，甚至编译器都是用的和 Chrome 一样的 V8，能在 Chrome 中 debug。那么对于前端来说，学习 Node.js 主要掌握差异化的部分和 API 即可。

首先我们要知道 Node.js 是什么。官网原话：

> Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.

> As an asynchronous event-driven JavaScript runtime, Node.js is designed to build scalable network applications.

那么这里有两个关键点，一是“基于 V8 的”，二是“异步的、事件驱动的”。

那么我们可以先来看下 Node.js 的构架，借用网上的一张图：

![](/images/bff/nodejs-architecture.jpg)

它包括了 Node.js 应用层、V8 引擎、Node.js Binding 层、以及异步 I/O 相关模块。

而作为对比， JavaScript 在浏览器环境下的构架大概是（以 Chrome 为例）：

![](/images/bff/browser-js-runtime.png)

对比二者，我们可以发现 V8 是二者都有的部分（事实上不同浏览器使用的 JavaScript 引擎不大一样，如果是 Chrome 就和 Node.js 一样）。Event Loop 也是二者都有，但是它们的实现方式有所区别：Node.js 靠的是 libuv，而在浏览器端它是由 [HTML 规范](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) 所定义的，具体交由各浏览器的开发者自行实现。另外浏览器端独有的是 Web API，这点我们应该都很熟悉。而 Node.js 独有的则是 Node.js Binding，它主要负责的是 Node.js 与操作系统的交互，这是后端语言必备的能力。举例来说，有可以读取系统信息的 API，可以用于性能监测；有读写文件的 API，这个是非常常用的。

那么这样分析我们大致可以了解到，在 Node.js 中 JavaScript 的执行机制其实和在 Chrome 浏览器中比较类似，然后在 Node.js 中我们不需要也不能使用 Web API，比如 window 全局对象、DOM API 等。那么接下来稍微讲讲 Node.js 中有区别的、独有的部分

### Event Loop

![](/images/bff/nodejs-event-loop.png)

这是 Node.js 大致的 event loop 流程图，来源于 Node.js 官网。这个图里面其实只描绘了 Macro Tasks 也就是宏任务的执行顺序，并没有我们前端常提到的微任务。实际上，在每次执行完一种宏任务之后，都会先去清空微任务的队列，例如执行完 timer （也就是 setTimeout 的 callback) 之后，会把所有的 Promise 执行一遍，再进入 pending callbacks 的阶段。而 Node.js 中有个特殊的任务，是 `process.nextTick()`，它的执行顺序优先于所有的微任务，会在每一次执行微任务前执行当前在调用栈中的 `process.nextTick()` 的 callback。简单描述就是：_先执行一种宏任务，每种宏任务中的每个宏任务执行完都会清空一次微任务队列，执行完一种宏任务的队列之后开始执行下一种宏任务。_ （注：Node.js v11 和 v10 的执行顺序是不一样的，你在网上查到的很多都是 v10 或以前的内容，两个是完全不一样的，可以参考[这篇文章](https://blog.fundebug.com/2019/04/02/nodejs-event-loop-has-changed/)。）

而相比之下，浏览器端其实也是类似的。我们之前常说先执行宏任务再执行微任务，但是严格来说并不是这么简单的，只是因为浏览器端只有 timer 这一个宏任务，所以我们简化了这个说法。事实上更严谨的说法应该是：_先执行一个宏任务(timer)，然后执行所有的微任务，如此循环。_

所以 Node.js 的 event loop 其实是浏览器的放大版，以及多了一个 `process.nextTick()`。

### Node.js Binding

顾名思义，这部分就是 Node.js 调用系统 API 的能力，例如查看 CPU 状态、内存状态等，进行进程的 kill、fork，以及我们最常用的 `fs`，也就是文件的读写。这部分的能力是由 C++ 实现的，想要研究的话可以去看 Node.js 的源码，在这里不赘述。

## 前端需要知道的 RPC

RPC(Remote Procedure Call Protocol)，中文名叫远程过程调用。实际上它就是两个服务之间的一种通信方式。而说到通信方式，主流的 RPC 框架分为基于 HTTP 和基于 TCP 的两种。HTTP 的缺点主要是协议传输效率低，短连接开销大。如果使用基于 TCP 的 RPC，我们可以对它进行灵活的定制，来解决性能和效率上的需求。业界中基于 HTTP 的 RPC 框架有非常著名的 gRPC，它是基于 HTTP/2 的，而基于 TCP 的方案则有很多，例如阿里的 Dubbo 和 HSF。而一个 RPC 调用的流程大概是：客户端寻址，客户端数据序列化并发送，服务端接收数据并反序列化，服务端执行本地代码并将返回数据序列化通过 TCP/HTTP 通道返回给调用方，客户端将数据反序列化并执行相应的业务代码。

这里我们提到了“序列化”和“反序列化”。在 HTTP 请求 RESTful 接口的时候，我们常用的往往是 application/json 的数据格式，这样我们前端只需要拼接出一个 JavaScript 对象，然后使用 fetch API 或者 axios 即可将其发送给后端。而在 RPC 中，为了效率，在数据传输方面，很多框架使用了二进制协议。在客户端向服务端发送数据的时候，HTTP 需要发送一个 JSON，包含众多键值对，而基于二进制协议的 RPC 则可以发送一个二进制流。而二进制数据序列化和反序列化的开销往往会比 JSON 更好，序列化后的数据包体积也往往更小（具体取决于不同框架的不同实现方式）。

## RPC 中的二进制数据

### Buffer

在 Node.js 中，我们通常会使用 Buffer 来实现数据的序列化和反序列化，实现 JavaScript 中的数据类型与二进制数据间的互相转换。当然在前端我们也有用于读写二进制数据的类似实现，ES6 中有 ArrayBuffer、TypedArray 和 DataView 的对象。它们常常应用于 JS 与显卡交互（WebGL）和与声卡交互（Web Audio API）等场景。而在 [Node.js 的文档](https://nodejs.org/dist/latest-v14.x/docs/api/buffer.html#buffer_buffers_and_typedarrays)中有相关说明，Node.js 的 Buffer 继承并且拓展了 JavaScript 的 Uint8Array，一个 Buffer 实例同样也是 Uint8Array 实例和 TypedArray 实例 (Node.js > 4.x)。

#### 创建

```JavaScript
// create
const buf1 = Buffer.alloc(10)
// create with initial values
const buf2 = Buffer.alloc(10, 1)
// convert from other JavaScript data structure
const buf3 = Buffer.from([1, 2, 3])
const buf4 = Buffer.from('tést')
const buf5 = Buffer.from('tést', 'latin1')
```

`Buffer.alloc` 即为创建一个固定长度的 buffer，而 `Buffer.from` 则是创建一个包含所给定数据的 buffer。

#### 读写

```JavaScript
const buf1 = Buffer.alloc(3)
buf.writeInt8(3) // <Buffer 03 00 00>

const buf2 = Buffer.alloc(3)
buf.writeInt8(3, 1) // <Buffer 00 03 00>

const buf3 = Buffer.alloc(3)
buf.writeInt16BE(0x0102, 1) // <Buffer 00 01 02>

const buf4 = Buffer.alloc(3)
buf.writeInt16LE(0x0304, 1) // <Buffer 00 04 03>

const buf4 = Buffer.alloc(3)
buf.writeInt16LE(0x0304, 2) // RangeError [ERR_OUT_OF_RANGE]: The value of "offset" is out of range. It must be >= 0 and <= 1. Received 2
```

读写的方法分为 `writeInt8`, `writeInt16BE`, `writeInt16LE` 等等。它们之间的区别是，它们表示了不同进制，比如 int8 就是八进制数，范围是 -254 到 255，占一个字节；int16 就是十六进制数，占两个字节，而 BE 和 LE 表示写入的顺序，是“从左往右”还是“从右往左”。当然要注意用 `Buffer.alloc` 的时候你是规定了一个 Buffer 的长度的，写入的内容必须在范围内，否则会报错。例如长度为 3 的 buffer 写入 int16 是无法从 offset = 2 的位置开始写入的，也无法写入 int32。当然想要扩展 Buffer 长度也不是没办法，比如 `Buffer.concat`：

```JavaScript
const buf1 = Buffer.alloc(3)
buf1.writeInt16LE(0x0304, 1)
const buf2 = Buffer.alloc(3)
buf2.writeInt16BE(0x0102, 1)
const targetBuf = Buffer.concat([buf1, buf2], 9) // <Buffer 00 04 03 00 01 02 00 00 00>
```

其余的使用方法请查阅 [Node.js 官方文档](https://nodejs.org/dist/latest-v14.x/docs/api/buffer.html#buffer_buffer)。

### Protobuf

如果在做业务的时候，我们每次都需要这样手动写入 Buffer，那样的工作效率显然太低了，因此我们就要依赖一些高效的序列化框架。例如 Protobuf，它是 Google 开发的一个结构化数据的格式，有在不同语言不同框架下的实现。其实这非常类似 GraphQL，但是 GraphQL 本质上仍然是 HTTP 请求而不是 RPC。
