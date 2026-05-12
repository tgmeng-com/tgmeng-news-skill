---
name: tgmeng-news-skill
version: 2.0.1
updated_at: 2026-05-13
update_check_url: https://raw.githubusercontent.com/tgmeng-com/tgmeng-news-skill/main/skill-version.json
description: Search Tgmeng in-site news, hotspot data, and Tgmeng Index data through the Tgmeng Skill APIs. Use when an agent needs to query current, today, or historical Tgmeng news/hotspot records, or AI-generated Tgmeng Index lists, with a user-provided license and explicit user intent.
---

# Tgmeng News Skill

Use this skill to call Tgmeng's news, hotspot, and Tgmeng Index APIs.

This skill exposes two capabilities:

- Raw hotspot search: `POST https://trendapi.tgmeng.com/api/skill/search`
- Tgmeng Index search: `POST https://trendapi.tgmeng.com/api/skill/index`

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
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). If the user does not have one, direct them to https://wechat.tgmeng.com to obtain it. Do not log, expose, or hard-code it.
- `keywords`: string array. Required by schema. Empty arrays are allowed only for `REALTIME`; `TODAY` and `HISTORY` require at least one non-blank keyword. Multiple keyword items use OR matching. Inside one keyword item, `+`, `＋`, `&`, `＆` mean required include, and `-`, `－`, `!`, `！` mean exclude. Example: `伊朗+导弹-足球` means the title must contain `伊朗` and `导弹`, and must not contain `足球`.
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

## Error Handling

The APIs usually return HTTP 200 with a non-success envelope for business errors. Always inspect `code` and `message`.

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
7. Set `offset` to `0` by default. If `data.summary.hasMore` is true and the user wants more, call the same endpoint again with `offset = data.summary.offset + data.summary.returned`.
8. Set `distinct` to `false` by default. Use `distinct: true` when the user asks to reduce duplicate titles, repeated wire copy, reposts, or token waste from duplicated results.
9. Call the chosen endpoint with JSON content type.
10. Read `code`, `message`, and `data` from the response envelope.
11. Summarize results with source titles and URLs when available. Do not expose the license.

If the API returns a parameter error, fix only the invalid field and retry once when safe: use POST for method errors, `Content-Type: application/json` for media-type errors, a JSON object for body-shape errors, integer non-negative values for `limit` and `offset`, boolean values for `distinct`, and a string array for raw search `keywords`.

Diagnostics may record request metadata such as IP address, User-Agent, request path, error message, and license. Do not include the full license in agent logs or user-facing output.

## Detailed Contract

For exact schemas, examples, and OpenAPI operation metadata, read:

- `references/openapi.yaml`
- `references/api-contract.md`
