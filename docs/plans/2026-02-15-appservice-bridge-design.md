# Appservice API 完善与国际 IM 桥接支持设计

## 目标

完善 palpo 的 Matrix Application Service API，使其能兼容 mautrix 系列等主流桥接软件，
支持 Telegram、Discord、WhatsApp、Signal、Slack、IRC 等国际 IM 平台的桥接接入。

## 策略：渐进式（方案 C）

先修复关键阻塞问题，用 mautrix-telegram 做集成测试验证，再逐步补齐高级功能。

## 可桥接平台全景

### Tier 1 — 成熟稳定
- Telegram: mautrix-telegram (Python, 1.6k stars)
- WhatsApp: mautrix-whatsapp (Go, 1.7k stars)
- Discord: mautrix-discord (Go, 398 stars)
- Signal: mautrix-signal (Go, 618 stars)
- IRC: Heisenbridge (Python)
- Slack: mautrix-slack (Go, 89 stars)

### Tier 2 — 可用
- Facebook Messenger / Instagram DM: mautrix-meta (Go)
- Twitter/X DM: mautrix-twitter (Go)
- Google Chat: mautrix-googlechat (Python)
- Google Messages (RCS/SMS): mautrix-gmessages (Go)
- LinkedIn: mautrix-linkedin (Go)
- Bluesky: mautrix-bluesky (Go)

### Tier 3 — 特殊用途
- XMPP: matrix-bifrost
- Email: Postmoogle
- Webhooks (GitHub/GitLab/Jira): matrix-hookshot
- iMessage: beeper-imessage (需 macOS)

## 当前 Appservice API 实现状态

### 已完成 ✅
- PUT /_matrix/app/v1/transactions/{txnId} — 端点存在但为空壳
- GET /_matrix/app/v1/rooms/{roomAlias} — 完整实现
- GET /_matrix/app/v1/users/{userId} — 完整实现
- POST /_matrix/app/v1/ping — 完整实现
- Third-party protocol 端点 — 存在但返回空数据
- POST /_matrix/client/v3/register (appservice type) — 完整实现
- User masquerading (?user_id=) — 完整实现，自动创建用户
- 事件分发到 appservice (sending.rs) — 基础实现
- Appservice 注册管理（数据库 + 文件系统）— 完整实现

### 需要修复 ❌
详见下方 Phase 分解。

## Phase 1：关键阻塞修复

### 1.1 Transaction Endpoint 事件处理
- 文件: `crates/server/src/routing/appservice/transaction.rs`
- 问题: 收到事件后直接返回 empty_ok()，不做处理
- 修复: 确认此端点的角色（HS→AS 推送 vs AS→HS 推送），完善事件处理逻辑

### 1.2 Ephemeral 事件支持
- 文件: `crates/core/src/appservice/` PushEventsReqBody
- 问题: ephemeral 字段被注释，桥接无法收到 typing/presence/receipts
- 修复: 启用 ephemeral 字段，在事件分发时推送给 receive_ephemeral=true 的 appservice

### 1.3 Namespace 验证
- 文件: `crates/server/src/routing/client/register.rs:334`
- 问题: 注册时不检查 appservice exclusive namespace
- 修复: 拒绝非 appservice 对 exclusive namespace 的注册

### 1.4 Rate Limiting 豁免
- 问题: rate_limited 字段存在但未使用
- 修复: 在 rate limiting 中间件中豁免 appservice 请求

## Phase 2：功能完善

### 2.1 Room Alias 查询
- 文件: `crates/server/src/room/alias.rs:100`
- 修复: 正确处理 appservice 返回的 alias 创建响应

### 2.2 Profile Masquerading
- 修复: 确保 ?user_id= 在 profile API 中完整工作

## Phase 3：高级功能（后续迭代）

### 3.1 To-Device 消息 (MSC4203)
### 3.2 Device List 更新 (MSC3202)
### 3.3 Device Management (MSC4190)

## 集成测试策略

### 测试环境
```
Docker Compose:
  ├── palpo (homeserver)
  ├── postgres (数据库)
  ├── mautrix-telegram (桥接)
  └── traefik (反向代理)
```

### 第一个测试目标: mautrix-telegram
- 已有 examples/with-telegram/ 示例
- 最成熟的桥接之一
- Telegram Bot API 容易获取测试账号

### Phase 1 验证检查点
- [ ] mautrix-telegram 成功注册为 appservice
- [ ] 桥接 bot 用户能被创建
- [ ] Telegram → Matrix 消息桥接
- [ ] Matrix → Telegram 消息桥接
- [ ] 虚拟用户（puppet）正确创建和显示

### Phase 2 验证检查点
- [ ] 虚拟用户 displayname/avatar 正确同步
- [ ] Room alias 被 appservice 正确创建
- [ ] Typing indicator 双向同步

### 后续桥接扩展顺序
1. mautrix-discord
2. mautrix-whatsapp
3. mautrix-signal
4. Heisenbridge (IRC)
5. mautrix-slack
