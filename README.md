# Tim Chen的技术成长记录

博客地址：<https://zhe-mingc.github.io/>。使用 Hexo 7 和本仓库内的 `dewjo` 主题。

## 本地写作

使用 Node.js 24 和 npm，依赖以 `package-lock.json` 为准。

```bash
nvm use                 # 已安装 nvm 时使用；否则自行安装 Node.js 24
npm ci
npm run server
```

浏览器打开 <http://localhost:4000>。

```bash
npx hexo new "文章标题"
npx hexo new draft "草稿标题"
npm run server -- --draft
npx hexo publish "草稿标题"
```

- 文章：`source/_posts/*.md`；草稿：`source/_drafts/*.md`。
- 文章图片：放入与文章同名的资源目录，正文用 `![说明](图片文件名.png)` 引用。
- 站点标题、时区、RSS、发布目标：`_config.yml`。
- 导航、主题功能：`themes/dewjo/_config.yml`。
- 关于、留言等独立页面：`source/<页面名>/index.md`。

已开启 Hexo 官方 Markdown 渲染器的 `postAsset` 支持，图片不依赖旧的 `hexo-asset-image` 插件。文章图片使用浏览器原生懒加载。

已经发布的文章改标题或日期时，应通过 front matter 的 `permalink` 固定历史网址。DPDK 文章保留了 `2025/08/16/深入浅出DPDK笔记/`。

## 源码与发布分支

同一 GitHub 仓库使用两个分支：

- `source`：当前本地分支，保存 Markdown、图片、主题、配置和依赖锁文件。
- `main`：现有 GitHub Pages 发布分支，保存 Hexo 生成的静态文件，由 `hexo-deployer-git` 更新。

源码仓库的 `origin` 与部署目标均为 `git@github.com:Zhe-MingC/Zhe-MingC.github.io.git`。首次推送或部署前，需要当前机器的 SSH key 已获该 GitHub 仓库写入权限。

备份源码（首次运行才会在远程创建 `source` 分支）：

```bash
git status
git add <本次修改的源码文件>
git commit -m "docs: update blog posts"
git push -u origin source
```

源码分支不要推送覆盖远程 `main`。`public/`、`node_modules/`、`db.json` 和 `.deploy_git/` 已被忽略。

主题文件直接纳入源码，克隆不需要子模块。原主题 Git 数据保存在本机 `.local-backups/dewjo.git/`，该目录被忽略；主题原有许可保留在 `themes/dewjo/LICENSE`。

## 构建与发布

先检查首页、搜索、关于、留言、友链、RSS 和文章图片：

```bash
npm run clean
npm run build
npm run server
```

确认内容后发布：

```bash
npm run deploy
```

这个命令会清理缓存、重新构建，然后将静态文件推送到远程 `main`，更新线上博客。当前没有自动发布工作流；普通本地构建不会发布。

## 凭据与功能配置

- 不在配置、远程 URL 或提交中保存 GitHub token；使用 SSH 身份验证。
- `.env*`、`.npmrc`、`_config.local.yml` 和私钥文件默认被忽略。提交前仍需检查 `git diff --cached`。
- 删除文件中的旧 token 不会撤销它。如果配置曾保存 token，需要在 GitHub 的 [Personal access tokens](https://github.com/settings/tokens) 中撤销对应凭据。
- 留言页使用已有公开邮箱，目前没有在线评论服务。Valine 和打赏保持关闭，配置完成后再启用。
- RSS 由 `hexo-generator-feed` 生成到 `/atom.xml`；主题负责提供订阅链接。
- 搜索在浏览器中读取 `/search.xml`，不需要后端服务。
- 搜索索引使用 `templates/search.xml`，统一处理固定网址的前导斜杠，避免把文章路径误生成为 `//2025/...` 这样的外站地址。
- “说说”页面目前为空，导航入口暂时隐藏；补充内容后可在主题配置中开启。

参考：[Hexo 资源目录](https://hexo.io/docs/asset-folders)、[文章 front matter](https://hexo.io/docs/front-matter)、[Feed 插件](https://github.com/hexojs/hexo-generator-feed)。
