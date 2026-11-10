---
name: tgmeng-news-skill
version: 3.0.0
updated_at: 2026-11-10
update_check_url: https://raw.githubusercontent.com/tgmeng-com/tgmeng-news-skill/main/skill-version.json
description: Search Tgmeng/糖果梦 hotspots and news, procurement and tender notices/招采公告, Tgmeng Index/糖果指数, and generated daily reports/日报 through the Tgmeng Open Platform. Use for current/today/history news searches, keyword or time-window queries, government procurement and bidding notices, AI rankings, and report Markdown. Authenticate with an Open Platform API Key from account.tgmeng.com/developer.
---

# Tgmeng News Skill v3

Tgmeng / 糖果梦开放平台 Skill for external agents. It exposes hotspot search, procurement notices, Tgmeng Index and daily reports through the new Open Platform API.

This version is a **breaking change** from v2:

- Authentication uses an Open Platform API Key, not a legacy License Code.
- Endpoints moved from `trendapi.tgmeng.com/api/skill/*` to `api-open.tgmeng.com/v1/*`.
- Image/file upload was removed. Do not call or describe the old image bed.

## Authentication

Every request uses:

```http
Authorization: Bearer tgm_open_xxxxxxxxxxxxxxxx
Content-Type: application/json
```

The key is created in the account center: `https://account.tgmeng.com/developer`.

- One account has one site-wide key.
- The full key is shown only when it is generated or reset.
- Resetting invalidates the previous key immediately.
- Never log, print, commit, or expose the complete key.

Base URL:

```text
https://api-open.tgmeng.com
```

Do not send a `license` field in JSON bodies.

## Version Check

Read `version` from this file's frontmatter. When network access is available, fetch `update_check_url` and compare its `latestVersion`.

- If the local version is missing or older, tell the user an update is available before relying on v3 behavior.
- Do not silently switch back to old endpoints or license authentication.
- If the version check fails, continue with the local version and mention that the check was skipped. A failed check must not block a valid API call.

## Capability Selection

| User intent | Endpoint |
|---|---|
| Hotspots, news, keywords, today/history | `POST /v1/hot/search` |
| Procurement, bidding, government purchase, tender notices | `POST /v1/procurement/search` |
| Tgmeng Index, AI-ranked list, hotScore | `POST /v1/candy/search` |
| Daily reports, Markdown content, report metadata | `POST /v1/daily/search` |

If the request is ambiguous, ask one short clarification before calling an endpoint.

## Common Response

Successful responses share this wrapper:

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

- `summary.total` is the total count known to the endpoint.
- `summary.returned` is the number of items in this page.
- `summary.hasMore` indicates whether a later page may exist.
- `requestId` is useful when reporting an API problem.

Errors use:

```json
{
  "requestId": "req_x",
  "error": {
    "code": "UNAUTHORIZED",
    "message": "API Key 缺失或无效"
  }
}
```

## Hotspot Search

`POST /v1/hot/search`

```json
{
  "mode": "REALTIME",
  "keywords": [],
  "rootCategories": [],
  "startTime": null,
  "endTime": null,
  "limit": 50,
  "offset": 0,
  "distinct": false
}
```

- `mode`: `REALTIME`, `TODAY`, or `HISTORY`.
- `REALTIME` reads the current snapshot and ignores time parameters.
- `TODAY` is limited to the current day; `startTime`/`endTime` are optional.
- `HISTORY` requires `startTime` and `endTime` and must stay inside one calendar month.
- `TODAY`/`HISTORY` require at least one keyword.
- `limit`: 1..500; default 500.
- `offset`: 0..5000; history offset max 2000.
- Keywords are ORed. Inside one expression, `+` means include and `-`/`!` means exclude, e.g. `伊朗+导弹-足球`.
- `distinct: true` removes duplicate titles.

Important item fields:

```json
{
  "id": "weibo:xxx",
  "title": "标题",
  "url": "https://...",
  "source": "微博",
  "platformId": "weibo",
  "channelId": "main",
  "category": "媒体",
  "rootCategory": "新闻",
  "rank": 1,
  "hotScore": "123456",
  "publishedAt": "2026-11-07 10:00:00",
  "observedAt": "2026-11-07T02:00:00Z"
}
```

## Procurement Search

`POST /v1/procurement/search`

Supported request fields:

- `mode`: `REALTIME`, `TODAY`, `HISTORY`
- `keywords`, `sourceCodes`, `sourceGroups`, `recordFamilies`, `regions`, `provinces`, `cities`, `rawTypes`
- `projectNumber`, `purchaser`, `agency`
- `startTime`, `endTime`, `publishedFrom`, `publishedTo`, `deadlineFrom`, `deadlineTo`
- `amountMin`, `amountMax`, `sort`, `limit`, `offset`, `distinct`

Sort values: `PUBLISHED_AT_DESC`, `PUBLISHED_AT_ASC`, `DEADLINE_ASC`, `DEADLINE_DESC`, `AMOUNT_DESC`, `AMOUNT_ASC`.

