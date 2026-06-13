# cychenme.org

记录生活、学习和个人作品的站点。

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

新建生活记录：

```bash
hugo new life/my-post.md
```

新建学习记录：

```bash
hugo new study/my-note.md
```

新建作品：

```bash
hugo new works/my-work.md
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
- 生活记录：[content/life](content/life)
- 学习记录：[content/study](content/study)
- 作品展示：[content/works](content/works)
- 关于页面：[content/about.md](content/about.md)
