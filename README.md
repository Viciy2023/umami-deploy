# umami-deploy

把 [Umami](https://github.com/umami-software/umami) 的官方镜像同步到两个 registry，供飞牛（FNOS）拉取：

| 目标 | 用途 |
| --- | --- |
| `ccy2026/viciy2023`（Docker Hub） | **飞牛实际拉取的那条** —— 走飞牛配的国内 mirror |
| `ghcr.io/viciy2023/umami`（GHCR） | 备用通道 |

## 为什么需要它

飞牛直连 `ghcr.io` 拉 umami 官方镜像会卡死：认证能通过（返回 401 是 registry 的正常未认证应答），但镜像层所在的主机 `pkg-containers.githubusercontent.com` 拉不动，`docker pull` 会长时间停在 `Waiting`。

飞牛上配的那 5 个加速源都是 **Docker Hub 的 mirror**（`registry-mirrors`），按 Docker 的设计只对 `docker.io` 生效，对 `ghcr.io` 无效 —— 所以 `postgres:16-alpine` 秒拉下来，`umami` 一动不动。

这台 runner 在美国，访问两个 registry 都快，所以由它定时搬运。

## 搬而不是建

Umami 官方已经在 `ghcr.io/umami-software/umami` 发布镜像，自己编译是重复劳动，而且要跑完整的 Next.js 16 + Prisma 7 构建链，既慢又容易失败。

`docker buildx imagetools create` 在两个 registry 之间直接复制 manifest，不落盘、保留多架构，复制出来的镜像与官方 digest 完全一致 —— 内容就是官方那份，中间没有引入差异的环节。

## 同步策略

- 每 6 小时检查一次上游镜像的 manifest digest（UTC 01/07/13/19，即北京 09/15/21/03 点）。
- 与 `last-sync.txt` 里的指纹比对，**只有变化才同步**，不会白跑。
- 每次同步打两个标签：`latest` 和一个日期标签 `YYYYMMDD`，留一个可以回滚的时间点。
- 上游用的是滚动标签 `postgresql-latest`（Umami v3 起不再发版本化 tag），所以 digest 比对是唯一可靠的变动检测方式。

## 需要的仓库 Secrets

| 名字 | 值 |
| --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub 用户名（`ccy2026`） |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token，权限 Read & Write |

没有配这两个时，workflow 只推 GHCR，不会失败。

## 在飞牛上使用

```bash
cd /vol1/1000/Docker/umami
docker compose pull && docker compose up -d
```

`docker-compose.yml` 里的镜像地址：

```yaml
image: ccy2026/viciy2023:latest
```

写成短名（不带 `docker.io/` 前缀）时 Docker 默认就是 Docker Hub，也就会走飞牛配的那几个加速源。数据库仍用 `postgres:16-alpine`。

升级和重装都只需要 `docker compose pull && docker compose up -d`：层缓存会保留，通常只拉变化的那几层。

## 手动触发

Actions 页面 → Sync Umami image → Run workflow。用于上游推送后想立刻同步，或想强制重跑一次。
