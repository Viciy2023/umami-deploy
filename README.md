# umami-deploy

把 [Umami](https://github.com/umami-software/umami) 的官方镜像搬到自己的 GHCR 命名空间，供飞牛（FNOS）拉取。

## 为什么需要它

飞牛直连 `ghcr.io` 拉 umami 官方镜像会卡死：认证能通过（返回 401 是 registry 的正常未认证应答），但镜像层所在的主机 `pkg-containers.githubusercontent.com` 拉不动，`docker pull` 会长时间停在 `Waiting`，30 秒零进度。

飞牛上配置的那 5 个加速源都是 **Docker Hub 的 mirror**（`registry-mirrors`），按 Docker 的设计只对 `docker.io` 生效，对 `ghcr.io` 无效 —— 所以 `postgres:16-alpine` 秒拉下来，`umami` 一动不动。

这台 runner 在美国，访问 `ghcr.io` 很快，所以由它定时把官方镜像复制一份到 `ghcr.io/viciy2023/umami`，FNOS 从这里拉。同一条路已经验证可用：`ghcr.io/viciy2023/address`、`ghcr.io/viciy2023/wb2panel-deploy` 都是这么发的，现在都跑在飞牛上。

## 搬而不是建

Umami 官方已经在 `ghcr.io/umami-software/umami` 发布镜像，自己编译是重复劳动，而且要跑完整的 Next.js 16 + Prisma 7 构建链，既慢又容易失败。

`docker buildx imagetools create` 在两个 registry 之间直接复制 manifest，不落盘、保留多架构，复制出来的镜像与官方 digest 完全一致 —— 内容就是官方那份，中间没有引入差异的环节。

## 同步策略

- 每 6 小时检查一次上游镜像的 manifest digest（UTC 01/07/13/19，即北京 09/15/21/03 点）。
- 与 `last-sync.txt` 里的指纹比对，**只有变化才同步**，不会每 6 小时白跑一次。
- 同步时打两个标签：`latest` 和一个日期标签 `YYYYMMDD`，留一个可以回滚的时间点。
- 上游用的是滚动标签 `postgresql-latest`（Umami v3 起不再发版本化 tag），所以 digest 比对是唯一可靠的变动检测方式。

## 在飞牛上使用

```bash
cd /vol1/1000/Docker/umami
docker compose pull && docker compose up -d
```

`docker-compose.yml` 里的镜像地址写成：

```yaml
image: ghcr.io/viciy2023/umami:latest
```

数据库仍然用 `postgres:16-alpine`，它走 Docker Hub mirror，本来就拉得动。

## 首次使用要确认包是公开的

GHCR 的包可见性跟随仓库。本仓库是 public，推送后包通常也是 public；如果飞牛拉取报 `denied`，到仓库右侧 **Packages → umami → Package settings** 把可见性改成 public，或者在飞牛上执行一次：

```bash
echo <你的_GITHUB_TOKEN> | docker login ghcr.io -u Viciy2023 --password-stdin
```

## 手动触发

Actions 页面 → Sync Umami image → Run workflow。用于上游推送后想立刻同步，或想强制重跑一次。
