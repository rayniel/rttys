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

### 使用 APISIX 的 HTTPS 访问 PVE 等内网 Web 服务

PVE 登录时，浏览器需要保存并发送 `PVEAuthCookie`。PVE 的 Web 界面会将这个
Cookie 标记为 `Secure`；如果浏览器打开的转发地址是 HTTP，登录后可能出现
`no ticket`。即使 rtty 到 PVE 使用 HTTPS，浏览器到代理的 HTTP 也不会变成 HTTPS。
应让浏览器访问的 **rttys 管理入口和 Web 代理入口都使用 HTTPS**。

下面是一个使用两个子域名、共同使用外部 443 端口的示例：

```text
浏览器 -- HTTPS --> APISIX -- HTTP --> rttys 管理端口 5913
浏览器 -- HTTPS --> APISIX -- HTTP --> rttys Web 代理端口 5914
                                       |
                                       +-- rtty 设备连接 --> 内网 PVE HTTPS :8006
```

无需为 Web 代理额外开放外部端口；APISIX 根据请求的域名选择上游。APISIX 的
证书和 PVE 的证书分别用于不同连接，不需要相同。APISIX 到 rttys 的 HTTP
连接仍是明文，应限制在可信网络；rtty 设备与 rttys 的连接也应单独配置 TLS
并限制访问。PVE 目标连接的 TLS 能力和证书验证行为取决于所用 rtty 客户端及
其配置，不能由 APISIX 的证书代替。

1. 在 DNS 中将 `rtty.example.com` 和 `proxy.example.com` 都指向 APISIX，
   并为两个域名配置浏览器信任的证书。将示例域名替换成自己控制的域名。

2. 在 APISIX 创建两条 HTTPS 路由，分别按 Host 匹配，并将所有路径转发到
   对应上游：

   | 外部入口 | 上游 | 用途 |
   | --- | --- | --- |
   | `https://rtty.example.com` | rttys 的 `5913` 端口 | 管理界面和 API |
   | `https://proxy.example.com` | rttys 的 `5914` 端口 | 设备 Web 服务 |

   上游地址应填写 **APISIX 能访问到的 rttys 地址**。若 APISIX 和 rttys
   在同一主机、且 APISIX 能访问主机回环地址，可以使用 `127.0.0.1:5913`
   和 `127.0.0.1:5914`；若 APISIX 在独立容器中，容器内的 `127.0.0.1`
   指向 APISIX 自身，应改用共享 Docker 网络中的服务名和容器端口，或使用
   可达的主机地址。为两条路由启用 WebSocket 支持；不要在代理路由前加
   路径前缀或剥除目标服务所需的路径。不要将这两个入口简单地放在同一域名的
   不同路径下，目标服务可能使用根路径及绝对 API 路径。

3. 在 `compose.yaml` 的 `rttys` 服务 `command` 列表中，保留已有参数并
   **追加**以下项目：

   ```yaml
      - --http-proxy-redir-url
      - https://proxy.example.com
      - --http-proxy-redir-domain
      - example.com
   ```

   第一个参数指定浏览器跳转到的 Web 代理 HTTPS 地址；第二个参数让管理入口
   设置的 `rtty-http-sid` Cookie 可发送给代理子域名。两个入口必须位于该
   Cookie 域名之下，不能填写协议或端口。建议为这两个入口使用专门控制的父域名：
   该域下其他子域名也可能收到此 Cookie，不要与不可信服务共享。若两个入口
   使用同一主机名、不同端口，则可省略 Cookie 域名参数，但需要另行区分端口。
   如 APISIX 可被不可信客户端访问，请在管理入口删除外部传入的
   `HttpProxyRedir` 和 `HttpProxyRedirDomain` 请求头，避免它们覆盖上述
   服务端配置；不要通过客户端提供的值决定跳转地址或 Cookie 域名。

4. 如果 APISIX 在主机上并通过回环地址连接，可将 Compose 中的 `5913`、
   `5914` 端口映射改为 `"127.0.0.1:5913:5913"` 和
   `"127.0.0.1:5914:5914"`。如果 APISIX 在容器中，不要直接照搬此
   回环绑定：使用共享 Docker 网络并确保服务可达，必要时移除这两个端口的
   主机发布。设备连接端口 `5912` 和临时分享端口的开放策略需要单独处理。
   修改后运行 `docker compose up -d` 重建服务。

5. 使用 `https://rtty.example.com` 登录管理界面，选择在线设备的 Web
   转发，协议选择 `https`，目标填写 **rtty 设备可访问的** PVE 内网 IPv4
   地址和端口 `8006`。成功跳转后，浏览器地址应以
   `https://proxy.example.com` 开头，再登录 PVE。

若仍出现 `no ticket`，使用浏览器开发者工具检查登录请求是否成功，以及后续
请求是否携带 `PVEAuthCookie`；查看浏览器是否提示 Cookie 被阻止。不要公开
Cookie 或登录票据的值。若出现 rttys 代理未授权，检查跳转地址、Cookie
域名、浏览器是否发送 `rtty-http-sid`，以及 APISIX 是否正确转发两个路由。
若代理页面无法连接 PVE，检查 rtty 设备到内网地址的连通性、HTTPS 支持和
证书配置。不要通过移除 Cookie 的 `Secure` 属性来绕过问题。

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
