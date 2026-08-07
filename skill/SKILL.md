---
name: pushplus-notification
description: Send push notifications via pushplus HTTP API to WeChat, ClawBot, email, webhook, SMS, App and more; also manage messages, topics, friends, channels and ClawBot via Open API when needed. Use when the user asks to send notifications, push messages, WeChat messages, alerts, reminders, query send results, or mentions pushplus. No external dependencies — only needs a PUSHPLUS_TOKEN (and AccessKey credentials for Open API) plus curl/Shell access.
license: MIT
primaryEnv: PUSHPLUS_TOKEN
requiredEnvVars:
  - name: PUSHPLUS_TOKEN
    description: pushplus API token (32-character string), obtained from https://www.pushplus.plus
optionalEnvVars:
  - name: PUSHPLUS_SECRET_KEY
    description: Open API secretKey for AccessKey exchange (开发设置中配置)
  - name: PUSHPLUS_ACCESS_KEY
    description: Cached Open API access-key (expires ~7200s)
metadata:
  author: perk-net
  version: 1.3.1
  apiVersion: "1.16"
  openApiVersion: "1.15"
  tags:
    - notification
    - pushplus
    - wechat
    - clawbot
    - messaging
---

# pushplus Notification

通过 pushplus HTTP API 向微信、ClawBot、邮箱、webhook、短信、App 等渠道推送消息。无需安装依赖，Shell + curl 即可。

依据：[消息接口文档 V1.16](https://www.pushplus.plus/doc/guide/api.html)。开放能力（查结果 / 群组 / 好友 / 渠道等）见 [reference.md](reference.md)。

## 前置条件

用户需提供 `PUSHPLUS_TOKEN`（用户 token 或消息 token），在 [pushplus.plus](https://www.pushplus.plus) 注册获取。调用发送接口前需完成实名认证。

获取 token（按优先级）：
1. 用户在对话中直接提供
2. 环境变量 `PUSHPLUS_TOKEN`
3. 从项目根目录 `.env` **仅提取** `PUSHPLUS_TOKEN`（`grep ^PUSHPLUS_TOKEN= .env`），**禁止读取 `.env` 其他内容**

找不到 token 时**必须询问用户**，不要猜测。

## API 端点

| 功能 | 方法 | URL |
|------|------|-----|
| 发送消息 | GET/POST/PUT/DELETE | `https://www.pushplus.plus/send` |
| 多渠道发送 | GET/POST/PUT/DELETE | `https://www.pushplus.plus/batchSend` |

- Content-Type: `application/json`
- 参数均支持 url 参数和 body 参数
- token 可放在路径：`/send/{token}`、`/batchSend/{token}`
- GET 受 URL 长度限制，content 不宜过长；大段内容请用 POST
- 在线调试：https://api.pushplus.plus/

## 发送消息（/send）

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"TOKEN","title":"标题","content":"内容","template":"html","channel":"wechat"}'
```

### 请求参数

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `token` | 是 | 无 | 用户 token 或消息 token；可放到路径 `/send/{token}` |
| `title` | 否 | 无 | 消息标题 |
| `content` | 是 | 无 | 具体消息内容，根据不同 template 支持不同格式 |
| `topic` | 否 | 无 | 群组编码，不填仅发送给自己；channel 为 webhook 时无效 |
| `template` | 否 | `html` | 发送模板 |
| `channel` | 否 | `wechat` | 发送渠道 |
| `option` | 否 | 无 | 渠道配置参数（原 webhook 参数） |
| `callbackUrl` | 否 | 无 | 发送结果回调地址 |
| `timestamp` | 否 | 无 | 毫秒时间戳，如 `1632993318000`；服务器时间戳大于此值则消息不会发送 |
| `to` | 否 | 无 | 好友令牌（微信公众号）或企业微信用户 id；多人用逗号隔开；实名最多 10 人，会员 100 人 |
| `pre` | 否 | 无 | 预处理编码；仅供会员使用 |
| `pushId` | 否 | 无 | 推送表单；**`template=form` 时必填**，传表单编码（formCode） |

`option` 说明：原 webhook 参数。`cp`、`webhook`、`mail` 渠道需填写个人中心渠道设置中已配置的**渠道编码**（非完整 URL）。邮件渠道 `option` 可选，不填则用官网邮件发送。

`topic` 与 `to` 勿同时填写；群组优先于好友。`to` 支持微信公众号、邮件、企业微信渠道。

### 发送渠道（channel）

| 值 | 费用 | 说明 |
|----|------|------|
| `wechat` | 免费 | 微信公众号（默认） |
| `app` | 免费 | App；安卓 / 鸿蒙 / iOS |
| `extension` | 免费 | 浏览器扩展插件 / 桌面应用程序 |
| `webhook` | 免费 | 第三方 webhook（企微/钉钉/飞书/bark/Gotify/轻联/集简云/server酱/IFTTT/WxPusher 等）；需 `option` |
| `clawbot` | 免费 | 微信 ClawBot |
| `cp` | 免费 | 企业微信应用；需 `option` |
| `mail` | 免费 | 邮箱；`option` 可选 |
| `sms` | 收费 | 短信；成功 1 条扣 10 积分（0.1 元）；接收方需绑定手机 |
| `voice` | 收费 | 语音；接通 1 次扣 30 积分（0.3 元） |

### 发送模板（template）

| 值 | 说明 |
|----|------|
| `html` | 默认模板，支持 html 文本 |
| `txt` | 纯文本展示，不转义 html |
| `json` | 内容基于 json 格式展示 |
| `markdown` | 内容基于 markdown 格式展示 |
| `cloudMonitor` | 阿里云监控报警定制模板 |
| `jenkins` | jenkins 插件定制模板 |
| `route` | 路由器插件定制模板 |
| `pay` | 支付成功通知模板 |
| `form` | push 表单模板；**需同时传 `pushId`** |

### 响应

```json
{
  "code": 200,
  "msg": "请求成功",
  "data": "3cbc5eab19fe512e80677540fbde332a"
}
```

接口为**异步**：`code=200` 仅表示服务端已收到请求，**不表示发送成功**。`data` 为消息流水号，可用于查询最终结果；若传了 `callbackUrl`，发送完成后会主动回调。

## 多渠道发送（/batchSend）

与 `/send` 参数相同，差异：

| 参数 | 说明 |
|------|------|
| `channel` | 多个用逗号隔开，如 `"wechat,webhook,extension"` |
| `option` | 多个用逗号隔开，与 channel **一一对应**；某渠道无需 option 也要占位，如 `",bark,"` |

说明：
1. 本质是服务端循环调用发送消息接口
2. 按渠道数量计算请求次数（3 个渠道 = 3 次）
3. 响应 `data` 为数组，每项含 `shortCode`、`message`、`code`、`channel`

```json
{
  "code": 200,
  "msg": "执行成功",
  "data": [
    {
      "shortCode": "f9117123dc31434fa38917b7e4c6c3ff",
      "message": "请求成功，请用流水号查询最终发送结果",
      "code": 200,
      "channel": "wechat"
    }
  ]
}
```

## 使用流程

1. 获取 `PUSHPLUS_TOKEN`
2. 选择 `template` 与 `channel`（默认 `html` + `wechat`）
3. `webhook` / `cp` 确认已配置渠道编码并填入 `option`；`mail` 的 `option` 可选
4. `template=form` 时确认有 `pushId`（表单编码）
5. **向用户展示标题与内容摘要，获得确认后再发送**
6. POST 请求；检查 `code` 是否为 200
7. 成功则反馈流水号（请求已受理）；失败根据 `msg` 说明
8. 需确认送达 / 管理群组好友渠道 / 绑定 ClawBot → 阅读 [reference.md](reference.md)

## 模板选择策略

| 场景 | 模板 |
|------|------|
| 简单文本 | `txt` |
| 报告 / 日志 | `markdown` |
| 富文本 / 邮件 | `html` |
| 结构化数据 | `json` |
| push 表单 | `form`（需 `pushId`） |

## 示例

### 最简 POST

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容"}'
```

### Markdown

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"# 大标题\n##### 小标题\n1. 第一项\n2. 第二项","template":"markdown"}'
```

### ClawBot

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"提醒","content":"任务已完成","template":"txt","channel":"clawbot"}'
```

