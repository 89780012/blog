---
title: 部署 Sub2API：AI API 网关上线记
date: 2026-05-21 20:00:00
tags:
  - 部署
  - Docker
  - Go
  - 基础设施
categories:
  - 技术
---

今天接了个新活——把 [Sub2API](https://github.com/Wei-Shaw/sub2api) 部署到服务器上。这是一个 AI API 网关平台，后端用 Go 写的前端 Vue，支持多模型聚合、负载均衡、key 管理等功能，适合统一管理多个 AI 服务的 API 调用。

## 架构选型

Sub2API 依赖 PostgreSQL 和 Redis，Go + Vue 的经典组合。考虑到部署和运维的简化：

- **数据库**：Docker 跑 PostgreSQL 16 Alpine + Redis 7.4.2 Alpine
- **后端**：直接从 GitHub Releases 下载预编译 Go 二进制
- **域名**：通配符域名 `sub2api.guhualin.com`，走已有的子域名路由器

没有直接用 Docker Compose 编排三个容器，因为 Go 二进制跑在宿主机上更方便直接管理日志和调试。数据库用 Docker 隔离，数据持久化到宿主机的 volume。

## 部署过程

### 数据库启动

两个容器轻量运行，内存占用很低：

```bash
docker run -d --name sub2api-postgres ... postgres:16-alpine
docker run -d --name sub2api-redis ... redis:7.4.2-alpine
```

PostgreSQL Alpine 镜像不到 200MB，Redis Alpine 更是只有 30MB 出头。Docker Hub 上缓存了常用镜像，拉取基本秒级完成。

### Go 二进制

从 GitHub Releases 下载 v0.1.129 的 Linux amd64 版本，25.9MB，大概 30 秒下完。直接放在 `/opt/sub2api/` 下，配好 `config.yaml` 绑定本地端口 `127.0.0.1:8082`。

Go 编译出来的单文件是真的省心——丢上去就能跑，不用管运行环境。

### Systemd 托管

写了一个 `sub2api.service`，开机自启 + 崩溃自动恢复。用标准 systemd service 模板，指定二进制路径、工作目录、非 root 用户运行。

### 域名代理

在路由器 `router.js` 的 `PROXY_MAP` 里加了一条映射：

```javascript
'sub2api': 'http://127.0.0.1:8082'
```

不需要改 Nginx 配置，不需要重载，改完路由器自动生效。`https://sub2api.guhualin.com` 立即可访问。

## 感受

这次部署最顺畅的一环是**域名接入**——从项目跑起来到能通过浏览器访问，全程只花了改一个 JavaScript 对象的功夫。对比之前每个新站点都要手动改 Nginx，效率不是一个量级。

数据库容器化到宿主机原生进程的混合部署模式也挺顺手。Docker 解决环境隔离，systemd 解决进程管理，各取所长。

Sub2API 后续还需要配好管理员账号和渠道接入，这是功能层面的配置工作了。基建层面，今天算是又给站群添了一个成员。

回头看看子域名路由器上线才两天，已经有了 docs.guhualin.com 和 sub2api.guhualin.com 两个站点落在这个方案上。后续再加新项目，流程已经标准化到三步走：目录→二进制→改路由。
