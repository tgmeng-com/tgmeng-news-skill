# Tgmeng News Skill API Contract

## Purpose

Expose Tgmeng in-site news, hotspot search, Tgmeng Index search, generated daily news reports, and TGMENG image/file upload to external intelligent agents. This skill is a contract wrapper around remote HTTP APIs and does not store license credentials.

## Agent Behavior Rules

### Capability Selection

Use raw hotspot search when the user clearly asks for original news/hotspot records, keyword search, realtime records, today's records, historical records, root categories, or a time-window search.

Use Tgmeng Index search when the user clearly asks for 糖果指数, index lists, 热度指数, AI-generated ranking, 综合热度, or cached Tgmeng Index data.

Use Daily News query when the user clearly asks for 日报, 每日总结, daily report, generated report content, category daily report, report Markdown, a report for a specific date, or a date-range list of generated reports.

Use Image/file upload when the user clearly asks to upload images or files, use the TGMENG image bed, integrate PicGo or another third-party upload client, batch upload, generate hosted image URLs, or call the public image upload API.

If the user's request is ambiguous, such as "查热点", "看看热榜", "最近有什么热搜", or "看看新闻总结", ask a short clarification before calling any API. Explain that raw hotspot search queries original news/hotspot data, Tgmeng Index search queries AI-generated and cached index lists with `hotScore`, Daily News query returns already generated daily reports with optional Markdown content, and Image/file upload sends local files to the TGMENG image bed and returns public links.

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
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

### Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `license` | string | Yes | Tgmeng universal license code (糖果梦通用密钥) used for authorization. If the user does not have one, direct them to `https://wechat.tgmeng.com`. Never log or reveal it. |
| `keywords` | string[] | Yes | Keyword array. Items are trimmed by the API. Blank items are ignored. Multiple keyword items use OR matching. Inside one item, `+`, `＋`, `&`, `＆` mean required include; `-`, `－`, `!`, `！` mean exclude. For a current full realtime feed, send `mode: "REALTIME"` with `keywords: []`; do not send `[""]` and do not omit `keywords`. `TODAY` and `HISTORY` require at least one non-blank keyword. |
| `mode` | enum | Yes | One of `REALTIME`, `TODAY`, `HISTORY`. Case-sensitive. |
| `startTime` | string/null | No | Inclusive time window start for `TODAY` and `HISTORY`. Ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`; date-only values are normalized to `00:00:00`. |
| `endTime` | string/null | No | Inclusive time window end for `TODAY` and `HISTORY`. Ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`; date-only values are normalized to `23:59:59`. |
| `rootCategories` | string[]/string/null | No | Filter by root category, matching `items[].rootCategory` exactly. Empty or omitted means all root categories. |
| `limit` | integer/null | No | Maximum returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit. |
| `offset` | integer/null | No | Result offset. Default is `0`. Use with a user-approved `limit` to fetch later pages. The next page offset is always `current offset + data.summary.returned`; for example, after requesting `limit: 100, offset: 0` and receiving `returned: 100`, request the next page with `offset: 100`. Negative values are invalid. |
| `distinct` | boolean/null | No | Whether to deduplicate by normalized title. Default is `false`. When `true`, deduplication runs after filters and before pagination, keeps the first item in the existing sort order, and does not use simHash. |

Currently known `rootCategories` values:

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

The platform may add or adjust root categories over time. Treat this list as the currently documented set; when filtering, prefer categories observed in `items[].rootCategory` or listed by the API's `rootCategories unsupported` error message.

### Modes

| Mode | Data source | Permission | Keyword rule |
| --- | --- | --- | --- |
| `REALTIME` | Current in-memory cache | `SEARCH` | May be empty. |
| `TODAY` | Today's persisted history | `SKILLHISTORY` | Must contain at least one non-blank keyword. |
| `HISTORY` | Long-term persisted history | `SKILLHISTORY` | Must contain at least one non-blank keyword. |

Licenses include the `SEARCH` permission required by `REALTIME` by default. To use `TODAY` or `HISTORY`, contact the administrator, explain the use case, and request `SKILLHISTORY` permission.

