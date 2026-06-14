---
name: tgmeng-news-skill
version: 2.2.1
updated_at: 2026-06-14
update_check_url: https://raw.githubusercontent.com/tgmeng-com/tgmeng-news-skill/main/skill-version.json
description: Search Tgmeng/糖果梦 in-site news, hotspot data, Tgmeng Index/糖果指数 data, generated daily news reports/日报, and upload images/files to the TGMENG image bed/图床 through the Tgmeng Open Platform Skill APIs. Use when an agent needs to query current, today, or historical news/hotspot records, AI-generated ranking/index lists, daily report Markdown, PicGo/third-party image upload, batch upload, or hosted image/file URLs with a user-provided license and explicit user intent.
---

# Tgmeng News Skill

Use this skill to call Tgmeng's news, hotspot, Tgmeng Index, daily report, and image upload APIs.

This skill exposes four capabilities:

- Raw hotspot search: `POST https://trendapi.tgmeng.com/api/skill/search`
- Tgmeng Index search: `POST https://trendapi.tgmeng.com/api/skill/index`
- Daily News query: `POST https://trendapi.tgmeng.com/api/skill/dailyNews`
- Image/file upload: `POST https://image.tgmeng.com/api/v1/upload`

## Agent Behavior Rules

### Skill Identity

Use `tgmeng-news-skill` as the stable skill name and installation directory name. The directory that contains this `SKILL.md` should be named `tgmeng-news-skill`, matching the `name` field in this file and the `name` field in `skill-version.json`.

If this skill is installed under an older or shortened directory name such as `tgmeng-news`, tell the user to reinstall or rename the skill directory to `tgmeng-news-skill` before relying on cron tasks, cross-environment sync, or exact skill references. Do not change the `name` field to match the old directory.

### Version Check

Before using this skill, check whether the local skill should be updated when network access is available.

Local version source:

- Read `version` from this `SKILL.md` frontmatter.
- If local `version` is missing, treat the local skill as outdated.

Remote version source:

- Fetch the JSON manifest from `update_check_url`.
- Compare remote `latestVersion` with local `version`.

If local version is missing or remote `latestVersion` is newer than local `version`:

1. Do not call the business API immediately.
2. Tell the user the local skill version is missing or outdated.
3. Show the local version, remote `latestVersion`, `updatedAt`, `releaseNotes`, `agentUsageTips`, and `breakingChanges` when present.
4. Ask whether the user wants to update the skill before continuing.
5. If the user declines, continue with the local skill.

If the version check fails because network access is unavailable or the remote manifest cannot be read, clearly tell the user that only the update check failed, the check has been skipped, and execution will continue with the local skill unless the user explicitly asked to update first. Do not treat update-check failure as a business API failure.

Use this version-check failure format:

```text
The tgmeng-news-skill update check failed because the remote version manifest could not be reached or read. I have skipped the update check and will continue with the local skill version. This does not affect the current API request.
```

Use this update reminder format:

```text
I detected that the local tgmeng-news-skill version is {localVersion}. The latest version is {latestVersion}, updated at {updatedAt}.

New features:
- ...

Agent usage tips:
- ...

Breaking changes:
- ...

Would you like to update the skill before I continue? If not, I can continue with the current local version.
```

### Capability Selection

Use raw hotspot search when the user clearly asks for original news/hotspot records, keyword search, realtime records, today's records, historical records, root categories, or a time-window search.

Use Tgmeng Index search when the user clearly asks for 糖果指数, index lists, 热度指数, AI-generated ranking, 综合热度, or cached Tgmeng Index data.

Use Daily News query when the user clearly asks for 日报, 每日总结, daily report, generated report content, category daily report, report Markdown, a report for a specific date, or a date-range list of generated reports.

Use Image/file upload when the user clearly asks to upload images, upload files to the image bed, use PicGo or a third-party upload client, batch upload, generate hosted image URLs, or call the public image upload API.

If the user's request is ambiguous, such as "查热点", "看看热榜", "最近有什么热搜", or "看看新闻总结", ask a short clarification before calling any API. Explain the options:

