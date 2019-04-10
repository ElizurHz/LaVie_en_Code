---
title: AWS Serverless 部署前端项目
date: 2019-03-24
draft: true
---

# CSR

# SSR

## 部署流程

1. 首先会先运行 `next build`，将文件打包至 `./build` 目录。由于在 `next.config.js` 中设置了 `target: 'serverless'`，所以打包出来的文件是 serverless 的版本，会与云主机、容器部署的版本不同。

2. 编译 `./server/lambda.js`. 并在 `./build` 目录下生成 `lambda_prod.js`。

3. 将所需文件打包成 `.zip`。压缩文件会在 `./.serverless` 目录下生成。 需要打包的内容可以在 `serverless.yml` 的如下字段中设置：

```yaml
package:
  exclude:
    - ./**
  include:
    - build/**
    - static/**
    - lang/**
```

4. 上传压缩文件至 S3，同时会自动修改 API Gateway 和 AWS Lambda 的相应设置。

## 原理

与 CSR App 不同的是，SSR App 部署完成后并不能直接访问静态页面，而是要通过 API Gateway 去访问（地址会在部署完之后显示在命令行，或者可以在 AWS 的 API Gateway 控制台查看）。当我们访问 API Gateway 时，API Gateway 会解析相应的请求（也就是路由），并转发至 AWS Lambda。而 AWS Lambda 则会根据我们写好的 Lambda 函数（`./server/lambda.js`）来访问 S3 上存储的对应的静态文件。

