# Tgmeng News Skill API Contract

## Purpose

Expose Tgmeng in-site news, hotspot search, and Tgmeng Index search to external intelligent agents. This skill is a contract wrapper around remote HTTP APIs and does not store license credentials.

## Agent Behavior Rules

### Capability Selection

Use raw hotspot search when the user clearly asks for original news/hotspot records, keyword search, realtime records, today's records, historical records, root categories, or a time-window search.

Use Tgmeng Index search when the user clearly asks for 糖果指数, index lists, 热度指数, AI-generated ranking, 综合热度, or cached Tgmeng Index data.

If the user's request is ambiguous, such as "查热点", "看看热榜", or "最近有什么热搜", ask a short clarification before calling any API. Explain that raw hotspot search queries original news/hotspot data, while Tgmeng Index search queries AI-generated and cached index lists with `hotScore`.

### Limit Handling

Agents must not add, infer, or reduce a concrete `limit` automatically. If the user did not explicitly request a result count, pass `limit: null` by default to mean no limit. Do not change it to a concrete number without user confirmation.

If an agent wants to add `limit` for performance, readability, token budget, or summarization reasons, it must ask the user first and get confirmation.

Only set `limit` to a concrete number when the user explicitly specifies a count, such as "top 10", "return 20", "只要前 5 条", or when the user confirms the agent's clarification question.

## Operation 1: Raw Hotspot Search

`searchTgmengNews`

```text
POST https://trendapi.tgmeng.com/api/skill/search
Content-Type: application/json
```

Use the fixed production endpoint `https://trendapi.tgmeng.com/api/skill/search`.

### Request

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI", "OpenAI"],
  "mode": "REALTIME",
  "startTime": null,
  "endTime": null,
  "rootCategories": ["科技"],
  "limit": null
}
```

### Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `license` | string | Yes | Tgmeng universal license code (糖果梦通用密钥) used for authorization. If the user does not have one, direct them to `https://wechat.tgmeng.com`. Never log or reveal it. |
| `keywords` | string[] | Yes | Keyword array. Items are trimmed by the API. Blank items are ignored. Multiple keyword items use OR matching. Inside one item, `+`, `＋`, `&`, `＆` mean required include; `-`, `－`, `!`, `！` mean exclude. `TODAY` and `HISTORY` require at least one non-blank keyword. |
| `mode` | enum | Yes | One of `REALTIME`, `TODAY`, `HISTORY`. Case-sensitive. |
| `startTime` | string/null | No | Inclusive time window start for `TODAY` and `HISTORY`. Ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`; date-only values are normalized to `00:00:00`. |
| `endTime` | string/null | No | Inclusive time window end for `TODAY` and `HISTORY`. Ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`; date-only values are normalized to `23:59:59`. |
| `rootCategories` | string[]/string/null | No | Filter by root category, matching `items[].rootCategory` exactly. Empty or omitted means all root categories. |
| `limit` | integer/null | No | Maximum returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit. |

Available `rootCategories` values:

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

### Modes

| Mode | Data source | Permission | Keyword rule |
| --- | --- | --- | --- |
| `REALTIME` | Current in-memory cache | `SEARCH` | May be empty. |
| `TODAY` | Today's persisted history | `SKILLHISTORY` | Must contain at least one non-blank keyword. |
| `HISTORY` | Long-term persisted history | `SKILLHISTORY` | Must contain at least one non-blank keyword. |

Licenses include the `SEARCH` permission required by `REALTIME` by default. To use `TODAY` or `HISTORY`, contact the administrator, explain the use case, and request `SKILLHISTORY` permission.

### Time Window

`TODAY` and `HISTORY` support optional `startTime` and `endTime` for more precise historical retrieval. When omitted, the API keeps the existing default range for that mode. `REALTIME` always reads the current in-memory cache and ignores time-window fields.

If only `startTime` is provided, the API searches from that time to now. If only `endTime` is provided, the API uses the mode's default start time and the provided end time. `startTime` must be before or equal to `endTime`.

### Matching

