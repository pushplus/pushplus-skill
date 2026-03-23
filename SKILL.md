---
name: pushplus-notification
description: Send push notifications via pushplus HTTP API to WeChat, email, webhook, SMS and more. Use when the user asks to send notifications, push messages, WeChat messages, alerts, reminders, or mentions pushplus. No external dependencies required — only needs a PUSHPLUS_TOKEN and curl/Shell access.
license: MIT
metadata:
  author: perk-net
  version: 1.0.0
  tags:
    - notification
    - pushplus
    - wechat
    - messaging
---

# pushplus Notification

通过 pushplus HTTP API 直接向微信、邮箱、webhook、短信等渠道推送消息。无需安装任何依赖，只需 Shell 工具 + curl 即可使用。

## 前置条件

用户需要提供 `PUSHPLUS_TOKEN`（32 位字符串），在 [pushplus.plus](https://www.pushplus.plus) 注册后获取。

获取 token 的方式（按优先级）：
1. 用户在对话中直接提供
2. 环境变量 `PUSHPLUS_TOKEN`
3. 项目根目录 `.env` 文件中的 `PUSHPLUS_TOKEN=xxx`

如果找不到 token，**必须询问用户**，不要猜测。

## API 端点

| 功能 | 方法 | URL |
|------|------|-----|
| 单条发送 | POST | `https://www.pushplus.plus/send` |
| 多渠道批量发送 | POST | `https://www.pushplus.plus/batchSend` |

Content-Type: `application/json`

## 发送消息

使用 Shell 工具执行 curl 命令：

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"TOKEN","title":"标题","content":"内容","template":"html","channel":"wechat"}'
```

### 请求参数（/send）

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `token` | 是 | — | 用户令牌，32 位字符串 |
| `title` | 是 | — | 消息标题，最大 100 字符 |
| `content` | 是 | — | 消息内容，根据 template 渲染 |
| `template` | 否 | `html` | 模板类型 |
| `channel` | 否 | `wechat` | 推送渠道 |
| `topic` | 否 | — | 群组编码，不填仅发送给自己 |
| `to` | 否 | — | 好友令牌，多人用逗号隔开 |
| `webhook` | 否 | — | 第三方 webhook 地址（channel 为 webhook 时） |
| `callbackUrl` | 否 | — | 消息回调地址 |
| `timestamp` | 否 | — | 毫秒时间戳，用于防重复 |

### 请求参数（/batchSend）

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `token` | 是 | — | 用户令牌 |
| `content` | 是 | — | 消息内容 |
| `channel` | 否 | `wechat` | 多个渠道用逗号隔开，如 `wechat,mail` |
| `title` | 否 | — | 消息标题 |
| `option` | 否 | — | 渠道配置参数，多个用逗号隔开对应 channel |
| `topic` | 否 | — | 群组编码 |
| `template` | 否 | `html` | 模板类型 |
| `callbackUrl` | 否 | — | 回调地址 |
| `timestamp` | 否 | — | 毫秒时间戳 |
| `to` | 否 | — | 好友令牌 |

## 模板类型

| 值 | 说明 |
|----|------|
| `html` | HTML 富文本（默认） |
| `txt` | 纯文本 |
| `markdown` | Markdown 格式 |
| `json` | JSON 可视化展示 |
| `cloudMonitor` | 阿里云监控 |
| `jenkins` | Jenkins 构建通知 |
| `route` | 路由模板 |
| `pay` | 支付模板 |

## 推送渠道

| 值 | 说明 |
|----|------|
| `wechat` | 微信公众号（默认） |
| `webhook` | 第三方 webhook |
| `cp` | 企业微信 |
| `mail` | 邮箱 |
| `sms` | 短信 |
| `voice` | 语音 |
| `extension` | 扩展渠道 |
| `app` | APP 推送 |

## 响应格式

```json
{"code": 200, "msg": "请求成功", "data": "消息流水号"}
```

- `code` 为 `200` 表示成功
- `data` 为消息流水号
- 非 200 时根据 `msg` 排查（常见：token 无效、余额不足）

## 使用流程

1. 获取用户的 `PUSHPLUS_TOKEN`
2. 根据场景选择模板和渠道
3. 构造 JSON 请求体（注意转义特殊字符）
4. 通过 Shell 工具执行 curl 命令
5. 检查响应 `code` 是否为 200，向用户反馈结果

## 模板选择策略

| 场景 | 模板 | 说明 |
|------|------|------|
| 简单文本通知 | `txt` | 最简洁 |
| 带格式的报告/日志 | `markdown` | 支持标题、列表、代码块 |
| 富文本/邮件通知 | `html` | 支持样式和排版 |
| 结构化数据 | `json` | 自动渲染为可视化表格 |

## 示例

### 发送 Markdown 通知

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"任务完成","content":"## 任务已完成\n\n- **任务**: 代码重构\n- **状态**: 成功\n- **耗时**: 5分钟","template":"markdown","channel":"wechat"}'
```

### 发送 HTML 告警

```bash
curl -s -X POST "https://www.pushplus.plus/send" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"构建失败","content":"<h2 style=\"color:red\">构建失败</h2><p>项目 my-app 构建失败，请检查日志。</p>","template":"html","channel":"wechat"}'
```

### 多渠道批量发送

```bash
curl -s -X POST "https://www.pushplus.plus/batchSend" \
  -H "Content-Type: application/json" \
  -d '{"token":"YOUR_TOKEN","title":"紧急通知","content":"服务器 CPU 超过 90%","channel":"wechat,mail","template":"txt"}'
```

## 注意事项

- content 中的双引号需要转义为 `\"`
- content 中的换行使用 `\n`
- Windows 环境下 curl 的单引号需改为双引号，内部双引号用 `\"` 转义
- 如果 curl 不可用，可用 Python、Node.js 等发送 HTTP POST 请求作为替代
- token 是敏感信息，不要在输出中明文展示完整 token
