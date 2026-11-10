# Tgmeng News Skill v3 - 糖果梦热榜、招采、糖果指数、日报 API

给外部智能体使用的糖果梦开放平台 Skill 包。v3 全部走新开放平台 API，不再使用旧 `/api/skill/*` 和 Legacy License Code。

## 能力

- 热榜/新闻热点搜索：`POST /v1/hot/search`
- 招采/招标公告搜索：`POST /v1/procurement/search`
- 糖果指数查询：`POST /v1/candy/search`
- 日报查询：`POST /v1/daily/search`

**图床上传已下线，不属于本 Skill。**

## 认证

```http
Authorization: Bearer tgm_open_xxxxxxxxxxxxxxxx
Content-Type: application/json
```

API Key 在账号中心获取：

1. 打开 `https://account.tgmeng.com/developer`
2. 每个账号自动有一把站内通用 Key
3. 完整 Key 只在生成或重置时显示一次
4. 重置后旧 Key 立即失效

不要把 Key 写进仓库、日志或提示词。请求体里不再有 `license` 字段。

Base URL：

```text
https://api-open.tgmeng.com
```

## 快速开始

### 热榜

```bash
curl -X POST "https://api-open.tgmeng.com/v1/hot/search" \
  -H "Authorization: Bearer tgm_open_xxx" \
  -H "Content-Type: application/json" \
  -d '{"mode":"REALTIME","keywords":[],"limit":50,"offset":0}'
```

历史查询：

```json
{
  "mode": "HISTORY",
  "keywords": ["AI", "芯片"],
  "startTime": "2026-11-01",
  "endTime": "2026-11-07",
  "limit": 50,
  "offset": 0
}
```

`HISTORY` 的 `startTime`/`endTime` 必须在同一个自然月内；`TODAY`/`HISTORY` 至少需要一个关键词。

### 招采

```bash
curl -X POST "https://api-open.tgmeng.com/v1/procurement/search" \
  -H "Authorization: Bearer tgm_open_xxx" \
  -H "Content-Type: application/json" \
  -d '{"mode":"TODAY","keywords":["教育"],"provinces":["吉林"],"limit":20,"offset":0}'
```

### 糖果指数

```bash
curl -X POST "https://api-open.tgmeng.com/v1/candy/search" \
  -H "Authorization: Bearer tgm_open_xxx" \
  -H "Content-Type: application/json" \
  -d '{"categories":["technology","ai"],"limit":50,"offset":0}'
```

### 日报

```bash
curl -X POST "https://api-open.tgmeng.com/v1/daily/search" \
  -H "Authorization: Bearer tgm_open_xxx" \
  -H "Content-Type: application/json" \
  -d '{"category":"technology","date":"2026-11-06","withContent":true}'
```

## 响应结构

```json
{
  "requestId": "req_x",
  "data": {
    "query": {},
    "summary": {
      "total": 1200,
      "returned": 50,
      "limit": 50,
      "offset": 0,
      "hasMore": true
    },
    "items": []
  }
}
```

错误响应：

```json
{
  "requestId": "req_x",
  "error": {
    "code": "UNAUTHORIZED",
    "message": "API Key 缺失或无效"
  }
}
```

## 参数速查

### 热榜

- `mode`: `REALTIME` / `TODAY` / `HISTORY`
- `keywords`: 数组，多个关键词 OR；单个表达式内 `+` 表示同时包含，`-`/`!` 表示排除，例如 `伊朗+导弹-足球`
- `rootCategories`: 根分类过滤
- `startTime` / `endTime`: TODAY 可选，HISTORY 必填
- `limit`: 1..500
- `offset`: 0..5000；HISTORY 最大 2000
- `distinct`: 是否按标题去重

### 招采

支持 `mode`、`keywords`、`sourceCodes`、`sourceGroups`、`recordFamilies`、`regions`、`provinces`、`cities`、`rawTypes`、`projectNumber`、`purchaser`、`agency`、`startTime`、`endTime`、`publishedFrom`、`publishedTo`、`deadlineFrom`、`deadlineTo`、`amountMin`、`amountMax`、`sort`、`limit`、`offset`、`distinct`。

`sort`: `PUBLISHED_AT_DESC` / `PUBLISHED_AT_ASC` / `DEADLINE_ASC` / `DEADLINE_DESC` / `AMOUNT_DESC` / `AMOUNT_ASC`。

`data.sourceStatuses` 会给出每个来源的采集状态、数据范围、最新发布日期和更新时间。REALTIME 的来源数据可能不是今天发布的，不要误称为今日数据。

### 糖果指数

- `keywords`
- `categories`: `all`、`dubang`、`alltalk`、`technology`、`finance`、`entertainment`、`car`、`sports`、`game`、`livelihood`、`ai`、`sideline`、`education`
- `limit`: 1..500
- `offset`: 0..5000
- `distinct`

### 日报

- `date`: 精确日期，优先级高于日期范围
- `startDate` / `endDate`
- `category`
- `limit`: 1..50，默认 20 条
- `offset`: 0..5000
- `withContent`: true 时返回 `contentMarkdown`

## 限流与错误

响应头包含：

```text
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
X-RateLimit-Scope
```

错误码：

| HTTP | code | 说明 |
|---|---|---|
| 400 | `INVALID_MODE` / `INVALID_LIMIT` / `INVALID_OFFSET` / `INVALID_WINDOW` / `INVALID_DATE` / `INVALID_KEYWORD` / `INVALID_PARAM` / `UNSUPPORTED_FILTER` | 参数错误 |
| 401 | `UNAUTHORIZED` | Key 缺失或无效 |
| 403 | `FORBIDDEN` | Key 禁用/过期，或套餐不支持 |
| 429 | `RATE_LIMITED` | 每分钟限流 |
| 429 | `QUOTA_EXCEEDED` | 每日额度用尽 |

## v2 到 v3 迁移

| v2 | v3 |
|---|---|
| `POST /api/skill/search` | `POST /v1/hot/search` |
| `POST /api/skill/procurement/search` | `POST /v1/procurement/search` |
| `POST /api/skill/index` | `POST /v1/candy/search` |
| `POST /api/skill/dailyNews` | `POST /v1/daily/search` |
| 请求体 `license` | 请求头 `Authorization: Bearer tgm_open_...` |
| `image.tgmeng.com` 图床 | 已移除 |

## 文件说明

- `SKILL.md`：智能体读取的主说明。
- `README.md`：人类可读的接入文档。
- `references/api-contract.md`：四个接口的详细契约。
- `references/openapi.yaml`：OpenAPI 3 描述。
- `agents/openai.yaml`：Agent 展示信息。
- `skill-version.json`：远程版本清单。
