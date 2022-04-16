# 准备

请在github的[设置页面](https://github.com/settings/tokens)上创建一个token，并确保有以下的权限：

- repo
- read:packages
- write:packages

> 请保存好该token，因为github将隐藏该值

在github的仓库的secrets设置页面(例：https://github.com/{your_username}/{your_repository_name}/settings/secrets/actions)里创建一个名为GHCR_PAT的secrets

![secrets设置页面](./../../assets/back-end/github-action-push-docker-image-to-github-packages-1.png 'secrets设置页面')

# 创建GitHub Action文件

在`.github/workflow`中创建一个`build-and-publish.yml`:


```yml
name: Build and Publish

on:
  push:
    branches: [ master ]

jobs:
  build-and-push-docker-image:
    name: Build Docker image and push to repositories
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to Github Packages
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_PAT }}

      - name: Build image and push to GitHub Container Registry
        uses: docker/build-push-action@v2
        id: docker_build
        with:
          context: .
          tags: |
            # 将usernmae和repository改成你自己的github账号和仓库名
            ghcr.io/{usernmae}/{repository}:${{ github.sha }}
            # 当且只有运行在master分支时才需要推送到github容器仓库
          push: ${{ github.ref == 'refs/heads/master' }}

      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

默认推送的镜像是私有，想要公开，可以在
`https://github.com/users/{your_username}/packages/container/{your_repository_name}/settings`下面的Danger Zone中的Change package visibility设置成*public*还是*private*。

# 参考资料

1. [How to build and push Docker image with GitHub actions?](https://event-driven.io/en/how_to_buid_and_push_docker_image_with_github_actions/)
2. [Hoteler](https://github.com/damingerdai/hoteler/blob/master/.github/workflows/build-and-publish.yml)