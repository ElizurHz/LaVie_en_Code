---
title: Apollo GraphQL 在 React 中的应用
date: 2020-05-22
categories: ["前端"]
tags: ["GraphQL", "React"]
---

这篇文章大概介绍下 GraphQL 的使用，Apollo 的使用，以及我在项目中遇到的一些场景和情况。

Apollo 是生产级的 GraphQL 框架（官方简介）。那么我使用 Apollo 的契机是，我们项目的后端（美国团队）已经长期在使用 GraphQL。在这个项目初期，后端是有提供 SDK 给部分接口的，但是后来由于业务的发展，我们还是全部接入 GraphQL，因为他们的很多服务都是提供 GraphQL 的 endpoint 的，封装成 SDK 相对困难。

## GraphQL

注：本节示例代码大多来自于 [GraphQL 官网](https://graphql.org/)。

提到 GraphQL，不明白的人可能会联想到 SQL。这确实没错，GraphQL 本身就是一种 query language。在 Apollo 中，我们需要借助 `gql` 这个包来写 queries 和 mutations。

```JavaScript
import gql from 'graphql-tag'

const SAMPLE_QUERY = gql`
  {
    hero {
      name
      height
    }
  }
`
```

上面的是不需要传入任何参数的 query。而我们知道，大部分接口都需要参数，所以使用 `gql` 包我们需要这么写：

```JavaScript
const SAMPLE_QUERY_WITH_VARIABLES = gql`
  query HeroNameAndFriends($episode: Episode) {
    hero(episode: $episode) {
      name
      friends {
        name
      }
    }
  }
`
```

`HeroNameAndFriends` 在 GraphQL 里叫做 `operation name`。

而在 GraphQL 中我们还会有一种常用的模式就是 Fragments。以官网的代码为例，使用 `gql` 包我们需要这么写：

```JavaScript
const sampleFragments = {
  fragments: {
    entry: gql`
      fragment comparisonFields on Character {
        name
        appearsIn
        friends {
          name
        }
      }
    `
  }
}
```

Fragments 可以用来定义重复使用的一种数据结构，这样我们可以不需要重复定义这些 schema。假设有个适用场景是用户管理系统，用户有一系列共有的属性，我们就可以将其定义为 Fragments。在进行各种 queries 和 mutations 的时候，例如请求用户列表、修改用户信息等等，我们就可以直接引用这个 Fragments，而不用重复写这段 schema，例如

```JavaScript
const SAMPLE_QUERY_WITH_FRAGMENTS = gql`
  query findHero($episode: Episode) {
    leftComparison: hero(episode: $episode) {
      ...comparisonFields
    }
    rightComparison: hero(episode: $episode) {
      ...comparisonFields
    }
  }
  ${sampleFragments.fragments.entry}
`
```

mutations 的写法和 query 非常相似，只需要把 `query` 关键字换成 `mutation` 即可。
这里非常容易遇到变量名字写错等等问题，写的时候还需要仔细参考模板。
另外关于 GraphQL 中的类型，例如上面示例代码中的 `Episode` 和 `Character`，这个一般是在后端定义的。至于如何获取类型名，这个后端是会在 endpoint 提供一个 playground 来显示有哪些 queries 哪些 mutations 可用的，包括需要传的参数都会写出来，也就相当于一个 API 文档。我们也可以在 playground 上测试我们的 queries 和 mutations 有没有写对。另外关于参数和类型，带感叹号的就是必传参数，其他并没有太多需要注意的地方。

## Apollo

### 使用

注：本节示例代码主要来自于 [Apollo 官网](https://www.apollographql.com/)，以 3.0beta 版本的 API 为准。

上面简单介绍了下怎样写 queries 和 mutations，但是我们需要一个工具来发起 GraphQL 请求。在 React 中，除了 Apollo，还有 Facebook 做的 Relay。我没用过 Relay，但是看评价是说它比较“重”。本文就着重介绍 Apollo。

Apollo 是个工具，它有各端的实现，而在 React 端它可以以组件的形式存在于 App 中。要使用 Apollo，我们需要先做两件事：写创建 `ApolloClient` 的代码，和将 `ApolloProvider` 加入到组件树中。

```JavaScript
/** apolloClient.js */
import { ApolloClient } from 'apollo-client';
import { InMemoryCache } from 'apollo-cache-inmemory';
import { HttpLink } from 'apollo-link-http';

// Instantiate required constructor fields
const cache = new InMemoryCache();
const link = new HttpLink({
  uri: 'http://yourbackendendpoint.com/graphql',
});

const client = new ApolloClient({
  // Provide required constructor fields
  cache: cache,
  link: link,

  // Provide some optional constructor fields
  name: 'react-web-client',
  version: '1.3',
  queryDeduplication: false,
  defaultOptions: {
    watchQuery: {
      fetchPolicy: 'cache-and-network',
    },
  },
});

export default client
```

按照上面的方式创建一个 `ApolloClient`，然后我们可以在组件中引用它，因为 `ApolloProvider` 需要它作为 prop 传入。

```JSX
import React from 'react';
import { render } from 'react-dom';
import client from './client' // sample

import { ApolloProvider } from '@apollo/react-hooks';

const App = () => (
  <ApolloProvider client={client}>
    <div>
      <h2>My first Apollo app 🚀</h2>
    </div>
  </ApolloProvider>
);

render(<App />, document.getElementById('root'));
```

题外话：需要注意一下 `ApolloProvider` 注入的位置，同时创建 `ApolloClient` 的时候需要导出一个创建 client 的函数（如果业务定制较为复杂的话可能需要这么做，例如绑定通用的错误处理和国际化）时，需要防止 `ApolloProvider` re-render 导致的 `ApolloClient` 被重复创建，因为这样相当于会清空 Apollo 的缓存，导致额外不必要的网络请求。缓存相关的内容会在下面提到。

而如果我们要发起请求，我们需要使用 Apollo 提供的 API。Apollo 提供了 `<Query />` 和 `<Mutation />` 的组件，它们是以 render callback 的模式来应用的。而我在项目中主要运用了 hooks。Apollo 的 hooks 主要常用的有 3 个，分别是 `useQuery`, `useLazyQuery` 和 `useMutation`。`useQuery` 和 `useLazyQuery` 的区别是，`useQuery` 在组件 render/re-render 的时候会自动执行（发起请求），而 `useLazyQuery` 则允许用户在特定的时候调用一个函数来发起请求。

```JSX
/** useQuery */
import { gql, useQuery } from '@apollo/client'; // 注： 3.0beta 将 gql 整合至 @apollo/client 包中，和上文的从 'graphql-tag' 包中引用的方式并不冲突

const GET_GREETING = gql`
  query GetGreeting($language: String!) {
    greeting(language: $language) {
      message
    }
  }
`;

function Hello() {
  const { loading, error, data } = useQuery(GET_GREETING, {
    variables: { language: 'english' },
  });
  if (loading) return <p>Loading ...</p>;
  return <h1>Hello {data.greeting.message}!</h1>;
}
```

```JSX
/** useLazyQuery */
import { gql, useLazyQuery } from "@apollo/client";

const GET_GREETING = gql`
  query GetGreeting($language: String!) {
    greeting(language: $language) {
      message
    }
  }
`;

function Hello() {
  const [loadGreeting, { called, loading, data }] = useLazyQuery(
    GET_GREETING,
    { variables: { language: "english" } }
  );
  if (called && loading) return <p>Loading ...</p>
  if (!called) {
    return <button onClick={() => loadGreeting()}>Load greeting</button>
  }
  return <h1>Hello {data.greeting.message}!</h1>;
}
```

```JSX
/** useMutation */
import { gql, useMutation } from '@apollo/client';

const ADD_TODO = gql`
  mutation AddTodo($type: String!) {
    addTodo(type: $type) {
      id
      type
    }
  }
`;

function AddTodo() {
  let input;
  const [addTodo, { data }] = useMutation(ADD_TODO);

  return (
    <div>
      <form
        onSubmit={e => {
          e.preventDefault();
          addTodo({ variables: { type: input.value } });
          input.value = '';
        }}
      >
        <input
          ref={node => {
            input = node;
          }}
        />
        <button type="submit">Add Todo</button>
      </form>
    </div>
  );
}
```

上面分别展示了 `useQuery`, `useLazyQuery` 和 `useMutation` 的用法。如果了解 hooks 的话，参考 Apollo 的 API 文档应该上手难度不大。那么在上一节提到的请求参数，我们就可以在这边以 `variables` 传入：

```JavaScript
const [loadGreeting, { called, loading, data }] = useLazyQuery(
  GET_GREETING,
  { variables: { language: "english" } } // 参数
)
```

而在 Apollo 中，`options` 是非常重要的，它可以定制 queries 和 mutations 的行为。举例来说，`variables` 其实就是 `options` 的一部分。在 queries 中有几个比较重要的参数：

- notifyOnNetworkStatusChange: `networkStatus` 变化的时候是否需要 re-render 组件
- fetchPolicy: 这关系到 Apollo 会不会缓存数据。设置为 `cache-first` 的话，针对同样的请求参数，Apollo 会直接读取 client 中的缓存而不会重新请求；设置为 `network-only` 的话，则会每次都请求新的数据
- pollInterval: 轮询间隔时间。虽然我们常规的业务不需要轮询，但是这个参数其实是可以用作刷新策略的
- onCompleted: 请求完成后的回调。我一般比较少用这个，因为之前有发现它会被重复调用的 bug。监听数据状态其实可以简单地用 `useEffect`

那么这些参数如何使用呢？假设我们有个场景，在一个用户列表打开一个弹窗，修改用户信息后需要重新获取用户列表：

```JSX
import React, { useState, useEffect } from 'react'

const UserModule = () => {
  const [pollInterval, setPollInterval] = useState(0)

  const [updateUserInfo, { data: mutationData }] = useMutation(UPDATE_USER_INFO); // updateUserInfo 会在某处被调用

  const [called, loading, data] = useQuery(
    GET_USERS,
    {
      variables: {}, // 省略
      pollInterval
    }
  )

  useEffect(() => {
    // mutation 成功
    if (mutationData.res === 'success') {
      setPollInterval(500)
    }
  }, [mutationData])

  useEffect(() => {
    // 重置 pollInterval 防止重复请求
    if (data && pollInterval !== 0) {
      setPollInterval(0)
    }
  }, [data])

  // 数据处理和渲染就不实现了
  return <></>
}
```

上面的代码逻辑是：发起 `UPDATE_USER_INFO` 的 mutation，请求成功后将 `pollInterval` 设置成 500ms，这样 `GET_USERS` 的 query 就会在 500ms 后重新请求。这个重新请求是无视缓存的，就算你设置缓存为 `cache-first` 也会重新请求。而 `fetchPolicy` 主要影响的是 `variables` 变化导致的行为，例如请求一个用户表格会有不同页码、每页数量、排序字段、过滤字段等参数，使用同样的参数是读取缓存还是重新获取数据。接着我们需要找一个时机将 `pollInterval` 重新设置成 0，防止无限循环的请求。另外建议不要在 React 组件中使用 `setTimeout` 去设置 `pollInterval`，因为我们看上面的代码涉及到了很多 state 变化和 effects，这些在 React 中某种程度上是“异步”的，如果在业务很复杂的场景下使用 `setTimeout`，反而会导致组件的行为难以控制，而且也可能会出现在已经 unmounted 的组件上更新 state 的 error，尽管对用户使用的影响不大，但是对性能是不利的。

### 单元测试

我们知道 React 组件的单元测试可以用 jest + enzyme 来完成。单元测试的时候我们一般会使用 shallow render。而在使用了 Apollo 相关的 API 比如 hooks 之后，jest 会提示我们该组件没有 `ApolloProvider`，这些 Apollo 相关的 API 只能在 `ApolloProvider` 内使用。Apollo 也提供了用于测试的 API：

```JavaScript
import { MockedProvider } from "@apollo/client/testing";
```

具体使用例如下：

```JavaScript
import { MockedProvider } from "@apollo/client/testing";
import { mount } from 'enzyme'

const mocks = [
  {
    request: {
      query: SOME_QUERY,
      variables: { first: 4 }
    },
    result: {
      data: {
        dog: {
          name: "Douglas"
        }
      }
    }
  },
  {
    request: {
      query: SOME_QUERY,
      variables: { first: 8}
    },
    error: new Error("Something went wrong")
  }
]

it("runs the mocked query", () => {
  const wrapper = mount(
    <MockedProvider mocks={mocks}>
      <MyQueryComponent />
    </MockedProvider>
  )
  // Run assertions on <MyQueryComponent/>
});
```

如果你是把组件实际渲染的部分和 `MockedProvider` 写在同一个组件内，那么测试的时候需要用 `mount`，否则无法将组件内部的内容渲染出来。对测试比较友好的方式可能是，将 queries/mutations 的部分独立出来做一个 container 组件，渲染部分作为 dumb 组件。但是这具体得看这方面的需求和代码规范是怎么制定的。

目前这个测试可能还是有点 bug 的状态，我目前项目还没迁移到 3.0beta，所以不大清楚会不会在新版修复。我在旧版中遇到过几种情况，例如 mock 的数据都写对，但实际上返回了 network error，提示参数有问题；还有遇到过 mocks 里写两个对象会出现读取错误的问题，也就是不方便同时 mock 多个 queries/mutations。
