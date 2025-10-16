# 辅助工具

这是助教工具仓库 [ZJU-OS/tool](https://github.com/ZJU-OS/tool) 的文档。

## 容器镜像

- 软件包只要发行版有并且能用，就用发行版的。
- 源码也从发行版获取。

### 多平台镜像构建

课程需要支持使用 amd64 和 arm64 平台的学生完成实验，因此使用 [Multi-arch build and images, the simple way | Docker](https://www.docker.com/blog/multi-arch-build-and-images-the-simple-way/) 构建多平台镜像，流程如下：

- 分别在 arm64 和 amd64 平台上执行

    ```shell
    make build-image
    make push-image
    ```

- 然后在其中一个地方执行

    ```shell
    make push-multi
    ```

    拉取在另一个平台上构建的镜像，使用 `docker manifest` 合成 `latest` 描述文件并推送。

相比用 Tag 区分不同架构，这么折腾一番可以让 Docker 自动匹配到对应架构，镜像名不用指定标签。

理论上来说，使用 Docker buildx 的 [Multi-platform builds](https://docs.docker.com/build/building/multi-platform/) 就能在一个平台上完成所有构建，更加简洁。但是：

- 笔者没有配置成功
- 跨架构运行实在是太慢了

但 Manifest 是 Docker 的实验性功能，有时候创建 `latest` 并不覆盖，挺奇怪的。

## GitLab CI 评测

`ZJU-OS/tool` 仓库下的 `zjugit-ci` 包含了评测 Runner 容器的配置，并且用 `Makefile` 包装了常用命令：

```shell
# （初次运行）注册 Runner
make TOKEN= register
# 启动 Runner
make
```

- 初次运行时，参考 [Tutorial: Create, register, and run your own project runner | GitLab Docs](https://docs.gitlab.com/tutorials/create_register_first_runner/#create-and-register-a-project-runner) 获得 Token。
- 我们使用 [Docker Executor](https://docs.gitlab.com/runner/executors/docker/)，[Run your CI/CD jobs in Docker containers | GitLab Docs](https://docs.gitlab.com/ci/docker/using_docker_images/)

    相关流程总结：

    - Prepare：
        - 创建和启动 service
        - 使用默认 Entrypoint 启动容器，然后 Attach 上去
    - Pre-job：克隆仓库、缓存和 artifact 处理
    - Job：
        - 将 `before_script`、`script`、`after_script` 组合成一个脚本
        - 通过 `stdin` 发送到容器 Shell 执行，并接收输出

    - CI 仓库会被放置在 `/build/<path>` 下，例如 `/build/os/fa25/jijiangming/os-123456`。

如何调试 CI？

- 在 `scripts` 加个 `sleep 1d`，然后去宿主机 attach 到容器
- 使用 `docker inspect` 查看容器的详细信息
