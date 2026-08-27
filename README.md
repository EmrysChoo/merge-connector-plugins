# Merge Declarative Connector Specification

Merge Connector 是一种**数据描述文件**，用于把没有 RSS 输出的网页或 JSON API 映射为 Merge 可以订阅的文章。插件不包含、下载或执行 JavaScript、Wasm、原生代码或其他可执行代码。

当前规范版本：`2`（完全兼容 v1）

## 能力边界与状态

| 能力 | 当前状态 | 说明 |
| --- | --- | --- |
| 发现订阅对象 | 已支持 | 仅用于作者、频道、播放列表、播客、仓库、合集或订阅源；不搜索文章。 |
| 订阅前预览 | 已支持 | 搜索结果可打开独立预览，查看源信息和最近内容，再决定是否订阅。 |
| 内容与媒体提取 | 已支持 | 支持正文、图片、视频、音频、作者、摘要与发布日期。 |
| 增量去重 | 已支持 | `externalID` 是稳定去重键；不要使用标题或发布时间作为 ID。 |
| 源级过滤 | 已支持 | `defaultFilters` 在订阅时转为可编辑的源级屏蔽规则。 |
| 登录与 Cookie | 已支持 | 仅宿主控制的 OAuth / 临时网页登录，敏感数据保存到 Keychain。 |
| 运行诊断 | 已支持 | App 记录操作结果、数量、耗时与错误摘要，不记录输入或响应内容。 |
| 评论提取、插件目录 | 规划中 | 当前请使用 App 的评论提取规则；插件不得声明页面自动化或脚本。 |

## 设计目标

- 让网站通过 HTML/CSS Selector 或 JSON Path 输出统一文章数据。
- 所有网络请求由 Merge 发起并执行权限检查。
- 插件不能读取文件、Keychain、剪贴板、位置、相册或其他系统数据；仅可由宿主代为使用明确声明的本地配置值。
- 安装前向用户展示发布者和允许访问的域名。
- 插件数据与 App 的文章模型解耦，不要求返回 RSS。
- 需要登录的服务由 App 托管授权和 Keychain 保存，插件不能读取 Cookie、Token 或系统 API。

## App Store 边界

本规范把远程插件限制为声明式数据，不提供通用解释器和原生 API 桥接。设计参考 Apple App Review Guidelines 2.5.2 与 4.7：

https://developer.apple.com/app-store/review/guidelines/

这降低了远程代码执行风险，但不能保证任何具体版本一定通过审核。提交审核时应在 Review Notes 中说明：

1. 插件仅是 JSON 数据映射规则。
2. App 不执行插件提供的代码。
3. App 控制全部网络请求、域名权限、响应大小和超时。
4. 插件不能扩展或调用原生平台 API。

## 发布结构

建议通过不可变的 GitHub Release 发布：

```text
plugin.json
README.md
LICENSE
```

用户在 Merge 中粘贴 `plugin.json` 的 HTTPS 下载地址。不要让安装链接指向会被静默改写的开发分支；每个版本应创建独立 Release，并保留旧版本以便回滚。

推荐同时发布一个固定版本的 `plugin.json` 和面向人阅读的 Release Notes。App 的“更新插件”会重新读取安装时保存的清单地址；更新不可更换插件 ID，也不应扩大未向用户解释的域名权限。

## 最小清单

```json
{
  "$schema": "https://example.com/schema/plugin.schema.json",
  "schemaVersion": 1,
  "runtime": "declarative-feed",
  "id": "com.example.connector",
  "name": "Example",
  "version": "1.0.0",
  "description": "Example connector",
  "publisher": { "name": "Example Author" },
  "permissions": {
    "domains": ["api.example.com"],
    "maximumResponseBytes": 1048576
  },
  "input": {
    "type": "username",
    "title": "用户名",
    "placeholder": "输入用户名",
    "encoding": "urlQuery"
  },
  "request": {
    "url": "https://api.example.com/posts?user={{input}}",
    "method": "GET"
  },
  "response": {
    "format": "json",
    "source": { "title": { "constant": "Example" } },
    "items": {
      "jsonPath": "items",
      "externalID": { "jsonPath": "id", "required": true },
      "url": { "jsonPath": "url", "required": true },
      "title": { "jsonPath": "title", "required": true }
    }
  }
}
```

完整字段约束见 [`schema/plugin.schema.json`](schema/plugin.schema.json)。v1 插件继续可用；声明 `authentication` 或 `configuration` 时必须使用 v2。

## 从发现到订阅的生命周期

1. 用户选择插件并输入查询词、用户名、主页或频道 ID。
2. Merge 渲染 `{{input}}` 与允许的 `{{config.fieldID}}`，检查目标域名后发起 GET 请求。
3. 插件将响应映射为源或订阅对象；若声明 `subscription`，结果行显示单独的订阅按钮。
4. 用户点结果行可预览源与最近内容；点订阅后，`feedURL` 中的 `{{externalID}}` 被替换为该对象的稳定 ID。
5. 后续刷新按文章 `externalID` 去重；源级过滤只作用于该订阅源。

