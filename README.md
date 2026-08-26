# Merge Declarative Connector Specification

Merge Connector 是一种**数据描述文件**，用于把没有 RSS 输出的网页或 JSON API 映射为 Merge 可以订阅的文章。插件不包含、下载或执行 JavaScript、Wasm、原生代码或其他可执行代码。

当前规范版本：`2`（完全兼容 v1）

## 设计目标

- 让网站通过 HTML/CSS Selector 或 JSON Path 输出统一文章数据。
- 所有网络请求由 Merge 发起并执行权限检查。
- 插件不能读取文件、Keychain、Cookie、剪贴板、位置、相册或其他系统数据。
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

完整字段约束见 [`schema/plugin.schema.json`](schema/plugin.schema.json)。v1 插件继续可用；只有声明 `authentication` 时才使用 v2。

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

用户可以在插件管理页退出登录；删除插件时，关联 Keychain 凭据会一并删除。需要 Cookie 会话的网站不能通过通用插件导入 Cookie，应为具体服务实现经过审核的宿主适配。

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

当 `itemsPath` 的数组混有不同类型的节点时，可使用 `embeddedJSON.itemObjectPath` 提取其中一种对象；例如 `"channelRenderer"`。字段规则可用 `jsonPaths` 提供按顺序尝试的 JSON 路径，以适配同一服务在 `simpleText` 与 `runs.0.text` 之间的结构变化。

```json
{ "selector": ".title a", "attribute": "href", "required": true }
```

```json
{ "jsonPath": "author.name" }
```

HTML 的 `attribute` 支持普通属性以及特殊值 `html`。媒体字段设置 `multiple: true` 可收集多个匹配结果。JSON Path v1 支持点分隔对象键和数组索引，例如 `data.items`、`authors.0.name`；列表路径可写成 `data.items` 或 `data.items[*]`。

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

需要 OAuth 或付费 API 的平台应等待未来由 Merge 提供受控账户能力；不要通过清单绕过认证限制。

## 版本兼容

- 修改字段映射但保持语义兼容：增加补丁版本；用户重新安装同一插件 ID 时会覆盖旧清单。
- 改变输入含义或稳定 ID：增加主版本并使用新的插件 ID。
- App 不认识 `schemaVersion` 时必须拒绝安装。

## 示例

- [`examples/html-list/plugin.json`](examples/html-list/plugin.json)：HTML + CSS Selector。
- [`examples/json-api/plugin.json`](examples/json-api/plugin.json)：JSON API。
- [`examples/oauth-api/plugin.json`](examples/oauth-api/plugin.json)：v2 受控 Bearer Token 登录示例（虚构域名，仅用于结构参考）。
- [`examples/youtube/plugin.json`](examples/youtube/plugin.json)：YouTube 公开频道 Atom Feed 示例，输入频道 ID 即可使用。
- [`examples/youtube-search/plugin.json`](examples/youtube-search/plugin.json)：YouTube 频道/博主搜索示例，输入关键词、选择频道后订阅该频道。
