---
title: 从零搭一个 Hexo 博客：记录我踩过的四个坑
date: 2026-11-27 21:30:00
categories:
  - 折腾记录
tags:
  - Hexo
  - GitHub Pages
  - Butterfly
description: 用 Hexo + Butterfly 主题 + GitHub Actions 搭一个免费博客的全过程，以及四个真实踩到的坑和解决办法。
---

一直想有个自己的地方放笔记，试过印象笔记、飞书文档，最后发现写完就再也不看了。想想还是搭个博客，至少公开写出来，会有一点「不能写得太烂」的压力。

这篇记录一下从零搭起来的过程。技术栈很普通：**Hexo 生成静态页 + Butterfly 主题 + GitHub Pages 托管 + GitHub Actions 自动构建**，全程零成本，不买服务器也不用备案。

<!-- more -->

## 为什么是这套组合

说白了就三个原因：

1. **免费**。GitHub Pages 托管静态站点不要钱，仓库本身就是备份。
2. **纯文本**。文章都是 Markdown，哪天想搬家，拎着一堆 `.md` 文件走就行，不用担心平台倒闭。
3. **写作和发布分离**。本地写好、预览没问题再推，不用对着网页编辑器写长文。

## 第一步：环境

需要的东西就三样：

```bash
git --version    # 版本控制，推送代码用
node -v          # Hexo 跑在 Node 上
npm -v           # 装 Hexo 和插件
```

然后全局装 Hexo 的命令行工具：

```bash
npm install -g hexo-cli
hexo -v
```

## 坑一：装完了，但 `hexo: command not found`

装的过程很顺利，`added 54 packages in 6s`，结果一敲 `hexo -v` 直接报 `command not found`。

**原因**：我为了让 C 盘干净，在 `.npmrc` 里把 npm 的全局安装目录改到了 E 盘：

```ini
prefix=E:\APP\tools\nodejs\node_global
cache=E:\APP\tools\nodejs\node_cache
registry=https://registry.npmmirror.com
```

包确实装到了 `E:\APP\tools\nodejs\node_global`，但系统 PATH 里登记的还是 npm 的默认目录 `C:\Users\用户名\AppData\Roaming\npm`——而且那个目录根本不存在。两个地方对不上，系统自然找不到 `hexo` 这个命令。

**解决**：把真正的全局目录加进用户 PATH，然后**关掉终端重开**（PATH 是进程启动时读的快照，老窗口改了也不认）。

```powershell
[Environment]::SetEnvironmentVariable(
  "Path",
  [Environment]::GetEnvironmentVariable("Path","User") + ";E:\APP\tools\nodejs\node_global",
  "User"
)
```

顺便记住一条通用原则：**prefix 指到哪，PATH 就要包含哪个目录**。以后换 pnpm、yarn，或者用 pip 装命令行工具，都是同一个道理。

## 坑二：`hexo init` 报 `EPERM: mkdir 'E:\'`

这一步最离谱。命令是这样的：

```bash
hexo init
```

输出：

```
INFO  Cloning hexo-starter https://github.com/hexojs/hexo-starter.git
fatal: unable to access '...': Recv failure: Connection was reset
WARN  git clone failed. Copying data instead
FATAL Something's wrong...
Error: EPERM: operation not permitted, mkdir 'E:\'
```

拆开看是两件事：

- **第一个是网络**。`hexo init` 本质是去 GitHub 克隆 `hexo-starter` 模板仓库，国内直连经常被重置。不过这不算致命——Hexo 自带一份同样的模板，克隆失败会自动改成「复制本地数据」。
- **第二个才是真凶**。它复制模板时，把目标目录算成了 **E 盘根目录**，去创建 `E:\`，Windows 当然不允许，于是 `EPERM`。

我单独验证过这个现象：目录明明存在，但对盘符根做 `mkdir` 也会报错。

```js
fs.statSync('E:\\')                    // ok，是目录
fs.mkdirSync('E:\\', {recursive: true}) // EPERM
```

**解决**：绕开这一步。先把模板在 C 盘临时目录生成好，再手工复制到目标目录，然后装依赖：

```bash
npm install
```

站点结构就齐了。顺带一提，`hexo generate` 和 `hexo server` 都正常，只有 `init` 会踩这个坑。

还有个衍生坑：**不要在博客目录的子目录里再执行 `hexo init`**。Hexo 会向上逐级找 `_config.yml`，找到父目录的配置后就会判定「目标目录不是空的」直接报错。

## 坑三：换了 Butterfly 主题，页面白屏或报错

默认的 landscape 主题太素，换成 Butterfly：

```bash
npm install hexo-theme-butterfly --save
```

然后把 `_config.yml` 里的 `theme` 改成 `butterfly`。

但 Butterfly 是用 **Pug 和 Stylus** 写的，Hexo 默认只装了 EJS 和 Markdown 渲染器，必须补上这两个，否则生成直接失败：

```bash
npm install hexo-renderer-pug hexo-renderer-stylus --save
```

主题配置单独放在根目录的 `_config.butterfly.yml`，不要直接改 `node_modules` 里的文件——升级主题会被覆盖。

## 坑四：搜索框点开没反应

配置里写了：

```yaml
search:
  use: local
```

结果页面上的搜索永远是空的。翻主题源码才发现，这个字段要填的是**插件名**，不是简写：

```yaml
search:
  use: local_search
```

同时还要装搜索数据生成器（否则没有 `search.xml` 可查）：

```bash
npm install hexo-generator-searchdb --save
```

以及在 `_config.yml` 里告诉它输出 xml：

```yaml
search:
  path: search.xml
  field: post
  content: true
  format: html
```

## 部署：交给 GitHub Actions

Hexo 传统做法是装 `hexo-deployer-git`，把生成的 `public` 推到另一个分支。但更省心的是让 GitHub 自己构建：**仓库里只存源码，Pages 由 Actions 现场生成**。

好处是写作流程退化成三步：

```bash
git add .
git commit -m "新增文章"
git push
```

推送后 GitHub 自动 `npm install` + `hexo generate` + 发布，一分多钟后线上就更新了。本地不用装部署插件，换电脑也不用重新配环境。

工作流文件放在 `.github/workflows/pages.yml`，核心就几行：检出代码 → 装 Node → `npm install` → `npx hexo generate` → 上传 `public` 作为 Pages 产物。

**务必记得 `.gitignore`**，不然 `node_modules` 会被一起提交，仓库瞬间几百兆：

```
node_modules/
public/
db.json
.deploy_git/
```

## 最后

真正搭起来大概花了两三个小时，其中一多半时间花在坑一和坑二上——都不是 Hexo 的问题，是 Windows 环境和路径配置的锅。

现在写文章的流程就是：`hexo new`、打开 md、写完、`git push`。剩下的交给流水线。

工具不复杂，难的是持续写。希望这个博客别在三个月后变成一片荒地。

> 本站地址：[https://ngngog.github.io](https://ngngog.github.io)