搜索插件的 `items.externalID` 必须是可生成订阅地址的稳定对象 ID，例如频道 ID，而不是展示名称或 URL 片段。

## 订阅对象与源级过滤

```json
{
  "subscription": {
    "mode": "feedFromExternalID",
    "targetKind": "channel",
    "feedURL": "https://example.com/feed?channel={{externalID}}",
    "defaultFilters": [
      { "id": "hide-shorts", "title": "排除短视频", "field": "title", "values": ["#Shorts", "Shorts"] }
    ]
  }
}
```

`targetKind` 取值为 `author`、`channel`、`playlist`、`podcast`、`repository`、`collection` 或 `feed`。`defaultFilters` 最多 12 条，每条最多 20 个关键词；它们是建议值，不会锁定用户，订阅后可在源设置中编辑或删除。

## 受控账户能力（v2）

登录能力只能声明 OAuth/Token 回调，不允许插件声明 `Cookie` 或 `Authorization` 请求头：

```json
{
  "schemaVersion": 2,
  "authentication": {
    "serviceName": "Example",
    "loginURL": "https://auth.example.com/authorize?redirect_uri={{redirectURI}}&response_type=token",
    "callbackScheme": "merge",
    "tokenQueryItem": "access_token",
    "scheme": "bearer",
    "required": true
  }
}
```

用户点击登录后，Merge 使用系统授权界面完成登录，将回调中的访问令牌写入本机 Keychain。请求由 Merge 统一发出，并且只为对应插件的声明域名添加短期 Bearer 认证。插件作者看不到令牌，插件清单也不能保存令牌。

用户可以在插件管理页退出登录；删除插件时，关联 Keychain 凭据会一并删除。

## 声明式配置（v2）

插件可声明最多 12 个由 Merge 原生界面渲染的配置项：`toggle`、`text`、`choice`、`cookie` 和 `webLogin`。点击插件管理页中的插件行即可修改。普通值仅保存在本机；Cookie 使用单独的本机 Keychain 项保存，不会写入清单、同步到其他设备，也不会展示给插件代码。

- `choice` 使用插件声明的固定选项，适合排序、地区、内容类型等有限选择。
- `webLogin` 打开插件声明的 HTTPS 登录页。用户自行在网页中登录，点击完成后 Merge 只提取 `cookieDomain` 范围内的 Cookie 保存到 Keychain；登录页面使用临时网页数据存储，不与 Safari 或其他插件共享。

Cookie 不能作为普通请求头声明。只有声明了 `cookie` 字段并由 `request.cookieFieldID` 精确引用时，Merge 才会将该字段作为 `Cookie` 请求头发送到已声明域名。配置值可在 URL 和普通请求头中用 `{{config.fieldID}}` 引用；Cookie 字段不得用于这些模板。

```json
{
  "schemaVersion": 2,
  "configuration": [
    { "id": "includeReplies", "title": "包含回复", "type": "toggle", "defaultValue": "false" },
    { "id": "region", "title": "地区", "type": "choice", "defaultValue": "global", "options": [{ "id": "global", "title": "全球" }, { "id": "cn", "title": "中国" }] },
    { "id": "session", "title": "登录服务", "type": "webLogin", "loginURL": "https://example.com/login" }
  ],
  "request": {
    "url": "https://example.com/feed?replies={{config.includeReplies}}&region={{config.region}}",
    "cookieFieldID": "session",
    "cookieDomain": "example.com"
  }
}
```

插件作者只能声明字段名称、用途和请求位置；不能读取、导出或执行 Cookie，也不能对用户未声明的域名发送它。

## 输入模板

请求 URL 中使用 `{{input}}`。编码由 `input.encoding` 决定：

- `raw`：原样替换，适合完整 URL，但插件仍只能请求白名单域名。
- `urlQuery`：作为查询参数编码。
- `pathComponent`：作为路径片段编码。

## 字段规则

HTML/XML 响应使用 `selector`，JSON 响应使用 `jsonPath`。XML v1 主要用于 Atom 等稳定的公开订阅格式：

```json
{ "format": "xml", "items": { "selector": "entry" } }
```

`htmlJSON` 用于网页中明确标记的 JSON 数据，例如 YouTube 的 `ytInitialData`。Merge 只提取并解析 JSON，不执行其中的 JavaScript。

当 `itemsPath` 的数组混有不同类型的节点时，可使用 `embeddedJSON.itemObjectPath` 提取其中一种对象；例如 `"channelRenderer"`。`itemsPaths` 和 `itemObjectPaths` 可分别声明多个备用路径和节点类型，以适配同一服务的桌面、移动版响应；字段规则可用 `jsonPaths` 提供按顺序尝试的 JSON 路径，以适配同一服务在 `simpleText` 与 `runs.0.text` 之间的结构变化。

