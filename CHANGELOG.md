# Changelog

本文件记录 `tgmeng-news-skill` 的重要版本变化。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循语义化版本。

## [2.2.1] - 2026-06-14

### Changed

- 优化 Skill SEO 和可发现性，README 标题补充“糖果梦热榜 API、日报 API、图床上传 API 与智能体接入”等核心关键词。
- README 新增“适用场景”，覆盖热榜 API、糖果指数 API、日报 API、PicGo 图床、第三方客户端上传、批量上传和 AI Agent 工具调用。
- `SKILL.md` frontmatter description、`references/openapi.yaml` description 和 `agents/openai.yaml` 展示文案补充中英文触发词，方便智能体和搜索索引识别。

## [2.2.0] - 2026-06-14

### Added

- 新增图床上传 API 能力，支持调用 `POST https://image.tgmeng.com/api/v1/upload` 上传图片/文件。
- 图床上传支持 `X-License-Code`、`Authorization: Bearer ...`、稳定 `X-Machine-Id`、`multipart/form-data`、PicGo/三方客户端字段兼容和批量上传。
- 上传响应文档新增 `url`、`data.url`、`data.urls`、`data.markdown`、`data.html` 和 `files[]` 单文件结果说明。
- `references/openapi.yaml` 新增 `/api/v1/upload` 路径和对应请求、响应 schema。
- `references/api-contract.md`、`README.md`、`SKILL.md` 和 `agents/openai.yaml` 新增图床上传的能力选择、参数契约、错误处理和调用示例。

### Changed

- Skill 能力说明从热榜数据搜索、糖果指数查询、日报数据查询扩展为热榜数据搜索、糖果指数查询、日报数据查询、图床上传 API 四类能力。
- 智能体安全规则区分 JSON 查询接口的请求体密钥传递与图床上传接口的 Header 密钥传递方式。

## [2.1.0] - 2026-05-20

### Added

- 新增日报数据查询能力，支持调用 `POST https://trendapi.tgmeng.com/api/skill/dailyNews` 查询已生成的糖果梦日报。
- 日报查询支持 `license`、`date`、`startDate`、`endDate`、`category`、`limit`、`offset`、`withContent` 参数。
- 日报查询返回标题、摘要、分类、日期、标签、AI 平台/模型信息、Token 消耗、热点数量以及可选的正文 Markdown。
- `references/openapi.yaml` 新增 `/api/skill/dailyNews` 路径和对应请求、响应 schema。
- `references/api-contract.md`、`README.md` 和 `SKILL.md` 新增日报查询的能力选择、参数契约、返回字段和调用示例。

### Changed

- Skill 能力说明从热榜数据搜索、糖果指数查询扩展为热榜数据搜索、糖果指数查询、日报数据查询三类能力。
- 智能体歧义处理规则新增日报选项：当用户明确要日报、每日总结、Markdown 正文或某天分类日报时，优先使用日报查询接口。

## [2.0.1] - 2026-05-13

### Changed

- 版本检查失败时，明确提示只是更新检查失败、已跳过版本检查并继续使用本地版本，不影响当前 API 请求。
- 明确 Skill 稳定名称和安装目录都应统一为 `tgmeng-news-skill`，旧目录 `tgmeng-news` 应迁移或重装，避免 cron 与跨环境同步依赖模糊匹配。
- 明确分页续拉公式为 `nextOffset = data.summary.offset + data.summary.returned`，并补充分页示例，避免用 `limit` 推导下一页。
- 说明 `rootCategories` 列表是当前已知集合，平台可能继续扩展或调整，后续筛选应优先参考实际返回的 `items[].rootCategory` 和接口错误提示。
- 明确当前全量实时热榜应使用 `mode: "REALTIME"` 和 `keywords: []`，不要传 `[""]` 或省略 `keywords`，并说明 `limit: null` 不代表突破服务端保护上限。

## [2.0.0] - 2026-05-10

### Changed

- 文档同步热榜数据搜索和糖果指数查询的关键词表达式能力：多个关键词项之间仍为 OR；单项内支持 `+`、`＋`、`&`、`＆` 表示必须包含，`-`、`－`、`!`、`！` 表示排除。
- 补充 `黄金+伊朗`、`黄金-伊朗`、`伊朗+导弹-足球` 等关键词表达式示例。
- 文档同步热榜数据搜索和糖果指数查询的 `limit + offset` 分页能力，并说明 `summary.hasMore` 与兼容字段 `summary.truncated`。
- 文档同步热榜数据搜索和糖果指数查询的 `distinct` 按标题去重能力，并说明 `summary.rawTotal` 与 `summary.duplicatesRemoved`。
- 文档补充方法、Content-Type、JSON 请求体、`limit`、`offset`、`distinct` 等参数错误提示，以及智能体安全重试建议。
- README 增加安装目录与缓存刷新提示，建议目录名保持为 `tgmeng-news-skill`，与 Skill name 一致。
## [1.1.1] - 2026-05-08

