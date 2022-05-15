# drone + gogs + docker实现持续自动化部署
## 准备

我们使用docker-compose来安装gogs和drone。

首先我们准备数据库，gogs支持mysql，postgres, sqlite, sql server和TiDB。
这里我们选择使用postgre:

```yaml
version: '3'
services:
  postgres:
    image: postgres:14.2
    container_name: postgres
    ports:
      - 5432:5432
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 123456
      POSTGRES_DB: postgres
    networks:
      - ci-network
 
networks:
  ci-network:
    external: true
```

创建docker网络并启动数据库

```shell
docker network create ci-network (仅第一次运行)

docker-compose up postgres
```

## Gogs

`Gogs`是一款极易搭建的自助 Git 服务, 使用go开发，可以很方便的在任何go支持的平台上部署

### 安装

在docker-compose.yaml上添加gogs

```yaml
  gogs:
    image: gogs/gogs:0.12.6
    container_name: 'gogs'
    ports:
      - '10022:22'
      - '13000:3000'
    volumes:
      - ./gogs:/data
      - gogs/gogs
    depends_on:
      - postgres
    networks:
      - ci-network
```

gogs使用22端口提供ssh， 使用3000提供web页面服务，这里我们分别映射到本地的10022和13000端口

启动gogs服务：
```
docker-compose up gogs
```