Keywords keep backward-compatible OR-style fuzzy matching between array items. A single keyword item can use lightweight expressions: `+`, `＋`, `&`, `＆` require included terms, while `-`, `－`, `!`, `！` exclude terms. Examples: `黄金+伊朗` means title contains both `黄金` and `伊朗`; `黄金-伊朗` means title contains `黄金` and does not contain `伊朗`; `伊朗+导弹-足球` means title contains `伊朗` and `导弹` and excludes `足球`.

Root category filters are AND-style with keyword matching. If `rootCategories` is provided, the result must match one of those root categories.

### Response

```json
{
  "code": 200,
  "message": "请求成功",
  "data": {
    "query": {
      "mode": "REALTIME",
      "keywords": ["AI", "OpenAI"],
      "permission": "SEARCH",
      "limit": null,
      "startTime": null,
      "endTime": null,
      "rootCategories": ["科技"]
    },
    "summary": {
      "total": 120,
      "returned": 120,
      "truncated": false
    },
    "items": []
  }
}
```

`code = 200` means success. For all other codes, treat the call as failed and display or summarize `message`. On success, read results from `data.items`, query metadata from `data.query`, and truncation status from `data.summary`. `data.items` is ordered by update time from newest to oldest, so earlier items are newer, rather than being sorted by hotspot popularity weight. This helps agents understand the result ordering logic.

### Raw Search Data Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.query.mode` | enum | Search mode used by the request. |
| `data.query.keywords` | string[] | Sanitized keyword array used by the request. |
| `data.query.permission` | string | Permission checked for this mode: `SEARCH` or `SKILLHISTORY`. |
| `data.query.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.query.startTime` | string/null | Normalized time-window start used by the query. |
| `data.query.endTime` | string/null | Normalized time-window end used by the query. |
| `data.query.rootCategories` | string[]/null | Root category filters used by the query. |
| `data.summary.total` | integer | Matched item count before limit. |
| `data.summary.returned` | integer | Returned item count after limit. |
| `data.summary.truncated` | boolean | Whether limit truncated the result set. |

### Raw Search Item Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | News or hotspot title. |
| `url` | string/null | Source URL when available. |
| `source` | string/null | Source platform name. |
| `category` | string/null | Platform category. |
| `rootCategory` | string/null | Root category. |
| `publishedAt` | string/null | Update time, usually `yyyy-MM-dd HH:mm:ss`. |
| `rank` | number/null | Source ranking order. |
| `simHash` | number/null | Historical similarity hash when available. |

## Operation 2: Tgmeng Index Search

`searchTgmengIndex`

```text
POST https://trendapi.tgmeng.com/api/skill/index
Content-Type: application/json
```

Use the fixed production endpoint `https://trendapi.tgmeng.com/api/skill/index`.

### Request

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "categories": ["all", "technology", "ai"],
  "limit": null
}
```

### Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `license` | string | Yes | Tgmeng universal license code (糖果梦通用密钥) used for authorization. Never log or reveal it. |
| `keywords` | string[]/string/null | No | Keyword filter for hotspot titles. Empty or omitted means no title filtering. Multiple keyword items use OR matching. Inside one item, `+`, `＋`, `&`, `＆` mean required include; `-`, `－`, `!`, `！` mean exclude. Examples: `黄金+伊朗`, `黄金-伊朗`, `伊朗+导弹-足球`. |
| `categories` | string[]/string/null | No | Tgmeng Index category filter. The API also accepts `category`, `type`, `platformCategory`, or `分类` as aliases. Empty or omitted means all Tgmeng Index categories. |
| `limit` | integer/null | No | Maximum returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit. |

Available Tgmeng Index category values:

```text
all, dubang, alltalk, technology, finance, entertainment, car, sports, game, livelihood, ai, sideline, education
```

Unknown categories are ignored. If no valid category remains, the API queries all Tgmeng Index categories.

### Response

```json
{
  "code": 200,
  "message": "请求成功",
  "data": {
    "query": {
      "mode": "INDEX",
      "keywords": ["AI"],
      "permission": "SEARCH",
      "limit": null,
      "categories": ["all", "technology", "ai"]
    },
    "summary": {
      "total": 120,
      "returned": 120,
      "truncated": false
    },
    "items": [
      {
        "title": "Hotspot title",
        "source": "糖果指数",
        "category": "糖果梦科技",
        "rootCategory": "糖果梦",
        "publishedAt": "2026-05-08 12:00:00",
        "hotScore": 9820
      }
    ]
  }
}
```

`hotScore` is the Tgmeng Index heat value. Larger values mean higher heat.

### Tgmeng Index Data Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.query.mode` | string | Fixed value `INDEX`. |
| `data.query.keywords` | string[] | Sanitized keyword array used by the request. |
| `data.query.permission` | string | Permission checked for this endpoint, usually `SEARCH`. |
| `data.query.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.query.categories` | string[] | Tgmeng Index categories used by the request. |
| `data.summary.total` | integer | Matched item count before limit. |
| `data.summary.returned` | integer | Returned item count after limit. |
| `data.summary.truncated` | boolean | Whether limit truncated the result set. |