For a current full realtime feed, call `REALTIME` with `keywords: []`, `limit: null`, and `offset: 0`. `limit: null` means the agent is not applying a client-side count limit; the actual returned count is still bounded by the current realtime cache size, license permissions, and any server-side protection limits. Use `data.summary.returned`, `data.summary.total`, and `data.summary.hasMore` to understand the returned size.

### Time Window

`TODAY` and `HISTORY` support optional `startTime` and `endTime` for more precise historical retrieval. When omitted, the API keeps the existing default range for that mode. `REALTIME` always reads the current in-memory cache and ignores time-window fields.

If only `startTime` is provided, the API searches from that time to now. If only `endTime` is provided, the API uses the mode's default start time and the provided end time. `startTime` must be before or equal to `endTime`.

### Matching

Keywords keep backward-compatible OR-style fuzzy matching between array items. A single keyword item can use lightweight expressions: `+`, `＋`, `&`, `＆` require included terms, while `-`, `－`, `!`, `！` exclude terms. Examples: `黄金+伊朗` means title contains both `黄金` and `伊朗`; `黄金-伊朗` means title contains `黄金` and does not contain `伊朗`; `伊朗+导弹-足球` means title contains `伊朗` and `导弹` and excludes `足球`.

Root category filters are AND-style with keyword matching. If `rootCategories` is provided, the result must match one of those root categories. Root category values may expand as the platform changes; prefer actual `items[].rootCategory` values when summarizing or constructing follow-up filters.

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
      "offset": 0,
      "distinct": false,
      "startTime": null,
      "endTime": null,
      "rootCategories": ["科技"]
    },
    "summary": {
      "total": 120,
      "rawTotal": 120,
      "returned": 120,
      "limit": null,
      "offset": 0,
      "duplicatesRemoved": 0,
      "hasMore": false,
      "truncated": false
    },
    "items": []
  }
}
```

`code = 200` means success. For all other codes, treat the call as failed and display or summarize `message`. On success, read results from `data.items`, query metadata from `data.query`, and pagination status from `data.summary`. `data.items` is ordered by update time from newest to oldest, so earlier items are newer, rather than being sorted by hotspot popularity weight. This helps agents understand the result ordering logic. If `data.summary.hasMore` is true, keep the same filters and set the next request's `offset` to `data.summary.offset + data.summary.returned`. Do not derive the next offset from `limit`, especially when `limit` is `null` or the API returns fewer items than requested. When `distinct` is true, `data.summary.total` is the deduplicated total and `data.summary.rawTotal` is the pre-deduplication total.

### Raw Search Data Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.query.mode` | enum | Search mode used by the request. |
| `data.query.keywords` | string[] | Sanitized keyword array used by the request. |
| `data.query.permission` | string | Permission checked for this mode: `SEARCH` or `SKILLHISTORY`. |
| `data.query.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.query.offset` | integer | Requested result offset. |
| `data.query.distinct` | boolean | Whether title deduplication was enabled. |
| `data.query.startTime` | string/null | Normalized time-window start used by the query. |
| `data.query.endTime` | string/null | Normalized time-window end used by the query. |
| `data.query.rootCategories` | string[]/null | Root category filters used by the query. |
| `data.summary.total` | integer | Matched item count after optional title deduplication and before pagination. |
| `data.summary.rawTotal` | integer | Matched item count before optional title deduplication. |
| `data.summary.returned` | integer | Returned item count after `offset` and `limit`. |
| `data.summary.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.summary.offset` | integer | Requested result offset. |
| `data.summary.duplicatesRemoved` | integer | Number of items removed by title deduplication. Zero when `distinct` is false. |
| `data.summary.hasMore` | boolean | Whether more matched results remain after this response. |
| `data.summary.truncated` | boolean | Compatibility field with the same meaning as `hasMore`. |

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
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

### Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `license` | string | Yes | Tgmeng universal license code (糖果梦通用密钥) used for authorization. Never log or reveal it. |
| `keywords` | string[]/string/null | No | Keyword filter for hotspot titles. Empty or omitted means no title filtering. Multiple keyword items use OR matching. Inside one item, `+`, `＋`, `&`, `＆` mean required include; `-`, `－`, `!`, `！` mean exclude. Examples: `黄金+伊朗`, `黄金-伊朗`, `伊朗+导弹-足球`. |
| `categories` | string[]/string/null | No | Tgmeng Index category filter. The API also accepts `category`, `type`, `platformCategory`, or `分类` as aliases. Empty or omitted means all Tgmeng Index categories. |
| `limit` | integer/null | No | Maximum returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit. |
| `offset` | integer/null | No | Result offset. Default is `0`. Use with a user-approved `limit` to fetch later pages. The next page offset is always `current offset + data.summary.returned`; for example, after requesting `limit: 100, offset: 0` and receiving `returned: 100`, request the next page with `offset: 100`. Negative values are invalid. |
| `distinct` | boolean/null | No | Whether to deduplicate by normalized title. Default is `false`. When `true`, deduplication runs after filters and before pagination, keeps the first item in the existing sort order, and does not use simHash. |

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
      "offset": 0,
      "distinct": false,
      "categories": ["all", "technology", "ai"]
    },
    "summary": {
      "total": 120,
      "rawTotal": 120,
      "returned": 120,
      "limit": null,
      "offset": 0,
      "duplicatesRemoved": 0,
      "hasMore": false,
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

`hotScore` is the Tgmeng Index heat value. Larger values mean higher heat. If `data.summary.hasMore` is true, keep the same filters and set the next request's `offset` to `data.summary.offset + data.summary.returned`. Do not derive the next offset from `limit`, especially when `limit` is `null` or the API returns fewer items than requested. When `distinct` is true, `data.summary.total` is the deduplicated total and `data.summary.rawTotal` is the pre-deduplication total.

### Tgmeng Index Data Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.query.mode` | string | Fixed value `INDEX`. |
| `data.query.keywords` | string[] | Sanitized keyword array used by the request. |
| `data.query.permission` | string | Permission checked for this endpoint, usually `SEARCH`. |
| `data.query.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.query.offset` | integer | Requested result offset. |
| `data.query.distinct` | boolean | Whether title deduplication was enabled. |
| `data.query.categories` | string[] | Tgmeng Index categories used by the request. |
| `data.summary.total` | integer | Matched item count after optional title deduplication and before pagination. |
| `data.summary.rawTotal` | integer | Matched item count before optional title deduplication. |
| `data.summary.returned` | integer | Returned item count after `offset` and `limit`. |
| `data.summary.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.summary.offset` | integer | Requested result offset. |
| `data.summary.duplicatesRemoved` | integer | Number of items removed by title deduplication. Zero when `distinct` is false. |
| `data.summary.hasMore` | boolean | Whether more matched results remain after this response. |
| `data.summary.truncated` | boolean | Compatibility field with the same meaning as `hasMore`. |

### Tgmeng Index Item Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string/null | Hotspot title. |
| `source` | string/null | Usually `糖果指数`. |
| `category` | string/null | Tgmeng Index category name. |
| `rootCategory` | string/null | Root category, usually `糖果梦`. |
| `publishedAt` | string/null | Index list update time, usually `yyyy-MM-dd HH:mm:ss`. |
| `hotScore` | integer/null | Tgmeng Index heat value. Larger values mean higher heat. |

## Operation 3: Daily News Query

`queryDailyNews`

```text
POST https://trendapi.tgmeng.com/api/skill/dailyNews
Content-Type: application/json
```

Use the fixed production endpoint `https://trendapi.tgmeng.com/api/skill/dailyNews`.

### Request

```json
{
  "license": "USER_LICENSE_CODE",
  "date": "2026-05-20",
  "category": "technology",
  "startDate": null,
  "endDate": null,
  "limit": null,
  "offset": 0,
  "withContent": true
}
```

### Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `license` | string | Yes | Tgmeng universal license code (糖果梦通用密钥) used for authorization. Never log or reveal it. |
| `date` | string/null | No | Exact report date in `yyyy-MM-dd`. When provided, it takes precedence over `startDate` and `endDate`. |
| `startDate` | string/null | No | Inclusive report-date range start in `yyyy-MM-dd`. Used only when `date` is absent. |
| `endDate` | string/null | No | Inclusive report-date range end in `yyyy-MM-dd`. Used only when `date` is absent. |
| `category` | string/null | No | Daily report category value. Empty or `all` means all categories. The API also accepts `type` or `分类` as request-field aliases. |
| `limit` | integer/null | No | Maximum returned reports. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Concrete values must be between `0` and `50`. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit. |
| `offset` | integer/null | No | Result offset. Default is `0`. Use with a user-approved `limit` to fetch later pages. The next page offset is always `current offset + data.summary.returned`. Negative values are invalid. |
| `withContent` | boolean/null | No | Whether to include `items[].contentMarkdown`. Use `true` when the user asks for report body, Markdown, article content, export, download, or learning material. Use `false` when the user only wants a report list or metadata. |

Known daily report category values:

```text
all, news, wool, media, tv, life, community, finance, stock, sports, technology, design, audiovideo, game, health, education, futures, ai, sideline
```

The endpoint checks the `SEARCH` license permission.

### Response

```json
{
  "code": 200,
  "message": "请求成功",
  "data": {
    "query": {
      "mode": "DAILY_NEWS",
      "permission": "SEARCH",
      "category": "technology",
      "date": "2026-05-20",
      "startDate": null,
      "endDate": null,
      "limit": null,
      "offset": 0,
      "withContent": true
    },
    "summary": {
      "total": 1,
      "returned": 1,
      "limit": null,
      "offset": 0,
      "hasMore": false,
      "truncated": false
    },
    "items": [
      {
        "id": 123,
        "title": "AI Agent落地加速，芯片竞争与消费电子价格战成主线",
        "summary": "今日科技热点集中在大模型应用、国产芯片和消费电子价格变化。",
        "category": "technology",
        "categoryName": "科技",
        "reportDate": "2026-05-20",
        "coverImageUrl": "https://example.com/image.jpg",
        "tags": ["AI Agent", "芯片竞争", "消费电子"],
        "aiPlatforms": "Sublyx",
        "aiModels": "gpt-5.5",
        "aiFroms": "糖果梦",
        "requestTypes": "summary",
        "totalTokens": 116541,
        "hotCount": 2000,
        "inputCount": 2000,
        "batchCount": 10,
        "aiUsedTimeSeconds": 420,
        "durationMs": 180000,
        "contentMarkdown": "# AI Agent落地加速..."
      }
    ]
  }
}
```

Reports are ordered by `reportDate` from newest to oldest, then by id from newest to oldest. If `data.summary.hasMore` is true, keep the same filters and set the next request's `offset` to `data.summary.offset + data.summary.returned`. `contentMarkdown` is `null` when `withContent` is false.

### Daily News Data Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.query.mode` | string | Fixed value `DAILY_NEWS`. |
| `data.query.permission` | string | Permission checked for this endpoint, usually `SEARCH`. |
| `data.query.category` | string/null | Category filter used by the request, or null for all categories. |
| `data.query.date` | string/null | Exact report date used by the request. |
| `data.query.startDate` | string/null | Report-date range start used by the request. |
| `data.query.endDate` | string/null | Report-date range end used by the request. |
| `data.query.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.query.offset` | integer | Requested result offset. |
| `data.query.withContent` | boolean | Whether Markdown content was requested. |
| `data.summary.total` | integer | Matched report count before pagination. |
| `data.summary.returned` | integer | Returned report count after `offset` and `limit`. |
| `data.summary.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.summary.offset` | integer | Requested result offset. |
| `data.summary.hasMore` | boolean | Whether more matched reports remain after this response. |
| `data.summary.truncated` | boolean | Compatibility field with the same meaning as `hasMore`. |

### Daily News Item Fields

| Field | Type | Description |
| --- | --- | --- |
| `id` | integer | Report id. |
| `title` | string/null | Daily report title. |
| `summary` | string/null | Daily report summary. |
| `category` | string/null | Daily report category value. |
| `categoryName` | string/null | Daily report category display name. |
| `reportDate` | string/null | Report date in `yyyy-MM-dd`. |
| `coverImageUrl` | string/null | Cover image URL when available. |
| `tags` | string[] | Report tags. |
| `aiPlatforms` | string/null | AI platform information used during report generation. |
| `aiModels` | string/null | AI model information used during report generation. |
| `aiFroms` | string/null | AI request source or provider information. |
| `requestTypes` | string/null | Request type information recorded during report generation. |
| `totalTokens` | integer/null | Total token usage recorded for report generation. |
| `hotCount` | integer/null | Number of hotspots included in report generation. |
| `inputCount` | integer/null | Number of input records used by the report-generation process. |
| `batchCount` | integer/null | Batch count used by the report-generation process. |
| `aiUsedTimeSeconds` | integer/null | AI processing time in seconds. |
| `durationMs` | integer/null | Total report-generation task duration in milliseconds. |
| `contentMarkdown` | string/null | Report body Markdown. Null when `withContent` is false. |

## Operation 4: Image/File Upload

`uploadImageFile`

```text
POST https://image.tgmeng.com/api/v1/upload
Content-Type: multipart/form-data
```

Use the fixed production endpoint `https://image.tgmeng.com/api/v1/upload`.

Use this operation for API upload, PicGo, third-party image hosting clients, scripts, batch uploads, and hosted public image/file links.

### Headers

| Header | Required | Description |
| --- | --- | --- |
| `X-License-Code` | Yes | Tgmeng universal license code. Recommended auth header. Never place it in a URL, log, or user-facing response. |
| `X-Machine-Id` | Yes | Stable device id used for the same TGMENG device-binding behavior as the web products. Reuse the same value for the same client/device. |
| `Authorization` | No | Compatibility auth form: `Bearer USER_LICENSE_CODE`. Use when a client supports bearer token but not custom license header. |

The endpoint requires BASIC license permission. Uploaded files are owned by the validated license; users can manage only files uploaded under their own license.

### Multipart Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | binary | Yes for single upload | Preferred single-file field. |
| `files[]` | binary[] | Yes for batch upload | Preferred repeated batch field. |
| `image`, `smfile`, `source`, `files`, `upload`, `media` | binary/binary[] | No | Compatible aliases for third-party clients. |
| `uploadFolder` | string | No | Target folder/path label. Aliases: `folder`, `dir`, `directory`, `path`. |
| `uploadNameType` | enum | No | Naming strategy: `default`, `index`, `origin`, or `short`. |
| `serverCompress` | boolean string | No | Use `"false"` to disable Telegram server compression. |
| `autoRetry` | boolean string | No | Use `"false"` to disable automatic retry. |

### Single Upload Example

```bash
curl -X POST "https://image.tgmeng.com/api/v1/upload" \
  -H "X-License-Code: USER_LICENSE_CODE" \
  -H "X-Machine-Id: USER_STABLE_MACHINE_ID" \
  -F "file=@/path/to/image.png" \
  -F "serverCompress=false"
```

### Batch Upload Example

```bash
curl -X POST "https://image.tgmeng.com/api/v1/upload" \
  -H "X-License-Code: USER_LICENSE_CODE" \
  -H "X-Machine-Id: USER_STABLE_MACHINE_ID" \
  -F "files[]=@/path/to/a.png" \
  -F "files[]=@/path/to/b.jpg"
```

### Response

```json
{
  "success": true,
  "partial": false,
  "code": "success",
  "message": "Upload success",
  "url": "https://image.tgmeng.com/file/1749820000000_test.png",
  "data": {
    "url": "https://image.tgmeng.com/file/1749820000000_test.png",
    "urls": ["https://image.tgmeng.com/file/1749820000000_test.png"],
    "markdown": "![test.png](https://image.tgmeng.com/file/1749820000000_test.png)",
    "html": "<img src=\"https://image.tgmeng.com/file/1749820000000_test.png\" alt=\"test.png\">"
  },
  "files": [
    {
      "success": true,
      "name": "test.png",
      "size": 12345,
      "id": "1749820000000_test.png",
      "url": "https://image.tgmeng.com/file/1749820000000_test.png",
      "markdown": "![test.png](https://image.tgmeng.com/file/1749820000000_test.png)"
    }
  ]
}
```

### Image Upload Response Fields

| Field | Type | Description |
| --- | --- | --- |
| `success` | boolean | Whether all uploaded files succeeded. |
| `partial` | boolean | Whether some files succeeded and some failed. |
| `code` | string | Machine-readable status, such as `success`. |
| `message` | string | Human-readable result or error message. |
| `url` | string/null | First successful file URL, PicGo-compatible. |
| `data.url` | string/null | First successful file URL. |
| `data.urls` | string[] | All successful file URLs. |
| `data.markdown` | string/null | Markdown for the first successful file. |
| `data.html` | string/null | HTML image tag for the first successful file. |
| `files[]` | object[] | Per-file result list with `success`, `name`, `size`, `id`, `url`, and `markdown`. |

Existing `/file/...` public links remain externally accessible even if the license later expires. License expiration blocks later upload/manage actions, not already published file delivery links.

Telegram-backed storage is intended for images and ordinary files. Files up to about 100 MB are the stable target range; hundreds of MB may be slow or unreliable, and very large video-like uploads are not recommended.

## Error Contract

Common parameter errors:

| Message | Meaning |
| --- | --- |
| `Request method GET is not supported, please use POST` | Unsupported HTTP method. Retry with POST. |
| `Content-Type must be application/json` | Request body must be sent as JSON with `Content-Type: application/json`. |
| `request body must be json object` | Request body must be a JSON object, not an array, string, or other JSON value. |
| `request body empty error` | Request body is missing. |
| `license empty error` | `license` is missing or blank. |
| `mode empty error` | Raw search `mode` is missing or null. |
| `参数mode不支持，请传[REALTIME, TODAY, HISTORY]` | Raw search `mode` is not a valid enum value. |
| `参数keywords格式不正确，请传字符串数组` | `keywords` is not a valid JSON array or accepted string form. |
| `TODAY/HISTORY mode keywords empty error` | Raw search `mode = TODAY` or `mode = HISTORY` but keywords is empty or blank-only. |
| `limit must be integer` | `limit` is not an integer. |
| `limit must be greater than or equal to 0` | `limit` is negative. |
| `limit is too large` | `limit` is greater than the maximum supported integer. |
| `offset must be integer` | `offset` is not an integer. |
| `offset must be greater than or equal to 0` | `offset` is negative. |
| `offset is too large` | `offset` is greater than the maximum supported integer. |
| `distinct must be boolean` | `distinct` is not a boolean-compatible value. |
| `rootCategories must be string array or string` | `rootCategories` is neither a JSON string array nor a string. |
| `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业` | `rootCategories` contains an unsupported value. |
| `category must be string array or string` | Tgmeng Index category input is neither a JSON string array nor a string. |
| `category must be string` | Daily News category input is not a string. |
| `date must be yyyy-MM-dd` | Daily News exact date is not in `yyyy-MM-dd` format. |
| `startDate must be yyyy-MM-dd` | Daily News start date is not in `yyyy-MM-dd` format. |
| `endDate must be yyyy-MM-dd` | Daily News end date is not in `yyyy-MM-dd` format. |
| `startDate must be before or equal to endDate` | Daily News date range is reversed. |
| `limit must be less than or equal to 50` | Daily News concrete limit is larger than the maximum allowed value. |
| `withContent must be boolean` | Daily News `withContent` is not a boolean-compatible value. |
| `startTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd` | `startTime` format is invalid. |
| `endTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd` | `endTime` format is invalid. |
| `startTime must be before or equal to endTime` | Time window is reversed. |
| `Invalid multipart/form-data request` | Image upload body is not valid multipart form data. |
| `No file provided` | Image upload request did not include any supported file field. |
| `File too large` | Uploaded file exceeds the current upload limit. |

Common permission errors:

| Situation | Expected handling |
| --- | --- |
| Invalid license | Report license validation failure. |
| `TODAY` or `HISTORY` without `SKILLHISTORY` | Report missing historical search permission. |
| Image upload without BASIC permission | Report missing upload permission; do not retry aggressively. |
| Image upload without stable machine id | Reuse the same `X-Machine-Id` for the same client/device and retry once when safe. |
| Image upload from blocked IP | Report HTTP 403 `Your IP is blocked`. |

## Security Rules For Agents

- Never hard-code a license in the skill. If the user does not have a license, direct them to `https://wechat.tgmeng.com` to obtain a Tgmeng universal license code (糖果梦通用密钥).
- Never print a full license in logs or user-facing output.
- For JSON news/index/daily APIs, pass the license only in the HTTPS request body.
- For image upload, pass the license only in HTTPS headers: prefer `X-License-Code`, or use `Authorization: Bearer ...` for compatible clients. Do not place the license in the query string or multipart fields.
- For image upload, pass a stable `X-Machine-Id` header and do not generate a new machine id for every request.
- Diagnostics may record request metadata such as IP address, User-Agent, request path, error message, and license.
- Do not retry aggressively on authorization failures.
- Do not call raw `TODAY` or `HISTORY` without a keyword.
- Set `limit` to `null` by default. Do not set a concrete limit unless the user explicitly requested a count or confirmed a limit.
- Set `offset` to `0` by default. Continue pagination only when `data.summary.hasMore` is true, using `nextOffset = data.summary.offset + data.summary.returned`.
- Set `distinct` to `false` by default. Use `distinct: true` only when the user asks to reduce duplicate titles, repeated wire copy, reposts, or token waste from duplicated results.
- For Daily News, use `date` for exact-day report requests, or `startDate` and `endDate` for date ranges. Use `withContent: true` when the user wants report body or Markdown; use `withContent: false` for lists or metadata.
- For Image/file upload, send `multipart/form-data`, include at least one supported file field, and treat `success: false` or `partial: true` as an upload problem to report clearly.
- For self-healing retries, fix only the invalid field and retry once when safe: use POST for method errors, JSON content type for JSON API media-type errors, multipart form data for image upload, a JSON object for body-shape errors, integer non-negative values for `limit` and `offset`, boolean values for `distinct`, and a string array for raw search `keywords`.

## Examples

### Raw Realtime Search

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "mode": "REALTIME",
  "rootCategories": ["AI"],
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

### Raw Full Realtime Feed

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": [],
  "mode": "REALTIME",
  "rootCategories": [],
  "limit": null,
  "offset": 0,
  "distinct": false
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
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

### Tgmeng Index Search

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "categories": ["all", "technology", "ai"],
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

### Daily News Exact-Date Query

```json
{
  "license": "USER_LICENSE_CODE",
  "date": "2026-05-20",
  "category": "technology",
  "limit": null,
  "offset": 0,
  "withContent": true
}
```

### Daily News Metadata List

```json
{
  "license": "USER_LICENSE_CODE",
  "startDate": "2026-05-01",
  "endDate": "2026-05-20",
  "category": "all",
  "limit": null,
  "offset": 0,
  "withContent": false
}
```

### Image Upload

```bash
curl -X POST "https://image.tgmeng.com/api/v1/upload" \
  -H "X-License-Code: USER_LICENSE_CODE" \
  -H "X-Machine-Id: USER_STABLE_MACHINE_ID" \
  -F "file=@/path/to/image.png"
```

### Image Batch Upload

```bash
curl -X POST "https://image.tgmeng.com/api/v1/upload" \
  -H "X-License-Code: USER_LICENSE_CODE" \
  -H "X-Machine-Id: USER_STABLE_MACHINE_ID" \
  -F "files[]=@/path/to/a.png" \
  -F "files[]=@/path/to/b.jpg"
```