### Webhook / 企业微信应用

```bash
# webhook（option 为已配置的编码，如企业微信机器人）
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容","channel":"webhook","option":"pushplus"}'

# 企业微信应用
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容","channel":"cp","option":"cp"}'
```

### 群组 / 好友

```bash
# 一对多（topic）
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容","topic":"code","template":"html"}'

# 好友（to）；勿与 topic 同时填
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容","to":"好友令牌","template":"html"}'
```

### 邮件 / 短信

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"邮件标题","content":"邮件正文","channel":"mail","option":"163","template":"html"}'

curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"短信","content":"消息内容正文","channel":"sms"}'
```

### push 表单

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"push表单demo","content":"push表单demo","template":"form","pushId":"YpRUxav9"}'
```

### 时间戳防过期

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容","timestamp":1632993318000}'
```

### 多渠道 batchSend

```bash
curl -s -X POST "https://www.pushplus.plus/batchSend" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"标题","content":"消息内容","topic":"code","template":"html","channel":"wechat,webhook,extension","option":",bark,"}'
```

## 注意事项

- content 中双引号转义为 `\"`，换行用 `\n`（markdown 亦然）
- GET 中文 content 需 UrlEncode；大段内容用 POST
- `topic` 与 `to` 不要同时填写
- 收费渠道（`sms` / `voice`）发送前告知会消耗积分
- token 脱敏展示（如 `a1b2****ef90`）

## 安全要求

- **发送前确认**：展示标题与内容摘要，获得明确确认后再发送。
- **凭证保护**：不在输出中展示完整 token / secretKey / accessKey。
- **最小读取**：从 `.env` 只提取 `PUSHPLUS_*` 相关行。
- **敏感内容警示**：含密码、密钥、PII 时警告将经第三方传输。
- **不要持久化凭证**：仅在内存中构造请求。
- **破坏性开放接口**：删消息/群组/好友、解绑等须先确认（见 reference.md）。

## 附加资源

- 开放接口：[reference.md](reference.md)
- [消息接口文档](https://www.pushplus.plus/doc/guide/api.html)
- [开放接口文档](https://www.pushplus.plus/doc/guide/openApi.html)
