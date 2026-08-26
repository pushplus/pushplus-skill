# pushplus 开放接口参考

文档版本：**V1.17**（[官方文档](https://www.pushplus.plus/doc/guide/openApi.html)）  
在线调试：https://api.pushplus.plus/doc-6905395

本文件供智能体在需要查询发送结果、管理群组/好友/黑名单/渠道/ClawBot/QQ 机器人、配置设置等场景时按需阅读。日常发消息请优先使用 [SKILL.md](SKILL.md) 中的 `/send` 与 `/batchSend`（用户 token，无需 AccessKey）。

## 何时使用开放接口

| 场景 | 用开放接口？ | 说明 |
|------|-------------|------|
| 发通知 / 告警 | 否 | 用 `/send` 或 `/batchSend` + `PUSHPLUS_TOKEN` |
| 查发送是否成功 | 是 | `GET .../message/sendMessageResult` |
| 列消息、删消息 | 是 | 消息接口 |
| 管群组 / 好友 / 黑名单 / webhook | 是 | 对应模块 |
| 绑定 ClawBot | 是 | ClawBot 接口 |
| 绑定 QQ 机器人 / 管群配置 | 是 | QQ 机器人接口 |
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
| 当日请求次数 | GET | `/sendCount` | 各渠道当日调用次数，含 `qqBotSendCount` |

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
| 删除用户 | POST | `/deleteTopicUser` | query: `topicRelationId`（订阅人列表中的 `id`） |
| 修改备注 | POST | `/editRemark` | `id`, `remark`（≤20 字） |
| 加入黑名单 | POST | `/addBlacklist` | query: `topicRelationId`（订阅人列表中的 `id`） |
| 黑名单列表 | POST | `/blacklistList` | `params.topicId` 必填；分页 |
| 解除黑名单 | POST | `/removeBlacklist` | query: `id`（**黑名单列表**中的 `id`，不是订阅人 `id`） |

拉黑说明：
- 加入后将该用户**移出群组**，且对方无法再加入该群组
- **积分群组不支持**黑名单
- 不能将自己加入黑名单
- 解除后**不会自动恢复订阅**，对方可重新加入

```bash
# 拉黑（topicRelationId = 订阅人列表 id）
curl -s -X POST "https://www.pushplus.plus/api/open/topicUser/addBlacklist?topicRelationId=1" \
  -H "access-key: ACCESS_KEY"

# 黑名单列表
curl -s -X POST "https://www.pushplus.plus/api/open/topicUser/blacklistList" \
  -H "Content-Type: application/json" \
  -H "access-key: ACCESS_KEY" \
  -d '{"current":1,"pageSize":20,"params":{"topicId":1}}'

# 解除（id = 黑名单列表 id）
curl -s -X POST "https://www.pushplus.plus/api/open/topicUser/removeBlacklist?id=1" \
  -H "access-key: ACCESS_KEY"
```

黑名单列表项：`id`（解除时用）、`userId`、`nickName`、`openId`、`headImgUrl`、`createTime`（拉黑时间）。

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

企微 `cpCode`、邮箱 `mailCode` 可作为 `/send` 的 `option`。QQ 机器人的群配置（`qqCode`）见「八、QQ 机器人接口」。

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

## 八、QQ 机器人接口

基址：`https://www.pushplus.plus/api/open/qqBot`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 获取绑定链接 | GET | `/getBindLink` | query: `refresh`（可选，true 使旧绑定码失效并重新生成） |
| 查询绑定状态 | GET | `/botInfo` | 无参数 |
| 解绑 | GET | `/unbind` | 无参数（不可逆，需确认） |
| 已加入的 QQ 群列表 | GET | `/groupList` | 无参数 |
| 群配置列表 | POST | `/list` | 分页 `current`, `pageSize` |
| 新增群配置 | POST | `/add` | `qqName`, `qqCode`, `qqGroupId` 必填 |
| 修改群配置 | POST | `/edit` | `id`, `qqName`, `qqGroupId`（`qqCode` 不可改） |
| 删除群配置 | DELETE | `/delete` | query: `id` |

`getBindLink` 返回：`url`（带参分享链接，用于生成二维码；已绑定用户可能为空）、`bindCode`（绑定码，已是好友时需私聊发给机器人；认领 QQ 群也用此码）、`expireSeconds`（默认 300）、`botAppId`、`botName`、`botAvatar`。

`botInfo` 返回：`isBind`（0 未绑定 / 1 已绑定）、`receiveStatus`（1 可接收 / 0 用户已关闭单聊接收）、`createTime`、`botInfo{botId, username, avatar, appId, shareUrl}`（`shareUrl` 可用于拉机器人进群）。

`groupList` 项：`id`（新增配置时作为 `qqGroupId`）、`groupOpenId`、`groupRemark`、`status`（1 在群 / 2 群消息接收关闭）、`groupName`、`groupFingerMemo`、`groupClassText`、`groupTags`、`groupMemberNum`、`createTime`。

群配置列表项：`id`、`qqName`、`qqCode`（**发送消息时作为 `option`**）、`sendType`（2 发到 QQ 群）、`qqGroupId`、`groupRemark`、`groupOpenId`、`groupName`、`updateTime`。

配置约束：仅用于发到 QQ 群（发给自己无需配置）；普通用户最多 5 个，会员最多 30 个；同一 QQ 群不可重复创建；`qqCode` 创建后不可修改（最多 32 字符，仅字母、数字、下划线、中划线）；`qqName` 最多 64 字符；目标群须允许机器人主动消息。

```bash
# 1. 取绑定链接，把 url 生成二维码给用户扫，或让用户私聊发送 bindCode
curl -s "https://www.pushplus.plus/api/open/qqBot/getBindLink" \
  -H "access-key: ACCESS_KEY"

# 2. 轮询绑定状态，isBind=1 即可用 channel=qq 发消息
curl -s "https://www.pushplus.plus/api/open/qqBot/botInfo" -H "access-key: ACCESS_KEY"

# 3. 查已加入的群，取 id 作为 qqGroupId 建配置
curl -s "https://www.pushplus.plus/api/open/qqBot/groupList" -H "access-key: ACCESS_KEY"

curl -s -X POST "https://www.pushplus.plus/api/open/qqBot/add" \
  -H "Content-Type: application/json" \
  -H "access-key: ACCESS_KEY" \
  -d '{"qqName":"交流群推送","qqCode":"qqgroup","qqGroupId":1}'
```

绑定流程：`/getBindLink` → 用户扫码加机器人好友（已是好友则私聊发送 `bindCode`）→ 轮询 `/botInfo` 直到 `isBind=1` → `/send` 且 `channel=qq`（不填 `option` 发给自己）。发到群还需：把机器人拉进群并允许主动消息 → `/groupList` 取群 `id` → `/add` 建配置 → 发送时 `option` 填 `qqCode`。

---

## 九、功能设置接口

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

默认渠道 `channel`：`wechat` / `cp` / `webhook` / `mail` / `sms` / `voice` / `extension` / `qq`。

---

## 十、好友功能接口

基址：`https://www.pushplus.plus/api/open/friend`

| 能力 | 方法 | 路径 | 关键参数 |
|------|------|------|----------|
| 个人二维码 | GET | `/getQrCode` | 可选 `appId`, `content`, `second`, `scanCount` |
| 好友列表 | POST | `/list` | 分页；项含 `token`（发好友消息用）、`friendId`（拉黑/删除用） |
| 删除好友 | GET | `/deleteFriend` | query: `friendId` |
| 修改备注 | POST | `/editRemark` | `id`（列表 `id`）, `remark` |
| 加入黑名单 | POST | `/addBlacklist` | query: `friendId`（好友列表中的 **`friendId`**，不是 `id`） |
| 黑名单列表 | POST | `/blacklistList` | 分页 |
| 解除黑名单 | POST | `/removeBlacklist` | query: `id`（**黑名单列表**中的 `id`，不是 `friendId`） |

发送好友消息：`/send` 的 `to` 填好友列表中的 `token`，勿与 `topic` 同填。

拉黑说明：
- 加入后将**解除双方好友关系**，对方无法再添加你
- 不能将自己加入黑名单；**仅可将已有好友**加入黑名单
- 解除后**不会自动恢复好友关系**，需重新扫码添加

```bash
# 拉黑（friendId = 好友列表 friendId）
curl -s -X POST "https://www.pushplus.plus/api/open/friend/addBlacklist?friendId=1" \
  -H "access-key: ACCESS_KEY"

# 黑名单列表
curl -s -X POST "https://www.pushplus.plus/api/open/friend/blacklistList" \
  -H "Content-Type: application/json" \
  -H "access-key: ACCESS_KEY" \
  -d '{"current":1,"pageSize":20}'

# 解除（id = 黑名单列表 id）
curl -s -X POST "https://www.pushplus.plus/api/open/friend/removeBlacklist?id=1" \
  -H "access-key: ACCESS_KEY"
```

黑名单列表项：`id`（解除时用）、`friendId`、`nickName`、`headImgUrl`、`createTime`（拉黑时间）。

---

## 十一、预处理信息接口（需会员）

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

## 十二、图片服务接口

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
2. **破坏性操作**（删消息、删群组、删好友、删 QQ 群配置、拉黑好友/订阅人、解绑 ClawBot / QQ 机器人、关发送）必须先向用户确认。拉黑会解除关系且对方无法再加入/添加，解除黑名单也不会自动恢复。
3. **分页**：多数列表 `pageSize` 最大 50。
4. **脱敏**：输出中遮盖 token、secretKey、accessKey、webhookUrl、邮箱密码等。
5. 完整字段与示例见 [开放接口文档](https://www.pushplus.plus/doc/guide/openApi.html)；发送参数见 [消息接口文档](https://www.pushplus.plus/doc/guide/api.html)。
