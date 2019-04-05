---
title: Next.js 实战：从 CSR 迁移到 SSR
date: 2019-03-26
categories: ["前端"]
tags: ["React", "SSR", "Javascript", "Next.js"]
---

> - CSR: 客户端渲染，Client Side Rendering
> - SSR: 服务端渲染，Server Side Rendering

## TL;DR

目前正在做一个项目，由于采用了敏捷开发，所以我们的自主权比较大，领导也让我们尝试使用 SSR，所以我们自然而然在原有的 create-react-app 创建的应用的基础上选择了 Next.js。

下面大致说明一下 SSR 项目与 CSR 项目到底有什么不同。

## 关于浏览器与 DOM

SSR 项目还是一个运行在浏览器的项目，所以是可以支持浏览器 API 的，但是它是由一个 Node.js 服务器来写入 html，而不是像 CSR 一样加载静态的 js 来渲染。这样会导致问题就是，如果在渲染时立即调用了浏览器/DOM 相关的 API，此时可能浏览器环境还未加载完成，那么就会出现 `document is not defined` 的错误。

关于 `document is not defined`，比较常遇到的情况有：

- 页面调用了 document、window 相关的 API
- 使用了 ReactDOM.createPortal。由于 ReactDOM.createPortal 必须有一个挂载的 DOM 节点，所以在浏览器环境完成前是无法使用的，解决方法就是在 `componentDidMount` 之后再加载：用变量进行条件渲染，参考[官方例子](https://github.com/zeit/next.js/tree/canary/examples/with-portals)。
- 某些组件如果使用 DOM API，且在被引用时未使用 `next/dynamic` 进行异步加载，也可能会导致这种问题

## Packages

大部分 CSR App 中关键的 packages 都可以直接在 Next.js 中使用，例如 redux, redux-saga, react-intl 等，但是使用的方式和在 CSR App 中的有所区别。另外，Next.js 中需要使用 `@zeit` 的一些 packages，它们与 SSR 相关。

## next.config.js

这个是放在根目录的 next 的配置文件，我们在这里可以配置一些 webpack loaders 的支持，例如 CSS（如果不配置就只能支持行内和 styled-jsx）, SASS, TypeScript 等，需要安装一些 `@zeit` 的 plugins。

根据项目需求，我做了如下配置：

```javascript
const withTypescript = require("@zeit/next-typescript");
const withCSS = require("@zeit/next-css");
const withSass = require("@zeit/next-sass");

module.exports = withSass(withCSS(withTypescript()));
```

### 样式

在 Next.js 中，样式默认使用的是 React 的 inline style 或者 Next.js 的 styled-jsx，它们只能写在组件文件中。如果需要使用独立的 css 文件或者其他 css 预处理器，那么需要安装相关的[插件](https://nextjs.org/docs/#importing-css--sass--less--stylus-files)。

就如上面代码所示，插件安装完之后需要在 `next.config.js` 中进行配置。

### TypeScript

Next.js 官方有 TypeScript 支持，需要安装 `@zeit/next-typescript` 这个插件，并配置 `tsconfig.json` 和 `next.config.js`。

## server.js

这是 SSR 和 CSR 最大的不同 —— 我们需要一个 Node.js 服务器。在 `package.json` 中，默认的 `scripts` 会运行 `next` 这个命令，启动的是 Next.js 内置的 SSR 服务器，而如果我们写一个 Node.js 服务器，并在 `scripts` 中运行它，同样可以实现 SSR 的功能，并且我们还可以在里面定制路由、渲染方式、缓存等等。

### 路由

在 Next.js 中，我们使用 Node.js 框架（Koa）的路由，所以 `react-router` 相关的代码都必须更改。详情可以参考[官方文档](https://nextjs.org/docs/#routing)。

- `react-router` 的 history 无法在 Next.js 中使用，我们可以使用 `next/router` 替代之（`import Router from 'next/router'`）。例如 `history.push()` 可以用 `Router.push()` 替代；也可以 `import { withRouter } from 'next/router'` 来把路由参数注入到组件的 props 中。
- 原本的 `react-router-dom` 中的 `<Link>` 可以被 `next/link` 的 `<Link>` 取代，但是 props 有所不同，比如 `to` 需要换成 `href`，以及 `<Link>` 中不能使用 text 节点等。
- 路由与 `.js/.tsx` 文件的绑定在 Koa 中完成，使用 `koa-router` 的写法，当浏览器访问路由时，由服务端的中间件设置对应的 return body 并返回给前端。具体可以参考 [koa-router](https://github.com/ZijianHe/koa-router)。

### 国际化

国际化的方式已经在上面的 `server.js` 的部分叙述过。在项目中我们一样需要一个存放 `.json` 翻译文件的地方。

下面是一段官方的代码示例，我在官方使用的 express 框架的基础上将其修改成了适用于 koa 的版本。

```javascript
const supportedLanguages = glob
  .sync("./lang/*.json")
  .map(f => basename(f, ".json"));
const localeDataCache = new Map();
const getLocaleDataScript = locale => {
  const lang = locale.split("-")[0];
  if (!localeDataCache.has(lang)) {
    const localeDataFile = require.resolve(`react-intl/locale-data/${lang}`);
    const localeDataScript = readFileSync(localeDataFile, "utf8");
    localeDataCache.set(lang, localeDataScript);
  }
  return localeDataCache.get(lang);
};

const getMessages = locale => {
  return require(`./lang/${locale}.json`);
};

const localization = ctx => {
  const { req } = ctx;
  const { query } = ctx.request;
  let locale;
  if (query.lang) {
    locale = query.lang;
  } else {
    const accept = accepts(req);
    locale = accept.language(accept.languages(supportedLanguages)) || "en";
  }
  req.locale = locale;
  req.localeDataScript = getLocaleDataScript(locale);
  req.messages = getMessages(locale);
  ctx.req = req;
  return ctx;
};
```

这段代码的目的是修改前端请求时的 context。来当浏览器访问路由时，服务端对应的 `Controller` 可以获取到 koa 的 `context` 对象，并通过一些方法把当前所需的国际化的文件内容加入到 `context.req` 中，这样前端页面就可以接收到这些数据。

在路由的 `Controller` 处理 ctx 的时候，我们只需要调用 `localization` 这个函数，对 ctx 进行一层“包装”即可。

### 缓存

React SSR 或者说 Next.js 里，最常用的页面缓存工具就是 `lru-cache` 了。使用 `lru-cache` 可以减少不必要的渲染以及加快加载的速度。

```javascript
const getCacheKey = req => `${req.url}`;

function renderAndCache(ctx, pagePath, queryParams) {
  const key = getCacheKey(ctx.req);

  // If we have a page in the cache, let's serve it
  if (ssrCache.has(key)) {
    console.log(`CACHE HIT: ${key}`);
    ctx.body = ssrCache.get(key);
    return;
  }

  // If not let's render the page into HTML
  return app
    .renderToHTML(ctx.req, ctx.res, pagePath, queryParams)
    .then(html => {
      // Let's cache this page
      console.log(`CACHE MISS: ${key}`);
      ssrCache.set(key, html);
      ctx.body = html;
    })
    .catch(err => {
      console.log("ERRR", err);
      return app.renderError(err, ctx.req, ctx.res, pagePath, queryParams);
    });
}
```

它的基本原理就是在向前端返回数据时进行判断，如果 存在 cache key，则调取缓存中相应的内容；否则服务端才会根据路由调用相应的 `Controller` 向前端返回数据。

如果需要使用它，我们可以将路由的 `Controller` 直接设成 `renderAndCache` 函数即可，例如：

```javascript
router.get("/your/path", async ctx => renderAndCache(ctx, "/your/path"));
```

如果需要结合国际化来使用，只要对 `renderAndCache` 的第一个参数 `ctx` 使用上面的 `localization` 函数进行处理即可。

## pages

根目录的 `pages` 文件夹是 Next.js 默认的路由对应的页面入口。

### \_app.tsx

这个文件相当于 CSR App 中的 `App.tsx`。如果你不创建这个文件，SSR App 也可以运行，因为 Next.js 中有一个默认的 `_app.tsx`。我们创建它是因为我们需要重写这个主入口组件，例如加入 Redux 的 `<Provider>` 和 `react-intl` 的 `<IntlProvider>` 等高阶组件。

### \_document.tsx

与 `_app.tsx` 一样，这也是个 Next.js 内置的组件，而如果我们需要实现某霞功能，也需要重写它。它相当于是 CSR App 中的 `index.html`。在 `_document.tsx` 中，有 `<Head />`、`<Main />`、`<NextScript />` 等组件，`<Head />` 对应的就是 html 的 `<head />` 标签，`<Main />` 的位置则会注入 `_app.tsx` 这个组件，`<NextScript />` 则是打包后一些 js 文件插入的位置。你可以在里面添加 style 和 script 等标签，它们将最终会渲染在页面上。`<NextScript />` 在 Next.js 的 production 环境中会默认启用 preload，这样可以节约加载的时间。

## 项目结构

我们的实践方式是，把 `pages` 作为 Container，在这里进行高阶组件的连接，包括 redux、国际化等等。其他的结构与 react + redux + redux-saga 的最佳实践无异。

## 构建方式

- `dev` 可以启动一个 dev server，它会自动构建项目到根目录下的 `.next` 文件夹中，可以在代码保存后热更新。
- 如果需要在 EC/Docker 之类的生产环境中运行，需要先 `build`，以生产的环境变量构建项目，再 `start` 启动服务。这里我们相当于是直接部署/使用了 Node.js 的 runtime，开启了一个 Node.js 进程，所以这一步可以用 pm2 等管理工具来替代。
- `next export` 是可以输出静态文件的。它是根据 pages 的目录结构输出的，可以不依赖于服务端来展示静态页面，当然正常的 JavaScript 也是可以运行的。但是有一点就是，如果在后端加了一些修改渲染的操作，比如上面提到的缓存和国际化，那在这个打包方式下，它们是完全不可用的。也就是说如果你有在 Node.js 层加入一些除路由外的额外功能，那你是不能使用静态页面的方式构建并直接上传到 github pages 或者 firebase 等托管服务的。

## 错误提示

由于 Next.js 运行时每次都需要编译，如果编译未报错而在运行时报错，那么错误提示只会显示编译完的文件的错误位置，而不像 CSR 一样能显示原始文件的错误位置，这对调试非常不友好。最常遇到的就是 `document is not defined`，但是大项目的迁移中很可能不记得自己那个地方用到了 DOM API 导致这样的报错。

## 单元测试

由于 `create-react-app` 自带 jest 并且将测试命令集成在了 react-scripts 中（inject 之前），我们不需要额外对 jest 的 match、mock、transform 等属性进行配置。在 Next.js 中，我们则需要重新对 jest 进行配置。

我使用的 jest 配置如下：

```javascript
module.exports = {
  moduleFileExtensions: ["ts", "tsx", "js"],
  transform: {
    "^.+\\.tsx?$": "ts-jest",
    "^.+\\.js$": "babel-jest"
  },
  testMatch: ["**/*.(test|spec).(ts|tsx)", "**/__tests__/*"],
  globals: {
    "ts-jest": {
      babelConfig: true,
      tsConfig: "jest.tsconfig.json"
    }
  },
  coveragePathIgnorePatterns: ["/node_modules/", "enzyme.js"],
  setupFilesAfterEnv: ["<rootDir>/enzyme.js"],
  coverageReporters: ["json", "lcov", "text", "text-summary"],
  moduleNameMapper: {
    "\\.(jpg|jpeg|png|gif|eot|otf|webp|svg|ttf|woff|woff2|mp4|webm|wav|mp3|m4a|aac|oga)$":
      "<rootDir>/__mocks__/fileMock.js",
    "\\.(css|less|scss)$": "<rootDir>/__mocks__/styleMock.js"
  }
};
```

由于我们项目使用了 TypeScript，所以需要用 ts-jest 来解析 TypeScript，同时我们需要配置一下运行单元测试是的 TypeScript 编译规则，也就是 `jest.tsconfig.json` 这个文件：

```json
{
  "compilerOptions": {
    "allowSyntheticDefaultImports": true,
    "module": "commonjs",
    "target": "esnext",
    "jsx": "react",
    "sourceMap": false,
    "experimentalDecorators": true,
    "noImplicitUseStrict": true,
    "removeComments": true,
    "moduleResolution": "node",
    "lib": ["es2017", "dom"],
    "typeRoots": ["node_modules/@types"]
  },
  "exclude": ["node_modules", "out", ".next"]
}
```

这样的配置仍然存在一个问题，就是：jest 中无法正常解析某些 import 语句。`"allowSyntheticDefaultImports": true` 这个 TypeScript 配置的目的是允许 default import。在某些包里是没有 default export 的，例如 React。在 JavaScript 中我们可以用 `import React from 'react'` 的原因是这个是直接从 react 包本体中引入的，而 react 包中是有 `module.exports` 的语句的。但是在 TypeScript 中，import 实际上解析到的并不是 react 包本体，而是 `@types/react` 这个包。而这个包的 index.d.ts 文件里是没有一个 default export 的，所以 TypeScript 中如果我们不使用 `"allowSyntheticDefaultImports": true` 的配置，`import React from 'react'` 是会报错的。而在 jest 执行的时候，我即使开启了配置，还是遇到了这样的报错。所以最后我们决定全部改成 `import * as React from 'react'` 的写法了。同样有这种情况的还有 `ReactDOM` 和 `proptypes` 这两个包。