Rules:

- `TODAY`/`HISTORY` require a time window; `HISTORY` must stay in one calendar month.
- `REALTIME` may omit keywords and returns each source's latest available data.
- `limit`: 1..500; default 100; `offset`: 0..5000.
- `data.sourceStatuses` reports each source's `collectionStatus`, `dataScope`, `latestPublicationDate`, `dataCount`, `dataUpdateTime`, and `lastCollectionTime`.
- `REALTIME` data from a source may represent an earlier publication date; preserve `items[].publishedAt` and do not call fallback data "today's data".
- A source with no notice today is not necessarily a failure. Read `collectionStatus` and `dataScope`.

Key item fields: `id`, `title`, `url`, `source`, `sourceCode`, `sourceGroup`, `recordFamily`, `originalType`, `region`, `province`, `city`, `projectNumber`, `purchaser`, `agency`, `amount`, `currency`, `publishedAt`, `deadlineAt`, `observedAt`, `sourceAttributes`.

Unsupported filters in the current version: `sourceLevels`, `sourceChannels`, `lifecycleStages`, `noticeActions`, `outcomeStatuses`, `domains`, `industries`, `subjectTypes`, `procurementMethods`, `transactionMethods`, `procurementChannels`, `specialTopics`, `districts`, `rawCategories`, `rawTypeCodes`, `organization`. Sending them returns `400 UNSUPPORTED_FILTER`.

## Tgmeng Index

`POST /v1/candy/search`

```json
{
  "keywords": [],
  "categories": ["technology", "ai"],
  "limit": 50,
  "offset": 0,
  "distinct": false
}
```

- `categories`: `all`, `dubang`, `alltalk`, `technology`, `finance`, `entertainment`, `car`, `sports`, `game`, `livelihood`, `ai`, `sideline`, `education`.
- `limit`: 1..500; `offset`: 0..5000.
- Item fields: `id`, `title`, `source`, `category`, `rootCategory`, `hotScore`, `publishedAt`.
- `hotScore` is the Tgmeng Index heat value; larger means hotter.

## Daily Reports

`POST /v1/daily/search`

```json
{
  "date": "2026-11-06",
  "category": "technology",
  "startDate": null,
  "endDate": null,
  "limit": 20,
  "offset": 0,
  "withContent": false
}
```

- `date` takes priority over `startDate`/`endDate`.
- `category` is optional; omit it for all categories.
- `limit`: 1..50; default 20; `offset`: 0..5000.
- `withContent: true` adds `contentMarkdown` to each item. Use `false` for list/meta-only responses.
- Item fields: `id`, `title`, `summary`, `category`, `categoryName`, `reportDate`, `coverImageUrl`, `tags`, `aiPlatforms`, `aiModels`, `aiFroms`, `totalTokens`, `hotCount`, `generatedAt`, `readMinutes`, and optional `contentMarkdown`.

## Limits and Errors

Every response carries rate-limit headers such as:

```text
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 119
X-RateLimit-Reset: 1762345678
X-RateLimit-Scope: feature
```

Common errors:

| HTTP | code | Meaning |
|---|---|---|
| 400 | `INVALID_MODE` / `INVALID_LIMIT` / `INVALID_OFFSET` / `INVALID_WINDOW` / `INVALID_DATE` / `INVALID_KEYWORD` / `INVALID_PARAM` / `UNSUPPORTED_FILTER` | Invalid request |
| 401 | `UNAUTHORIZED` | Missing or invalid API Key |
| 403 | `FORBIDDEN` | Key disabled/expired, or plan does not allow the feature |
| 429 | `RATE_LIMITED` | Per-minute limit exceeded |
| 429 | `QUOTA_EXCEEDED` | Daily quota exhausted |

Use `Retry-After` when present. Do not aggressively retry 401/403; report the problem to the user.

## Agent Behavior Rules

1. Never ask for, store, log, or print a legacy License Code. Ask the user for an Open Platform API Key only if it is not already configured.
2. Read the key from the agent's secure configuration, never from a repository or prompt.
3. Use `limit` conservatively for exploratory calls; do not fetch large pages unless the user asked for them.
4. For `HISTORY`, confirm the time range is inside one calendar month.
5. Prefer `withContent: false` for daily list queries; request content only when the user needs Markdown.
6. Do not claim a procurement source with fallback/older data is today's data.
7. Report `requestId` and the error code when an API call fails.
8. Image/file upload is not part of this Skill. Do not suggest the old `image.tgmeng.com` service.

## Migration from v2

| v2 | v3 |
|---|---|
| `POST /api/skill/search` | `POST /v1/hot/search` |
| `POST /api/skill/procurement/search` | `POST /v1/procurement/search` |
| `POST /api/skill/index` | `POST /v1/candy/search` |
| `POST /api/skill/dailyNews` | `POST /v1/daily/search` |
| `license` in JSON | `Authorization: Bearer tgm_open_...` |
| Image bed upload | Removed |
