---
title: 个人博客搭建教程
date: 2026-09-08
draft: false
description: 从 0 到 1 用 Hugo + Stack 主题 + GitHub Pages 搭建免费个人博客的完整记录，Windows 环境全程可复制命令
image: cover.jpg
categories:
  - 教程
tags:
  - Hugo
  - GitHub Pages
  - Stack
toc: true
---

> 本文记录我从 0 到 1 搭建这个博客的完整过程：**Hugo 生成静态页面，GitHub Pages 免费托管，push 即自动上线**。不需要服务器，不需要花钱，全程在 Windows 下操作。

## 为什么是 Hugo + GitHub Pages

市面上静态博客方案很多（Hexo、Hugo、Astro 系的 Fuwari……），我最终选了 Hugo + Stack 主题 + GitHub Pages，理由很直接：

| 需求 | 这套方案的回答 |
|---|---|
| 免费 | GitHub Pages 托管完全免费，还自带 HTTPS 和 CDN |
| 不想买服务器 | 纯静态网站，GitHub 全权托管 |
| 不想手动构建 | GitHub Actions 自动构建，`git push` 后约 1 分钟网站自动更新 |
| 要能长期写下去 | 文章就是 `content/post/` 下的 Markdown 文件，主题随时可换，内容永远是自己的 |

对比我考虑过的 Fuwari（Astro）：颜值确实高，但它需要 fork 整个仓库改源码，主题升级要手动合并冲突，而且每次改完都得手动 `pnpm build`。Hugo 把「内容 / 配置 / 主题」三层分得很开，长期维护省心得多。

## 一、安装本地环境

Windows 下用系统自带的 `winget` 三条命令搞定（PowerShell 执行）：

```powershell
winget install --id Git.Git -e
```

```powershell
winget install --id Hugo.Hugo.Extended -e
```

```powershell
winget install --id GoLang.Go -e
```

几个要点：

- **必须是 Hugo extended 版本**。Stack 主题用了 SCSS，普通版会报
  `TOCSS: failed to transform "scss/style.scss"`，这是我踩的第一个坑
- Go 是给 Hugo Modules 下载主题用的；如果你的主题用 git submodule 方式安装（本文就是），Go 可以不装
- 装完**重开终端**让环境变量生效，然后验证：

```powershell
hugo version
```

输出里必须带 `+extended` 字样：

```text
hugo v0.164.0+extended windows/amd64 BuildDate=...
```

## 二、创建仓库并开启 GitHub Pages

1. 登录 GitHub，新建一个仓库，名字必须是 **`<你的用户名>.github.io`**（比如我的就是 `DrStone-youqiong.github.io`），这样最终网址就是干净的 `https://<你的用户名>.github.io/`
2. 仓库保持 **Public**——免费账户的 Pages 只能托管公开仓库
3. 进入仓库 **Settings → Pages → Build and deployment → Source**，选择 **GitHub Actions**（这一步最容易漏，不选的话 push 了也不会部署）

## 三、初始化 Hugo 站点并安装主题

本地挑个目录：

```powershell
hugo new site myblog
cd myblog
git init
```

把 Stack 主题作为 git 子模块加进来（子模块的好处：主题和你自己的内容分开版本管理，升级主题不影响你的修改）：

```powershell
git submodule add https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack
```

然后在站点根目录写配置 `config.yaml`，最核心的几个字段：

```yaml
baseURL: "https://<你的用户名>.github.io/"
languageCode: "zh-cn"
theme: "hugo-theme-stack"
title: "你的博客名"

params:
  mainSections:
    - post                  # 首页展示 content/post 下的文章
  sidebar:
    subtitle: "一句话签名"
    avatar: img/avatar.png  # 对应 assets/img/avatar.png
  comments:
    enabled: false          # 评论系统，玩熟了再开
```

> **TIP**：`baseURL` 结尾的 `/` 不能少，后面很多坑都出在这个字段上。

## 四、本地预览

```powershell
hugo server
```

浏览器打开 `http://localhost:1313/` 就能看了。这个命令开着不用动，改配置、写文章、保存，页面自动热更新；`Ctrl + C` 停止。

两个常用变体：

```powershell
hugo server -D     # 把 draft: true 的草稿也渲染出来
hugo server --disableFastRender    # 页面没变化时强制完整重渲染
```

## 五、写文章

文章都放在 `content/post/` 下，推荐用**页面包（page bundle）**结构，文章和图片放一个文件夹里管理：

```text
content/post/
└── my-first-post/
    ├── index.md      # 文章本体，必须叫 index.md
    └── cover.jpg     # 封面图、正文配图
```

`index.md` 开头的 front matter 是文章的元信息：

```markdown
---
title: 文章标题
date: 2026-09-08
draft: false
description: 摘要，显示在列表页
categories:
  - 分类
tags:
  - 标签一
  - 标签二
---

正文用 Markdown 随便写。
```

注意 `draft: false`——**草稿状态的文章线上不会显示**，这是「本地看得到、线上找不到」的高频原因。`date` 也别写成未来时间，Hugo 默认不渲染未到期的文章。

## 六、自动部署：push 即上线

在仓库 `.github/workflows/hugo.yml` 放一个部署工作流，核心内容：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]     # push 到 main 就触发
  workflow_dispatch:       # 也允许在 Actions 页面手动触发

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.164.0   # 与本地版本保持一致
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive   # 关键：把主题子模块一起拉下来
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Hugo
        run: |
          hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

它做的事：每次 push，GitHub 云端服务器装好同版本 Hugo → 拉取你的仓库（含主题子模块）→ 构建出纯静态 HTML → 发布到 Pages。**本地只是写作环境，构建永远发生在云端。**

以后写文章的日常就三连：

```powershell
git add .
git commit -m "post: 新文章"
git push
```

到仓库的 **Actions** 标签页看进度：黄点是构建中，绿勾就是上线了，整个过程一分多钟。

## 七、踩坑记录

**① 白屏 / 样式全丢、图片 404**
九成是 `baseURL` 不对。用 `<用户名>.github.io` 之外的仓库名时，必须写成 `https://<用户名>.github.io/<仓库名>/`。

**② 本地预览正常，线上文章 404**
检查 `draft` 是不是 `false`、`date` 是否写成了未来时间。

**③ Actions 构建红叉**
点进对应那次运行看日志，报错一般写得很直白，最常见的还是配置文件 YAML 语法问题（少引号、缩进错）。

**④ 仓库越吃越大**
`hugo` 构建产物 `public/` 和 `resources/_gen/` 不要提交进仓库，加进 `.gitignore`：

```gitignore
/public/
/resources/_gen/
.hugo_build.lock
```

如果之前已经提交过，用 `git rm -r --cached public` 从跟踪中移除（本地文件保留）。

## 结语

整套方案搭完，写作体验就是：开 `hugo server`，写 Markdown，`git push`，一分钟上线。没有服务器要维护，没有数据库要备份，文章永远是纯文本文件——换电脑、换主题、甚至换托管平台，把文件夹带走就行。

希望这篇能帮到同样想拥有自己小站的你。
