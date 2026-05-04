# Changelog

本文件记录 `tgmeng-news-skill` 的重要版本变化。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循语义化版本。

## [1.0.1] - 2026-05-05

### Added

- `TODAY` 和 `HISTORY` 查询模式新增可选 `startTime`、`endTime` 时间窗口参数，用于更精准地检索非实时历史数据。
- 文档说明时间参数支持 `yyyy-MM-dd HH:mm:ss` 和 `yyyy-MM-dd` 两种格式。
- 返回结构的 `data.query` 新增 `startTime`、`endTime` 回显字段。

### Changed

- 更新 `SKILL.md`、`README.md`、`references/openapi.yaml`、`references/api-contract.md`，同步非实时模式的时间窗口调用说明。

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
- 服务端调用日志仅记录脱敏密钥和密钥哈希，不保存明文密钥。