1. Raw hotspot search: searches original news/hotspot records. It supports `REALTIME`, `TODAY`, `HISTORY`, keywords, root categories, and optional time windows.
2. Tgmeng Index search: searches AI-generated and cached Tgmeng Index lists. It returns indexed hotspot titles with `hotScore` and category, suitable for viewing overall heat.
3. Daily News query: searches already generated Tgmeng daily reports. It supports date, date range, category, and optional Markdown content.
4. Image/file upload: uploads local images or files to the TGMENG image bed and returns public links, Markdown, HTML, and batch result metadata.

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
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). If the user does not have one, direct them to https://wechat.tgmeng.com to obtain it. Do not log, expose, or hard-code it.
- `keywords`: string array. Required by schema. Empty arrays are allowed only for `REALTIME`; `TODAY` and `HISTORY` require at least one non-blank keyword. For a current full realtime feed, send `mode: "REALTIME"` with `keywords: []`; do not send `[""]` and do not omit `keywords`. Multiple keyword items use OR matching. Inside one keyword item, `+`, `＋`, `&`, `＆` mean required include, and `-`, `－`, `!`, `！` mean exclude. Example: `伊朗+导弹-足球` means the title must contain `伊朗` and `导弹`, and must not contain `足球`.
- `mode`: enum string. Required. Must be exactly one of `REALTIME`, `TODAY`, or `HISTORY`.
- `startTime`: string or null. Optional. Inclusive time window start for `TODAY` and `HISTORY`; ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`. Date-only values are normalized to `00:00:00`.
- `endTime`: string or null. Optional. Inclusive time window end for `TODAY` and `HISTORY`; ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`. Date-only values are normalized to `23:59:59`.
- `rootCategories`: string array or string. Optional. Filter by root category, matching the `items[].rootCategory` field exactly. Empty or omitted means all root categories.
- `limit`: integer or null. Optional. Maximum number of returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit.
- `offset`: integer or null. Optional. Result offset. Use `0` by default. Use it with a user-approved `limit` to fetch later pages. The next page offset is always `current offset + data.summary.returned`; for example, after requesting `limit: 100, offset: 0` and receiving `returned: 100`, the next request should use `offset: 100`. Negative values are invalid.
- `distinct`: boolean or null. Optional. Default is `false`. When `true`, the API deduplicates by normalized title after keyword/category/time filtering and before pagination, keeping the first item in the existing sort order. It does not use simHash.

Currently known `rootCategories` values:

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

The platform may add or adjust root categories over time. Treat this list as the currently documented set; when filtering, prefer categories observed in `items[].rootCategory` or listed by the API's `rootCategories unsupported` error message.

Mode behavior:

- `REALTIME`: Query current in-memory hotspot cache. Requires `SEARCH` license permission.
- `TODAY`: Query today's persisted hotspot history, optionally narrowed by `startTime` and `endTime`. Requires `SKILLHISTORY` license permission and non-empty `keywords`.
- `HISTORY`: Query long-term persisted hotspot history, optionally narrowed by `startTime` and `endTime`. Requires `SKILLHISTORY` license permission and non-empty `keywords`.

For a current full realtime feed, call `REALTIME` with `keywords: []`, `limit: null`, and `offset: 0`. `limit: null` means the agent is not applying a client-side count limit; the actual returned count is still bounded by the current realtime cache size, license permissions, and any server-side protection limits. Use `data.summary.returned`, `data.summary.total`, and `data.summary.hasMore` to understand the returned size.

Licenses have `SEARCH` permission for `REALTIME` by default. If the user needs `TODAY` or `HISTORY`, tell them to contact the administrator, explain the use case, and request `SKILLHISTORY` permission.

Keyword matching keeps backward-compatible OR-style matching between keyword array items. A single keyword item may use lightweight expressions: `+`, `＋`, `&`, `＆` require included terms, while `-`, `－`, `!`, `！` exclude terms. For example, `伊朗+导弹-足球` means the title must contain `伊朗` and `导弹`, and must not contain `足球`; `股票＆黄金！AI` means the title must contain `股票` and `黄金`, and must not contain `AI`.
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

