---
layout:     post
title:      Docker Compose 部署 Kong, 8443 端口无法访问
subtitle:   docker-compose.yaml 参数的问题
date:       2021-08-25
author:     JerAxxxxxxx
header-img: img/background/Clouds_landscapes_nature_trees.jpg
catalog: true
tags:
- kong
---



### 问题状况

使用 docker 直接部署 kong 的时候, 不会出现此问题, ipv4 和 ipv6 都会被监听

当使用 docker swarm 部署时, 只会监听 ipv6, 且 8000 和 8001 这两个端口可以 telnet 通

**而 8443 端口 telnet 不通.**

**BUT**

> 在我们阿里云的生产机器上部署, 此问题无法复现

### 配置文件

#### 单节点部署

单节点部署时, 直接使用 docker run 通过镜像启动一个容器即可

```bash
  docker run -d --name ${KONG_NAME} --network=${NETWORK_NAME} \\\\
    -v ${KONG_VOLUME_NAME}:${KONG_VOLUME_DIR} \\\\
    -e "KONG_DATABASE=postgres" \\\\
    -e "KONG_PG_HOST=${KONG_DATABASE_NAMES}" \\\\
    -e "KONG_PG_PASSWORD=kong" \\\\
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \\\\
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \\\\
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \\\\
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \\\\
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \\\\
    -e "KONG_ADMIN_GUI_URL=http://${HOST}" \\\\
    -p 8000:8000 \\\\
    -p 8443:8443 \\\\
    -p 8001:8001 \\\\
    -p 8444:8444 \\\\
    -p 8002:8002 \\\\
    -p 8445:8445 \\\\
    -p 8003:8003 \\\\
    -p 8004:8004 \\\\
    ${KONG_IMAGE_NAME}
}
```

#### docker swarm 部署

docker swarm 的部署文件截取了部署 kong 的一部分.

```yaml
  kong:
    image: ${KONG_IMAGE_NAME}
    restart: always
    networks:
      - kong-net
    volumes:
      - kong-data:/usr/local/kong
      - kong-cfg:/etc/kong
    deploy:
      replicas: 3
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kongSekorm
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    depends_on:
      - kong-migration
    links:
      - kong-database
    ports:
      - "8001:8001"
      - "8000:8000"
      - "8443:8443"
```

### 问题原因猜测

#### 系统内核版本

阿里云机器和测试内网机器的内核版本不一致, 通过升级内核, 再次部署.

**结果:**

> 部署后 8443 端口依旧无法访问

#### ipv6 的原因

一开始觉得传统通过镜像启动容器没问题，而通过 docker-compose 部署却又有问题。

通过`lsof -i:${port}`命令查看端口监听情况，发现 docker-compose 部署的没有监听 ipv4, 通过更改内核参数，禁止 ipv6 的转发，问题依旧复现


### yaml 配置文件

最后排查的是安装的脚本，在脚本中找到了这个参数`KONG_PROXY_LISTEN`, 便怀疑有问题。

最终查看 GitHub 一个 [issue](https://github.com/Kong/kong/issues/4181) 找到了以下的配置方法。
更改之后问题消失。

```yaml
KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl
KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
```

### 原因总结

通过查看 Kong GitHub 上的一个 [issue](https://github.com/Kong/kong/issues/4181) , 找到了正确的参数配置

yaml 文件中 `KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443` 参数是无效的，不正确的。
应该更换为 `KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl`


----

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

