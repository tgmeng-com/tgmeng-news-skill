# Tgmeng News Skill

这是一个给外部智能体使用的 Tgmeng 站内新闻/热点搜索 Skill 包。

它本身不保存业务数据，也不保存密钥，只负责把 Tgmeng 搜索接口整理成标准、清晰、稳定的调用契约，方便龙虾、Hermes 或其他支持工具调用的智能体接入。

## 能力说明

当前 Skill 暴露两个核心能力：

```text
1. 热榜数据搜索：按密钥、关键词数组、查询模式和可选时间窗口，搜索 Tgmeng 站内新闻和热点数据
2. 糖果指数查询：按密钥、关键词和分类，查询糖果指数榜单
```

底层实际调用接口：

```text
POST https://trendapi.tgmeng.com/api/skill/search
POST https://trendapi.tgmeng.com/api/skill/index
```

## 目录结构

```text
.
├── SKILL.md
├── skill-version.json
├── agents/
│   └── openai.yaml
└── references/
    ├── api-contract.md
    └── openapi.yaml
```

文件说明：

| 文件 | 作用 |
| --- | --- |
| `SKILL.md` | Skill 主说明文件，给 Codex/智能体读取使用。 |
| `skill-version.json` | 远程版本清单，给智能体在执行 Skill 前检查本地版本是否落后，并展示新版功能和使用建议。 |
| `agents/openai.yaml` | Skill 的展示信息和默认调用提示。 |
| `references/openapi.yaml` | OpenAPI 3.1 标准接口契约，适合工具系统读取。 |
| `references/api-contract.md` | 中文/人类可读的详细 API 契约说明。 |

## 版本检查

`SKILL.md` 的 frontmatter 中声明了当前本地版本和远程版本清单地址：

```yaml
version: 1.1.1
updated_at: 2026-05-08
update_check_url: https://raw.githubusercontent.com/tgmeng-com/tgmeng-news-skill/main/skill-version.json
```

智能体每次执行 Skill 前，如果网络可用，应读取 `update_check_url` 指向的 `skill-version.json`，比较本地 `version` 和远程 `latestVersion`。

## 安装目录与缓存

为避免部分 agent 框架出现 `skills_list` 可见但 `skill_view` 偶发 `Skill not found` 的缓存问题，安装目录建议固定为：

```text
tgmeng-news-skill
```

目录名应与 `SKILL.md` 的 `name: tgmeng-news-skill`、`skill-version.json` 的 `name` 保持一致。更新 Skill 后，如果 agent 仍读取旧索引，请刷新/重启 agent 的 skill 缓存后再调用。

- 如果本地版本不存在，视为需要更新。
- 如果远程 `latestVersion` 大于本地 `version`，提醒用户本地 Skill 已落后。
- 提醒时展示远程版本、更新时间、`releaseNotes`、`agentUsageTips` 和 `breakingChanges`。
- 提醒后询问用户是否先更新 Skill；如果用户不更新，再继续使用本地版本。
- 如果版本检查因网络不可用或远程清单读取失败而失败，简短说明检查失败，然后继续使用本地版本，除非用户明确要求先更新。

## 请求参数

请求体包含以下业务参数：

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

