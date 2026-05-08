---
name: tgmeng-news-skill
description: Search Tgmeng in-site news, hotspot data, and Tgmeng Index data through the Tgmeng Skill APIs. Use when an agent needs to query current, today, or historical Tgmeng news/hotspot records, or AI-generated Tgmeng Index lists, with a user-provided license and explicit user intent.
---

# Tgmeng News Skill

Use this skill to call Tgmeng's news, hotspot, and Tgmeng Index APIs.

This skill exposes two capabilities:

- Raw hotspot search: `POST https://trendapi.tgmeng.com/api/skill/search`
- Tgmeng Index search: `POST https://trendapi.tgmeng.com/api/skill/index`

## Agent Behavior Rules

### Capability Selection

Use raw hotspot search when the user clearly asks for original news/hotspot records, keyword search, realtime records, today's records, historical records, root categories, or a time-window search.

Use Tgmeng Index search when the user clearly asks for 糖果指数, index lists, 热度指数, AI-generated ranking, 综合热度, or cached Tgmeng Index data.

If the user's request is ambiguous, such as "查热点", "看看热榜", or "最近有什么热搜", ask a short clarification before calling any API. Explain the options:

1. Raw hotspot search: searches original news/hotspot records. It supports `REALTIME`, `TODAY`, `HISTORY`, keywords, root categories, and optional time windows.
2. Tgmeng Index search: searches AI-generated and cached Tgmeng Index lists. It returns indexed hotspot titles with `hotScore` and category, suitable for viewing overall heat.

Ask the user which capability they want to use, then call the matching endpoint.

### Limit Handling

Do not infer, add, or reduce a concrete `limit` automatically.

If the user did not explicitly request a result count, set `limit` to `null` by default, which means no limit. Do not change it to a concrete number without user confirmation.

If an agent wants to add `limit` for performance, readability, token budget, or summarization reasons, it must ask the user first and get confirmation.

Only set `limit` to a concrete number when:

- the user explicitly specifies a count, such as "top 10", "return 20", "只要前 5 条";
- or the user confirms the agent's clarification question about limiting result count.

## Raw Hotspot Search

Call:

```text
POST https://trendapi.tgmeng.com/api/skill/search
```

Send JSON with these business parameters:

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

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). If the user does not have one, direct them to https://wechat.tgmeng.com to obtain it. Do not log, expose, or hard-code it.
- `keywords`: string array. Required by schema. Empty arrays are allowed only for `REALTIME`; `TODAY` and `HISTORY` require at least one non-blank keyword.
- `mode`: enum string. Required. Must be exactly one of `REALTIME`, `TODAY`, or `HISTORY`.
- `startTime`: string or null. Optional. Inclusive time window start for `TODAY` and `HISTORY`; ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`. Date-only values are normalized to `00:00:00`.
- `endTime`: string or null. Optional. Inclusive time window end for `TODAY` and `HISTORY`; ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`. Date-only values are normalized to `23:59:59`.
- `rootCategories`: string array or string. Optional. Filter by root category, matching the `items[].rootCategory` field exactly. Empty or omitted means all root categories.
- `limit`: integer or null. Optional. Maximum number of returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit.

Available `rootCategories` values:

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

Mode behavior:

- `REALTIME`: Query current in-memory hotspot cache. Requires `SEARCH` license permission.
- `TODAY`: Query today's persisted hotspot history, optionally narrowed by `startTime` and `endTime`. Requires `SKILLHISTORY` license permission and non-empty `keywords`.
- `HISTORY`: Query long-term persisted hotspot history, optionally narrowed by `startTime` and `endTime`. Requires `SKILLHISTORY` license permission and non-empty `keywords`.

Licenses have `SEARCH` permission for `REALTIME` by default. If the user needs `TODAY` or `HISTORY`, tell them to contact the administrator, explain the use case, and request `SKILLHISTORY` permission.

Keyword matching is OR-style fuzzy title matching: an item is returned when its title contains any keyword.
Root category filtering is AND-style with keyword matching: an item must match the requested `rootCategories` if the filter is provided. Multiple root category values use OR matching.

For non-realtime modes, if only `startTime` is provided, the API searches from that time to now. If only `endTime` is provided, the API uses the mode's default start time and the provided end time. `startTime` must be before or equal to `endTime`.

