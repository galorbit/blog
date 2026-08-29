# Repository Guidelines

## Project Structure & Module Organization

Firefly is an Astro 7 site with Svelte islands and TypeScript configuration. Main source code lives in `src/`: routes in `src/pages`, layouts in `src/layouts`, reusable UI in `src/components`, styles in `src/styles`, content in `src/content`, helpers in `src/utils`, and Markdown/HTML plugins in `src/plugins`. Site configuration is split across `src/config` with matching type definitions in `src/types`; prefer imports from `@/config` when available. Static files served directly belong in `public`, source-managed images in `src/assets`, docs in `docs` and `Firefly-Docs`, and automation in `scripts`.

## Build, Test, and Development Commands

Use `pnpm`; the `preinstall` script enforces it.

- `pnpm dev` or `pnpm start`: run the local Astro dev server.
- `pnpm check`: run Astro diagnostics.
- `pnpm type-check`: run TypeScript with `--noEmit`.
- `pnpm format`: format `src` with Biome.
- `pnpm lint`: run Biome checks and safe fixes on `src`.
- `pnpm build`: generate icons, LQIPs, the Astro build, font subsets, and Pagefind search output in `dist`.
- `pnpm preview`: preview the production build locally.
- `pnpm new-post`: scaffold a new content post.

## Coding Style & Naming Conventions

Biome is the formatter and linter. It uses tabs for indentation and double quotes for JavaScript/TypeScript strings. Keep Astro and Svelte components in `PascalCase` (`PostCard.astro`, `Search.svelte`), config modules in `camelCase` ending with `Config.ts`, and utilities in descriptive kebab case such as `date-utils.ts`. Keep `src/types` aligned with `src/config`. Avoid unrelated formatting churn.

## Testing Guidelines

There is no dedicated unit-test framework configured. Before submitting changes, run `pnpm check`, `pnpm type-check`, and `pnpm build` for rendering, content, or generated asset work. For visual or interactive changes, verify with `pnpm dev` or `pnpm preview` and include screenshots in the PR. Name future tests near the feature they cover, using the local file name as the stem.

## Commit & Pull Request Guidelines

Use Conventional Commits, matching the current history: `feat: ...`, `fix: ...`, and `chore: ...`. Keep commits and PRs focused on one concern. PRs should include a concise summary, linked issues when relevant, validation commands run, and screenshots for UI changes. Discuss major features or design changes in an issue or discussion before implementation.

## Security & Configuration Tips

Do not commit secrets, tokens, or service keys in config files. Keep deployment-specific settings in the target platform environment, and review generated files such as `dist`, `src/constants/lqips.json`, and `src/constants/icons.ts` before committing them.

## 推送与部署规范(用户约定)

- 远程: `github` = SSH, `origin` = Gitea https+token
- 笔记仓库(`greetingsyi/markdown-notes`)只推 Gitea(私有),不上 GitHub
- **博客仓库(公开)推送仅当用户明确说"推送博客"时才执行**: `git push github main && git -c http.sslVerify=false push origin main`
- 构建: 博客目录 `rm -rf node_modules/.astro .astro dist && pnpm build`(pnpm 11 需 `CI=true pnpm build`)
- 内容源: 笔记流水线 `02-final/`(Gitea markdown-notes 仓库本地克隆 `/opt/Project/notes-pipeline/repo/`)

## 笔记上传(用户说"检查新笔记并推送到博客"时,笔记流水线模式 B 的第二步)

前置:笔记流水线已先完成 02-final 处理并推 Gitea(模式 A),本流程只做"同步到博客"这一步。

1. `cd /opt/Project/notes-pipeline/repo && git pull origin main` 拉最新
2. 检查 `02-final/` 的 frontmatter(必须有 `title/category/tags/published`;`slug`/`description` 建议有)
   - 用 python 批量校验+采样;若有 `date:` 需转成 `published:`(博客 schema 只认 published)
3. `rm -rf src/content/posts/* && cp -r /opt/Project/notes-pipeline/repo/02-final/* src/content/posts/`(分类子目录照搬)
4. 构建验证: `CI=true pnpm build`,确认 `dist/posts/` 下文章数与 02-final 一致,URL 为英文 slug(`/posts/<slug>/`)
5. 提交 + 推送双远程(命令见上),Vercel 自动部署
