---
title: Docker常用命令
date: "2018-01-08"
tags: ["Docker"]
---
# Docker note
- `docker info`显示详细的信息
- docker 命令行格式`docker <command> <sub-command> (option)` command 一般指manage command,比如docker container run
- `--detach\-d`参数，使container在后台运行
- `docker container run` 会一直新建一个container,使用`docker container start`启用一个已存在的
- `docker container --help`
- `docker container logs [containerName]`显示container 日志信息
- `docker container top [containerName]`显示container进程
- ` docker container run -d -p 3306:3306 --name db -e MYSQL_RANDOM_ROOT_PASSWORD=yes  mysql` 新建一个mysql, `docker container logs db` 查看生成的密码
- `docker container inspect` container配置详细
- `docker container stats`查看所有container 状态
- `docker container exec -it [name] bash` 进入container
- `docker container port [name]`
- `docker image history [imagename:tag]` 查看历史
- `docker image tag ...`更换标签

## dockerfile build
- `docker build -f some-dockerfile`
- `docker image build -t [tag name] .`

```
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log
# forward request and error logs to docker log collector
```

- `WORKDIR` 切换目录
- `docker volume ls` 查看挂在目录

## docker compose
- `docker-compose -f`

```
version: '3.1'  # if no version is specificed then v1 is assumed. Recommend v2 minimum

services:  # containers. same as docker run
  servicename: # a friendly name. this is also DNS name inside network
    image: # Optional if you use build:
    command: # Optional, replace the default CMD specified by the image
    environment: # Optional, same as -e in docker run
    volumes: # Optional, same as -v in docker run
  servicename2:

volumes: # Optional, same as docker volume create

networks: # Optional, same as docker network create
```

- `docker-compose ps`

