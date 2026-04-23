# GitLab Runner Docker Compose 部署

这个目录提供了一套可直接落地的 GitLab Runner Docker Compose 配置，默认使用 Docker executor，适合快速部署一个自托管 Runner。

默认特性：

- 配置持久化到 `./config`
- 构建缓存持久化到 `./cache`
- 通过宿主机 `/var/run/docker.sock` 调度 Job 容器
- 提供一次性注册服务 `gitlab-runner-register`
- 提供 `config.template.toml` 作为注册时的默认 Runner 模板

## 目录结构

```text
gitlab-runner/
├── .env.example
├── README.md
├── cache/
├── config/
│   └── config.template.toml
└── docker-compose.yml
```

注册完成后，GitLab Runner 会自动生成：

- `config/config.toml`
- `config/.runner_system_id`

## 前置条件

- 已安装 Docker Engine 和 Docker Compose 插件
- 宿主机可以访问你的 GitLab 地址
- 你已经在 GitLab 中创建 Runner，并拿到了 token

说明：

- 推荐使用 Runner authentication token，也就是通常以 `glrt-` 开头的 token
- 截至 2026-04-23，GitLab 官方仍保留 registration token 的兼容注册方式，但它已经是 deprecated，计划在 GitLab 20.0 移除

## 快速开始

### 1. 准备环境变量

```bash
cd /root/my-deploy-env/gitlab-runner
cp .env.example .env
```

修改 `.env` 中至少这几个值：

- `CI_SERVER_URL`：你的 GitLab 地址，例如 `https://gitlab.example.com`
- `RUNNER_NAME`：Runner 名称
- `RUNNER_TOKEN`：推荐使用，填写 GitLab 创建 Runner 后得到的认证 token

如果你的 GitLab Web 地址是 `https://...`，但 Runner 日志里实际下发的仓库地址却是 `http://...git`，建议额外设置：

- `RUNNER_CLONE_URL`：例如 `https://gitlab.example.com`

这个参数可以避免 Job 在拉取私有仓库源码时因为 `http -> https` 重定向而失败。

如果你的 GitLab 还在使用旧的 registration token，则：

- 清空 `RUNNER_TOKEN`
- 填写 `RUNNER_REGISTRATION_TOKEN`
- 根据需要修改 `RUNNER_TAG_LIST`、`RUNNER_RUN_UNTAGGED`、`RUNNER_LOCKED`

### 2. 如果 GitLab 使用自签证书

把 CA 证书放到下面这个路径，然后再执行注册：

```text
gitlab-runner/config/certs/ca.crt
```

如果你希望按主机名区分，也可以放到：

```text
gitlab-runner/config/certs/<gitlab-host>.crt
```

放好之后重启容器即可生效。

### 3. 注册 Runner

```bash
docker compose --profile register run --rm gitlab-runner-register
```

注册成功后，会在 `config/config.toml` 中写入实际 Runner 配置。

### 4. 启动 Runner

```bash
docker compose up -d gitlab-runner
```

### 5. 验证 Runner

```bash
docker compose exec gitlab-runner gitlab-runner verify
docker compose exec gitlab-runner gitlab-runner list
```

如果 GitLab 项目里的 Job 配置了 `tags`，记得让 Runner 标签和 Job 标签一致，否则不会被调度。

## 常用命令

查看日志：

```bash
docker compose logs -f gitlab-runner
```

重启 Runner：

```bash
docker compose restart gitlab-runner
```

停止并删除容器：

```bash
docker compose down
```

升级镜像并重建：

```bash
docker compose pull
docker compose up -d gitlab-runner
```

## 验证 Runner 接单

仓库根目录已经提供了一个示例文件：

- `../.gitlab-ci.yml`

这个示例包含：

- `workflow: rules`：只在“Merge Request 管道”或“Git tag 管道”时触发
- `.mr-job`：MR 专用 job 模板
- `.tag-job`：Git tag 专用 job 模板
- `mr-action-*`：只会在 Merge Request 管道中执行
- `tag-action-*`：只会在 Git tag 管道中执行

使用方式：

1. 把根目录的 `.gitlab-ci.yml` 提交到你的 GitLab 项目
2. 如果你的 Runner 在 GitLab 页面里配置了标签，并且没有开启 `run untagged`，把 `.gitlab-ci.yml` 顶部注释掉的 `tags` 打开，并改成和 Runner 一致的标签
3. 推送代码，触发一条新的 Pipeline
4. 在 GitLab Web 页面查看 Job 状态，或在 Runner 主机上执行下面命令看日志

