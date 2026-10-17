---
layout: post
title: "把博客部署到 GitHub Pages：BG6HMA 的建站记录"
date: 2026-10-17 23:50:00 +0800
categories: 折腾
---

这是 **BG6HMA** 的新博客，第一件正经事，就是把这套站点是怎么搭起来的记下来。

## 用到的三样东西

1. **域名** `bg6hma.top` —— 注册在阿里云，DNS 托管在 Cloudflare。
2. **GitHub 仓库** —— 公开仓库 `ntlaoshi/blog`，用来放站点源码。
3. **Jekyll** —— GitHub Pages 原生支持，不用本地装环境。

## 搭建步骤

### 1. 新建仓库

在 GitHub 上建一个 public 仓库，名字叫 `blog`。

### 2. 放三个文件

```
_config.yml                    站点配置
index.md                       首页
_posts/2026-10-17-hello.md     第一篇文章
```

`_config.yml` 里指定主题：

```yaml
title: BG6HMA 的博客
description: BG6HMA 的折腾记录与随笔
url: "https://blog.bg6hma.top"
theme: minima
lang: zh-CN
author: BG6HMA
```

### 3. 开启 GitHub Pages

仓库 **Settings → Pages**：

- Source 选 **Deploy from a branch**
- Branch 选 **main**，目录 **/ (root)**，保存

### 4. 绑定自己的域名

在 Cloudflare 里加一条记录：

| 类型 | 名称 | 内容 | 代理状态 |
|---|---|---|---|
| CNAME | `blog` | `ntlaoshi.github.io` | **仅 DNS（灰云）** |

然后在 Pages 设置里填 **Custom domain**：`blog.bg6hma.top`，保存。

> 这里特意用了子域名 `blog.`，因为根域名 `bg6hma.top` 上还挂着邮件用的 MX 记录，拿根域名做 Pages 要加一堆 A 记录，容易互相影响。

### 5. 等证书并开启 HTTPS

GitHub 会做 DNS 检查，检查通过后自动签发证书。等 `Enforce HTTPS` 从灰色变成可勾选，勾上就完事。

## 踩过的坑

- **Cloudflare 的橙云必须关掉**。开了代理之后，GitHub 的证书校验和回源都会出问题，保持「仅 DNS」最稳。
- **第一次访问 404 是正常的**。Pages 首次构建要一两分钟，构建完成后刷新就好。
- **证书不是立刻有的**。DNS 检查通过后通常几分钟到半小时，期间只能先用 HTTP。

## 以后怎么发文章

在仓库里新建 `_posts/YYYY-MM-DD-标题.md`，开头写上：

```yaml
---
layout: post
title: "文章标题"
date: 2026-10-17 23:50:00 +0800
categories: 折腾
---
```

提交后等一两分钟，GitHub 会自动构建发布。

---

> **说明**：本文由 AI 生成，经 BG6HMA 校对后发布。

73 de **BG6HMA**。
