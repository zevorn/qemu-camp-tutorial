# CLAUDE.md

MkDocs/Zensical 文档站点，禁用 CI/CD，构建与发布全部手工完成。

## 项目结构

- `docs/tutorial/<year>/ch*/` — 主课程讲义
- `docs/exercise/<year>/stage*/` — 练习题
- `docs/blogs/`、`docs/news/`、`docs/image/` — 博客、新闻、资产
- `docs/plugins/`、`docs/template/` — 站点插件与模板
- `video/<topic>-口播稿.md` — 教学视频口播稿（用 `/qemu-camp-video-script` skill 生成）
- `site/` — 生成目录，禁止手工改动

## 课程阶段

| 目录 | 阶段 | 说明 |
|------|------|------|
| `ch0` | 导学 | 课程导读 |
| `ch1` | 基础 | QEMU 基础概念（QOM、MR、调试等） |
| `ch2` | 专业 | QEMU 内部机制（TCG、CPU 建模、设备、softmmu 等） |
| `ch3` | 项目 | 实战项目（CXL、K230、Wine CE 等） |

## 编码风格

- Markdown 以单一 `#` 一级标题开头，文件名 `kebab-case`，章节入口 `index.md`
- 提示框用 admonition（`!!! note`、`!!! warning`），块级缩进 4 空格
- 图片必须带配字并紧随 `{: .caption }`
- 章节作者用 `!!! note "主要贡献者"` 提示框标注

## Commit 规范

- 消息格式：`scope: action`（英文），如 `docs/tutorial/2026/ch2: add qemu-tcg article`
- 单次提交保持原子性，只覆盖一类改动
- PR 遵循 `.github/pull_request_template.md`

## 快速命令

```bash
zensical serve                # 本地预览 http://127.0.0.1:8000
autocorrect .                 # 中文排版检查
markdownlint-cli2 "**/*.md"  # Markdown 规范检查
```