使用浏览器打开[http://127.0.0.1:13000/install](http://127.0.0.1:13000/install)

![gogs的install页面](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-install-ui.png)

设置postgresql数据库，

![gogs的install的postgreql设置](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-install-ui-postgreq-settings.png)


设置应用基本设置

![gogs的install的应用设置](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-install-ui-application-settings.png)

跳过邮件服务设置， 关闭·启用验证码服务,

![gogs的install的邮件设置](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-install-ui-email-settings.png)

设置管理员账号,

![gogs的install的管理员设置](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-install-ui-admin-settings.png)

最后我们可以点击安装按钮了，浏览器将会跳转到`http://host.docker.internal:3000`, 但是我们其实不能访问这个链接，不过不用担心，可以使用[http://127.0.0.1:13000](http://127.0.0.1:13000)

### 配置ssh

在[http://127.0.0.1:13000/user/settings/ssh](http://127.0.0.1:13000/user/settings/ssh)页面上设置你的SSH 密钥：

![gogs的SSH密钥配置](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-ssh-settings.png)
![gogs的SSH密钥配置](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-ssh-settings-result.png)


### 创建仓库

![gogs的创建仓库](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-create-new-repo-1.png)
![gogs的创建仓库](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-create-new-repo-2.png)
![gogs的创建仓库](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-create-new-repo-3.png)

这里我使用我的[hoteler-web](https://github.com/damingerdai/hoteler-web)作为demo：

```
git clone https://github.com/damingerdai/hoteler-web.git
git remote add gogs ssh://git@localhost:10022/daming/hoteler-web.git
git push gogs master
```

![gogs的创建仓库](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-create-new-repo-4.png)

换一个目录, 把hoteler-web拉到本地，

```shell
git clone ssh://git@localhost:10022/daming/hoteler-web.git
```

![gogs的hoteler-web仓库](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-ui-hoteler-web-clone-termial.png)


## Drone

`Drone`是一个持续集成和持续交付的平台，可以与Docker完美集成。相对于`Jenkins`来说更加轻量，可以配合轻量的Gogs来实现持续集成

### 安装

```yaml
  drone-server:
    image: drone/drone:2.11
    container_name: drone
    ports:
      - 10081:80
      - 9000
    volumes:
      - ./drone:/var/lib/drone/
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
    environment:
      - DRONE_OPEN=true
      - DRONE_SERVER_HOST=drone-server
      - DRONE_SERVER_PROTO=http
      - DRONE_LOGS_TRACE=true
      - DRONE_LOGS_DEBUG=true
      - DRONE_GOGS=true
      - DRONE_GOGS_SKIP_VERIFY=false
      - DRONE_GOGS_SERVER=http://host.docker.internal:13000
      - DRONE_PROVIDER=gogs
      - DRONE_DATABASE_DATASOURCE=/var/lib/drone/drone.sqlite
      - DRONE_DATABASE_DRIVER=sqlite3
      - DRONE_RPC_SECRET=MWckgvhjqg4E3eQ0psgZX4iNCxoQiyU4LLvO4eXFFuHtrTkIy8vwcAc3erB5f9reM
      - DRONE_SECRET=ALQU2M0KdptXUdTPKcEw
      - DRONE_USER_CREATE=username:daming,admin:true
    networks:
      - ci-network
  drone-runner-docker:
    image: drone/drone-runner-docker:1.8.1
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./pipe/docker_engine://./pipe/docker_engine
    restart: always
    ports:
      - 13001:3000
    environment:
      - DRONE_RPC_HOST=drone-server
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_SECRET=MWckgvhjqg4E3eQ0psgZX4iNCxoQiyU4LLvO4eXFFuHtrTkIy8vwcAc3erB5f9reM
      - DRONE_LOGS_TRACE=true
      - DRONE_LOGS_DEBUG=true
      - DRONE_RUNNER_CAPCAITY=2
      - DRONE_RUNNER_NAME=docker-runner
      - DRONE_SECRET=ALQU2M0KdptXUdTPKcEw
    networks:
      - ci-network

```

然后执行

```zsh
docker-compose up drone-server drone-runner-docker
```

在浏览器中打开[http://127.0.0.1:10081](http://127.0.0.1:10081)。

drone没有自己的账号信息, 直接使用gogs的账号信息就好了

![drone登录页面](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/drone-ui-login.png)
![drone注册页面](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/drone-ui-register.png)

激活`hoteler-web`仓库

![drone的hoteler-web](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/drone-ui-hoteler-web-dashboard.png)
![drone的hoteler-web](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/drone-ui-hoteler-web-active-repository.png)

我们需要设置`Project Settings`中开启*Truste*配置：

![drone的hoteler-web](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/drone-ui-hoteler-web-active-repository-trusted.png)

### 配置drone.yml

```yml
kind: pipeline
name: hoteler-web

steps:
  - name: frontend
    image: node:16
    commands:
      - yarn install && yarn build
trigger:
  branch:
  - master
```

并提交到gogs仓库里。

### 配置gogs的wehook

在浏览器中进入[http://127.0.0.1:13000/daming/hoteler-web/settings/hooks/1](http://127.0.0.1:13000/daming/hoteler-web/settings/hooks/1)

我们把`http://drone-server/hook`改成`http://host.docker.internal:10081/hook`, 点击更新web钩子的按钮。

最后我们可以点击测试推送的按钮：

![drone的webhook测试](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/gogs-web-hook-test.png)

在drone我们可以看到：

![drone的webhook结果](https://raw.githubusercontent.com/damingerdai/damingerdai.github.io/master/assets/software/drone-ci-cd-result.png)

> 第一次ci/cd可能是网络问题，导致开启`node:16`镜像太慢了，上面的截图其实是第二次的结果


## 问题

1. drone无法clone的gogs的地址问题
在`gogs/gogs/conf/app.ini`中将*EXTERNAL_URL*修改为*http://host.docker.internal:13000/*（仅仅作为本地使用）

2. 容器无法访问localhost
windows或者mac用户可以使用host.docker.internal，其他系统尚未测试过。

## docker-compose.yaml

```yaml
version: '3'
services:
  postgres:
    image: postgres:14.2
    container_name: postgres
    ports:
      - 5432:5432
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 123456
      POSTGRES_DB: postgres
    networks:
      - ci-network
  gogs:
    image: gogs/gogs:0.12.6
    container_name: 'gogs'
    ports:
      - '10022:22'
      - '13000:3000'
    volumes:
      - ./gogs:/data
      - gogs/gogs
    depends_on:
      - postgres
    networks:
      - ci-network
  drone-server:
    image: drone/drone:2.11
    container_name: drone
    ports:
      - 10081:80
      - 9000
    volumes:
      - ./drone:/var/lib/drone/
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
    environment:
      - DRONE_OPEN=true
      - DRONE_SERVER_HOST=drone-server
      - DRONE_SERVER_PROTO=http
      - DRONE_LOGS_TRACE=true
      - DRONE_LOGS_DEBUG=true
      - DRONE_GOGS=true
      - DRONE_GOGS_SKIP_VERIFY=false
      - DRONE_GOGS_SERVER=http://host.docker.internal:13000
      - DRONE_PROVIDER=gogs
      - DRONE_DATABASE_DATASOURCE=/var/lib/drone/drone.sqlite
      - DRONE_DATABASE_DRIVER=sqlite3
      - DRONE_RPC_SECRET=MWckgvhjqg4E3eQ0psgZX4iNCxoQiyU4LLvO4eXFFuHtrTkIy8vwcAc3erB5f9reM
      - DRONE_SECRET=ALQU2M0KdptXUdTPKcEw
      - DRONE_USER_CREATE=username:daming,admin:true
    networks:
      - ci-network
  drone-runner-docker:
    image: drone/drone-runner-docker:1.8.1
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./pipe/docker_engine://./pipe/docker_engine
    restart: always
    ports:
      - 13001:3000
    environment:
      - DRONE_RPC_HOST=drone-server
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_SECRET=MWckgvhjqg4E3eQ0psgZX4iNCxoQiyU4LLvO4eXFFuHtrTkIy8vwcAc3erB5f9reM
      - DRONE_LOGS_TRACE=true
      - DRONE_LOGS_DEBUG=true
      - DRONE_RUNNER_CAPCAITY=2
      - DRONE_RUNNER_NAME=docker-runner
      - DRONE_SECRET=ALQU2M0KdptXUdTPKcEw
    networks:
      - ci-network

networks:
  ci-network:
    external: true
```

## 参考资料

1. [gogs官网](https://gogs.io)
2. [gogs安装与说明(docker)](https://www.cnblogs.com/shanfeng1000/p/14622319.html)
3. [Drone+Gogs+docker实现持续自动化部署](https://zhuanlan.zhihu.com/p/108842332)
4. [使用drone和gogs搭建自己的CI/CD系统](https://kanda.me/2019/03/06/building-ci-cd-system-by-drone/)
5. [Setup CI/CD service with gogs & registry & drone](https://zhuanlan.zhihu.com/p/81904099)
6. [Drone 教程](https://www.cnblogs.com/manastudent/p/15938616.html)
7. [轻量快速的CI工具Drone快速入门](https://my.oschina.net/keking/blog/3053769)