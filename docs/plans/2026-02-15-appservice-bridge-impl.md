# Appservice API 完善实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 完善 palpo 的 Application Service API，使 mautrix 系列桥接软件能正常工作。

**Architecture:** palpo 已有 appservice 基础设施（注册管理、namespace 匹配、事件分发、用户伪装）。需要修复 4 个关键缺陷：ephemeral 事件字段被注释、namespace 验证缺失、sending.rs 中 EDU 被忽略、rate limiting 未豁免 appservice。

**Tech Stack:** Rust, Diesel (PostgreSQL), Salvo web framework, serde, regex

---

### Task 1: 启用 PushEventsReqBody 的 ephemeral 字段

**Files:**
- Modify: `crates/core/src/appservice/event.rs:54-108`

**Step 1: 取消 ephemeral 字段的注释**

在 `PushEventsReqBody` 结构体中（第 54-108 行），ephemeral 字段被注释掉了（第 93-99 行）。
需要将其启用：

```rust
// 在 PushEventsReqBody 中，将注释掉的 ephemeral 字段替换为：
    /// A list of ephemeral events (typing, presence, receipts).
    #[serde(
        default,
        skip_serializing_if = "<[_]>::is_empty",
        rename = "de.sorunome.msc2409.ephemeral"
    )]
    pub ephemeral: Vec<RawJson<EphemeralData>>,
```

注意：`EphemeralData` 类型已在同文件中定义（第 139-213 行），`RawJson` 已在 imports 中。
需要确认 `RawJson<EphemeralData>` 是否可行，或者是否需要用 `serde_json::Value`。

**Step 2: 验证编译通过**

Run: `cargo check -p palpo 2>&1 | head -30`
Expected: 可能需要修复 sending.rs 中构造 PushEventsReqBody 的地方（Task 2 处理）

**Step 3: Commit**

```bash
git add crates/core/src/appservice/event.rs
git commit -m "feat(appservice): enable ephemeral events field in PushEventsReqBody"
```

---

### Task 2: 在事件推送中包含 ephemeral 数据

**Files:**
- Modify: `crates/server/src/sending.rs:314-370` (send_events 函数的 Appservice 分支)

**Step 1: 修复 PushEventsReqBody 构造**

在 `sending.rs` 第 345-348 行，当前构造 `PushEventsReqBody` 时没有 ephemeral 字段。
Task 1 添加了 ephemeral 字段后，这里需要补上：

```rust
let req_body = PushEventsReqBody {
    events: pdu_jsons.clone(),
    ephemeral: vec![],  // 新增：暂时为空，后续 Task 可以填充实际数据
    to_device: vec![],
};
```

同时注意第 325-327 行的注释 `// Appservices don't need EDUs (?)`——这是错误的。
Appservice 如果设置了 `receive_ephemeral: true`，是需要 EDU 的。
但 EDU 的收集逻辑较复杂，先确保编译通过，后续再实现 EDU 收集。

**Step 2: 验证编译通过**

Run: `cargo check -p palpo 2>&1 | head -30`
Expected: PASS（无编译错误）

**Step 3: Commit**

```bash
git add crates/server/src/sending.rs
git commit -m "feat(appservice): add ephemeral field to appservice event push"
```

---

### Task 3: 添加 username availability 的 appservice namespace 检查

**Files:**
- Modify: `crates/server/src/routing/client/register.rs:324-337`

**Step 1: 实现 namespace 验证**

在第 334 行的 `// TODO add check for appservice namespaces` 处，添加检查逻辑：

```rust
// 替换第 334 行的 TODO 注释为：
    // Check if the username is reserved by an appservice exclusive namespace
    if crate::appservice::is_exclusive_user_id(&user_id)? {
        return Err(MatrixError::exclusive("Username is reserved by an application service.").into());
    }
```

`is_exclusive_user_id` 函数已在 `crates/server/src/appservice.rs:300-307` 中实现。
`MatrixError::exclusive` 已在第 112 行使用过，确认可用。

**Step 2: 验证编译通过**

Run: `cargo check -p palpo 2>&1 | head -30`
Expected: PASS

**Step 3: Commit**

```bash
git add crates/server/src/routing/client/register.rs
git commit -m "fix(appservice): validate exclusive namespace on username availability check"
```

---

### Task 4: 确认 rate limiting 现状并添加 appservice 豁免

**Files:**
- Explore: `crates/server/src/` 搜索 rate limit 相关代码
- Possibly modify: rate limiting 中间件（如果存在）

**Step 1: 调查 rate limiting 实现**

palpo 当前可能没有实现 rate limiting 中间件（hoops/ 目录下只有 auth.rs）。
需要搜索确认：

Run: `grep -rn "rate" crates/server/src/ --include="*.rs" | grep -i limit | head -20`

如果没有 rate limiting 实现，则此 Task 标记为"不需要修复"（因为没有限速就不需要豁免）。
如果有实现，需要在限速逻辑中检查 `AuthedInfo.appservice.is_some()` 来跳过。

**Step 2: 根据调查结果决定是否需要修改**