字段说明：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `license` | string | 是 | 糖果梦通用密钥。由调用方传入，Skill 内不保存、不写死。没有密钥时，可前往 `https://wechat.tgmeng.com` 获取。 |
| `keywords` | string[] | 是 | 关键词数组。多个关键词项之间保持 OR；单个关键词项内支持 `+`、`＋`、`&`、`＆` 表示必须包含，`-`、`－`、`!`、`！` 表示排除。`REALTIME` 可以为空数组，`TODAY` 和 `HISTORY` 必须至少包含一个非空关键词。 |
| `mode` | enum | 是 | 查询模式，只能是 `REALTIME`、`TODAY`、`HISTORY`。 |
| `startTime` | string/null | 否 | `TODAY` 和 `HISTORY` 的时间窗口开始时间，`REALTIME` 会忽略。支持 `yyyy-MM-dd HH:mm:ss` 或 `yyyy-MM-dd`，仅日期会补为当天 `00:00:00`。 |
| `endTime` | string/null | 否 | `TODAY` 和 `HISTORY` 的时间窗口结束时间，`REALTIME` 会忽略。支持 `yyyy-MM-dd HH:mm:ss` 或 `yyyy-MM-dd`，仅日期会补为当天 `23:59:59`。 |
| `rootCategories` | string[]/string/null | 否 | 按根分类过滤，精确匹配返回结果里的 `items[].rootCategory`。不传或为空表示不过滤。 |
| `limit` | integer/null | 否 | 最大返回条数。默认传 `null` 表示不限制；接口也兼容不传或 `0` 表示不限制；负数非法。智能体不要自行添加、推断或缩小为具体数字，只有用户明确指定数量或确认限制条数后才可以传具体数字。 |
| `offset` | integer/null | 否 | 结果偏移量，默认 `0`。配合用户确认过的 `limit` 获取下一批数据，例如 `limit: 100, offset: 100` 表示从第 101 条命中结果开始返回；负数非法。 |
| `distinct` | boolean/null | 否 | 是否按标题去重，默认 `false`。为 `true` 时，在关键词、分类、时间过滤之后、分页之前按规范化标题去重，保留当前排序中最靠前的一条；不使用 `simHash`。 |

`rootCategories` 可选值：

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

调用时传上面这些值，比如 `科技`、`新闻`、`AI`。

## mode 可选值

| mode | 含义 | 数据范围 | 权限要求 | keywords 要求 |
| --- | --- | --- | --- | --- |
| `REALTIME` | 实时数据 | 当前内存缓存热点 | `SEARCH` | 可以为空数组 |
| `TODAY` | 今日数据 | 当天持久化历史数据 | `SKILLHISTORY` | 必须至少有一个非空关键词 |
| `HISTORY` | 历史数据 | 长期持久化历史数据 | `SKILLHISTORY` | 必须至少有一个非空关键词 |

注意：`mode` 只支持英文大写值，不支持中文。

密钥默认具有 `REALTIME` 查询所需的 `SEARCH` 权限。如果需要调用 `TODAY` 或 `HISTORY`，请联系管理员说明使用场景，审核后开启 `SKILLHISTORY` 权限。

## 分类过滤

可以用 `rootCategories` 缩小结果范围。根分类过滤和关键词匹配是 AND 关系；传多个根分类值时是 OR 关系。

为了兼容不同调用方，接口也接受单个字符串形式，并兼容 `rootCategory`、`platformCategoryRoot` 这些字段名。

## 关键词表达式

热榜数据搜索和糖果指数查询的 `keywords` 都兼容旧 OR 逻辑，同时支持单个关键词项内的轻量表达式：

- 多个关键词项之间仍然是 OR，例如 `["伊朗", "股票"]` 表示标题包含“伊朗”或“股票”。
- `+`、`＋`、`&`、`＆` 表示必须包含，例如 `黄金+伊朗` 表示标题同时包含“黄金”和“伊朗”。
- `-`、`－`、`!`、`！` 表示排除，例如 `黄金-伊朗` 表示标题包含“黄金”且不包含“伊朗”。
- 示例：`伊朗+导弹-足球` 表示标题同时包含“伊朗”和“导弹”，并排除“足球”。
- 示例：`股票＆黄金！AI` 表示标题同时包含“股票”和“黄金”，并排除“AI”。

## 时间窗口

当 `mode` 不是 `REALTIME` 时，可以传 `startTime` 和 `endTime` 来缩小查询范围。这个能力适合用户明确想查某个小时、某半天、某几天内的热点时使用。

- 只传 `startTime`：从开始时间查到当前时间。
- 只传 `endTime`：使用当前模式默认开始时间，查到结束时间。
- 两个都传：按指定闭区间查询。
- `startTime` 必须早于或等于 `endTime`。

## 返回格式