### Added

- 新增 `skill-version.json` 远程版本清单，用于智能体执行前检查本地 Skill 是否落后。

### Changed

- `SKILL.md` frontmatter 新增 `version`、`updated_at`、`update_check_url`，并新增执行前版本检查规则。
- `README.md` 新增版本检查说明，解释智能体如何比较本地版本和远程 `latestVersion`，以及如何展示新版功能和使用建议。

## [1.1.0] - 2026-05-08

### Added

- 新增糖果指数查询能力，支持调用 `POST https://trendapi.tgmeng.com/api/skill/index` 查询糖果指数榜单。
- 糖果指数接口支持 `license`、`keywords`、`categories`、`limit` 参数。
- 糖果指数返回结构新增 `hotScore` 字段，用于表示糖果指数热度值，数值越大表示热度越高。
- 文档新增糖果指数分类值和常见中文别名说明。
- `references/openapi.yaml` 新增 `/api/skill/index` 路径和对应请求、响应 schema。
- `agents/openai.yaml` 默认提示新增糖果指数能力说明。

### Changed

- `SKILL.md`、`README.md`、`references/api-contract.md` 重构为双能力说明：热榜数据搜索和糖果指数查询。
- 新增智能体能力选择规则：用户明确说糖果指数、指数榜、热度指数时使用糖果指数接口；用户明确说原始热搜、历史、关键词、时间窗口检索时使用热榜数据搜索接口。
- 新增歧义处理规则：当用户只说“查热点”“看看热榜”“最近有什么热搜”等不明确需求时，智能体应先解释两种能力并询问用户选择。
- `limit` 使用规则调整为默认传 `"limit": null` 表示不限制；智能体不得自行设置具体数字，除非用户明确指定数量或确认限制条数。
- 明确热榜数据搜索结果按更新时间从最新到最久排列，越靠前越新，不是按照热点热度权重排序，方便智能体知道结果排序逻辑。
- 放宽 OpenAPI 中糖果指数分类 schema，允许工具型智能体传入中文别名或其他后端兼容的分类字段值。

## [1.0.1] - 2026-05-05

### Added

- `TODAY` 和 `HISTORY` 查询模式新增可选 `startTime`、`endTime` 时间窗口参数，用于更精准地检索非实时历史数据。
- 文档说明时间参数支持 `yyyy-MM-dd HH:mm:ss` 和 `yyyy-MM-dd` 两种格式。
- 返回结构的 `data.query` 新增 `startTime`、`endTime` 回显字段。
- 新增 `rootCategories` 根分类过滤参数，支持按新闻、科技、AI 等根分类缩小搜索范围。
- 文档补充 `rootCategories` 可选值列表，OpenAPI 契约同步增加可选值约束。
- 返回结构的 `data.query` 新增 `rootCategories` 回显字段。

### Changed

- 更新 `SKILL.md`、`README.md`、`references/openapi.yaml`、`references/api-contract.md`，同步非实时模式的时间窗口调用说明。
- 分类过滤只对外支持根分类，不暴露平台分类过滤参数。

## [1.0.0] - 2026-05-03

### Added

- 新增糖果梦热榜 Skill 主说明文件 `SKILL.md`。
- 新增中文使用说明 `README.md`。
- 新增 OpenAPI 3.1 接口契约 `references/openapi.yaml`。
- 新增详细 API 契约文档 `references/api-contract.md`。
- 支持调用 `POST https://trendapi.tgmeng.com/api/skill/search` 查询糖果梦站内新闻和热点数据。
- 支持 `REALTIME`、`TODAY`、`HISTORY` 三种查询模式。
- 支持关键词数组匹配。
- 支持 `limit` 参数限制返回数量，`null`、不传或 `0` 表示不限制。
- 说明糖果梦通用密钥获取方式：`https://wechat.tgmeng.com`。
- 说明 `SEARCH` 与 `SKILLHISTORY` 权限要求。
- 说明智能体调用时的密钥安全要求。

### Changed

- 返回结构统一为智能体友好的 `query`、`summary`、`items`。
- 搜索结果不再嵌套重复的 `raw` 字段。

### Security

- Skill 文件不保存、不写死用户密钥。
- 文档要求智能体不得记录、展示或泄露完整密钥。