Treat `data.items` as the result list. Items are ordered by update time from newest to oldest; earlier items in the array are newer, rather than being sorted by hotspot popularity weight. This helps agents understand the result ordering logic. Use `data.query` to understand what was searched and `data.summary.hasMore` to detect whether more results are available. `data.summary.truncated` is kept for compatibility and has the same meaning as `hasMore`. If `hasMore` is true and the user wants more results, keep the same filters and set the next request's `offset` to `data.summary.offset + data.summary.returned`. For example, if the current response has `offset: 100` and `returned: 100`, the next request should use `offset: 200`. When `distinct` is true, `data.summary.total` is the deduplicated total and `data.summary.rawTotal` is the pre-deduplication total. Some item fields may be absent or null depending on source and mode.

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
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). Do not log, expose, or hard-code it.
- `keywords`: string array, string, or null. Optional. Filters hotspot titles. Empty or omitted means no title filtering. Multiple keyword items use OR matching. Inside one keyword item, `+`, `＋`, `&`, `＆` mean required include, and `-`, `－`, `!`, `！` mean exclude. Examples: `黄金+伊朗`, `黄金-伊朗`, `伊朗+导弹-足球`.
- `categories`: string array, string, or null. Optional. Filters Tgmeng Index categories. The API also accepts `category`, `type`, `platformCategory`, or `分类` as aliases. Empty or omitted means all Tgmeng Index categories.
- `limit`: integer or null. Optional. Maximum number of returned items. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. Negative values are invalid. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit.
- `offset`: integer or null. Optional. Result offset. Use `0` by default. Use it with a user-approved `limit` to fetch later pages. The next page offset is always `current offset + data.summary.returned`; for example, after requesting `limit: 100, offset: 0` and receiving `returned: 100`, the next request should use `offset: 100`. Negative values are invalid.
- `distinct`: boolean or null. Optional. Default is `false`. When `true`, the API deduplicates by normalized title after keyword/category filtering and before pagination, keeping the first item in the existing sort order. It does not use simHash.

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

Treat `data.items` as the Tgmeng Index result list. `hotScore` is the Tgmeng Index heat value; larger values mean higher heat. Use `data.summary.hasMore` to detect whether more results are available. `data.summary.truncated` is kept for compatibility and has the same meaning as `hasMore`. If `hasMore` is true and the user wants more results, keep the same filters and set the next request's `offset` to `data.summary.offset + data.summary.returned`. For example, if the current response has `offset: 100` and `returned: 100`, the next request should use `offset: 200`. When `distinct` is true, `data.summary.total` is the deduplicated total and `data.summary.rawTotal` is the pre-deduplication total.

## Daily News Query

Call:

```text
POST https://trendapi.tgmeng.com/api/skill/dailyNews
```

Send JSON with these business parameters:

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

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). Do not log, expose, or hard-code it.
- `date`: string or null. Optional. Exact report date in `yyyy-MM-dd`. When `date` is provided, it takes precedence over `startDate` and `endDate`.
- `startDate`: string or null. Optional. Inclusive report-date range start in `yyyy-MM-dd`. Used only when `date` is absent.
- `endDate`: string or null. Optional. Inclusive report-date range end in `yyyy-MM-dd`. Used only when `date` is absent.
- `category`: string or null. Optional. Daily report category value, such as `all`, `news`, `wool`, `finance`, `technology`, `audiovideo`, or `ai`. Empty or `all` means all categories.
- `limit`: integer or null. Optional. Maximum number of returned reports. Use `null` by default to mean no limit. The API also accepts omitted or `0` as no limit. The maximum concrete limit is `50`. Do not set a concrete number unless the user explicitly requested a count or confirmed a limit.
- `offset`: integer or null. Optional. Result offset. Use `0` by default. Use it with a user-approved `limit` to fetch later pages. The next page offset is always `current offset + data.summary.returned`.
- `withContent`: boolean or null. Optional. Whether to include `items[].contentMarkdown`. Use `true` when the user asks for report body, Markdown, article content, export, download, or learning material. Use `false` when the user only wants a report list or metadata. If unclear and response size matters, ask the user before disabling content.

Known daily report category values:

```text
all, news, wool, media, tv, life, community, finance, stock, sports, technology, design, audiovideo, game, health, education, futures, ai, sideline
```

The Daily News endpoint requires the `SEARCH` license permission.

### Daily News Response

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

Treat `data.items` as the generated report list. Items are ordered by report date from newest to oldest, then by id from newest to oldest. `contentMarkdown` is null when `withContent` is false. If `data.summary.hasMore` is true and the user wants more reports, keep the same filters and set the next request's `offset` to `data.summary.offset + data.summary.returned`.

