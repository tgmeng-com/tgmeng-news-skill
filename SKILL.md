---
name: tgmeng-news-skill
description: Search Tgmeng in-site news and hotspot data through the Tgmeng Skill Search API. Use when an agent needs to query current, today, or historical Tgmeng news/hotspot records with a user-provided license, keyword array, and strict mode enum for external assistants such as Longxia, Hemes, or other tool-calling agents.
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
  "limit": 50
}
```

Parameter contract:

- `license`: string. Required. Tgmeng universal license code (糖果梦通用密钥). If the user does not have one, direct them to https://wechat.tgmeng.com to obtain it. Do not log, expose, or hard-code it.
- `keywords`: string array. Required by schema. Empty arrays are allowed for `REALTIME` and `TODAY`; `HISTORY` requires at least one non-blank keyword.
- `mode`: enum string. Required. Must be exactly one of `REALTIME`, `TODAY`, or `HISTORY`.
- `limit`: integer or null. Optional. Maximum number of returned items. `null`, omitted, or `0` means no limit. Negative values are invalid.

Mode behavior:

- `REALTIME`: Query current in-memory hotspot cache. Requires `SEARCH` license permission.
- `TODAY`: Query today's persisted hotspot history. Requires `SKILLHISTORY` license permission.
- `HISTORY`: Query long-term persisted hotspot history. Requires `SKILLHISTORY` license permission and non-empty `keywords`.

Keyword matching is OR-style fuzzy title matching: an item is returned when its title contains any keyword.

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
      "limit": 50
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
- `历史模式 keywords empty error`: `mode` is `HISTORY` but `keywords` has no non-blank values.

Common authorization errors:

- Missing or invalid license: license validation fails.
- Missing `SKILLHISTORY`: `TODAY` or `HISTORY` was requested but the license lacks historical search permission.

## Recommended Agent Workflow

1. Validate request locally before calling the API: `license` non-blank, `keywords` is an array, and `mode` is a valid enum.
2. For `HISTORY`, require at least one non-blank keyword.
3. Call the endpoint with JSON content type.
4. Read `code`, `message`, and `data` from the response envelope.
5. Summarize results with source titles and URLs. Do not expose the license.

Server-side diagnostics may record request metadata such as IP address, User-Agent, request path, error message, masked license, and license hash. Do not include the full license in agent logs or user-facing output.

## Detailed Contract

For exact schemas, examples, and OpenAPI operation metadata, read:

- `references/openapi.yaml`
- `references/api-contract.md`






