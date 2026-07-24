# 自动化部署

本项目通过 GitHub Actions 将 `master` 分支的提交部署到生产服务器。工作流定义在 [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml)。

```text
git push origin master
        |
        v
GitHub Actions
        |
        v
GitHub Runner：打包指定提交并通过 SSH 上传到服务器
        |
        v
服务器：切换发布目录、构建 Docker 镜像、重启 Compose 服务、执行 HTTP 健康检查
```

## 部署前提

服务器应具备以下目录和服务：

```text
/data/apps/toonflow-app/
  app/                    当前运行版本的源码目录
  docker-compose.yml      生产 Compose 配置
  volumes/data/           持久化应用数据
  releases/               已部署、失败和待发布的源码归档
```

工作流假定 `app/Dockerfile.production` 为服务器维护的生产 Dockerfile，不会删除它。`volumes/data/` 会挂载到容器的 `/app/data`，因此其中的项目数据、上传文件和本地配置应纳入备份策略。

## GitHub Secrets

在仓库的 **Settings → Secrets and variables → Actions** 中添加以下 Secrets：

| Secret | 说明 |
| --- | --- |
| `DEPLOY_SSH_HOST` | 服务器 IP 或域名。 |
| `DEPLOY_SSH_PORT` | SSH 端口，通常为 `22`。 |
| `DEPLOY_SSH_USER` | 专用部署账号。 |
| `DEPLOY_SSH_KEY` | 部署私钥的完整内容。 |

不要将 SSH 密码写入 Actions Secrets、工作流、仓库文件或命令历史。建议创建仅用于部署的 SSH 密钥，并为其设置最小权限。

工作流通过固定的 `known_hosts` 条目校验服务器的 ED25519 主机公钥。该公钥是公开的身份标识，不属于 Secret；服务器更换 SSH 主机密钥时，需要在工作流中同步更新。可在可信网络中使用以下命令核对：

```bash
ssh-keyscan -t ed25519 <server-host> | ssh-keygen -lf - -E sha256
```

## 服务器初始化

在部署服务器创建专用账号和部署密钥的公钥授权。账号需要读取 `/data/apps/toonflow-app`、运行该目录的 Docker Compose 命令，并访问 Docker socket。不要将日常登录账号或密码复用于 CI。

首次使用前，在服务器上确认：

```bash
cd /data/apps/toonflow-app
docker compose config --quiet
docker compose ps
curl --fail http://127.0.0.1:10588/
```

## 安全机制与失败处理

部署工作流会在下列情况中止并恢复上一版本：

- 服务器缺少待上传的源码归档或 `Dockerfile.production`；
- Docker Compose 构建或启动失败；
- Docker Compose 启动后 60 秒内未通过本机 HTTP 健康检查。

工作流由 GitHub Runner 下载提交并通过 SCP 上传，因此服务器不需要访问 GitHub。失败后，先通过 Actions 日志确认原因；工作流会保留失败版本和上一版本在 `releases/` 目录中，便于排查与手动回滚。

## 日常发布流程

```bash
yarn lint
yarn build
git add .
git commit -m "描述本次修改"
git push origin master
```

推送后在 GitHub Actions 中查看 `Deploy production` 的执行结果。若只需要重新部署当前 `master`，可从 Actions 页面手动运行该工作流。
