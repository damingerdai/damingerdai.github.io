---
title: 在k3s中配置私有镜像仓库
date: 2025-03-06 20:55:33
tags: [k3s, docker registry]
categories: [软件]
---

# 前言

本文的目的是实现在k3s中可以访问使用docker部署[registry](https://hub.docker.com/_/registry)。

# 部署registry

启动一个一次性容器用于创建账号密码.密码文件路径以/root/registry/htpasswd为例,账号密码以admin和12345678为例.

```bash
docker run --rm --entrypoint \
    htpasswd httpd:2 -Bbn \
    admin 12345678 > ./registry/htpasswd
```

编写docker compose的yaml文件用于启动registry。

```yaml
services:
  registry:
    image: registry:2
    container_name: registry
    volumes:
      # - ./config.yml:/etc/docker/registry/config.yml
      - ./htpasswd:/auth/htpasswd
      - ./registry:/var/lib/registry
      - /etc/localtime:/etc/localtime
    ports:
      - 5000:5000
    environment:
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
      - REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm
      - REGISTRY_STORAGE_DELETE_ENABLED=true
    networks:
      - registry-network
    restart: always
    extra_hosts:
      - host.docker.internal:host-gateway
networks:
  registry-network:
    external: false
```

resistry默认提供web ui使用，我们可以使用[docker-registry-ui](https://hub.docker.com/r/joxit/docker-registry-ui)部署一个web ui。

```yaml
registry-ui:
image: joxit/docker-registry-ui:main
container_name: registry-ui
restart: always
ports:
  - 5001:80
environment:
  - SINGLE_REGISTRY=true
  - REGISTRY_TITLE=Docker Registry UI
  - DELETE_IMAGES=true
  - SHOW_CONTENT_DIGEST=true
  - NGINX_PROXY_PASS_URL=http://registry:5000
  - SHOW_CATALOG_NB_TAGS=true
  - CATALOG_MIN_BRANCHES=1
  - CATALOG_MAX_BRANCHES=1
  - TAGLIST_PAGE_SIZE=100
  - REGISTRY_SECURED=false
  - CATALOG_ELEMENTS_LIMIT=1000
networks:
  - registry-network
extra_hosts:
  - host.docker.internal:host-gateway
```

然后通过·docker compose up -d·就可以启动了。

# docker连接私有仓库

docker默认不支持http协议，需要额外配置, 通过在/etc/docker/daemon.json 中将私有仓库地址添加进入就好了。

```json
{
  "insecure-registries": ["192.168.31.220:5000"]
}
```

然后可以通过docker login去登陆私有仓库。

```bash
docker login 192.168.31.220:5000
```

# k3s连接私有仓库

如果k3s使用docker运行时就不需要额外的操作了，就可以操作了应该（没试过）

如果k3s使用containerd运行时，需要自行配置/var/lib/rancher/k3s/agent/etc/containerd/config.toml。

一般而言不鼓励直接修改/var/lib/rancher/k3s/agent/etc/containerd/config.toml文件，而是通过 /etc/rancher/k3s/registries.yaml让k3s自行生成containerd的配置文件。

```bash
mirrors:
  "192.168.31.220:5000":
    endpoint:
      - "http://192.168.31.220:5000"

configs:
  "192.168.31.220:5000":
    auth:
      username: admin
      password: password
    tls:
      insecure_skip_verify: true
```

mirrors定义镜像仓库的镜像规则（mirror），用于指定如何访问特定的私有仓库。当容器运行时尝试拉取镜像时，会根据 mirrors 的配置决定从哪个地址拉取。
configs定义私有仓库的认证和 TLS 配置。当访问私有仓库时，容器运行时会使用这里配置的用户名、密码和 TLS 设置。

有的时候k3s不一定能够实时自动监听registries.yaml的改动，所以最好的方式还是重启k3s:

```bash
sudo systemctl restart k3s-agent   # worker 节点
# 或者
sudo systemctl restart k3s         # server 节点
```

现在可以通过创建一个pod去测试k3s是否正常访问私有仓库。

这里推荐使用crictl去直接拉取私有仓库的镜像。

```bash
sudo crictl pull 192.168.31.220:5000/hoteler-api:07e1b87e82ccfc49e5bee7d3d88cf2c304376056
```

# 参考资料

1. [Docker login 登录私服，报错； http: server gave HTTP response to HTTPS client](https://blog.csdn.net/tergou/article/details/120422445)
2. [Private Registry Configuration](https://docs.k3s.io/zh/installation/private-registry)

