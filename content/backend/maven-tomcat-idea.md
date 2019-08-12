---
title: 用 IntelliJ IDEA 配置 Maven Project 并使用 Tomcat 在本机运行
date: 2019-03-24
categories: ["后端"]
tags: ["Java"]
---

# 用 IntelliJ IDEA 配置 Maven Project 并使用 Tomcat 在本机运行

作为一个前端开发，时不时会参与维护一些老的前后端未分离的项目，而服务端渲染的老项目大多还是以 Java 为主，这样就起码需要懂得怎样配置项目，怎样起服务，否则天天需要服务端的同学帮忙岂不是很麻烦。

## Java 环境

首先上官网下载 JDK，然后安装。我这里使用 1.8 的版本，因为太高版本好像不支持。

然后下载 IntelliJ IDEA。

接着打开项目。这里以 Windows 10 为例（公司内网服务只能用有线连接的台式机访问很蛋疼不能用 Mac 开发）。首先 File => New => Project from Existing Sources...

![open](/images/maven-tomcat-idea/open.png)

然后选择项目并使用默认配置即可。

## 配置 Maven

点开右侧栏中的 Maven Projects，点击设置按钮

![maven-1](/images/maven-tomcat-idea/maven-1.png)

按如图所示选择 Maven home directory（内有相应的依赖配置）。另外在 User settings file 右侧勾选 override，然后选择 Maven home directory 中的 `/conf/settings.xml` 文件。然后 Apply 即可。

![maven-2](/images/maven-tomcat-idea/maven-2.jpg)

接着右下角会提示你是否需要 Import，选择 Import 即可。或者你可以在 Maven 面板中先双击 clean，然后再双击 install。这一步会完成 Maven 依赖的安装。

## 配置 Tomcat

首先找到右上角的工具栏，红框所示位置如果你没有配置 Tomcat，会显示 Add Configuration。点击这里即可。

![tomcat-1](/images/maven-tomcat-idea/tomcat-1.png)

然后点击 ➕ ，找到 Tomcat Server，添加 Local。

![tomcat-2](/images/maven-tomcat-idea/tomcat-2.png)

接着我们需要配置 Tomcat home。点击面板右侧的 Application server 右侧的 `Configure...`，

![tomcat-3](/images/maven-tomcat-idea/tomcat-3.png)

配置 Tomcat home，选择你下好的 Tomcat 的包的根目录。

![tomcat-4](/images/maven-tomcat-idea/tomcat-4.jpg)

接着切换到 Deployment 标签，➕ => Artifact...

![tomcat-5](/images/maven-tomcat-idea/tomcat-5.png)

这里你可以看到当前可用的 war 包，选择“war exploded”的那个，然后点 OK，Apply。

![tomcat-6](/images/maven-tomcat-idea/tomcat-6.png)

最后点击右上角的 Debug（虫子）按钮，等待 Tomcat 服务器启动即可。IntelliJ IDEA 会在你指定的浏览器里自动打开你配置的那个本地地址，在那里即可访问你做好的模板页面。