### Raw Hotspot Response

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
    "items": [
      {
        "title": "News title",
        "url": "https://example.com/item",
        "source": "Platform name",
        "category": "Platform category",
        "rootCategory": "Root category",
        "publishedAt": "2026-05-03 01:00:00",
        "rank": 1,
        "simHash": 123456789
      }
    ]
  }
}
```

Treat `data.items` as the result list. Items are ordered by update time from newest to oldest; earlier items in the array are newer, rather than being sorted by hotspot popularity weight. This helps agents understand the result ordering logic. Use `data.query` to understand what was searched and `data.summary` to detect truncation. Some item fields may be absent or null depending on source and mode.

## Tgmeng Index Search

Call:

```text
POST https://trendapi.tgmeng.com/api/skill/index
```

Send JSON with these business parameters:

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "categories": ["all", "technology", "ai"],
  "limit": null
}
```

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). Do not log, expose, or hard-code it.
- `keywords`: string array, string, or null. Optional. Filters hotspot titles. Empty or omitted means no title filtering. Multiple keywords use OR matching.
- `categories`: string array, string, or null. Optional. Filters Tgmeng Index categories. The API also accepts `category`, `type`, `platformCategory`, or `分类` as aliases. Empty or omitted means all Tgmeng Index categories.
- `limit`: integer or null. Optional. Maximum number of returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit.

Available Tgmeng Index category values:

```text
all, dubang, alltalk, technology, finance, entertainment, car, sports, game, livelihood, ai, sideline, education
```

Common aliases:

- `all`: 综合, 糖果梦综合
- `dubang`: 毒榜, 糖果梦毒榜
- `alltalk`: 争议榜, 争议, 糖果梦争议榜
- `technology`: 科技, tech, 糖果梦科技
- `finance`: 财经, 糖果梦财经
- `entertainment`: 娱乐, 糖果梦娱乐
- `car`: 汽车, 糖果梦汽车
- `sports`: 体育, sport, 糖果梦体育
- `game`: 游戏, 糖果梦游戏
- `livelihood`: 民生, 糖果梦民生
- `ai`: AI, 人工智能, 糖果梦AI
- `sideline`: 副业, fuye, 糖果梦副业
- `education`: 教育, jiaoyu, 糖果梦教育

Unknown categories are ignored. If no valid category remains, the API queries all Tgmeng Index categories.

### Tgmeng Index Response

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

Treat `data.items` as the Tgmeng Index result list. `hotScore` is the Tgmeng Index heat value; larger values mean higher heat. Use `data.summary.truncated` to detect whether an explicit user-approved limit truncated the result set.

## Error Handling

The APIs usually return HTTP 200 with a non-success envelope for business errors. Always inspect `code` and `message`.

Common parameter errors:

- `request body empty error`: request body is missing.
- `license empty error`: `license` is missing or blank.
- `mode empty error`: `mode` is missing or null for raw hotspot search.
- `参数mode不支持，请传[REALTIME, TODAY, HISTORY]`: `mode` is not one of the allowed raw search enum values.
- `参数keywords格式不正确，请传字符串数组`: `keywords` is not a valid JSON array or accepted string form.
- `limit must be integer`: `limit` is not an integer.
- `limit must be greater than or equal to 0`: `limit` is negative.
- `rootCategories must be string array or string`: `rootCategories` is neither a JSON string array nor a string.
- `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业`: `rootCategories` contains an unsupported value.
- `category must be string array or string`: Tgmeng Index category input is neither a JSON string array nor a string.
- `TODAY/HISTORY mode keywords empty error`: raw search mode is `TODAY` or `HISTORY` but `keywords` has no non-blank values.
- `startTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd`: `startTime` format is invalid.
- `endTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd`: `endTime` format is invalid.
- `startTime must be before or equal to endTime`: time window is reversed.

Common authorization errors:

- Missing or invalid license: license validation fails.
- Missing `SKILLHISTORY`: `TODAY` or `HISTORY` was requested but the license lacks historical search permission.

## Recommended Agent Workflow

1. Determine whether the user wants raw hotspot search or Tgmeng Index search.
2. If the request is ambiguous, explain both capabilities and ask the user which one to use before calling an API.
3. Validate `license` locally before calling the API.
4. Set `limit` to `null` by default. Do not set a concrete limit unless the user explicitly requested a count or confirmed a limit.
5. For raw `TODAY` and `HISTORY`, require at least one non-blank keyword.
6. For raw `TODAY` or `HISTORY`, pass `startTime` and `endTime` when the user asks for a precise period.
7. Call the chosen endpoint with JSON content type.
8. Read `code`, `message`, and `data` from the response envelope.
9. Summarize results with source titles and URLs when available. Do not expose the license.

Diagnostics may record request metadata such as IP address, User-Agent, request path, error message, and license. Do not include the full license in agent logs or user-facing output.

## Detailed Contract

For exact schemas, examples, and OpenAPI operation metadata, read:

- `references/openapi.yaml`
- `references/api-contract.md`