接口返回 Tgmeng 标准响应包：

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
        "title": "新闻标题",
        "url": "https://example.com/news",
        "source": "平台名称",
        "category": "平台分类",
        "rootCategory": "根分类",
        "publishedAt": "2026-05-03 01:00:00",
        "rank": 1,
        "simHash": 123456789
      }
    ]
  }
}
```

返回字段说明：

| 字段 | 说明 |
| --- | --- |
| `code` | 业务状态码，`200` 表示成功。 |
| `message` | 返回信息或错误提示。 |
| `data` | 智能体友好的搜索结果对象，包含 `query`、`summary`、`items`。 |
| `data.query` | 本次查询信息，包括 `mode`、`keywords`、`permission`、`limit`、`offset`、`distinct`、`startTime`、`endTime`、`rootCategories`。 |
| `data.summary` | 结果统计信息，包括 `total`、`rawTotal`、`returned`、`limit`、`offset`、`duplicatesRemoved`、`hasMore`、`truncated`。 |
| `data.summary.total` | 当前返回策略下的命中总数。`distinct=true` 时是去重后的总数；否则等于 `rawTotal`。 |
| `data.summary.rawTotal` | 按关键词、分类和时间过滤后的原始命中总数，去重前统计。 |
| `data.summary.returned` | 本次实际返回条数。 |
| `data.summary.limit` | 本次请求的最大返回条数，`null` 或 `0` 表示不限制。 |
| `data.summary.offset` | 本次请求的结果偏移量。 |
| `data.summary.duplicatesRemoved` | `distinct=true` 时去掉的重复标题数量；未开启去重时为 `0`。 |
| `data.summary.hasMore` | 是否还有后续结果；为 `true` 时可保持同样条件并增大 `offset` 继续获取。 |
| `data.summary.truncated` | 兼容旧字段，含义与 `hasMore` 一致。 |
| `data.items` | 搜索结果数组，按更新时间从最新到最久排列；越靠前的结果越新，而不是按照热点热度权重排序，方便智能体知道结果的排序逻辑。 |
| `data.items[].title` | 新闻或热点标题。 |
| `data.items[].url` | 来源链接，部分数据可能为空。 |
| `data.items[].source` | 来源平台名称。 |
| `data.items[].category` | 平台分类。 |
| `data.items[].rootCategory` | 平台根分类。 |
| `data.items[].publishedAt` | 数据更新时间。 |
| `data.items[].rank` | 来源榜单排序。 |
| `data.items[].simHash` | 历史数据相似度哈希，部分模式可能为空。 |

## 糖果指数查询

糖果指数接口用于查询糖果指数榜单，适合查看综合热点、分类热点和对应热度指数。

接口地址：

```text
POST https://trendapi.tgmeng.com/api/skill/index
```

请求示例：

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

字段说明：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `license` | string | 是 | 糖果梦通用密钥。由调用方传入，Skill 内不保存、不写死。 |
| `keywords` | string[]/string/null | 否 | 关键词过滤。支持字符串数组或单个字符串；为空表示不过滤标题。多个关键词项之间保持 OR；单个关键词项内支持 `+`、`＋`、`&`、`＆` 表示必须包含，`-`、`－`、`!`、`！` 表示排除。 |
| `categories` | string[]/string/null | 否 | 糖果指数分类过滤。也兼容 `category`、`type`、`platformCategory`、`分类` 这些入参名；为空表示查询全部分类。 |
| `limit` | integer/null | 否 | 最大返回条数。默认传 `null` 表示不限制；接口也兼容不传或 `0` 表示不限制；负数非法。智能体不要自行添加、推断或缩小为具体数字，只有用户明确指定数量或确认限制条数后才可以传具体数字。 |
| `offset` | integer/null | 否 | 结果偏移量，默认 `0`。配合用户确认过的 `limit` 获取下一批数据，例如 `limit: 100, offset: 100` 表示从第 101 条命中结果开始返回；负数非法。 |
| `distinct` | boolean/null | 否 | 是否按标题去重，默认 `false`。为 `true` 时，在关键词、分类过滤之后、分页之前按规范化标题去重，保留当前排序中最靠前的一条；不使用 `simHash`。 |

`categories` 可选值：

```text
all, dubang, alltalk, technology, finance, entertainment, car, sports, game, livelihood, ai, sideline, education
```

常见别名：

| 分类值 | 含义 / 常见别名 |
| --- | --- |
| `all` | 综合、糖果梦综合 |
| `dubang` | 毒榜、糖果梦毒榜 |
| `alltalk` | 争议榜、争议、糖果梦争议榜 |
| `technology` | 科技、tech、糖果梦科技 |
| `finance` | 财经、糖果梦财经 |
| `entertainment` | 娱乐、糖果梦娱乐 |
| `car` | 汽车、糖果梦汽车 |
| `sports` | 体育、sport、糖果梦体育 |
| `game` | 游戏、糖果梦游戏 |
| `livelihood` | 民生、糖果梦民生 |
| `ai` | AI、人工智能、糖果梦AI |
| `sideline` | 副业、fuye、糖果梦副业 |
| `education` | 教育、jiaoyu、糖果梦教育 |

未知分类会被忽略；没有有效分类时，接口会查询全部糖果指数分类。

返回示例：

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
        "title": "热点标题",
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

糖果指数返回字段说明：

| 字段 | 说明 |
| --- | --- |
| `data.query.mode` | 固定为 `INDEX`，表示糖果指数查询。 |
| `data.query.keywords` | 本次实际使用的关键词数组。 |
| `data.query.permission` | 本次接口校验的权限，通常为 `SEARCH`。 |
| `data.query.limit` | 本次请求的最大返回条数。`null` 或 `0` 表示不限制。 |
| `data.query.offset` | 本次请求的结果偏移量，默认 `0`。 |
| `data.query.distinct` | 是否按标题去重。 |
| `data.query.categories` | 本次实际使用的糖果指数分类。 |
| `data.summary` | 结果统计信息，包括 `total`、`rawTotal`、`returned`、`limit`、`offset`、`duplicatesRemoved`、`hasMore`、`truncated`。 |
| `data.summary.total` | 当前返回策略下的命中总数。`distinct=true` 时是去重后的总数；否则等于 `rawTotal`。 |
| `data.summary.rawTotal` | 按关键词和分类过滤后的原始命中总数，去重前统计。 |
| `data.summary.returned` | 本次实际返回条数。 |
| `data.summary.limit` | 本次请求的最大返回条数，`null` 或 `0` 表示不限制。 |
| `data.summary.offset` | 本次请求的结果偏移量。 |
| `data.summary.duplicatesRemoved` | `distinct=true` 时去掉的重复标题数量；未开启去重时为 `0`。 |
| `data.summary.hasMore` | 是否还有后续结果；为 `true` 时可保持同样条件并增大 `offset` 继续获取。 |
| `data.summary.truncated` | 兼容旧字段，含义与 `hasMore` 一致。 |
| `data.items` | 糖果指数结果数组。 |
| `data.items[].title` | 热点标题。 |
| `data.items[].source` | 固定为 `糖果指数`。 |
| `data.items[].category` | 糖果指数分类名称。 |
| `data.items[].rootCategory` | 根分类，通常为 `糖果梦`。 |
| `data.items[].publishedAt` | 该指数榜单的更新时间。 |
| `data.items[].hotScore` | 糖果指数热度值，数值越大表示热度越高。 |

## 常见错误

| message | 说明 |
| --- | --- |
| `license empty error` | 密钥为空。 |
| `mode empty error` | 查询模式为空。 |
| `参数mode不支持，请传[REALTIME, TODAY, HISTORY]` | `mode` 不在允许值中。 |
| `参数keywords格式不正确，请传字符串数组` | `keywords` 不是字符串数组。 |
| `limit must be integer` | `limit` 不是整数。 |
| `limit must be greater than or equal to 0` | `limit` 是负数。 |
| `offset must be integer` | `offset` 不是整数。 |
| `offset must be greater than or equal to 0` | `offset` 是负数。 |
| `distinct must be boolean` | `distinct` 不是布尔值。 |
| `rootCategories must be string array or string` | `rootCategories` 不是字符串数组或字符串。 |
| `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业` | `rootCategories` 包含不支持的分类。 |
| `category must be string array or string` | 糖果指数分类参数不是字符串数组或字符串。 |
| `TODAY/HISTORY mode keywords empty error` | `TODAY` 或 `HISTORY` 模式下没有传有效关键词。 |
| `startTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd` | `startTime` 格式错误。 |
| `endTime format error, expected yyyy-MM-dd HH:mm:ss or yyyy-MM-dd` | `endTime` 格式错误。 |
| `startTime must be before or equal to endTime` | 开始时间晚于结束时间。 |
| 授权码校验异常 | 密钥无效、过期或缺少对应权限。 |

注意：业务错误通常也可能以 HTTP 200 返回，所以调用方必须检查响应体里的 `code` 和 `message`。

## 调用示例

### 实时搜索

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

### 今日搜索

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["OpenAI", "机器人"],
  "mode": "TODAY",
  "rootCategories": ["科技"],
  "startTime": "2026-05-05 00:00:00",
  "endTime": "2026-05-05 12:00:00",
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

### 历史搜索

```json
{
  "license": "USER_LICENSE_CODE",
  "keywords": ["OpenAI"],
  "mode": "HISTORY",
  "rootCategories": ["AI"],
  "startTime": "2026-04-01",
  "endTime": "2026-04-30",
  "limit": null,
  "offset": 0,
  "distinct": false
}
```

## 给智能体的调用建议

### 能力选择

当用户明确说“糖果指数”“指数榜”“热度指数”“AI 汇总榜”“综合热度”时，优先使用糖果指数接口 `/api/skill/index`。

当用户明确说“热搜原始数据”“新闻热点”“查某关键词”“查今天或历史”“按时间窗口检索”时，优先使用热榜数据搜索接口 `/api/skill/search`。

当用户需求不明确时，例如只说“查热点”“看看热榜”“最近有什么热搜”，智能体应先询问用户要使用哪种能力：

1. 热榜数据搜索：查询原始新闻/热点数据，支持实时、今日、历史、关键词、根分类和时间窗口。
2. 糖果指数查询：查询糖果指数榜单，返回标题、分类和 `hotScore`，适合按综合热度查看热点。

用户确认后再调用对应接口。

### limit 规则

智能体不要自行添加、推断或缩小为具体数字。如果用户没有明确要求返回条数，默认传 `"limit": null` 表示不限制，不要自行改成具体数字。

如果智能体因为性能、可读性、上下文长度或摘要需要想添加 `limit`，必须先询问用户并获得确认。

只有在以下情况才可以把 `limit` 设为具体数字：

- 用户明确指定数量，例如“前 10 条”“返回 20 条”“只要 5 条”；
- 或用户确认了智能体关于限制条数的追问。

### offset 分页规则

默认传 `"offset": 0`。当响应里的 `data.summary.hasMore` 为 `true`，并且用户需要继续获取剩余结果时，保持其他查询条件不变，把 `offset` 增加本次 `returned` 或已确认的 `limit` 后再次调用。

### distinct 去重规则

默认传 `"distinct": false`。当用户希望减少重复标题、通稿转载或 token 浪费时，可以传 `"distinct": true`。接口会在分页前按标题去重，保留当前排序中最靠前的一条；`data.summary.rawTotal` 是去重前数量，`data.summary.total` 是去重后数量。

### 热榜数据搜索

智能体调用前应先做本地参数校验：

1. `license` 必须是非空字符串。
2. `keywords` 必须是字符串数组。
3. `mode` 必须是 `REALTIME`、`TODAY` 或 `HISTORY`。
4. 当 `mode = TODAY` 或 `mode = HISTORY` 时，`keywords` 必须至少包含一个非空字符串。
5. 当用户指定非实时的时间范围时，优先使用 `startTime` 和 `endTime` 缩小查询范围。
6. 当用户要求某类新闻、热点或平台范围时，优先使用 `rootCategories`。
7. 不要记录、展示或泄露完整 `license`。
8. 权限失败时不要无限重试，应把错误信息反馈给调用者。

## 安全要求

- 不要把用户密钥写死在 Skill 文件里。
- 不要在日志或回复中输出完整密钥。
- 不要把密钥提交到 Git 仓库。
- 外部智能体应通过运行时参数、环境变量或用户输入传入密钥。没有密钥时，引导用户前往 `https://wechat.tgmeng.com` 获取糖果梦通用密钥。
- 密钥默认只具备 `REALTIME` 查询权限；需要 `TODAY` 或 `HISTORY` 权限时，联系管理员说明情况后开启。
- 接口会记录调用排查信息，包括访问 IP、User-Agent、请求路径、错误信息和密钥。
- `TODAY` 和 `HISTORY` 查询必须带关键词；`HISTORY` 查询可能较重。

## 相关文件

- `SKILL.md`：Skill 主入口说明。
- `references/openapi.yaml`：OpenAPI 3.1 标准定义。
- `references/api-contract.md`：详细接口契约。
