---
title: Hexo + GitHub Pages 从零搭博客（附踩坑记录）
date: 2026-10-31 23:30:00
tags:
  - Hexo
  - GitHub
  - 博客
  - 教程
categories:
  - 折腾
---

## 前言

记录一次从零搭建博客的全过程：Windows 11 + Hexo + GitHub Actions + GitHub Pages。
全程免费、不需要服务器，也不需要买域名（默认域名 `用户名.github.io`）。

这篇文章既是教程，也是我自己的操作笔记，包含几个真实踩过的坑。

---

## 一、方案选型

```
本地写 Markdown
      │  git push
      ▼
GitHub 仓库（main 分支，Hexo 源码）
      │  GitHub Actions 自动构建
      ▼
public/ 静态文件 ──> GitHub Pages ──> https://用户名.github.io
```

- **Hexo**：Node.js 的静态博客框架，主题多、插件多。
- **GitHub Actions**：推送后自动 `hexo generate` 并发布，本地不用保存 token。
- **GitHub Pages**：免费托管静态网站。

替代方案：Hugo（更快）、VitePress（偏文档）、WordPress（要服务器）。选 Hexo 主要是生态成熟、中文资料多。

---

## 二、环境准备

### 2.1 安装 Node.js 和 Git

```powershell
winget install --id OpenJS.NodeJS.LTS -e --accept-source-agreements --accept-package-agreements
winget install --id Git.Git -e --accept-source-agreements --accept-package-agreements
```

装完**必须关掉终端重新打开**，再验证：

```powershell
node -v
npm -v
git --version
```

### 2.2 换 npm 镜像

```powershell
npm config set registry https://registry.npmmirror.com
npm config get registry
```

### 2.3 安装 Hexo CLI

```powershell
npm install -g hexo-cli
hexo -v
```

> **踩坑 1：装完仍提示“不是内部或外部命令”**
>
> 原因：已经打开的 PowerShell 窗口不会重新读取系统 PATH。
> 解决：关掉窗口重开；或在当前窗口手动刷新：
>
> ```powershell
> $env:Path = [Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [Environment]::GetEnvironmentVariable("Path","User")
> ```

> **踩坑 2：`npm warn install-scripts ... hexo-util`**
>
> npm 11 默认禁止依赖执行安装脚本。`hexo-util` 的 `postinstall` 用于预编译代码高亮数据。
> 放行：
>
> ```powershell
> npm config set allow-scripts=hexo-util --location=user
> npm install -g hexo-cli
> ```

---

## 三、创建站点

```powershell
cd $env:USERPROFILE
hexo init blog
cd blog
npm install
```

预览：

```powershell
hexo server
# http://localhost:4000
```

目录结构：

```
blog/
├── _config.yml          # 站点配置
├── package.json
├── scaffolds/           # 文章模板
├── source/
│   ├── _posts/          # 文章 Markdown
│   └── about/
├── public/              # 构建产物（不提交）
└── .github/workflows/   # GitHub Actions
```

---

## 四、站点配置

编辑 `_config.yml`：

```yaml
title: 我的博客
subtitle: ''
description: ''
author: 你的名字
language: zh-CN
timezone: Asia/Shanghai

# 用户主页（仓库名 用户名.github.io）
url: https://用户名.github.io
root: /
```

注意事项：

- 冒号后必须有空格；
- 缩进只能用空格，不能用 Tab；
- 仓库若是 `blog` 这种项目页，`url` 要带 `/blog`，`root` 要写 `/blog/`，否则样式和图片全部 404。

---

## 五、部署到 GitHub

### 5.1 仓库命名

| 仓库名 | 网站地址 |
| --- | --- |
| `用户名.github.io` | `https://用户名.github.io` |
| `blog` | `https://用户名.github.io/blog` |

必须是 **Public**，创建时不要勾选 README / .gitignore / license。

### 5.2 GitHub Actions 工作流

`.github/workflows/pages.yml`：

