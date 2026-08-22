# 笔记修复与博客导入 实施计划

> **For Hermes:** 实施时用 delegate_task 按任务批次派发子代理(快速模型 qwen3.6-flash),每批完成后人工抽检。

**Goal:** 将 `/root/markdown-notes` 的 65 篇技术笔记修复格式、修正内容错误、优化标题、重新分类,全程三态备份到 Gitea 私有库,最终以带 frontmatter/tag 的状态导入博客 `src/content/posts/`。

**Architecture:** 三目录流水线(`01-original` → `02-repaired` → `03-final`)+ 批量子代理修复(qwen3.6-flash,每篇独立处理,输出修复报告)+ 脚本化 frontmatter 注入 + `astro build` 构建验证。笔记源文件仓库与博客仓库分离,私有库存源文件,博客仓库只存成品。

**Tech Stack:** Python 3 脚本(文件处理/frontmatter)、qwen3.6-flash(token-plan 百炼端点,已验证 HTTP 200 可用)、Gitea API + HTTPS/token 推送、Astro/pnpm 构建验证。

---

## 一、需求理解(用户原话 → 我的理解)

| # | 用户要求 | 我的理解 |
|---|---|---|
| 0 | 先在 Gitea 建**私有库**存笔记源文件 | 新建私有仓库(如 `notes-source`),与博客仓库 `blog` 分开;Gitea API 有 token,可自动建仓 |
| 0 | 私有库分三个目录 | ① 修复前原始文件(现状快照)② 修复后**未**加 frontmatter ③ 修复后**已**加 frontmatter(即博客成品源) |
| 1 | 严格 Markdown 格式修复 | 补一级标题、修复标题层级、代码块闭合/语言标注、表格规范化、清除 HTML 标签、列表/空行规范 |
| 2 | 修复内容错误 | ⚠️ **高风险项**,见"开放问题 Q1"。我的默认方针:只修明显错误(命令拼写、前后矛盾、明显过时写法),不做主观改写,每篇输出修改清单 |
| 3 | 优化标题 | 检查现有标题(多为文件名)是否贴切,重命名为简洁准确的标题(约 8~25 字) |
| 4 | 按内容重新建文件夹分类 | 不沿用现有 10 个目录,按文章内容重新规划分类体系;分类同时写入 frontmatter 的 `category` 字段 |
| 5 | 完成后备份私有库 + 加 tag/frontmatter | 三态全部推入私有库;成品文章按 Firefly schema 注入 frontmatter(title/published/description/tags/category/slug) |
| — | 用子进程 + 快速模型 | 已验证 `qwen3.6-flash` 在 token-plan 端点可用(HTTP 200);按批次派发 delegate_task 处理 |

## 二、现状调研结论(只读探测已完成)

### 笔记源 `/root/markdown-notes`
- **65 篇** .md,10 个目录:`服务部署`(12)、`网络相关`(11)、`磁盘相关`(10)、`驱动相关`(8)、`大模型相关`(6)、`服务器硬件`(6)、`NAS应用部署`(6)、`其他`(5)、`存储配置`(1)、`系统配置`(**空目录**)
- **图片全是外链** `pic.byt3.ro/pic/...`(约 30+ 处),无本地图片,不需要迁移图床
- 56 篇含代码块、11 篇含表格、**7 篇含 HTML 标签**(需清理)
- **35 篇无一级标题**、28 篇开头非标题
- **65 篇全部无有效 frontmatter**(初查以为 3 篇有,实为误判——开头 `---` 是正文分隔线)
- 文件 mtime 全部是 2026-08-04(今天复制来的),**不能用作发布日期**

### 博客侧 Firefly schema(`src/content/config.ts`)
- 必填:`title`(string)、`published`(date)
- 可选:`description`、`tags[]`、`category`、`slug`、`image`、`pinned`、`draft`、`updated` 等
- URL 规则:glob loader 以路径为 id;frontmatter 可用 `slug:` 覆盖 → **中文文件名必须指定英文 slug**,否则 URL 含中文路径
- 上游 demo frontmatter 样板已取到(`title/published/pinned/description/tags/category/image/slug`)

### 环境
- Gitea:有 `$gitea_access` token,API 建仓可用;**SSH 4443 被 reset,推送必须走 HTTPS + token + `http.sslVerify=false`**
- 博客仓库:`github` + `origin`(Gitea)双远程已就位,main=`a187a4e2`
- 快速模型:`qwen3.6-flash` @ `token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`,key 用主模型 api_key,已实测 200

## 三、开放问题

- **Q1 内容错误修复的边界**:✅ 已定(用户 2026-08-04 拍板)——只修**客观明显错误**(命令拼写、自相矛盾、笔误);原文**不留标注**;修复完成后输出一份《改动清单》供用户抽查
- **Q2 发布日期**:✅ 已定——65 篇全部统一 `published: 2026-08-04`,以后新文章用真实日期
- **Q3 分类体系**:现有 10 目录是否可参考?我初步拟按内容合并为 6~8 类(如:系统运维、服务部署、网络、存储与磁盘、GPU与驱动、大模型部署、NAS),最终分类表在实施第一步产出、交用户确认后再批量执行
- **Q4 slug 策略**:每篇生成英文 slug(如 `linux-raid-setup`),写入 frontmatter;文件名保持中文(私有库可读性)——是否认可?
- **Q5 博客仓库目录结构**:`src/content/posts/` 下按分类建子目录(Firefly 支持,归档页按 frontmatter category 归类,目录仅组织作用)——是否认可?

