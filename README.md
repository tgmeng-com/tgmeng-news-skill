# Tgmeng News Skill

这是一个给外部智能体使用的 Tgmeng 站内新闻/热点搜索 Skill 包。

它本身不保存业务数据，也不保存密钥，只负责把 Tgmeng 搜索接口整理成标准、清晰、稳定的调用契约，方便龙虾、Hermes 或其他支持工具调用的智能体接入。

## 能力说明

当前 Skill 暴露的核心能力是：

```text
按密钥、关键词数组、查询模式和可选时间窗口，搜索 Tgmeng 站内新闻和热点数据
```

底层实际调用接口：

```text
POST https://trendapi.tgmeng.com/api/skill/search
```

## 目录结构

```text
.
├── SKILL.md
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
| `agents/openai.yaml` | Skill 的展示信息和默认调用提示。 |
| `references/openapi.yaml` | OpenAPI 3.1 标准接口契约，适合工具系统读取。 |
| `references/api-contract.md` | 中文/人类可读的详细 API 契约说明。 |

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
  "limit": 50
}
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `license` | string | 是 | 糖果梦通用密钥。由调用方传入，Skill 内不保存、不写死。没有密钥时，可前往 `https://wechat.tgmeng.com` 获取。 |
| `keywords` | string[] | 是 | 关键词数组，使用 OR 逻辑模糊匹配标题。 |
| `mode` | enum | 是 | 查询模式，只能是 `REALTIME`、`TODAY`、`HISTORY`。 |
| `startTime` | string/null | 否 | `TODAY` 和 `HISTORY` 的时间窗口开始时间，`REALTIME` 会忽略。支持 `yyyy-MM-dd HH:mm:ss` 或 `yyyy-MM-dd`，仅日期会补为当天 `00:00:00`。 |
| `endTime` | string/null | 否 | `TODAY` 和 `HISTORY` 的时间窗口结束时间，`REALTIME` 会忽略。支持 `yyyy-MM-dd HH:mm:ss` 或 `yyyy-MM-dd`，仅日期会补为当天 `23:59:59`。 |
| `rootCategories` | string[]/string/null | 否 | 按根分类过滤，精确匹配返回结果里的 `items[].rootCategory`。不传或为空表示不过滤。 |
| `limit` | integer/null | 否 | 最大返回条数。`null`、不传或 `0` 表示不限制；负数非法。 |

`rootCategories` 可选值：

```text
新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业
```

调用时传上面这些值，比如 `科技`、`新闻`、`AI`。

## mode 可选值

| mode | 含义 | 数据范围 | 权限要求 | keywords 要求 |
| --- | --- | --- | --- | --- |
| `REALTIME` | 实时数据 | 当前内存缓存热点 | `SEARCH` | 可以为空数组 |
| `TODAY` | 今日数据 | 当天持久化历史数据 | `SKILLHISTORY` | 可以为空数组 |
| `HISTORY` | 历史数据 | 长期持久化历史数据 | `SKILLHISTORY` | 必须至少有一个非空关键词 |

注意：`mode` 只支持英文大写值，不支持中文。

## 分类过滤

可以用 `rootCategories` 缩小结果范围。根分类过滤和关键词匹配是 AND 关系；传多个根分类值时是 OR 关系。

为了兼容不同调用方，接口也接受单个字符串形式，并兼容 `rootCategory`、`platformCategoryRoot` 这些字段名。

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
| `data.query` | 本次查询信息，包括 `mode`、`keywords`、`permission`、`limit`、`startTime`、`endTime`、`rootCategories`。 |
| `data.summary` | 结果统计信息，包括 `total`、`returned`、`truncated`。 |
| `data.items` | 搜索结果数组。 |
| `data.items[].title` | 新闻或热点标题。 |
| `data.items[].url` | 来源链接，部分数据可能为空。 |
| `data.items[].source` | 来源平台名称。 |
| `data.items[].category` | 平台分类。 |
| `data.items[].rootCategory` | 平台根分类。 |
| `data.items[].publishedAt` | 数据更新时间。 |
| `data.items[].rank` | 来源榜单排序。 |
| `data.items[].simHash` | 历史数据相似度哈希，部分模式可能为空。 |

## 常见错误

| message | 说明 |
| --- | --- |
| `license empty error` | 密钥为空。 |
| `mode empty error` | 查询模式为空。 |
| `参数mode不支持，请传[REALTIME, TODAY, HISTORY]` | `mode` 不在允许值中。 |
| `参数keywords格式不正确，请传字符串数组` | `keywords` 不是字符串数组。 |
| `limit must be integer` | `limit` 不是整数。 |
| `limit must be greater than or equal to 0` | `limit` 是负数。 |
| `rootCategories must be string array or string` | `rootCategories` 不是字符串数组或字符串。 |
| `rootCategories unsupported, available values: 新闻, 羊毛, 媒体, 电视, 生活, 社区, 财经, 股讯, 体育, 科技, 设计, 影音, 游戏, 健康, 教育, 期货, AI, 副业` | `rootCategories` 包含不支持的分类。 |
| `历史模式 keywords empty error` | `HISTORY` 模式下没有传有效关键词。 |
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
  "limit": 50
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
  "endTime": "2026-05-05 12:00:00"
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
  "limit": 50
}
```

## 给智能体的调用建议

智能体调用前应先做本地参数校验：

1. `license` 必须是非空字符串。
2. `keywords` 必须是字符串数组。
3. `mode` 必须是 `REALTIME`、`TODAY` 或 `HISTORY`。
4. 当 `mode = HISTORY` 时，`keywords` 必须至少包含一个非空字符串。
5. 当用户指定非实时的时间范围时，优先使用 `startTime` 和 `endTime` 缩小查询范围。
6. 当用户要求某类新闻、热点或平台范围时，优先使用 `rootCategories`。
7. 不要记录、展示或泄露完整 `license`。
8. 权限失败时不要无限重试，应把错误信息反馈给调用者。

## 安全要求

- 不要把用户密钥写死在 Skill 文件里。
- 不要在日志或回复中输出完整密钥。
- 不要把密钥提交到 Git 仓库。
- 外部智能体应通过运行时参数、环境变量或用户输入传入密钥。没有密钥时，引导用户前往 `https://wechat.tgmeng.com` 获取糖果梦通用密钥。
- 接口会记录调用排查信息，包括访问 IP、User-Agent、请求路径、错误信息和密钥。
- `HISTORY` 查询可能较重，必须带关键词。

## 相关文件

- `SKILL.md`：Skill 主入口说明。
- `references/openapi.yaml`：OpenAPI 3.1 标准定义。
- `references/api-contract.md`：详细接口契约。