```yaml
name: Deploy Hexo to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npx hexo generate

      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 5.3 提交与推送

`.gitignore` 至少包含：

```
node_modules/
public/
.deploy_git/
db.json
*.log
```

```powershell
git init -b main
git add .
git commit -m "init: hexo blog"
git remote add origin https://github.com/用户名/用户名.github.io.git
git push -u origin main
```

> **踩坑 3：`Author identity unknown`**
>
> Git 第一次使用要设置提交者身份：
>
> ```powershell
> git config --global user.name "你的用户名"
> git config --global user.email "用户名@users.noreply.github.com"
> ```

> **踩坑 4：`LF will be replaced by CRLF`**
>
> Windows 换行符提示，不是错误，忽略即可。

### 5.4 开启 Pages

仓库 → **Settings** → **Pages** → **Build and deployment** → **Source** 选 **GitHub Actions**。

然后到 **Actions** 看 `Deploy Hexo to GitHub Pages` 是否变绿。

> **踩坑 5：Actions 里多出一个红色的 `pages build and deployment`**
>
> 这是 GitHub 对 `用户名.github.io` 仓库自动启用的**旧版 Jekyll 构建**，它把仓库根目录当静态站点，而根目录是 Hexo 源码，所以必然失败。
> 只要把 Pages 的 Source 改成 **GitHub Actions**，这个流程就不会再跑；历史失败记录会保留，但不影响网站。
>
> 判断标准：真正干活的是 `Deploy Hexo to GitHub Pages`，它绿了网站就是好的。

---

## 六、日常写作

写文章的本质：**在 `source/_posts/` 放一个带 front-matter 的 `.md`，提交到 `main`**。不一定非要用命令行。

### 方式 A：GitHub 网页（最省事）

1. 仓库 → `source` → `_posts` → **Add file** → **Create new file**
2. 文件名用英文，如 `hello-hexo.md`（会变成 URL 的一部分）
3. 写内容：

```markdown
---
title: 文章标题
date: 2026-11-01 10:00:00
tags:
  - 标签
categories:
  - 分类
---

正文……
```

4. **Commit changes** → Actions 自动发布

### 方式 B：GitHub Desktop

克隆仓库 → 本地用任意编辑器写 `.md` → Commit → Push。全程点鼠标。

### 方式 C：命令行

```powershell
cd $env:USERPROFILE\blog
hexo new "文章标题"
# 编辑 source\_posts\文章标题.md
hexo server     # 本地预览
git add .
git commit -m "post: 文章标题"
git push
```

### 插入图片

图片放 `source/images/`，文章里引用：

```markdown
![说明](/images/photo.png)
```

---

## 七、常见问题速查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `hexo` 不是命令 | PATH 未生效 | 重开终端，或刷新 PATH |
| `npm install` 很慢 / 超时 | 没换镜像 | `npm config set registry https://registry.npmmirror.com` |
| Actions `npm ci` 失败 | 没提交 `package-lock.json` | 本地 `npm install` 后提交 lock 文件 |
| `Get Pages site failed` | Pages Source 不是 Actions | Settings → Pages 改过来后 Re-run |
| 网站 404 | 仓库名 / url / root 不匹配 | 对照第五节检查 |
| 页面无样式 | `root` 写错 | 用户主页 `/`，项目页 `/仓库名/` |
| 中文乱码 | 编码或 language | 文件存 UTF-8，`language: zh-CN` |
| 图片不显示 | 路径不对 | 用 `/images/xxx.png` 绝对路径 |

---

## 八、参考

- Hexo 文档：<https://hexo.io/zh-cn/docs/>
- GitHub Pages：<https://docs.github.com/zh/pages>
- GitHub Actions 部署 Pages：<https://docs.github.com/zh/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site>

---

## 总结

整套流程的关键只有三点：

1. **仓库根目录放 Hexo 源码**，构建产物 `public/` 不进仓库；
2. **Pages 的 Source 必须是 GitHub Actions**，否则会跑旧版 Jekyll 构建；
3. **写文章 = 往 `source/_posts/` 加 Markdown 并 push**，命令行只是其中一种方式。

从零到上线大约 40 分钟，之后每篇文章的成本就是写 Markdown + push。
