---
title: 前后端分离项目的容器化
date: 2019-03-24
---

# 前后端分离项目的容器化

## docker

CentOS 7 上

```bash
yum install -y docker
```

### 基本命令

```bash
docker images # 列出当前本地所有的 docker 镜像
```

```bash
docker search [docker-name] # 在 docker hub 中搜索镜像
```

```bash
docker pull [docker-name] # 在 docker hub 中拉取镜像到本机
```

```bash
docker run \ # 启动镜像
-d \ # --detach
-h example.com  \ # --hostname，主机
-p 8443:443 -p 8888:80 -p 2222:22 \ # --publish 端口，格式：本机端口:容器端口，将容器端口映射到本机端口
--name example \ # 容器的名字
-e XXX=xxx \ # 指定环境变量
-v /your/local/path:/your/image/path \ # --volume，格式：本机路径:容器路径，挂载卷，可以使本机的目录和容器中的目录互通、同步，可以用于持久化容器数据
[docker-name] # 镜像名字，如 mysql, gitlab/gitlab-ce 等
```

```bash
docker ps # 查看所有已创建的镜像
docker ps -a # 查看所有运行中的镜像
```

```bash
docker start/stop/restart/rm [container id/container name] # 启动、停止、重启、删除某个容器，可以使用容器 id 或者容器名
```

容器在 docker run 之后会在 `docker ps -a` 的列表中，如果需要重新运行这个容器，必须先停止它再 rm。