## Image/File Upload

Call:

```text
POST https://image.tgmeng.com/api/v1/upload
Content-Type: multipart/form-data
```

Use this endpoint when the user wants API upload, PicGo integration, third-party image hosting clients, scripted uploads, batch uploads, or public image/file links.

Send authentication in HTTPS headers:

```text
X-License-Code: USER_LICENSE_CODE
X-Machine-Id: USER_STABLE_MACHINE_ID
Authorization: Bearer USER_LICENSE_CODE
```

Header contract:

- `X-License-Code`: string. Required and recommended. Tgmeng universal license code. Do not send it in query strings, logs, or user-facing output.
- `X-Machine-Id`: string. Required. Stable device identifier for the same user/device binding logic used by TGMENG sites. Generate or reuse a stable value per client/device; do not rotate it per request.
- `Authorization`: string. Optional compatibility form. `Bearer USER_LICENSE_CODE` may be used instead of `X-License-Code` when a client only supports bearer tokens.

Send files as `multipart/form-data` fields. Preferred field names:

- `file`: preferred single-file field.
- `files[]`: preferred repeated batch field.
- Compatible aliases: `image`, `smfile`, `source`, `files`, `upload`, `media`.

Optional form fields:

- `uploadFolder`: target folder/path label. Aliases: `folder`, `dir`, `directory`, `path`.
- `uploadNameType`: naming strategy. Supported values: `default`, `index`, `origin`, `short`.
- `serverCompress`: boolean string. Use `"false"` to disable Telegram server compression.
- `autoRetry`: boolean string. Use `"false"` to disable auto-retry.

Single upload example:

```bash
curl -X POST "https://image.tgmeng.com/api/v1/upload" \
  -H "X-License-Code: USER_LICENSE_CODE" \
  -H "X-Machine-Id: USER_STABLE_MACHINE_ID" \
  -F "file=@/path/to/image.png" \
  -F "serverCompress=false"
```

Batch upload example:

```bash
curl -X POST "https://image.tgmeng.com/api/v1/upload" \
  -H "X-License-Code: USER_LICENSE_CODE" \
  -H "X-Machine-Id: USER_STABLE_MACHINE_ID" \
  -F "files[]=@/path/to/a.png" \
  -F "files[]=@/path/to/b.jpg"
```

### Image/File Upload Response

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

Treat `url` and `data.url` as the first successful file URL, `data.urls` as all successful URLs, and `files` as per-file batch status. `partial: true` means at least one file failed while at least one file succeeded.

The endpoint requires BASIC license permission. Files are owned by the validated license, and users can manage only files uploaded under their own license. Existing `/file/...` public links remain externally accessible even if the license later expires; expiration blocks later upload/manage actions, not already published file delivery links.

Telegram-backed storage is intended for images and ordinary files. Files up to about 100 MB are the stable target range; hundreds of MB may be slow or unreliable, and very large video-like uploads are not recommended.

## Error Handling

The JSON query APIs usually return HTTP 200 with a non-success envelope for business errors. Always inspect `code` and `message`. The Image/file upload API may use HTTP status codes such as 400, 401, 403, 413, and 500 in addition to its JSON response body.

Common parameter errors:

