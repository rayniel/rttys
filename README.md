# rttys

## Docker image distribution

This fork distributes unmodified source from `zhaojh329/rttys` to
`ghcr.io/rayniel/rttys`, for both `linux/amd64` and `linux/arm64`.
The `master` branch tracks upstream; this workflow and its documentation live
on the `docker-images` default branch so the scheduled workflow can run without
adding distribution changes to `master`.

| Tag | Source |
| --- | --- |
| `latest` | Latest upstream non-prerelease release |
| `5.6.0` (example) | Corresponding upstream release tag, without the `v` prefix |
| `edge` | Latest upstream `master` commit |
| `edge-<full commit SHA>` | Corresponding upstream `master` commit |

```sh
docker pull ghcr.io/rayniel/rttys:latest
docker pull ghcr.io/rayniel/rttys:edge
```

### Docker Compose 部署

使用 Docker Engine 和 Docker Compose v2 或更新版本，无需数据库。以下文件位于
`docker-images` 分支，不会合并到跟踪上游的 `master`。

```sh
git clone --branch docker-images --single-branch https://github.com/rayniel/rttys.git
cd rttys
cp .env.example .env
chmod 600 .env
openssl rand -hex 32
```

编辑 `.env`，将占位值替换为自己的凭据：

```dotenv
RTTYS_TOKEN=替换为上面生成的随机字符串
RTTYS_PASSWORD=替换为独立的强密码
```

`RTTYS_TOKEN` 是设备连接凭据，服务端不会自动生成；每台 rtty 客户端需设置相同
token。`RTTYS_PASSWORD` 是网页管理密码，不是设备 token。两者为空或未设置时，
Compose 会拒绝启动，但示例占位值仍需手动替换。建议使用随机十六进制值；
若密码包含 `$` 等特殊字符，请按 Compose `.env` 语法使用单引号包裹。
`.env` 已加入 `.gitignore`，不要公开或提交真实凭据。凭据会通过启动参数传入，
主机管理员仍可通过容器检查读取；不要公开 `docker compose config` 的展开结果，
也不要启用会输出完整配置的 debug 日志。

```sh
docker compose up -d
docker compose ps
docker compose logs -f rttys
```

浏览器访问 `http://服务器地址:5913`，使用 `RTTYS_PASSWORD` 登录。
[`compose.yaml`](compose.yaml) 默认发布以下 TCP 端口，且绑定所有主机网络接口：

| 端口 | 用途 |
| --- | --- |
| `5912` | rtty 设备连接 |
| `5913` | Web 管理界面 |
| `5914` | 访问设备 Web 服务的 HTTP 代理，已固定监听端口以适配 Docker |
| `20000-21000` | 临时 SSH / TCP 分享，内外端口保持一致 |

根据需要配置云安全组和主机防火墙，不要将不需要的端口暴露到公网。
Docker 发布端口可能绕过部分主机防火墙规则，应在 Docker 对应的过滤链或云安全组中限制来源。
若不使用临时分享，删除 `20000-21000` 映射即可关闭其外部访问；
若只需少量分享，可同步缩小端口映射和 `--share-port-start` / `--share-port-end` 范围。
Docker 内网或反向代理后使用分享时，可添加 `--share-public-host` 参数指定客户端可达的主机名。
TCP 分享没有额外分享密码，目标服务需要自行提供认证。

此示例没有启用 TLS，不应通过不可信网络直接传输 token 或登录凭据。
公网部署应为 Web 和 HTTP 代理配置 HTTPS 反向代理（终端连接需支持 WebSocket），
并为设备连接配置 TLS：挂载证书文件并添加 `--sslcert`、`--sslkey` 参数，
客户端同步启用 TLS。使用反向代理时可将 Web / HTTP 代理映射改为
`127.0.0.1:5913:5913` 和 `127.0.0.1:5914:5914`；HTTP 代理的对外地址可通过
`--http-proxy-redir-url` 配置。完整参数见 [`deploy/rttys.conf`](deploy/rttys.conf)。

