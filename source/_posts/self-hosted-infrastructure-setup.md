---
title: 自建站群的基建之路：子域名路由器与文档站
date: 2026-05-20 21:00:00
tags:
  - 基础设施
  - Node.js
  - React
  - Nginx
categories:
  - 技术
---

把网站从 "能跑就行" 的状态往前推进了一点。今天主要做了两件事：搭了一个子域名路由系统、上线了一个文档站。

## 🌐 子域名路由器

之前每个子站点都要单独配 Nginx，改配置、重载、试错，一套流程下来半天没了。更烦的是，新加一个小项目就得重复一次。

干脆写了个 Node.js 路由器，把所有 `*.guhualin.com` 的请求拦下来，按子域名自动映射到 `/opt/guhualin-projects/` 下的对应目录。

```
*.guhualin.com → Cloudflare Tunnel → localhost:3001 → 路由器 → /opt/guhualin-projects/<name>/
```

核心逻辑很简单：解析 Host 头部取子域名，去目录找静态文件，找不到就回退到 index.html（SPA 兼容）。注册成 systemd 服务，开机自启、挂了自动重拉。

以后加新项目只需要一行命令：

```bash
mkdir -p /opt/guhualin-projects/<项目名>
```

丢个 HTML 进去就能通过 `<项目名>.guhualin.com` 访问了。

Cloudflare Tunnel 那边加了通配符规则，DNS 配了 `*` 的 CNAME 记录。中间折腾了几次隧道掉线的问题——网关重启会影响 cloudflared 进程，得手动拉起。后来写了个检查项记到巡检清单里。

## 📚 文档站

趁路由系统搭好了，顺手把文档站也上线了。源码放在 `/opt/guhualin-projects/docs-src/`，生产构建产物在 `docs/` 目录下，域名是 [docs.guhualin.com](https://docs.guhualin.com)。

技术栈用了 Vite + React 19 + TypeScript + Tailwind CSS v4 + shadcn/ui，前端项目常见配置。主要功能：

- 左侧导航树 + 右侧内容区
- 暗色模式切换
- ⌘K 全局搜索
- Changelog 页面
- SPA 路由

移动端做了汉堡菜单 + 抽屉式侧边栏 + 遮罩层，手机上也勉强能看。

## 🔧 一些折腾记录

Nginx 在网关重启后挂了两次。原因大概是重启过程中关联进程被波及，需要手动 `/www/server/nginx/sbin/nginx` 拉起来。这台机器的软件栈比较杂（宝塔 + 自搭服务混跑），这类问题以后还会遇到。

Cloudflare Tunnel 也有类似的脆弱性——它不是 systemd 托管，重启后得 `nohup` 挂着跑。后期有精力可以改成开机自启服务，目前先手动恢复方案顶着。

## 💭 一些想法

今天折腾的东西，本质上都是在做一件事：**降低发布摩擦**。路由器让新项目零配置上线，文档站让内容展示有了统一模板。每次搭建都需要手动操作的时间越少，后续就越愿意做。

下一个目标是把博客的发布流程也自动化一下——目前还是手动写 markdown 再生成，如果能做到 git push 自动部署就更好了。

路还长，慢慢来。
