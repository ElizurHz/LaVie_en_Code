---
title: ECS 上的 git 管理工具踩坑记
date: 2019-03-24
categories: ["运维/部署"]
---

# ECS 上的 git 管理工具踩坑记

## gitlab

### 在 CentOS 上安装 gitlab

参考 [8 小时外实践系列(六) - 在阿里云服务器上搭建 GitLab(草稿) - CSDN 博客](https://blog.csdn.net/caib1109/article/details/51756246)

> gitlab 非常吃性能，在 1 核 1G 的服务器上安装后会导致机子非常卡，同时域名/IP 访问也会出现 502，查资料之后发现是配置太低的问题。官方推荐的配置是 2 核心 8G 内存，我们普通个人用户显然难以支付这么昂贵的服务器配置，更不用说海外主机了。

### 使用 docker 部署 gitlab

只需要在 docker hub 中 `docker pull gitlab/gitlab-ce` 就行，然后我启动容器的的代码如下：

```bash
sudo docker run --detach \
  --hostname gitlab.mydomain.com \
  --publish 8443:443 --publish 80:80 --publish 2222:22 \
  --name gitlab \
  --restart always \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce
```

如需修改 gitlab.rb 的配置，只需要 `vim /srv/gitlab/config/gitlab.rb` 。

一般需要配置的有：

```bash
external_url '你的域名或者IP' # 改成你的域名或者 IP，如果你的端口映射不是 80，那也不能在这里加端口
gitlab_rails['gitlab_shell_ssh_port'] = 2222 # 防止和本机 22 端口冲突
unicorn['port'] = 8999 # 如果你的 8080 端口被占用的话
```

**另外记住，如果单开某个端口，需要在 ECS 实例的安全组中允许这个端口被某个或者某些 IP 访问。**

配置完成之后需要在容器中 reconfigure: `docker exec -it gitlab gitlab-ctl reconfigure`

> 然而我按照上面的方式完成之后，通过域名或者 IP 进入仍是 502。
> 后来我尝试了在不同机子上部署。例如在群辉 DS218+ 上用同样的方式配置会出现 容器意外停止 的问题；在我的 iMac 本机上用 docker 部署就可以正常访问和进行代码管理

## gogs

gogs 不像 gitlab 那样吃性能，功能也很完善，不过 gogs 需要有额外的数据库依赖，如 MySQL。具体的部署方式参考以下两篇文章：

[docker 安装 mysql - 简书](https://www.jianshu.com/p/5f5e419b5de8)

[docker 安装 gogs - 简书](https://www.jianshu.com/p/2a7acb07b352)

> 勘误：在《docker 安装 mysql - 简书》中，创建网络的命令不是 `docker create network backend` 而是 `docker network create backend`

我的容器启动代码如下：

```bash
# 启动 MySQL
docker run -d --name mysql57 \
--net=backend \ # 加入创建的 backend 网络中
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=root \
-v /srv/mysql/data:/var/lib/mysql \
-v /srv/mysql/conf:/etc/mysql/conf.d \
# 启动 gogs
docker run -d \
-p 10022:22 -p 3000:3000 \
--name=gogs \
--net=backend \ # 加入创建的 backend 网络中
-v /srv/gogs/:/data gogs/gogs
```

此时访问服务器的 3000 端口即可进入设置界面，记住要把 localhost 全部改成服务器 ip。

### 关于 gogs 的数据库设置

首先由于本例使用了 docker network，所以可以直接使用 容器名:端口 来访问，例如 mysql57:3306。

另外我们需要在 mysql 容器中创建一个数据库，否则 gogs 无法完成设置。

```bash
docker exec -it mysql57 /bin/bash
mysql -u root -p
# 然后输入密码，如果你没更改密码，则密码是 root
```

接着在 mysql 命令行中创建数据库，gogs 默认数据库名就是 gogs

```sql
CREATE DATABASE gogs;
```

接着我们在网页中就可以完成 gogs 的初始化设置了。

如果想要使用 gogs，首先我们要注册一个账号，注意第一个注册的账号默认是拥有最高权限的账号。

> 在实际使用中，我遇到了 git clone 时发现有 not a repository 的问题，可能是 IP/域名、端口等等设置的问题，仍在研究中

---

参考：

1. [8 小时外实践系列(六) - 在阿里云服务器上搭建 GitLab(草稿) - CSDN 博客](https://blog.csdn.net/caib1109/article/details/51756246)
2. [docker 安装 mysql - 简书](https://www.jianshu.com/p/5f5e419b5de8)
3. [docker 安装 gogs - 简书](https://www.jianshu.com/p/2a7acb07b352)