## 四、实施步骤(任务拆解)

### 阶段 0:Gitea 私有库 + 原始备份(脚本,~5 分钟)
1. Gitea API 创建私有仓库 `notes-source`(`"private": true`)
2. 初始化仓库结构:`01-original/`、`02-repaired/`、`03-final/`(各含 README 说明)
3. 将 `/root/markdown-notes` **完整只读复制**到 `01-original/`(保留目录结构)
4. 提交并推送(HTTPS + token + sslVerify=false),SHA 验证
5. 此后**原始目录永不再改**——所有修复在 `02-repaired` 工作副本上进行

### 阶段 1:分类体系设计(需用户确认 Q3)
6. 用快速模型通读 65 篇(标题+前 300 字),产出分类建议表:每篇 → 建议分类 + 建议标题 + 建议 slug
7. 汇总表交用户审阅调整 → 锁定《分类/标题/slug 对照表》(存为 CSV,后续步骤的唯一事实来源)

### 阶段 2:格式与内容修复(子代理批处理,核心阶段)
8. 编写修复任务 prompt 模板(含:Markdown 规范清单、"只修明显错误"边界、输出格式要求:修复后全文 + 修改清单 JSON)
9. 按每批 5~8 篇派发 delegate_task(子代理内用 qwen3.6-flash 逐篇处理),输出到 `02-repaired/`
10. 每批完成后运行**自动校验脚本**:代码块闭合、标题层级、表格格式、无 HTML 残留、非空、UTF-8;不合格的回炉重修
11. 汇总 65 篇《修改清单报告》交用户抽查(重点看"内容错误修复"部分,Q1 风控点)

### 阶段 3:frontmatter 注入 + 成品目录
12. 编写注入脚本:读《对照表》→ 为 `02-repaired/` 每篇生成 frontmatter(title/优化后标题、published=Q2 决议值、description=模型生成摘要、tags=模型生成 3~6 个、category=对照表分类、slug=对照表 slug)→ 输出到 `03-final/`(按新分类建子目录)
13. 校验:用 Firefly 的 zod schema 逻辑本地预检 65 篇 frontmatter(必填字段、类型)
14. 三态全部提交推送私有库,SHA 验证

### 阶段 4:导入博客 + 构建验证
15. 将 `03-final/` 复制到博客 `src/content/posts/`(按分类子目录)
16. `rm -rf node_modules/.astro dist && pnpm build`(注意:必须清缓存,上次踩过坑)
17. 构建成功 → 启动预览,抽检:首页列表、分类页、标签页、随机 3 篇文章渲染(代码块/表格/图片外链)
18. `biome check` + `astro check` 通过
19. 提交博客仓库(amend 进定制提交或新提交,视改动量定),推送 GitHub + Gitea 双远程

### 阶段 5:收尾
20. 最终汇报:65 篇清单(旧标题→新标题→分类→slug)+ 修改报告位置 + 私有库/博客 SHA

## 五、涉及文件

| 路径 | 动作 |
|---|---|
| `/root/notes-pipeline/`(新建工作区) | 流水线脚本、对照表 CSV、修复报告、prompt 模板 |
| `/root/notes-pipeline/notes-source-repo/` | Gitea 私有库本地克隆(三目录) |
| `/root/markdown-notes/` | **只读**,绝不修改 |
| `/opt/Project/blog/src/content/posts/` | 阶段 4 导入成品文章 |
| `/opt/Project/blog/` | 构建验证 + 提交推送 |

## 六、验证标准

- [ ] 私有库三目录内容齐全,65×3 篇文件数一致
- [ ] `02-repaired` 全部通过自动格式校验(代码块/标题/表格/无 HTML)
- [ ] `03-final` 全部通过 frontmatter schema 预检
- [ ] `pnpm build` 成功(65 篇,pagefind 索引 65+ 页)
- [ ] 预览抽检无渲染异常
- [ ] 博客双远程 SHA 一致

## 七、风险与对策

| 风险 | 对策 |
|---|---|
| 模型"修内容"引入新错误 | Q1 方案 A 边界约束 + 每篇修改清单 + 用户抽查;原始文件永久保留可回滚 |
| 子代理批处理质量不稳定 | 每批自动校验脚本兜底,不合格回炉;批间抽检 |
| 中文路径/slug 问题 | 强制英文 slug 写入 frontmatter(对照表锁定) |
| 构建缓存脏数据 | 每次构建前清 `node_modules/.astro` + `dist`(已验证的坑) |
| Gitea SSH 不通 | 全程 HTTPS + token + sslVerify=false(已验证可行) |
| 图片外链失效 | 构建后抽检确认 `pic.byt3.ro` 图片可加载;失效的在报告中标注 |

## 八、预计耗时

阶段 0:5 分钟 · 阶段 1:10 分钟(+用户确认)· 阶段 2:30~50 分钟(9~13 批子代理)· 阶段 3:10 分钟 · 阶段 4:15 分钟 · 阶段 5:5 分钟。**合计约 1.5 小时**(不含用户审阅等待)。