插件搜索只用于发现可订阅的源、频道或作者，不提供“搜索文章”能力。声明 `subscription.mode: "feedFromExternalID"` 的搜索插件会被呈现为可订阅对象列表：可用 `targetKind` 标记作者、频道、播放列表、播客、仓库、合集或订阅源；应用使用每个结果的 `images`、`title`、`author` 和 `summary` 显示头像、名称与辅助信息，并在每一行提供独立的订阅按钮。未声明该能力的插件只显示订阅源确认信息，不会在添加订阅页展示文章列表。

`subscription.defaultFilters` 可声明订阅时应用到该源的建议过滤规则，字段为 `title`、`content` 或 `author`。它们会转换为源设置中可见、可编辑、可删除的屏蔽规则，不会影响其他订阅源。

```json
{ "selector": ".title a", "attribute": "href", "required": true }
```

```json
{ "jsonPath": "author.name" }
```

HTML 的 `attribute` 支持普通属性以及特殊值 `html`。媒体字段设置 `multiple: true` 可收集多个匹配结果。JSON Path v1 支持点分隔对象键和数组索引，例如 `data.items`、`authors.0.name`；列表路径可写成 `data.items` 或 `data.items[*]`。

### 内容、媒体与日期

`items.content` 应返回可阅读的 HTML 或文本；`images`、`videos`、`audio` 可使用 `multiple: true` 返回多个 URL。所有文章 URL 与媒体 URL 都必须是安全 HTTPS 地址。对于视频源，应同时提供可在系统浏览器打开的文章 URL；不要提供流媒体解密地址或临时鉴权 URL。

`publishedAt` 推荐 ISO 8601；Unix 秒须显式写为 `{ "jsonPath": "timestamp", "dateFormat": "unix" }`。`externalID` 必须长期稳定：同一篇内容在多次刷新中应返回完全相同的值，才能避免重复推送。

## 必填输出

每篇文章必须输出：

- `externalID`：在该订阅源中长期稳定且唯一。
- `url`：文章的 HTTPS 原文链接。
- `title`：可显示标题。

推荐同时输出 `publishedAt`、`author`、`summary` 与正文。日期默认按 ISO 8601 解析；也可指定 `dateFormat`，Unix 秒使用 `"dateFormat": "unix"`。

## 权限和安全

- 仅允许 HTTPS。
- v1 仅允许 GET。
- 域名必须预先列入 `permissions.domains`。
- 子域名自动继承对应根域名权限。
- 禁止 `Authorization`、`Cookie`、`Host`、`Proxy-Authorization` 请求头。
- 拒绝 localhost、`.local`、回环、链路本地和常见私有 IPv4 地址。
- 清单最大 512 KiB，内容响应硬上限 5 MiB，最多处理 300 篇文章。
- 插件不得要求用户输入密码、Cookie、Token 或 API Secret。

`cookie` 与 `webLogin` 配置字段是例外：它们只能由 Merge 的受控界面收集或保存，插件清单和作者都无法读取其值。不要把 Cookie、密码、Token 或 API Key 写入示例、README、默认值或请求模板。

需要 OAuth 或付费 API 的平台应等待未来由 Merge 提供受控账户能力；不要通过清单绕过认证限制。

## 版本兼容

- 修改字段映射但保持语义兼容：增加补丁版本；用户重新安装同一插件 ID 时会覆盖旧清单。
- 改变输入含义或稳定 ID：增加主版本并使用新的插件 ID。
- App 不认识 `schemaVersion` 时必须拒绝安装。

## 作者自检清单

- 用真实但不含私人数据的输入验证：空结果、单个结果、多结果、非 ASCII 查询词。
- 验证每个结果都有唯一 `externalID`、HTTPS `url` 和非空 `title`。
- 分别验证桌面/移动网页结构；需要时使用 `itemsPaths`、`itemObjectPaths` 与 `jsonPaths` 声明备用路径。
- 验证 `feedURL` 能直接在无登录状态下返回可解析订阅源；若需要登录，使用声明式受控登录，不要伪造请求头。
- 更新版本后在 Merge 中使用“更新订阅插件”，查看“运行诊断”中的结果数量、耗时与错误摘要。
- 发布前通过 JSON Schema 验证，并确认所有实际请求和重定向都在 `permissions.domains` 内。

## 示例

- [`examples/html-list/plugin.json`](examples/html-list/plugin.json)：HTML + CSS Selector。
- [`examples/json-api/plugin.json`](examples/json-api/plugin.json)：JSON API。
- [`examples/oauth-api/plugin.json`](examples/oauth-api/plugin.json)：v2 受控 Bearer Token 登录示例（虚构域名，仅用于结构参考）。
- [`examples/youtube/plugin.json`](examples/youtube/plugin.json)：YouTube 公开频道 Atom Feed 示例，输入频道 ID 即可使用。
- [`examples/youtube-search/plugin.json`](examples/youtube-search/plugin.json)：YouTube 频道/博主搜索示例，输入关键词、选择频道后订阅该频道。