命名卷 `rttys-data` 挂载到 `/var/lib/rttys`。首次创建 SSH 分享时自动生成的
SSH 主机密钥保存在该卷，重建容器后会复用；token 和网页密码则来自主机上的 `.env`。
这不是数据库，也不保存在线会话或临时分享，容器重启后这些内存状态会清空。

更新镜像并重建容器：

```sh
docker compose pull
docker compose up -d
```

默认使用正式版 `latest`；使用开发版时将 `image` 改为
`ghcr.io/rayniel/rttys:edge`，固定版本可用 `:5.6.0`，精确固定构建产物需使用镜像 digest。
`pull_policy: always` 会在启动时检查镜像，但不会自动更新正在运行的容器。
停止服务使用 `docker compose down`，默认保留命名卷；不要使用 `down -v`，除非确定要删除
持久化密钥。迁移时需保存 `.env` 并备份该卷。

The **Distribute upstream Docker images** workflow runs daily at 03:23 UTC
and can also be started using **Actions > Run workflow**. It resolves upstream
refs to exact commits before building with the upstream Dockerfile. No fork
sync or Docker Hub credentials are required; publishing uses `GITHUB_TOKEN`
with `packages: write`. The fork's own code changes are not included in these
images. Daily builds also refresh dependencies allowed by the upstream Dockerfile;
use the published digest to pin an exact image.

Enable Actions on this fork if GitHub has disabled them. After the first
successful publication, set the `rttys` package visibility to **Public** in its
GitHub package settings to allow anonymous pulls. GitHub creates new GHCR
packages as private by default. Public repositories' scheduled workflows may
be disabled after 60 days without repository activity; re-enable the workflow
if needed. Changes to upstream build paths or release tag conventions may
require updating this workflow.

[1]: https://img.shields.io/badge/license-MIT-brightgreen.svg?style=plastic
[2]: /LICENSE
[3]: https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=plastic
[4]: https://github.com/zhaojh329/rttys/pulls
[5]: https://img.shields.io/badge/Issues-welcome-brightgreen.svg?style=plastic
[6]: https://github.com/zhaojh329/rttys/issues/new
[7]: https://img.shields.io/badge/release-5.6.0-blue.svg?style=plastic
[8]: https://github.com/zhaojh329/rttys/releases
[9]: https://github.com/zhaojh329/rttys/workflows/build/badge.svg
[10]: https://img.shields.io/github/downloads/zhaojh329/rttys/total
[12]: https://deepwiki.com/badge.svg
[13]: https://deepwiki.com/zhaojh329/rttys
[14]: https://goreportcard.com/badge/github.com/zhaojh329/rttys/v5
[15]: https://goreportcard.com/report/github.com/zhaojh329/rttys/v5

[![license][1]][2]
[![PRs Welcome][3]][4]
[![Issue Welcome][5]][6]
[![Release Version][7]][8]
![Build Status][9]
[![Go Report Card][14]][15]
![Downloads][10]
[![Ask DeepWiki][12]][13]

> **🔗 This is the server component of the rtty project. For complete information please visit the main [rtty client repository](https://github.com/zhaojh329/rtty).**

![](/img/terminal.gif)
![](/img/file.gif)
![](/img/web.gif)
![](/img/virtual-keyboard.jpg)

## 📖 About

**rttys** is the server-side component of the [rtty](https://github.com/zhaojh329/rtty) remote terminal system. It provides a web-based management interface and handles connections from rtty clients running on remote devices.

## 🏗️ Project Structure

This repository contains only the server-side components:
- **Server binary**: The main rttys server program
- **Web UI**: Browser-based management interface
- **API**: RESTful API for device management
```

## ⭐ Star History
[![Star History Chart](https://api.star-history.com/svg?repos=zhaojh329/rttys&type=Date)](https://www.star-history.com/#zhaojh329/rttys&Date)

## 🤝 Contributing
If you would like to help making [rttys](https://github.com/zhaojh329/rttys) better,
see the [CONTRIBUTING.md](https://github.com/zhaojh329/rttys/blob/master/CONTRIBUTING.md) file.

## ❤️ [Donation](https://zhaojh329.github.io/zhaojh329/)
