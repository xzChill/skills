---
name: xuanim-api
display_name: 喧喧 IM 集成 API
display_name_en: XuanIM Integration API
description: 喧喧 IM（xuanim）服务端 xxd 直连集成 API 开发技能。当用户需要调用喧喧服务端 `/im/...` HTTP 接口时使用此技能，涵盖获取讨论组、获取成员、推送通知消息、发送聊天消息等场景。包含请求格式、签名机制、字段约束及接口定义。当用户提到喧喧、xuanim、消息推送、群通知、IM 集成等关键词时应优先使用此技能。
description_zh: 喧喧 IM 服务端 xxd 直连 API 集成技能，覆盖获取讨论组、获取成员、推送通知、发送聊天消息，含请求格式、签名算法与接口定义。
description_en: XuanIM server-side xxd integration API skill covering chat groups, members, notifications and chat messages, with request format, signature algorithm and endpoint definitions.
version: 1.0.0
author: 喧喧
---

# 喧喧IM（xxd）应用集成 API

> 本技能描述的是 `xxd` 提供的直连 HTTP 接口 `/im/...`。

## 前提条件

调用 API 前需在喧喧后台设置应用集成，获取：

- **应用代号**：`code`
- **应用密匙**：`key`

请求通常发往 `xxd` 的 `CommonPort`，默认端口为 `11443`。如果前面有反向代理，也可以直接使用代理后的 `80/443` 地址。

---

## 一、请求格式

所有 API 请求地址格式为：

```text
{baseUrl}{path}?code={code}&{params}&token={token}
```

- `{baseUrl}`：`xxd` 对外服务基础地址，例如 `https://im.example.com:11443`、`https://im.example.com`、`http://10.0.0.8:11443`
- `{path}`：接口路径，见下方接口列表
- `code`：应用代号，放在 query 中
- `params`：接口额外参数，如 `gid=xxx`
- `token`：签名，放在 query 中

可用路径：

- `/im/getChatGroups`
- `/im/getChatUsers`
- `/im/sendNotification`
- `/im/sendChatMessage`

---

## 二、签名算法

`xxd` 的签名不是对 `m=im&f=...` 做签名，而是对下面这个“规范化字符串”做签名：

```text
processedQuery = trimLeadingSlash(path) + "?" + canonicalQueryWithoutToken
token = md5(md5(processedQuery) + key)
```

其中：

- `path`：请求路径，例如 `/im/getChatUsers`
- `canonicalQueryWithoutToken`：移除 `token` 后，按标准 URL query 编码并按 key 排序后的 query 字符串
- `key`：应用密匙

示例：

```text
path  = /im/getChatUsers
query = gid=30683aea-7a1f-4ec8-a6d6-834e0310fd7d&code=myAppCode

canonicalQueryWithoutToken = code=myAppCode&gid=30683aea-7a1f-4ec8-a6d6-834e0310fd7d
processedQuery             = im/getChatUsers?code=myAppCode&gid=30683aea-7a1f-4ec8-a6d6-834e0310fd7d
token                      = md5(md5(processedQuery) + key)
```

> 项目中可使用 `scripts/sign.{py,js,php,sh}` 中的 `buildUrl/build_url` 直接构造完整请求 URL。

---

## 三、可用接口速览

| 路径 | 用途 | 请求方式 | 核心参数 |
|------|------|----------|----------|
| `/im/getChatGroups` | 获取讨论组列表 | GET | 无 |
| `/im/getChatUsers` | 获取成员信息 | GET | `gid`（可选） |
| `/im/sendNotification` | 向通知中心推送消息 | POST (JSON) | `users`, `title`, `sender` |
| `/im/sendChatMessage` | 向讨论组发送消息 | POST (JSON) | `gid`, `title`, `sender` |

返回结构不是完全统一的：

- `GET` 成功通常返回：
```json
{ "result": "success", "data": {} }
```
- `POST` 成功通常返回：
```json
{ "result": "success", "message": "" }
```
- 失败通常返回：
```json
{ "result": "fail", "message": "..." }
```

### 触发场景指南

- 用户说“获取喧喧讨论组/群组列表” → 使用 `/im/getChatGroups`
- 用户说“获取讨论组成员/成员列表” → 使用 `/im/getChatUsers`
- 用户说“推送通知消息/小喧喧通知” → 使用 `/im/sendNotification`
- 用户说“发送群消息/讨论组消息” → 使用 `/im/sendChatMessage`

---

## 四、如何使用本技能的更多资源

### 需要查看某个接口的完整参数、返回值结构和代码示例时

每个接口独立一个文件，按需读取：

| 接口 | 参考文件 |
|------|----------|
| `/im/getChatGroups` | `references/get-chat-groups.md` |
| `/im/getChatUsers` | `references/get-chat-users.md` |
| `/im/sendNotification` | `references/send-notification.md` |
| `/im/sendChatMessage` | `references/send-chat-message.md` |

各文件包含对应接口的完整参数表、返回值结构和 JavaScript 示例代码。

### 需要在代码中计算签名时

使用已准备好的脚本可直接计算 token，或直接生成完整 URL：

- **Python**：`python scripts/sign.py <processed_query> <key>`
- **Node.js**：`node scripts/sign.js <processed_query> <key>`
- **PHP**：`php scripts/sign.php <processed_query> <key>`
- **Bash**：`bash scripts/sign.sh <processed_query> <key>`

示例：

```text
node scripts/sign.js "im/getChatUsers?code=myAppCode&gid=xxx" "3cd0914d656e90ab181f1d52ff352cfe"
```

脚本还提供以下辅助函数：

- `buildUrl()` / `build_url()`：构造完整 URL
- `buildSignPayload()` / `build_sign_payload()`：构造签名原串
- `xuanimSign()` / `xuanim_sign()`：对签名原串计算 token

---

## 参考资料

- 官方文档 - 应用集成 API：https://www.xuanim.com/book/dev/139
- 官方文档 - API 格式和签名：https://www.xuanim.com/book/dev/140
- 官方文档 - API 定义：https://www.xuanim.com/book/dev/141
