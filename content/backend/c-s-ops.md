---
title: 前后端分离项目在阿里云/七牛云的简单部署
date: 2019-03-24
categories: ["后端"]
---

# 前后端分离项目在阿里云/七牛云的简单部署

## 购买主机

我购买了两台主机，都是的是 1 核 + 1GHz + 1Mbps 的配置，系统一台是为 Ubuntu 14.04，另一台是 CentOS 7.4。直连实测 700 多 KB 的 Webpack 生产打包后的 React 的 js 文件需要十多秒才能加载，速度大约 60KB/s。

## 启动实例

购买后实例是自动启动的，但是需要进行一些初始化的设置。

进入 ECS 的实例页面，首先如果需要从控制台远程连接，阿里云会提供一个远程连接密码，这个密码需要记住，每次从控制台远程连接时会需要。

我们正常的开发一般是从 Terminal 或者 FileZilla 等 SFTP 客户端去连接服务器，为了进行此操作，我们需要设置初始密码。控制台的实例页面点击 “更多” => “密码/密钥” => “重置密码” 即可重置 Linux 登录密码，也是 ssh 的密码。重置后我们就可以使用刚刚输入的密码，通过 `ssh root@${yourServerIp}` 登陆。如果你已经在购买的 ECS 时候设置了密码或者密钥，那可以不需要在此设置密码。

而 FileZilla 也是一样，建立一个 SFTP 连接，使用用户 root 和更改好的密码即可。

## 系统配置

首先我们需要把包管理器源替换成国内的源，Ubuntu 的 apt 可以使用清华的源，CentOS 的 yum 可以使用阿里云的源。

1. Ubuntu

```bash
cd /etc/apt/
sudo cp sources.list sources.list.bak  # 备份 source.list
sudo vim sources.list
```

接着将 source.list 中的源替换为清华的源

```bash
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse
```

最后，更新源

```bash
sudo apt-get update # 更新源
sudo apt-get upgrade # 更新软件
```

2. CentOS

```bash
cd /etc/yum.repos.d/
wget http://mirrors.aliyun.com/repo/Centos-7.repo # 下载 repo
mv CentOs-Base.repo CentOs-Base.repo.bak # 备份
mv Centos-7.repo CentOs-Base.repo # 替换
# 更新
yum clean all
yum makecache
yum update
```

### 安装 nginx

需要安装的依赖：

- gcc gcc-c++
- pcre
- openssl
- zlib
- zlib-devel

以上可以都用 yum 安装。安装完成之后 `yum install -y nginx` 即可成功安装 nginx。如果以上依赖未完全安装，在安装 nginx 时则会提示某个包未安装。

### 配置 Nginx

根据版本的不同，nginx 的配置文件可以放在 site-enable 或者 conf.d 中。

1. 代理端口、域名

2. try_files

3. https
4. SSL 证书

5. 强制转写为 https

6. 强制转跳
   例如一级域名转跳到 www 二级域名

### 部署服务端

我的项目服务端用的是 Koa2，托管在 bitbucket 上。

1. 从 bitbucket 上将项目拉取下来。
2. `npm install` 安装需要的 npm packages。
3. 通过 pm2 启动服务，我的项目用的是 Koa2 的脚手架，使用命令 `pm2 start ./bin/www --watch` 即可。
4. 可以通过 `pm2 list` 查看正在运行的服务，例如我有部署多个后端服务；通过 `pm2 describe ${id}` 和 `pm2 show ${id}` 可以查看某个服务的详情。

### MongoDB

如果需要在本机（非服务器所在内网）测试访问 MongoDB，则需要修改 MonogoDB 配置。

## 云解析 DNS

## 安全组

## SSL