### Tgmeng Index Item Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string/null | Hotspot title. |
| `source` | string/null | Usually `糖果指数`. |
| `category` | string/null | Tgmeng Index category name. |
| `rootCategory` | string/null | Root category, usually `糖果梦`. |
| `publishedAt` | string/null | Index list update time, usually `yyyy-MM-dd HH:mm:ss`. |
| `hotScore` | integer/null | Tgmeng Index heat value. Larger values mean higher heat. |

## Error Contract

Common parameter errors:

| Message | Meaning |
| --- | --- |
| `request body empty error` | Request body is missing. |
| `license empty error` | `license` is missing or blank. |
| `mode empty error` | Raw search `mode` is missing or null. |
| `参数mode不支持，请传[REALTIME, TODAY, HISTORY]` | Raw search `mode` is not a valid enum value. |
| `参数keywords格式不正确，请传字符串数组` | `keywords` is not a valid JSON array or accepted string form. |
| `TODAY/HISTORY mode keywords empty error` | Raw search `mode = TODAY` or `mode = HISTORY` but keywords is empty or blank-only. |
| `limit must be integer` | `limit` is not an integer. |
| `limit must be greater than or equal to 0` | `limit` is negative. |
| `rootCategories must be string array or string` | `rootCategories` is neither a JSON string array nor a string. |
| `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业` | `rootCategories` contains an unsupported value. |
| `category must be string array or string` | Tgmeng Index category input is neither a JSON string array nor a string. |
| `startTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd` | `startTime` format is invalid. |
| `endTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd` | `endTime` format is invalid. |
| `startTime must be before or equal to endTime` | Time window is reversed. |

Common permission errors:

| Situation | Expected handling |
| --- | --- |
| Invalid license | Report license validation failure. |
| `TODAY` or `HISTORY` without `SKILLHISTORY` | Report missing historical search permission. |

## Security Rules For Agents

- Never hard-code a license in the skill. If the user does not have a license, direct them to `https://wechat.tgmeng.com` to obtain a Tgmeng universal license code (糖果梦通用密钥).
- Never print a full license in logs or user-facing output.
- Pass the license only in the HTTPS request body.
- Diagnostics may record request metadata such as IP address, User-Agent, request path, error message, and license.
- Do not retry aggressively on authorization failures.
- Do not call raw `TODAY` or `HISTORY` without a keyword.
- Set `limit` to `null` by default. Do not set a concrete limit unless the user explicitly requested a count or confirmed a limit.

## Examples

### Raw Realtime Search

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "mode": "REALTIME",
  "rootCategories": ["AI"],
  "limit": null
}
```

### Raw Today Search

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["机器人", "OpenAI"],
  "mode": "TODAY",
  "rootCategories": ["科技"],
  "startTime": "2026-05-05 00:00:00",
  "endTime": "2026-05-05 12:00:00",
  "limit": null
}
```

### Tgmeng Index Search

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "categories": ["all", "technology", "ai"],
  "limit": null
}
```
