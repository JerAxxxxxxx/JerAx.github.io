---
layout:     post
title:      Kong&Konga docker 部署
subtitle:   附一键安装脚本
date:       2021-08-18
author:     JerAxxxxxxx
header-img: img/background/under_construction_sign_work.jpg
catalog: true
tags:
- kong
---


Kong 是目前比较主流的 API 网关，最近公司也在生产中落地，打算记录下来，包括安装部署以及后面 lua 插件的开发和 go 插件的开发。

鉴于网上的部署教程层次不齐，甚至有的教程甚至跑不起来，自己备份了一份安装脚本，方便部署测试，即使是小白也能部署成功的脚本。



以下是部署脚本，sleep 主要是等待数据库初始化完毕，否则可能会连接失败

```shell
#!/bin/bash
# docker 网络名称
export NET_WORK_NAME=kong-net
# postgres 镜像名
export POSTGRES_IMAGE_NAME=postgres:9.6
# kong 镜像名
export KONG_IMAGE_NAME=kong:latest
# konga 镜像名
export KONGA_IMAGE_NAME=pantsel/konga:latest

# 创建网络并安装 postgres
function install_postgres() {
  docker network create "${NET_WORK_NAME}"

  docker run -d --name kong-database \
  --network=${NET_WORK_NAME} \
  -p 5432:5432 \
  -e "POSTGRES_USER=kong" \
  -e "POSTGRES_DB=kong" \
  -e "POSTGRES_PASSWORD=kong" \
  "${POSTGRES_IMAGE_NAME}"
}

# 初始化数据库
function prep_kong_db() {
  docker run --rm \
  --network=${NET_WORK_NAME} \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_USER=kong" \
  -e "KONG_PG_PASSWORD=kong" \
  ${KONG_IMAGE_NAME} kong migrations bootstrap
}

# 启动 kong
function run_kong() {
  docker run -d --name kong \
  --network=${NET_WORK_NAME} \
  -e "KONG_DATABASE=postgres" \
  -e "KONG_PG_HOST=kong-database" \
  -e "KONG_PG_USER=kong" \
  -e "KONG_PG_PASSWORD=kong" \
  -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
  -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
  -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 8444:8444 \
  ${KONG_IMAGE_NAME}
}

# 启动 konga 管理界面
function start_konga() {
  docker run -d -p 1337:1337 \
  --network ${NET_WORK_NAME} \
  -e "DB_ADAPTER=postgres" \
  -e "DB_HOST=kong-database" \
  -e "DB_USER=kong" \
  -e "DB_PASSWORD=kong" \
  --name konga \
  ${KONGA_IMAGE_NAME}
}

install_postgres && \
sleep 4 && \
prep_kong_db && \
run_kong && \
start_konga
```





----

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。