```bash
cd /root/my-deploy-env/gitlab-runner
docker compose logs -f gitlab-runner
```

如果一切正常，你会看到：

- 创建或更新 Merge Request 时，只会运行 `mr-action-*`
- 推送 Git tag 时，只会运行 `tag-action-*`
- 两类动作彼此隔离，不会串着跑

说明：

- 现在你的 Runner `concurrent = 10`，因此同一时间最多可并行处理 10 个 Job
- `.gitlab-ci.yml` 里的 `tags:` 是 Runner 调度标签，和 Git 仓库的版本 tag 不是一回事
- 现在这份示例会在创建或更新 Merge Request 时触发 MR 动作，也会在推送 Git tag 时触发 tag 动作
- 你可以直接把 `mr-action-*` 和 `tag-action-*` 里的 `script` 替换成你自己的实际命令
- 如果 Pipeline 一直处于 `pending`，优先检查 Job 标签和 Runner 标签是否一致
- 如果 Job 被 Runner 接到后失败，再根据日志定位镜像、网络、脚本或权限问题

重新注册：

```bash
rm -f ./config/config.toml ./config/.runner_system_id
docker compose --profile register run --rm gitlab-runner-register
docker compose up -d gitlab-runner
```

## 环境变量说明

| 变量 | 作用 | 默认值 |
| :--- | :--- | :--- |
| `TZ` | 容器时区 | `Asia/Shanghai` |
| `GITLAB_RUNNER_IMAGE` | Runner 镜像 | `gitlab/gitlab-runner:alpine` |
| `CI_SERVER_URL` | GitLab 地址 | `https://gitlab.com` |
| `RUNNER_CLONE_URL` | 可选，强制 Runner 使用指定地址拉取仓库 | 空 |
| `RUNNER_NAME` | Runner 名称 | `docker-runner` |
| `RUNNER_EXECUTOR` | 执行器类型 | `docker` |
| `RUNNER_DOCKER_IMAGE` | Job 默认基础镜像 | `alpine:3.20` |
| `RUNNER_TOKEN` | 推荐使用的认证 token | 空 |
| `RUNNER_REGISTRATION_TOKEN` | 旧版注册 token，仅兼容使用 | 空 |
| `RUNNER_TAG_LIST` | 旧版 token 注册时写入的标签 | `docker` |
| `RUNNER_RUN_UNTAGGED` | 旧版 token 是否接收无标签任务 | `true` |
| `RUNNER_LOCKED` | 旧版 token 注册后是否锁定到当前项目 | `false` |
| `RUNNER_ACCESS_LEVEL` | 旧版 token 的访问级别 | `not_protected` |

## 模板配置说明

`config/config.template.toml` 会在注册时注入默认 Runner 配置，目前包含：

- `FF_NETWORK_PER_BUILD=1`
- `request_concurrency = 10`
- Docker executor 默认镜像 `alpine:3.20`
- `pull_policy = if-not-present`
- 持久化缓存目录 `/cache`

注意：

- 这个模板只在“注册时”生效，不会自动覆盖已经生成的 `config/config.toml`
- 如果你后续改了模板，需要重新注册，或者直接编辑 `config/config.toml` 后重启容器

## 如果 Job 里需要执行 Docker 命令

当前配置已经把宿主机的 `/var/run/docker.sock` 挂到 Runner 容器中，Runner 自己可以用它来创建 Job 容器。

但如果你的 CI Job 内部也要直接执行 `docker build`、`docker push` 之类的命令，你通常还需要二选一：

- 方案一：在 `config/config.toml` 的 `[runners.docker]` 里增加 `"/var/run/docker.sock:/var/run/docker.sock"` 挂载，让 Job 容器也能访问宿主机 Docker
- 方案二：改成 Docker-in-Docker，并把 `privileged = true`

如果只是普通的编译、测试、打包，当前默认配置通常已经够用。

## 安全提醒

`/var/run/docker.sock` 暴露给 Runner 后，等同于把宿主机 Docker 管理权限交给这个 Runner。只建议在可信项目、可信仓库和可信成员环境中使用。

## 参考文档

- GitLab 官方文档: https://docs.gitlab.com/runner/install/docker/
- GitLab Runner 注册: https://docs.gitlab.com/runner/register/
