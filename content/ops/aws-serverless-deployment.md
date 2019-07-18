---
title: Amazon AWS Serverless 部署 React 项目
date: 2019-07-18
---

# 写在前面

## 为什么使用 Serverless

因为它是按量(请求数)付费的，相比于 EC 或者容器的按时间付费会节省很多成本，并且 EC 往往会遇到带宽限制的问题，增加带宽会提高很多成本。

## Prerequisite

如果要使用 Amazon AWS Serverless 来部署 React 项目，我们需要一个叫做 `serverless` 的 npm package。同时我们还需要在项目下编写 `serverless.yml`。具体的写法可以参考 `serverless` 的[官方文档](https://serverless.com/)。

# CSR(create-react-app)

客户端渲染，也就是我们最常用的 create-react-app 搭出来的项目，使用 Serverless 的方式部署到 AWS 其实很简单。要让我们的部署能够运行起来，在 AWS 上我们只需要一个服务：S3。S3 有一个静态页面托管的服务，并且会自动生成一个可访问的地址。这个静态站点可以运行我们打包好的 web app。

## 手动部署

首先只要在 AWS S3 的控制面板里创建存储桶就可以了，配置可以都用默认配置。

![](/images/aws-serverless-deployment/aws-1.png)

其次，我们需要进入已经建好的存储桶，把静态网站托管打开。

![](/images/aws-serverless-deployment/aws-2.jpg)

接着我们需要注意，一定要把 error.html 指向 index.html。因为我们可能在项目中使用 react-router，而在 S3 的静态文件托管中，任何子路径都会被认为是 error，因为我们使用 create-react-app 创建的项目，入口文件只有一个 index.html，而不是有其他 html 文件的多页面应用。只有把错误页面指向 index.html，才能正常使用 react-router。

![](/images/aws-serverless-deployment/aws-3.png)

这里会显示该存储桶的访问地址，通过这个地址就可以访问了。接着我们在存储桶的“概述”标签页上传我们打包好的文件（index.html 需要在存储桶的根目录），这样我们的站点就可以访问了。

如果需要 https，我们可以在 CloudFront 上创建一个域名，把源指向我们的 S3 bucket 就可以了。这个过程无需我们申请任何的证书。

## serverless 命令行部署

首先我们需要安装 serverless：运行 `yarn add --global serverless`(如果你安装了 yarn) 或者 `npm install -g serverless`。

serverless 这个 npm package 可以使用插件来辅助进行部署。这里推荐使用 `serverless-finch` 进行部署。我们需要编写一个 `serverless.yml` 文件，serverless 包会读取这个文件的配置进行部署。参考配置如下：

```yaml
service: your-app-name

frameworkVersion: "=1.38.0"

# AWS 的配置
provider:
  name: aws
  runtime: nodejs6.10
  stage: dev
  region: us-east-2
  profile: default

plugins:
  - serverless-finch

# 以下为 serverless-finch 的配置
custom:
  client:
    bucketName: your-bucket-name # 需要和 S3 上创建的 bucket 同名
    distributionFolder: build # 本地打包的输出目录
    indexDocument: index.html # 同 S3 上静态文件托管中设置的入口文档
    errorDocument: index.html # 同 S3 上静态文件托管中设置的错误文档
```

然后我们需要在 AWS Console 的 IAM 上创建一个能读写 S3 的角色。

![](/images/aws-serverless-deployment/aws-4.png)

进入 IAM 的控制台，左侧菜单选择“用户”，然后添加用户。接着填写用户名，并且需要勾选“编程访问”。

![](/images/aws-serverless-deployment/aws-5.png)

接着要给创建的用户附加权限，这边推荐可以选择 AmazonS3FullAccess。

![](/images/aws-serverless-deployment/aws-6.png)

完成创建之后，这里会出现 ID 和密钥，接着我们需要用它：

```bash
serverless config credentials --provider aws --key [YOUR_KEY] --secret [YOUR_SECRET]
```

这里的 key 填写 ID，secret 填写密钥。这一步是设置我们 serverless 的当前用户权限。**注意如果需要切换用户，需要用 AWS CLI 来切换，实测 serverless 中覆盖掉 credentials 的配置并不会生效。**

这样我们就完成了配置。需要部署的时候，我们只需要打包好文件，然后运行 `serverless client deploy` 即可。

# SSR(Next.js)

由于篇幅限制，并且我个人踩坑踩了好几个星期，很多细节没法回忆起来或者无法详细说明。并且作为前端开发者，我们其实并不需要了解整个 AWS Serverless 的工作原理，我们只需要配合后端/运维人员把我们代码层面需要做的做好即可。所以这里只提供一个大概的方案。

## 原理

与 CSR App 不同的是，SSR App 部署完成后并不能直接访问静态页面，而是要通过 API Gateway 去访问（地址会在部署完之后显示在命令行，或者可以在 AWS 的 API Gateway 控制台查看）。当我们访问 API Gateway 时，API Gateway 会解析相应的请求（也就是路由），并转发至 AWS Lambda。而 AWS Lambda 则会根据我们写好的 Lambda 函数（`./server/lambda.js`）来访问 S3 上存储的对应的静态文件。

当然我们也可以设置自定义的域名，但是最好是在 Route53 申请，这样才能享受到 AWS 的一系列服务（全家桶）。

## API Gateway

API Gateway 有几个基本概念/配置项。

首先是 stage，它代表的是不同的阶段，例如开发环境、测试环境、生产环境等等。在 API Gateway 的访问域名中会有一个 `/{stage}` 的路径。

其次是"资源"，这里就是配置 API 访问的了。我们只需要设置一个通用的访问如 `/{proxy+}` 即可，因为我们有许多路由，不可能每个都配置到这里来，而不可访问的路由最好使用 Next.js 中的 `_error.js` 来做，而不是在这里限制。

## serverless

这里会使用到一些不同的插件，原来的 serverless-finch 将不再使用，因为它是设置静态文件托管的。这里使用到了两个：`serverless-apigw-binary` 和 `serverless-domain-manager`。`serverless-apigw-binary` 是用于配置 API Gateway 的，而 `serverless-domain-manager` 可以把自定义域名到 S3 bucket 的整个流程的配置都配置好。由于涉及到机密问题，这里就不把 `serverless.yml` 放出来了，具体可以去搜索一下这两个包的使用方式。

## Lambda

Lambda 其实就是一个压缩文件，里面是各种函数包。它实际上有点类似于一个小型的虚拟机，所以 Next.js 中每个页面打包出来都有 5.9MB。AWS 上的 Lambda 平台会做一个从 API Gateway 到我们上传的函数包中对应的文件的映射，这样就实现了路由功能。

Lambda 在 SSR(Next.js) 中其实就是那个进行渲染和路由导航的 Node.js 服务。而实际上它就是一个 Node 服务器，可以用任意的 Node 框架来写。这里我使用的是 Express。示例代码如下：

```javascript
const express = require("express");
const routes = require("./routes"); // next-routes 做的自定义路由

const app = express();

// host the static files
app.use("/_next/static", express.static(path.join(__dirname, "/static")));
app.use("/static", express.static("static"));

app.get("*", (req, res) => {
  const parsedUrl = parse(req.url, true);
  const { pathname, query, search } = parsedUrl;

  // Map url to page directory
  const matchRoute = routes.match(pathname);
  if (matchRoute && matchRoute.route) {
    const page = matchRoute.route.page;
    const params = matchRoute.params;

    if (params) {
      if (search) {
        req.url += "&" + querystring.stringify(params);
      } else {
        req.url += "?" + querystring.stringify(params);
      }

      try {
        __non_webpack_require__(`./serverless/pages${page}`).render(
          req,
          res,
          parsedUrl,
          Object.assign(params, query)
        );
      } catch (err) {
        __non_webpack_require__("./serverless/pages/_error").render(
          req,
          res,
          parsedUrl,
          Object.assign(params, query)
        );
      }
    } else {
      try {
        __non_webpack_require__(`./serverless/pages${page}`).render(
          req,
          res,
          parsedUrl
        );
      } catch (err) {
        __non_webpack_require__("./serverless/pages/_error").render(
          req,
          res,
          parsedUrl
        );
      }
    }
  } else {
    __non_webpack_require__("./serverless/pages/_error").render(
      req,
      res,
      parsedUrl
    );
  }
});
```

这边实际上就是在做路由的映射，然后载入 Next.js 打包出来对应的页面入口，接着就可以使用 React 的能力进行渲染。关于 Next.js，可以参考我的另一篇博文：[Next.js 实战：从 CSR 迁移到 SSR](https://elizurhz.cn/frontend/mirgation-from-csr-to-ssr/)。但是在实际生产中，这个文件是需要被编译成 ES5 的。所以我们还需要写一个 Webpack 配置来打包它，只需要在根目录建一个 `webpack.config.js` 即可：

```javascript
// webpack.config.js
const path = require("path");
const CopyWebpackPlugin = require("copy-webpack-plugin");

const LAMBDA_FILE_LOC = path.join(__dirname, "./server/lambda.js");
const LAMBDA_DIR = path.join(__dirname, "./build");
const LAMBDA_FILE_PROD = "./lambda_prod.js";

const plugins = [
  new CopyWebpackPlugin([
    // your custom config
  ])
];

// 打包配置
module.exports = {
  entry: LAMBDA_FILE_LOC,
  externals: ["aws-sdk"], // 设置成 AWS Lambda 支持的格式
  output: {
    libraryTarget: "commonjs",
    filename: LAMBDA_FILE_PROD,
    path: LAMBDA_DIR
  },
  target: "node",
  node: {
    __filename: false,
    __dirname: false
  },
  mode: "production",
  plugins
};
```

然后运行 `npx webpack` 即可进行打包。使用 `CopyWebpackPlugin` 的原因主要是，我们会遇到一些文件无法被 Next.js 打包，但是它们又是 Web App 运行必需的，或者是 AWS Lambda 的函数包需要的。所以我们只能手动把它们拷贝到 Next.js 的 build 目录，因为 serverless 会根据这个目录来打包出 AWS Lambda 所需的函数包。

## 部署流程

1. 首先会先运行 `next build`，将文件打包至 `./build` 目录。由于在 `next.config.js` 中设置了 `target: 'serverless'`，所以打包出来的文件是 serverless 的版本，会与云主机、容器部署的版本不同。

2. 编译 `./server/lambda.js`. 并在 `./build` 目录下生成 `lambda_prod.js`。这一步主要是要把 ES6 写的 lambda 函数编译成 ES5，这样 AWS Lambda 才能正常访问它。

3. 将所需文件打包成 `.zip`。压缩文件会在 `./.serverless` 目录下生成。 需要打包的内容可以在 `serverless.yml` 的如下字段中设置：

```yaml
package:
  exclude:
    - ./**
  include:
    - build/**
    - static/**
```

4. 上传压缩文件至 S3，同时会自动修改 API Gateway 和 AWS Lambda 的相应设置。
