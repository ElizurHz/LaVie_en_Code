---
title: 用 Webpack 从零开始学习打包 React + TypeScript 的项目
date: 2019-08-14
categories: ["前端"]
tags: ["webpack", "React", "TypeScript"]
draft: true
---

# TL;DR

Webpack 在现在的业务开发中好像重要程度越来越低，一般的项目打包我们其实直接用脚手架就可以了，因为脚手架的打包是直接打包成可以部署到各种服务上的形式。简单的定制化处理也非常简单，只需要使用 `react-app-rewired` 来覆盖一些 webpack 配置即可，例如 ant-design 的使用。而如果需要做被引用的组件、npm 包这种，那么仅仅使用脚手架自带的 webpack 配置显然是不行的。

而我现在其实相当于从零开始学习，所以打包出来的东西不免会做不到最优化，所以会持续去优化它。

# 理解”需要达成什么目标“

Webpack 的工作，实际上就是对文件、代码进行解析，并转化为指定的输出内容。

首先在这里，我需要处理的是一个 React + TypeScript 的项目，输出的是一个 JavaScript 版本的，能被其他 React 项目引用的模块/组件。其次，我使用了 less 进行样式管理，同样地，依赖的 ant-design 也使用了 less，而为了兼容性，我需要把它转换成 css 文件。

# webpack 核心构件

## entry

## output

## modules

### TypeScript

TypeScript 的 loader 可以使用 ts-loader，不过现在社区有一个更好的解决方案就是 awesome-typescript-loader。

### 样式文件

style-loader/MiniCssExtractPlugin.loader, css-loader(, less-loader/sass-loader...)，运行顺序从右往左

## resolve

## optimization

## plugins

# In practice

## TypeScript 配置

TypeScript 的打包配置不仅仅在 `webpack.config.js` 中，如果你只尝试去修改这里面的配置，那很可能达不到想要的效果。事实上，TypeScript 的 webpack loader 会读取项目中的 `tsconfig.json`，根据里面的配置去编译。当然，在这种打包的需求场景下，建议单独建一个 `tsconfig.json` 文件，可以放在不同的目录，也可以有不同的命名。

我们知道 tsc 中可以通过 `--p` 或者 `--project` 来指定编译的 TypeScript 文件。而在 webpack 的 loader 中，我们需要写 query string，例如：`awesome-typescript-loader?configFileName=tsconfig.webpack.json`，`ts-loader?configFile=tsconfig.webpack.json`。`awesome-typescript-loader` 的指定参数是 `configFileName`，`ts-loader` 的则是 `configFile`。参数可以指定目录，例如要指定配置文件为 `./config/tsconfig.json`，需要在 loader 里写 `awesome-typescript-loader?configFileName=config/tsconfig.json`。
