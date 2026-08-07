# pushplus 开放接口参考

文档版本：**V1.15**（[官方文档](https://www.pushplus.plus/doc/guide/openApi.html)）  
在线调试：https://api.pushplus.plus/doc-6905395

本文件供智能体在需要查询发送结果、管理群组/好友/渠道/ClawBot、配置设置等场景时按需阅读。日常发消息请优先使用 [SKILL.md](SKILL.md) 中的 `/send` 与 `/batchSend`（用户 token，无需 AccessKey）。

## 何时使用开放接口

| 场景 | 用开放接口？ | 说明 |
|------|-------------|------|
| 发通知 / 告警 | 否 | 用 `/send` 或 `/batchSend` + `PUSHPLUS_TOKEN` |
| 查发送是否成功 | 是 | `GET .../message/sendMessageResult` |
| 列消息、删消息 | 是 | 消息接口 |
| 管群组 / 好友 / webhook | 是 | 对应模块 |
| 绑定 ClawBot | 是 | ClawBot 接口 |
| 上传图片 | 是 | 图片服务 |

## 认证（AccessKey）

开放接口权限较高，默认禁用。用户须在开发设置中开启，并配置 `secretKey` 与安全 IP。

### 获取 AccessKey

```bash
curl -s -X POST "https://www.pushplus.plus/api/common/openApi/getAccessKey" \
  -H "Content-Type: application/json" \
  -d '{"token":"USER_TOKEN","secretKey":"SECRET_KEY"}'
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `token` | 是 | **用户 token**（不支持消息 token） |
| `secretKey` | 是 | 用户密钥 |

响应：

```json
{"code":200,"msg":"请求成功","data":{"accessKey":"...","expiresIn":7200}}
```

### 调用规则

1. 后续请求 Header 必须带：`access-key: <accessKey>`
2. 有效期约 7200 秒；重复获取会使旧 key 失效；刷新有约 5 分钟新旧并存窗口
3. 建议中控统一刷新，避免多端互相覆盖
4. 请求 IP 须在安全 IP 列表内，否则 403
5. 凭证获取：优先环境变量 `PUSHPLUS_TOKEN` + `PUSHPLUS_SECRET_KEY`；缺失则询问用户。**禁止**在输出中展示完整 secretKey / accessKey

凭证优先级：
1. 对话中用户提供
2. 环境变量 `PUSHPLUS_TOKEN`、`PUSHPLUS_SECRET_KEY`（可选已有 `PUSHPLUS_ACCESS_KEY`）
3. `.env` 中**仅提取**上述变量行

通用响应：`{"code":200,"msg":"...","data":...}`；`code!=200` 时根据 `msg` 排查。

---

## 一、消息接口

基址：`https://www.pushplus.plus/api/open/message`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 消息列表 | POST | `/list` | body: `current`, `pageSize`（最大 50） |
| 查询发送结果 | GET | `/sendMessageResult` | query: `shortCode`（必填） |
| 删除消息 | DELETE | `/deleteMessage` | query: `shortCode`（必填；不可撤销） |
| 消息详情 HTML | GET | `https://www.pushplus.plus/shortMessage/{shortCode}` | 返回 HTML，无需 access-key |

### 查询发送结果（发消息后常用）

```bash
curl -s "https://www.pushplus.plus/api/open/message/sendMessageResult?shortCode=SHORT_CODE" \
  -H "access-key: ACCESS_KEY"
```

| data 字段 | 说明 |
|-----------|------|
| `status` | `0` 未投递 / `1` 发送中 / `2` 已发送 / `3` 发送失败 |
| `errorMessage` | 失败原因 |
| `updateTime` | 更新时间 |

消息列表项：`shortCode`, `title`, `channel`, `messageType`（1 一对一 / 2 一对多）, `topicName`, `updateTime`。

---

## 二、用户接口

基址：`https://www.pushplus.plus/api/open/user`

| 能力 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取用户 token | GET | `/token` | data 为用户 token 字符串 |
| 个人资料 | GET | `/myInfo` | 含积分、实名、会员等 |
| 解封剩余时间 | GET | `/userLimitTime` | `sendLimit`: 1 无限制 / 2 短期 / 3 永久 |
| 当日请求次数 | GET | `/sendCount` | 各渠道当日调用次数 |

`myInfo` 要点：`verifyStatus`（0 未实名 / 1 已实名）、`points`、`vipInfo.isVip` / `vipInfo.lastDay`、`token`、`phoneNumber`、`email`。

---

## 三、消息 token 接口

基址：`https://www.pushplus.plus/api/open/token`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 列表 | POST | `/list` | `current`, `pageSize` |
| 新增 | POST | `/add` | `name`（必填）, `expireTime`（默认 2999-12-31） |
| 修改 | POST | `/edit` | `id`, `name`, `expireTime` |
| 删除 | DELETE | `/deleteToken` | query: `id` |
| 下拉列表 | GET | `/selectTokenList` | query: `type`：0 全部 / 1 未配默认渠道 |

新增成功时 `data` 为新建的消息 token 字符串。

---

## 四、群组接口

基址：`https://www.pushplus.plus/api/open/topic`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 列表 | POST | `/list` | `params.topicType`：0 我创建的 / 1 我加入的 |
| 我创建的详情 | GET | `/detail` | `topicId` |
| 我加入的详情 | GET | `/joinTopicDetail` | `topicId` |
| 新增 | POST | `/add` | `topicCode`, `topicName`, `contact`, `introduction` 必填 |
| 修改 | POST | `/editTopic` | `topic`（群组编号）, `topicCode`, `topicName` 必填 |
| 二维码 | GET | `/qrCode` | `topicId`；`second` 默认 604800（最长 30 天）；`scanCount` 默认 -1（无限） |
| 退出 | GET | `/exitTopic` | `topicId` |
| 删除 | GET | `/delete` | `topicId` |
| 积分群上下架 | POST | `/isOpen` | `topic`, `isOpen`（1 上架 / 0 下架） |

`topicType`：0 普通 / 1 积分 / 2 公开。列表含 `topicCode`（发送消息时 `topic` 参数用此编码）。

---

## 五、群组用户接口

基址：`https://www.pushplus.plus/api/open/topicUser`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 订阅人列表 | POST | `/subscriberList` | `params.topicId` 必填 |
| 删除用户 | POST | `/deleteTopicUser` | query: `topicRelationId`（列表中的 `id`） |
| 修改备注 | POST | `/editRemark` | `id`, `remark`（≤20 字） |

---

## 六、渠道配置接口

### Webhook

基址：`https://www.pushplus.plus/api/open/webhook`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 列表 | POST | `/list` | 分页 |
| 详情 | GET | `/detail` | `webhookId` |
| 新增 | POST | `/add` | `webhookCode`, `webhookName`, `webhookType`, `webhookUrl` |
| 修改 | POST | `/edit` | 另加 `id` |

`webhookType`：1 企微机器人 / 2 钉钉 / 3 飞书 / 4 Server酱 / 50 bark / 6 企微应用 / 7 腾讯轻联 / 8 IFTTT / 9 集简云 / 10 Gotify / 11 WxPusher / 12 自定义。

自定义类型（12）可额外传 `httpMethod`、`headers`、`body`。

发送消息时 `/send` 的 `option` 填列表中的 `webhookCode`。

### 其他渠道只读列表

| 能力 | 方法 | 路径 |
|------|------|------|
| 微信公众号列表 | POST | `/api/open/mp/list` |
| 企业微信应用列表 | POST | `/api/open/cp/list` |
| 邮箱渠道列表 | POST | `/api/open/mail/list` |
| 邮箱详情 | GET | `/api/open/mail/detail?mailId=` |

企微 `cpCode`、邮箱 `mailCode` 可作为 `/send` 的 `option`。

---

## 七、微信 ClawBot 接口

基址：`https://www.pushplus.plus/api/open/clawBot`

| 能力 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取二维码 | GET | `/getBotQrcode` | 返回 `url`, `qrcode` |
| 扫码结果 | GET | `/getQrcodeStatus` | query: **`qrcode`**（二维码编号） |
| 绑定详情 | GET | `/botInfo` | `createTime`, `haveContextToken` |
| 解绑 | GET | `/unbind` | 无参数 |
| 获取发送消息 | GET | `/getMsg` | 数组：`type` 1 文字 / 3 语音；`text` 内容 |

绑定流程：调用 `/getBotQrcode` → 展示 `url` 给用户扫码 → 轮询 `/getQrcodeStatus?qrcode=` → 成功后可用 `/send` 且 `channel=clawbot`。

---

## 八、功能设置接口

基址：`https://www.pushplus.plus/api/open/setting`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 默认配置列表 | POST | `/listUserDefault` | 分页 |
| 默认配置详情 | GET | `/detailUserDefault` | `id` |
| 新增默认配置 | POST | `/addUserDefault` | `channel`, `option`, `pre`, `tokenId`（用户令牌为 0） |
| 修改默认配置 | POST | `/editUserDefault` | 另加 `id` |
| 删除默认配置 | DELETE | `/deleteUserDefault` | query: `id` |
| 接收限制 | GET | `/changeRecevieLimit` | `recevieLimit`：0 全部 / 1 不接收（参数名按官方拼写） |
| 开/关发送 | GET | `/changeIsSend` | `isSend`：0 禁用 / 1 启用 |
| 打开方式 | GET | `/changeOpenMessageType` | `openMessageType`：0 H5 / 1 小程序 |
| 插件转发 | GET | `/extension` | `forward`：0 否 / 1 是（微信消息同步插件/桌面端） |

默认渠道 `channel`：`wechat` / `cp` / `webhook` / `mail` / `sms` / `voice` / `extension`。

---

## 九、好友功能接口

基址：`https://www.pushplus.plus/api/open/friend`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 个人二维码 | GET | `/getQrCode` | 可选 `appId`, `content`, `second`, `scanCount` |
| 好友列表 | POST | `/list` | 分页；项含 `token`（发好友消息用） |
| 删除好友 | GET | `/deleteFriend` | `friendId` |
| 修改备注 | POST | `/editRemark` | `id`, `remark` |

发送好友消息：`/send` 的 `to` 填好友列表中的 `token`，勿与 `topic` 同填。

---

## 十、预处理信息接口（需会员）

基址：`https://www.pushplus.plus/api/open/pre`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 列表 | POST | `/list` | 分页 |
| 详情 | GET | `/detail` | `preId` |
| 新增 | POST | `/add` | `content`, `preName`, `preCode`, `contentType`（1=JS） |
| 修改 | POST | `/edit` | 另加 `id` |
| 删除 | DELETE | `/delete` | query: `preId` |
| 测试代码 | POST | `/test` | `content`, `contentType`, `message` |

发送时 `/send` 的 `pre` 填 `preCode`。

---

## 十一、图片服务接口

图片约 30 天有效；可主动删除。

| 能力 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取上传凭证 | GET | `/api/open/userImage/uploadToken` | 返回 `uploadToken`, `uploadUrl`, `expiresIn` 等 |
| 上传图片 | POST | 凭证中的 `uploadUrl` | multipart：`token`=`uploadToken`，`file`=图片；**无需** access-key |
| 图片列表 | POST | `/api/open/userImage/list` | 分页 |
| 删除图片 | DELETE | `/api/open/userImage/delete` | query: `id` |

上传成功响应（七牛表单）含 `url`、`thumbnail`；`errno=0` 表示成功。可将 `url` 用于消息 content / `icon`。

---

## Agent 调用要点

1. **先发消息用 token，再查结果用 AccessKey**：`/send` 返回 `shortCode` → 开放接口查 `status`。
2. **破坏性操作**（删消息、删群组、删好友、解绑 ClawBot、关发送）必须先向用户确认。
3. **分页**：多数列表 `pageSize` 最大 50。
4. **脱敏**：输出中遮盖 token、secretKey、accessKey、webhookUrl、邮箱密码等。
5. 完整字段与示例见 [开放接口文档](https://www.pushplus.plus/doc/guide/openApi.html)；发送参数见 [消息接口文档](https://www.pushplus.plus/doc/guide/api.html)。
