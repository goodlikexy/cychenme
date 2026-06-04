# cychenme.org

个人主页、技术博客、学习笔记和项目展示站点。

## 技术栈

- Hugo
- Hugo Ladder Theme
- Markdown
- Cloudflare Pages

## 本地开发

```bash
git submodule update --init --recursive
hugo server -D
```

本地预览地址：

```text
http://localhost:1313
```

## 新建内容

新建技术博客：

```bash
hugo new blog/my-post.md
```

新建学习笔记：

```bash
hugo new notes/my-note.md
```

新建项目记录：

```bash
hugo new projects/my-project.md
```

## Cloudflare Pages

推荐配置：

```text
Framework preset: Hugo
Build command: hugo
Build output directory: public
Production branch: main
```

If Cloudflare asks for a deploy command, use:

```text
npx wrangler deploy
```

The Wrangler config is stored in [wrangler.jsonc](wrangler.jsonc) and deploys Hugo's `public` directory.

建议在 Cloudflare Pages 的环境变量中设置：

```text
HUGO_VERSION=0.128.0
```

域名绑定：

```text
cychenme.org
www.cychenme.org
```

## 主要配置

- 站点配置：[config.yml](config.yml)
- 技术博客：[content/blog](content/blog)
- 学习笔记：[content/notes](content/notes)
- 项目展示：[content/projects](content/projects)
- 关于页面：[content/about.md](content/about.md)
