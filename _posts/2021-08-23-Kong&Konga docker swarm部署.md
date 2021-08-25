---
layout:     post
title:      Kong&Konga docker swarm 部署
subtitle:   附一键安装脚本
date:       2021-08-23
author:     JerAxxxxxxx
header-img: img/background/abstract_digital_art_Iron_Man.jpg
catalog: true
tags:
- kong
---

#### Kong docker swarm 部署

这次分享下 Kong&Konga 通过 docker swarm 部署的脚本，因为目前没有用上 K8S，目前还是用比较轻量的 docker swarm 进行部署。

```shell
#!/bin/bash
# kong镜像名称,需带 tag, 若不带 tag 会默认查找 latest 版本
export KONG_IMAGE_NAME="kong:jerax"
# konga 镜像名称
export KONGA_IMAGE_NAME="konga:jerax"
# postgres 镜像名称
export POSTGRES_IMAGE_NAME="postgres:9.6"
# 本机 ip
export HOST_IP="127.0.0.1"
# 当前文件夹
SHELL_FOLDER=$(
  cd "$(dirname "$0")" || exit
  pwd
)

function build_yaml() {
  echo -e "\033[32m Start building docker-compose.yaml files \033[0m"
  cat <<EOF >"${SHELL_FOLDER}"/docker-compose.yaml
version: "3.8"

networks:
 kong-net:
  driver: overlay

services:
  kong-database:
    image: ${POSTGRES_IMAGE_NAME}
    restart: always
    networks:
      - kong-net
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongSekorm
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 5s
      timeout: 5s
      retries: 5

  kong-migration:
    image: ${KONG_IMAGE_NAME}
    command: "kong migrations bootstrap"
    networks:
      - kong-net
    restart: on-failure
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-database
      - KONG_PG_DATABASE=kong
      - KONG_PG_PASSWORD=kongSekorm
    links:
      - kong-database
    depends_on:
      - kong-database

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
      KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl
      KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
    depends_on:
      - kong-migration
    links:
      - kong-database
    ports:
      - "8001:8001"
      - "8000:8000"
      - "8443:8443"
      - "8444:8444"

  konga-prepare:
    image: ${KONGA_IMAGE_NAME}
    command: "-c prepare -a postgres -u postgresql://kong:kongSekorm@kong-database:5432/konga"
    networks:
      - kong-net
    restart: on-failure
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-database
      - KONG_PG_DATABASE=konga
      - KONG_PG_PASSWORD=kongSekorm
    links:
      - kong-database
    depends_on:
      - kong-database

  konga:
    image: ${KONGA_IMAGE_NAME}
    restart: always
    networks:
     - kong-net
    environment:
      DB_ADAPTER: postgres
      DB_URI: postgresql://kong:kongSekorm@kong-database:5432/konga
      NODE_ENV: production
    links:
      - kong-database
    depends_on:
      - kong
      - konga-prepare
    ports:
      - "1337:1337"

volumes:
  postgres-data:
  kong-data:
  kong-cfg:
EOF
  echo -e "\033[32m Yaml file build complete \033[0m"
}

function start_kong() {
  # 初始化 docker swarm
  result=$(docker info)
  if [[ ${result} =~ "Swarm: active" ]]; then
    echo -e "\033[31m Docker swarm already active \033[0m"
  else
    echo -e "\033[32m Start init docker swarm \033[0m"
    docker swarm init --advertise-addr "${HOST_IP}"
  fi
  echo -e "\033[32m Start building the Kong service... \033[0m"
  docker stack deploy -c docker-compose.yaml kong
}

build_yaml &&
  start_kong
```

#### docker-compose.yaml
由于博客上 shell 的格式对于 yaml 文件不太友好，重新把 docker-compose.yaml 文件内容贴一下
```yaml
version: "3.8"

networks:
 kong-net:
  driver: overlay

services:
  kong-database:
    image: ${POSTGRES_IMAGE_NAME}
    restart: always
    networks:
      - kong-net
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongSekorm
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 5s
      timeout: 5s
      retries: 5

  kong-migration:
    image: ${KONG_IMAGE_NAME}
    command: "kong migrations bootstrap"
    networks:
      - kong-net
    restart: on-failure
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-database
      - KONG_PG_DATABASE=kong
      - KONG_PG_PASSWORD=kongSekorm
    links:
      - kong-database
    depends_on:
      - kong-database

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
      KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl
      KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
    depends_on:
      - kong-migration
    links:
      - kong-database
    ports:
      - "8001:8001"
      - "8000:8000"
      - "8443:8443"
      - "8444:8444"

  konga-prepare:
    image: ${KONGA_IMAGE_NAME}
    command: "-c prepare -a postgres -u postgresql://kong:kongSekorm@kong-database:5432/konga"
    networks:
      - kong-net
    restart: on-failure
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-database
      - KONG_PG_DATABASE=konga
      - KONG_PG_PASSWORD=kongSekorm
    links:
      - kong-database
    depends_on:
      - kong-database

  konga:
    image: ${KONGA_IMAGE_NAME}
    restart: always
    networks:
     - kong-net
    environment:
      DB_ADAPTER: postgres
      DB_URI: postgresql://kong:kongSekorm@kong-database:5432/konga
      NODE_ENV: production
    links:
      - kong-database
    depends_on:
      - kong
      - konga-prepare
    ports:
      - "1337:1337"

volumes:
  postgres-data:
  kong-data:
  kong-cfg:
```


----

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。