- `request body empty error`: request body is missing.
- `request body must be json object`: request body is not a JSON object. Use an object such as `{ "license": "...", "keywords": [] }`.
- `Request method GET is not supported, please use POST`: the endpoint was called with GET or another unsupported method. Retry with POST.
- `Content-Type must be application/json`: the request body was not sent as JSON. Retry with `Content-Type: application/json`.
- `license empty error`: `license` is missing or blank.
- `mode empty error`: `mode` is missing or null for raw hotspot search.
- `参数mode不支持，请传[REALTIME, TODAY, HISTORY]`: `mode` is not one of the allowed raw search enum values.
- `参数keywords格式不正确，请传字符串数组`: `keywords` is not a valid JSON array or accepted string form.
- `limit must be integer`: `limit` is not an integer.
- `limit must be greater than or equal to 0`: `limit` is negative.
- `limit is too large`: `limit` is greater than the maximum supported integer.
- `offset must be integer`: `offset` is not an integer.
- `offset must be greater than or equal to 0`: `offset` is negative.
- `offset is too large`: `offset` is greater than the maximum supported integer.
- `distinct must be boolean`: `distinct` is not a boolean-compatible value.
- `rootCategories must be string array or string`: `rootCategories` is neither a JSON string array nor a string.
- `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业`: `rootCategories` contains an unsupported value.
- `category must be string array or string`: Tgmeng Index category input is neither a JSON string array nor a string.
- `category must be string`: Daily News category input is not a string.
- `date must be yyyy-MM-dd`: Daily News exact date format is invalid.
- `startDate must be yyyy-MM-dd`: Daily News start date format is invalid.
- `endDate must be yyyy-MM-dd`: Daily News end date format is invalid.
- `startDate must be before or equal to endDate`: Daily News date range is reversed.
- `limit must be less than or equal to 50`: Daily News concrete limit is larger than the maximum allowed value.
- `withContent must be boolean`: Daily News `withContent` is not a boolean-compatible value.
- `TODAY/HISTORY mode keywords empty error`: raw search mode is `TODAY` or `HISTORY` but `keywords` has no non-blank values.
- `startTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd`: `startTime` format is invalid.
- `endTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd`: `endTime` format is invalid.
- `startTime must be before or equal to endTime`: time window is reversed.

Common authorization errors:

- Missing or invalid license: license validation fails.
- Missing `SKILLHISTORY`: `TODAY` or `HISTORY` was requested but the license lacks historical search permission.
- Image upload missing/invalid license or missing BASIC permission: report upload authorization failure and do not retry aggressively.
- Image upload missing/rotated `X-Machine-Id`: reuse the stable device machine id instead of generating a new one per request.

Common image upload HTTP errors:

- HTTP 400 `Invalid multipart/form-data request`: request is not valid multipart form data.
- HTTP 400 `No file provided`: no supported file field was sent.
- HTTP 401 `Unauthorized`: license or machine id authorization failed.
- HTTP 403 `Your IP is blocked`: source IP is blocked by protection rules.
- HTTP 413 `File too large`: file exceeded the current upload limit.
- HTTP 500 upload/storage failures: retry once only when the request is idempotent and the user accepts a retry.

## Recommended Agent Workflow

1. Determine whether the user wants raw hotspot search, Tgmeng Index search, Daily News query, or Image/file upload.
2. If the request is ambiguous, explain the available capabilities and ask the user which one to use before calling an API.
3. Validate `license` locally before calling the API.
4. Set `limit` to `null` by default. Do not set a concrete limit unless the user explicitly requested a count or confirmed a limit.
5. For raw `TODAY` and `HISTORY`, require at least one non-blank keyword.
6. For raw `TODAY` or `HISTORY`, pass `startTime` and `endTime` when the user asks for a precise period.
7. For Daily News, use `date` for exact-day report requests, or `startDate` and `endDate` for report-date ranges.
8. For Daily News, set `withContent` to `true` when the user wants report body or Markdown; set it to `false` only for report lists or metadata.
9. For Image/file upload, require at least one file path/blob and a stable `X-Machine-Id`; send `multipart/form-data`, not JSON.
10. Set `offset` to `0` by default. If `data.summary.hasMore` is true and the user wants more, call the same endpoint again with `offset = data.summary.offset + data.summary.returned`.
11. Set `distinct` to `false` by default for raw hotspot search and Tgmeng Index search. Daily News and Image/file upload do not support `distinct`.
12. Call the chosen news/index/daily endpoint with JSON content type, or the image upload endpoint with multipart form data.
13. Read `code`, `message`, and `data` from JSON APIs; for image upload, read `success`, `partial`, `url`, `data.urls`, and `files`.
14. Summarize results with source titles, report titles, URLs, or Markdown content when available. Do not expose the license.

If the API returns a parameter error, fix only the invalid field and retry once when safe: use POST for method errors, `Content-Type: application/json` for JSON media-type errors, `multipart/form-data` for image upload, a JSON object for body-shape errors, integer non-negative values for `limit` and `offset`, boolean values for `distinct`, and a string array for raw search `keywords`.

Diagnostics may record request metadata such as IP address, User-Agent, request path, error message, and license. Do not include the full license in agent logs or user-facing output.

## Detailed Contract

For exact schemas, examples, and OpenAPI operation metadata, read:

- `references/openapi.yaml`
- `references/api-contract.md`
