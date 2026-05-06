---
name: tgmeng-news-skill
description: Search Tgmeng in-site news and hotspot data through the Tgmeng Skill Search API. Use when an agent needs to query current, today, or historical Tgmeng news/hotspot records with a user-provided license, keyword array, strict mode value, optional non-realtime time window, and optional result limit for external assistants such as Longxia, Hermes, or other tool-calling agents.
---

# Tgmeng News Skill

Use this skill to call Tgmeng's in-site news and hotspot search API.

## Endpoint

Call:

```text
POST https://trendapi.tgmeng.com/api/skill/search
```

## Required Input

Send JSON with these business parameters:

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI", "OpenAI"],
  "mode": "REALTIME",
  "startTime": null,
  "endTime": null,
  "rootCategories": ["科技"],
  "limit": 50
}
```

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). If the user does not have one, direct them to https://wechat.tgmeng.com to obtain it. Do not log, expose, or hard-code it.
- `keywords`: string array. Required by schema. Empty arrays are allowed for `REALTIME` and `TODAY`; `HISTORY` requires at least one non-blank keyword.
- `mode`: enum string. Required. Must be exactly one of `REALTIME`, `TODAY`, or `HISTORY`.
- `startTime`: string or null. Optional. Inclusive time window start for `TODAY` and `HISTORY`; ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`. Date-only values are normalized to `00:00:00`.
- `endTime`: string or null. Optional. Inclusive time window end for `TODAY` and `HISTORY`; ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`. Date-only values are normalized to `23:59:59`.
- `rootCategories`: string array or string. Optional. Filter by root category, matching the `items[].rootCategory` field exactly. Empty or omitted means all root categories.
- `limit`: integer or null. Optional. Maximum number of returned items. `null`, omitted, or `0` means no limit. Negative values are invalid.

Available `rootCategories` values:

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

Use these values exactly. For example, pass `科技` or `新闻`.

Mode behavior:

- `REALTIME`: Query current in-memory hotspot cache. Requires `SEARCH` license permission.
- `TODAY`: Query today's persisted hotspot history, optionally narrowed by `startTime` and `endTime`. Requires `SKILLHISTORY` license permission.
- `HISTORY`: Query long-term persisted hotspot history, optionally narrowed by `startTime` and `endTime`. Requires `SKILLHISTORY` license permission and non-empty `keywords`.

Keyword matching is OR-style fuzzy title matching: an item is returned when its title contains any keyword.
Root category filtering is AND-style with keyword matching: an item must match the requested `rootCategories` if the filter is provided. Multiple root category values use OR matching.

For non-realtime modes, if only `startTime` is provided, the API searches from that time to now. If only `endTime` is provided, the API uses the mode's default start time and the provided end time. `startTime` must be before or equal to `endTime`.

## Response Shape

The API returns Tgmeng's standard envelope:

```json
{
  "code": 200,
  "message": "请求成功",
  "data": {
    "query": {
      "mode": "REALTIME",
      "keywords": ["AI", "OpenAI"],
      "permission": "SEARCH",
      "limit": 50,
      "startTime": null,
      "endTime": null,
      "rootCategories": ["科技"]
    },
    "summary": {
      "total": 120,
      "returned": 50,
      "truncated": true
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

Treat `data.items` as the result list. Use `data.query` to understand what was searched and `data.summary` to detect truncation. Some item fields may be absent or null depending on source and mode.

## Error Handling

The API usually returns HTTP 200 with a non-success envelope for business errors. Always inspect `code` and `message`.

Common parameter errors:

- `license empty error`: `license` is missing or blank.
- `mode empty error`: `mode` is missing or null.
- `参数mode不支持，请传[REALTIME, TODAY, HISTORY]`: `mode` is not one of the allowed enum values.
- `参数keywords格式不正确，请传字符串数组`: `keywords` is not a JSON array.
- `limit must be integer`: `limit` is not an integer.
- `limit must be greater than or equal to 0`: `limit` is negative.
- `rootCategories must be string array or string`: `rootCategories` is neither a JSON string array nor a string.
- `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业`: `rootCategories` contains an unsupported value.
- `历史模式 keywords empty error`: `mode` is `HISTORY` but `keywords` has no non-blank values.
- `startTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd`: `startTime` format is invalid.
- `endTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd`: `endTime` format is invalid.
- `startTime must be before or equal to endTime`: Time window is reversed.

Common authorization errors:

- Missing or invalid license: license validation fails.
- Missing `SKILLHISTORY`: `TODAY` or `HISTORY` was requested but the license lacks historical search permission.

## Recommended Agent Workflow

1. Validate request locally before calling the API: `license` non-blank, `keywords` is an array, and `mode` is a valid enum.
2. For `HISTORY`, require at least one non-blank keyword.
3. For `TODAY` or `HISTORY`, pass `startTime` and `endTime` when the user asks for a precise period.
4. Call the endpoint with JSON content type.
5. Read `code`, `message`, and `data` from the response envelope.
6. Summarize results with source titles and URLs. Do not expose the license.

Diagnostics may record request metadata such as IP address, User-Agent, request path, error message, and license. Do not include the full license in agent logs or user-facing output.

## Detailed Contract

For exact schemas, examples, and OpenAPI operation metadata, read:

- `references/openapi.yaml`
- `references/api-contract.md`