如果需要修改，在 rate limiting 检查中添加：
```rust
// 如果请求来自 appservice，跳过 rate limiting
if authed.appservice.is_some() {
    return Ok(());
}
```

**Step 3: Commit（如果有修改）**

```bash
git commit -m "feat(appservice): exempt appservices from rate limiting"
```

---

### Task 5: 完善 room alias 的 appservice 查询响应处理

**Files:**
- Modify: `crates/server/src/room/alias.rs:87-114`

**Step 1: 审查当前实现**

`resolve_appservice_alias` 函数（第 87-114 行）当前逻辑：
1. 遍历所有 appservice
2. 如果 alias 匹配 appservice namespace 且 appservice 有 URL
3. 发送查询请求到 appservice
4. 如果返回 `Ok(Some(_))`，尝试 `resolve_local_alias`

问题：appservice 收到 alias 查询后应该创建房间并设置 alias，然后 homeserver 再查本地 alias。
当前代码在第 106-108 行的逻辑看起来是正确的——它在 appservice 返回成功后重新查本地 alias。

需要验证：appservice 返回成功后，alias 是否已经被创建到本地数据库中。
如果 appservice 通过 Client API 创建了房间和 alias，那么 `resolve_local_alias` 应该能找到。

**Step 2: 添加更好的错误处理和日志**

```rust
async fn resolve_appservice_alias(room_alias: &RoomAliasId) -> AppResult<OwnedRoomId> {
    for appservice in crate::appservice::all()?.values() {
        if appservice.aliases.is_match(room_alias.as_str())
            && let Some(url) = &appservice.registration.url
        {
            let request = query_room_alias_request(
                url,
                QueryRoomAliasReqArgs {
                    room_alias: room_alias.to_owned(),
                },
            )?
            .into_inner();

            match crate::sending::send_appservice_request::<Option<()>>(
                appservice.registration.clone(),
                request,
            )
            .await
            {
                Ok(Some(_)) => {
                    // Appservice acknowledged the alias, try resolving locally now
                    match resolve_local_alias(room_alias) {
                        Ok(room_id) => return Ok(room_id),
                        Err(e) => {
                            warn!(
                                "Appservice {} claimed alias {} but it wasn't created locally: {}",
                                appservice.registration.id, room_alias, e
                            );
                        }
                    }
                }
                Ok(None) => {
                    debug!("Appservice {} did not claim alias {}", appservice.registration.id, room_alias);
                }
                Err(e) => {
                    warn!("Failed to query appservice {} for alias {}: {}", appservice.registration.id, room_alias, e);
                }
            }
        }
    }

    Err(MatrixError::not_found("resolve appservice alias not found").into())
}
```

**Step 3: 验证编译通过**

Run: `cargo check -p palpo 2>&1 | head -30`
Expected: PASS

**Step 4: Commit**

```bash
git add crates/server/src/room/alias.rs
git commit -m "fix(appservice): improve alias resolution error handling and logging"
```

---

### Task 6: 完善 Docker Compose 集成测试环境

**Files:**
- Modify: `examples/with-telegram/compose.yml`

**Step 1: 取消 postgres 注释并添加 palpo 服务**

```yaml
services:
  postgres:
    hostname: postgres
    image: postgres:18
    restart: always
    ports:
      - 5432:5432
    volumes:
      - ./data/pgsql:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: root
      POSTGRES_USER: postgres
      POSTGRES_DB: palpo_local
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - internal

  palpo:
    build:
      context: ../../
      dockerfile: Dockerfile
    restart: unless-stopped
    volumes:
      - ./palpo.toml:/etc/palpo/palpo.toml
      - ./appservices:/etc/palpo/appservices
      - ./data/palpo:/data
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - 6006:6006
    networks:
      - internal

  mautrix-telegram:
    image: dock.mau.dev/mautrix/telegram:latest
    restart: unless-stopped
    volumes:
      - ./data/mautrix-telegram:/data
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - 29317:29317
    networks:
      - internal

networks:
  internal:
    attachable: true
```

**Step 2: 验证 compose 文件语法**

Run: `docker compose -f examples/with-telegram/compose.yml config --quiet 2>&1`
Expected: 无错误输出

**Step 3: Commit**

```bash
git add examples/with-telegram/compose.yml
git commit -m "feat(examples): complete telegram bridge docker compose setup"
```

---

### Task 7: 集成测试 — 启动环境并验证 appservice 注册

**Step 1: 启动 Docker Compose 环境**

```bash
cd examples/with-telegram
docker compose up -d
```

**Step 2: 验证 palpo 启动并加载 appservice 注册**

```bash
docker compose logs palpo | grep -i appservice
```
Expected: 日志中显示加载了 telegram appservice 注册

**Step 3: 验证 mautrix-telegram 能连接到 palpo**

```bash
docker compose logs mautrix-telegram | grep -i "connected\|registered\|started"
```
Expected: mautrix-telegram 成功连接到 palpo 的 appservice API

**Step 4: 记录测试结果**

如果有问题，记录错误信息并创建修复任务。
