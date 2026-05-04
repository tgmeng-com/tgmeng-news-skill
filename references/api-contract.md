# Tgmeng News Skill API Contract

## Purpose

Expose Tgmeng in-site news and hotspot search to external intelligent agents. This skill is a contract wrapper around the remote HTTP API and does not store license credentials.

## Operation

`searchTgmengNews`

```text
POST https://trendapi.tgmeng.com/api/skill/search
Content-Type: application/json
```

Use the fixed production endpoint `https://trendapi.tgmeng.com/api/skill/search`.

## Request

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI", "OpenAI"],
  "mode": "REALTIME",
  "startTime": null,
  "endTime": null,
  "limit": 50
}
```

### Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `license` | string | Yes | Tgmeng universal license code (糖果梦通用密钥) used for authorization. If the user does not have one, direct them to `https://wechat.tgmeng.com`. Never log or reveal it. |
| `keywords` | string[] | Yes | Keyword array. Items are trimmed by the API. Blank items are ignored. |
| `mode` | enum | Yes | One of `REALTIME`, `TODAY`, `HISTORY`. Case-sensitive. |
| `startTime` | string/null | No | Inclusive time window start for `TODAY` and `HISTORY`. Ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`; date-only values are normalized to `00:00:00`. |
| `endTime` | string/null | No | Inclusive time window end for `TODAY` and `HISTORY`. Ignored for `REALTIME`. Accepts `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd`; date-only values are normalized to `23:59:59`. |
| `limit` | integer/null | No | Maximum returned items. `null`, omitted, or `0` means no limit. Negative values are invalid. |

### Modes

| Mode | Data source | Permission | Keyword rule |
| --- | --- | --- | --- |
| `REALTIME` | Current in-memory cache | `SEARCH` | May be empty. |
| `TODAY` | Today's persisted history | `SKILLHISTORY` | May be empty. |
| `HISTORY` | Long-term persisted history | `SKILLHISTORY` | Must contain at least one non-blank keyword. |

### Time Window

`TODAY` and `HISTORY` support optional `startTime` and `endTime` for more precise historical retrieval. When omitted, the API keeps the existing default range for that mode. `REALTIME` always reads the current in-memory cache and ignores time-window fields.

If only `startTime` is provided, the API searches from that time to now. If only `endTime` is provided, the API uses the mode's default start time and the provided end time. `startTime` must be before or equal to `endTime`.

### Matching

Keywords use OR-style fuzzy title matching. A result matches when its title contains at least one keyword.

## Response

All responses use the Tgmeng response envelope:

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
      "endTime": null
    },
    "summary": {
      "total": 120,
      "returned": 50,
      "truncated": true
    },
    "items": []
  }
}
```

`code = 200` means success. For all other codes, treat the call as failed and display or summarize `message`. On success, read results from `data.items`, query metadata from `data.query`, and truncation status from `data.summary`.

### Data Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.query.mode` | enum | Search mode used by the request. |
| `data.query.keywords` | string[] | Sanitized keyword array used by the request. |
| `data.query.permission` | string | Permission checked for this mode: `SEARCH` or `SKILLHISTORY`. |
| `data.query.limit` | integer/null | Requested limit. `null` or `0` means no limit. |
| `data.query.startTime` | string/null | Normalized time-window start used by the query. |
| `data.query.endTime` | string/null | Normalized time-window end used by the query. |
| `data.summary.total` | integer | Matched item count before limit. |
| `data.summary.returned` | integer | Returned item count after limit. |
| `data.summary.truncated` | boolean | Whether limit truncated the result set. |

## Item Fields

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


## Error Contract

Common parameter errors:

| Message | Meaning |
| --- | --- |
| `license empty error` | `license` is missing or blank. |
| `mode empty error` | `mode` is missing or null. |
| `参数mode不支持，请传[REALTIME, TODAY, HISTORY]` | `mode` is not a valid enum value. |
| `参数keywords格式不正确，请传字符串数组` | `keywords` is not a JSON string array. |
| `历史模式 keywords empty error` | `mode = HISTORY` but keywords is empty or blank-only. |
| `limit must be integer` | `limit` is not an integer. |
| `limit must be greater than or equal to 0` | `limit` is negative. |
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
- Server-side diagnostics may record request metadata such as IP address, User-Agent, request path, error message, masked license, and license hash. Full license values must not be logged by agents.
- Do not retry aggressively on authorization failures.
- Do not call `HISTORY` without a keyword.

## Examples

### Realtime

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["AI"],
  "mode": "REALTIME",
  "limit": 50
}
```

### Today

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["机器人", "OpenAI"],
  "mode": "TODAY",
  "startTime": "2026-05-05 00:00:00",
  "endTime": "2026-05-05 12:00:00"
}
```

### History

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["OpenAI"],
  "mode": "HISTORY",
  "startTime": "2026-04-01",
  "endTime": "2026-04-30",
  "limit": 50
}
```






