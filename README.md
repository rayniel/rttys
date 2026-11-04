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
