---
title: 使用systemd部署podman的容器
date: 2025-07-20 09:57:01
tags: [podman, 容器, systemd]
categories: [后端]
---

# podman的问题

podman是一个容器运行时环境，提供与 Docker 非常相似的功能并不需要在你的系统上运行任何守护进程。没有守护进程就意味着如果你想使用podman部署容器，是没有办法使用类似docker的方式去实现自动重启功能。如果使用k8s部署容器那问题不大，如果就纯容器部署，可以使用podman给出的方案：

```bash
podman generate systemd --help
[DEPRECATED] Generate systemd units

Description:
  Generate systemd units for a pod or container.
  The generated units can later be controlled via systemctl(1).

DEPRECATED command:
It is recommended to use Quadlets for running containers and pods under systemd.

Please refer to podman-systemd.unit(5) for details.


Usage:
  podman generate systemd [options] {CONTAINER|POD}

Examples:
  podman generate systemd CTR
  podman generate systemd --new --time 10 CTR
  podman generate systemd --files --name POD

Options:
      --after stringArray         Add dependencies order to the generated unit file
      --container-prefix string   Systemd unit name prefix for containers (default "container")
  -e, --env stringArray           Set environment variables to the systemd unit files
  -f, --files                     Generate .service files instead of printing to stdout
      --format string             Print the created units in specified format (json)
  -n, --name                      Use container/pod names instead of IDs
      --new                       Create a new container or pod instead of starting an existing one
      --no-header                 Skip header generation
      --pod-prefix string         Systemd unit name prefix for pods (default "pod")
      --requires stringArray      Similar to wants, but declares stronger requirement dependencies
      --restart-policy string     Systemd restart-policy (default "on-failure")
      --restart-sec uint          Systemd restart-sec
      --separator string          Systemd unit name separator between name/id and prefix (default "-")
      --start-timeout uint        Start timeout override
      --stop-timeout uint         Stop timeout override (default 10)
      --template                  Make it a template file and use %i and %I specifiers. Working only for containers
      --wants stringArray         Add (weak) requirement dependencies to the generated unit file
```

# systemd

systemd是Linux系统的一套基本构建模块。它提供了一个系统和服务管理器，作为PID 1运行并启动系统的其余部分。systemd是在引导期间启动的第一个守护进程，也是在关闭期间终止的最后一个守护进程。利用这个特性，可以使用systemd去监控容器的运行状态

# 例子

这里使用[health-master](https://github.com/damingerdai/health-master)作为例子。

```bash
# 获取名为 health-master 的容器 ID
CONTAINER_ID=$(podman ps -a --filter "name=health-master" --format "{{.ID}}")

if [ -z "$CONTAINER_ID" ]; then
  echo "未找到名为 health-master 的容器。" >&2
  exit 1
fi

echo "找到容器 health-master，ID 为 $CONTAINER_ID"

# 定义服务文件的存储路径
SERVICE_FILE_PATH="$HOME/.config/systemd/user"

SERVICE_NAME="podman-health-master"

# 使用 podman generate systemd 创建服务文件
echo "正在生成 systemd 服务文件..."
podman generate systemd --name --restart-policy=on-failure -t 10 "$CONTAINER_ID" >"$SERVICE_FILE_PATH/$SERVICE_NAME.service"

if [ $? -ne 0 ]; then
  echo "错误: 无法生成 $SERVICE_NAME 服务。" >&2
  exit 1
fi

# 修复 SELinux 上下文（如启用 SELinux）
restorecon -RvF "$SERVICE_FILE_PATH/$SERVICE_NAME.service"

# 启用用户级 systemd 支持（首次需运行）
loginctl enable-linger "$USER"

# 重新加载 systemd 配置
echo "重新加载 systemd 配置..."
systemctl --user daemon-reload

if [ $? -ne 0 ]; then
  echo "错误: 无法重新加载 systemd 配置。" >&2
  exit 1
fi

# 启用服务
echo "启用服务 $SERVICE_NAME..."
systemctl --user enable --now "$SERVICE_NAME"

if [ $? -ne 0 ]; then
  echo "错误: 无法启用服务 $SERVICE_NAME。" >&2
  exit 1
fi

# 启动服务
echo "启动服务 $SERVICE_NAME..."
systemctl --user start "$SERVICE_NAME"

if [ $? -ne 0 ]; then
  echo "错误: 无法启动服务 $SERVICE_NAME。" >&2
  exit 1
fi

echo "服务 $SERVICE_NAME 已成功创建并启动。"
echo "你现在可以使用以下命令管理服务:"
echo "  systemctl --user status $SERVICE_NAME  # 查看服务状态"
echo "  systemctl --user stop $SERVICE_NAME    # 停止服务"
echo "  systemctl --user restart $SERVICE_NAME # 重启服务"
```

# 问题

podman generate systemd在podman 4.4的时候就已经弃用了，目前推荐使用[Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)

# 引用

1. [使用 Podman 以非 root 用户身份运行 Linux 容器](https://zhuanlan.zhihu.com/p/47706426)
2. [System and Service Manager](https://systemd.io/)
3. [Linux Systemd基础教程](https://segmentfault.com/a/1190000044854265)
4. [How to replace the "podman generate systemd" command since its deprecated](https://github.com/containers/podman/discussions/20218)