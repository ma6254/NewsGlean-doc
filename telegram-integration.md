# Telegram 渠道集成（规划）

> 状态：**规划阶段（待决策）**，尚未实现代码。
> 关联文档：`PLAN.md`（主设计规格，阶段 14/15/20）、`rsshub-quick-add.md`、`bilibili-cli-integration.md`。
> 本文**不改采集契约（`Connector`/`Item`/`Cursor` 接口），但需在核心层补一条「推送模式」调度接线**——与 rsshub/bilibili 的「零契约改动」不同，见 §2。

## 1. 定位与核心结论

**一句话**：新增 `telegram`（官方 Bot API）与 `telegram_user`（MTProto 用户账号）**两个渠道类型**，让用户在加源时自选；两者都服务「订阅特定频道/群」与「实时聊天内容」两个数据需求，差别在**覆盖范围**与**风险**。

**关键结论：**

- ✅ 契约层（`Connector`/`Item`/`Cursor`/`State`）**零改动**——接口早已预留：`Run`（推送）、`ErrPushUnsupported`（回退拉取）、`Metadata.Pull/Push`（注册元数据）、`Cursor` 注释里已写「Telegram 存 update_id」、`Item.GUID` 注释已写「TG 的 chat_id+message_id」。
- ⚠️ 核心层要**补一条推送调度接线**。当前 `app.go:177` 只调 `Fetch`、`scheduler` 只做 ticker 拉取，`Run` 尚未被任何调度路径使用。这是本项目**第一个推送型渠道**，实时聊天内容依赖这条接线，是本次规划里除渠道包之外最重要的新工作（§5）。
- 「订阅特定频道/群」与「实时聊天内容」两个需求，**两个类型都能满足**；真正的分水岭是「bot 只能看到被拉进去的对话」vs「用户账号能看到全部」，以及「官方无风险」vs「违反 ToS 有封号风险」。

## 2. 现状与缺口（已核对代码）

| 事实 | 位置 | 结论 |
|---|---|---|
| `Connector` 已有 `Run`/`ErrPushUnsupported`/`Fetch`/`Init`/`Close` | `internal/source/source.go` | 契约已就绪，无需改接口 |
| `Metadata{Pull, Push}` 已存在，`feed` 注册 `Pull:true, Push:false` | `internal/source/registry.go` | 类型元数据已支持推送标记 |
| 采集只走拉取：`refreshOne` 仅 `conn.Fetch(...)` | `internal/app/app.go:177` | **推送未接线** |
| 调度只做 ticker 轮询 | `internal/scheduler/scheduler.go` | 推送源会被错误地当拉取源轮询 |
| `ingestItem`（GUID→URL→ContentHash 三层去重 + 写库）已独立可复用 | `internal/app/app.go:230` | 推送 emit 可直接复用，零重复实现 |
| `SourceTypeTelegram = "telegram"` 常量已存在 | `internal/database/source.go:14` | 缺 `telegram_user` 常量 |
| 技术选型：Telegram 用**自写 HTTP 客户端**（Bot API），不引 SDK | `PLAN.md` 技术选型 | bot 侧零新增依赖 |
| `.gomodcache` 内**无 gotd**；本机沙箱网络被拦（实测抓 GitHub 报 schannel 错误） | 本机环境 | `telegram_user` 开工前需**联网 `go get github.com/gotd/td`**，是硬阻塞 |
| 前端类型列表硬编码 `TYPES=[{value:'feed',...}]`，表单只渲染 feed 字段 | `NewsGlean-web/src/components/SourceForm.jsx` | 需扩展类型 + 动态配置字段 |

## 3. 两个渠道类型

### 3.1 对比总览

| 维度 | `telegram`（Bot API） | `telegram_user`（MTProto 用户账号） |
|---|---|---|
| 协议 | 官方 Bot API（HTTPS） | MTProto（长连接） |
| 数据范围 | 仅 bot 被拉进去的频道/群（频道需设管理员） | 账号能看到的**全部**实时消息（私聊+群+频道） |
| 实时性 | 长轮询近实时 / webhook 真实时 | 长连接 update 事件，真实时 |
| 封号风险 | 无（bot 独立于个人账号） | **违反 ToS，有封号风险** |
| 新增依赖 | 无（自写 HTTP 客户端） | gotd（`github.com/gotd/td`，tdl 同款底层） |
| 项目既定态度 | ✅ 阶段 15 已规划 | ⛔ PLAN.md 渠道清单标「暂不支持」 |
| 落地难度 | 低 | 高 |

### 3.2 `telegram`（Bot API）

- **操作前提**：@BotFather 建 bot 取 token → 把 bot 加进目标频道/群（频道需给管理员权限）。
- **配置形态**：

```jsonc
// type = "telegram"
{
  "chats": ["@some_channel", "@some_group"],   // 订阅的频道/群列表
  "mode": "poll"                                // poll | webhook
}
// token 不入 config，走 source_credentials（见 §6）
```

- **游标**：`update_id`（`getUpdates` 的 offset = 上次 update_id + 1），存 `source_cursors`。
- **实时方式**：
  - `mode=poll`：`getUpdates` 长轮询（timeout 参数），近实时，**走现有拉取调度即可**（interval 设短）。
  - `mode=webhook`：`setWebhook` → NewsGlean 暴露回调端点，真实时推送，**依赖 §5 的推送接线 + `server.base_url` 公网可达**。
- **探测能力（`Prober`）**：实现 `Probe` 调 `getChat` 返回频道/群标题，支撑前端「自动获取显示名」。

### 3.3 `telegram_user`（用户账号 MTProto）

- **凭据**：`api_id`/`api_hash`（my.telegram.org 申请，半公开）+ 手机号登录 + 验证码 + 2FA，会话落盘（最高风险秘密）。
- **配置形态**：

```jsonc
// type = "telegram_user"
{
  "dialogs": ["@ch1", "@ch2"],   // 订阅的对话列表；为空或 "all" 表示账号可见的全部
  "session_file": "tg_user.session"
}
```

- **实时方式**：`Run` 推送模式——gotd 长连接 `Updates` handler → `emit(Item)`，真实时。
- **回填（防掉线漏消息）**：启动时按游标（各 chat 的 `message_id`）拉历史补上离线期间的消息；否则「只有在线时才收到」。
- **风险隔离**：违反 ToS。建议**独立可选模块**（见 §9 决策点 ③），并在 UI 显著标注封号风险。

## 4. 契约映射（原始消息 → `Item`）

| `Item` 字段 | 来源（Telegram） | 备注 |
|---|---|---|
| `GUID` | `"<type>:<chat_id>:<message_id>"` | 稳定唯一，主去重键；type 为 `telegram`/`telegram_user` |
| `URL` | 公开频道给 `https://t.me/<username>/<message_id>`；私聊为空 | 空则走 GUID 去重分支（PLAN.md 已约定） |
| `Title` | 群/频道标题 + 发送者（或仅标题） | 供列表展示 |
| `Author` | 发送者显示名（first_name + last_name） | |
| `PublishedAt` | 消息 date（unix 秒 → `time.Time`） | 不缺失，无需 InferredTime |
| `Summary` | 空（或消息首行） | |
| `Content` | 消息文本；媒体消息取 caption（无 caption 则空） | 媒体文件本体首版**不下载**，见 §9 ④ |
| `ContentType` | `text/plain`（默认）/ `text/html`（含格式实体） | |
| `Tags` | `[chat 标题或 username, 话题主题(forum topic)]` | |
| `Extra` | `chat_id`/`message_id`/`sender_id`/`reply_to_msg_id`/`forward_from`/`media_type`/`media_file_id` | 逃生口，前端按需展示 |

## 5. 核心层改动：补「推送模式」调度接线

这是本次规划**唯一需要动核心层**的地方，但属于「接线」而非「改契约」——`Run`/`ErrPushUnsupported`/`Metadata.Push` 都已在接口里。

### 5.1 目标数据流

```
拉取源（现有）：scheduler ticker ──> refreshOne ──> conn.Fetch(cursor) ──> ingestItem
推送源（新增）：pushRunner 常驻 ──> conn.Run(ctx, emit) ──> emit(Item) ──> ingestItem（复用）
```

### 5.2 改动点

| 层 | 文件 | 改动 |
|---|---|---|
| 应用服务层 | `internal/app/app.go` | 新增 `pushRunner`：对 `Push=true` 的源调用 `conn.Run(ctx, emit)`；`emit` 包裹 `ingestItem`（去重/写库/进度事件白拿） |
| 调度层 | `internal/scheduler/scheduler.go` | 分流：`Push=true` 的源**不进 ticker**，交给 pushRunner 常驻；启停/启禁用联动 |
| 数据层 | `internal/database/source.go` | 新增 `SourceTypeTelegramUser = "telegram_user"` 常量 |
| 采集日志 | `internal/database/fetch_log.go` | 语义扩展：常驻型源记「连接建立/断开/重连」事件，而非每轮拉取一次 |

### 5.3 生命周期与健壮性

- **启动/启用**时拉起 pushRunner；**禁用/删除/停机**时 `ctx` 取消收敛（复用现有 `wg`/`Stop` 机制）。
- **断线自动重连** + 指数退避（复用阶段 2 的健康度降频思路，但作用对象是「重连间隔」）。
- **手动 `refresh`** 对推送源跳过或降级为「重同步/回填」，并在 UI 提示「该源为实时推送，无需手动刷新」。
- SSE 进度事件沿用 `progressHub`；前端对推送源显示「连接状态」而非「下次刷新时间」。

## 6. 凭据与安全

| 项 | 处理 | 依据 |
|---|---|---|
| Bot Token | 走 `--token-env`，数据库只存环境变量名（如 `NEWSGLEAN_TG_TOKEN`） | 现有 `credentials.store: env` 约定（PLAN.md 凭据处理） |
| 用户账号会话 | `api_id`/`api_hash` + 会话文件路径；文件放工作区私有目录 | 最高风险秘密，待阶段 20 凭据加密 |
| 日志脱敏 | token/会话一律替换为 `***` | 现有约定 |
| 封号风险 | `telegram_user` 加源表单必须显示醒目风险横幅；文档标注「违反 ToS」 | PLAN.md 第 984 行「账号风险渠道默认不并主线」 |

## 7. 后端 / 前端设计

### 7.1 后端

```
internal/source/telegram/          # Bot API
├── register.go                    #   source.Register("telegram", {Pull:true, Push:true}, New, extraFlag)
├── connector.go                   #   Connector：Validate/Init/Fetch/Run/Close
├── botapi.go                      #   自写 getUpdates/setWebhook/getChat/getChatAdministrators
└── testdata/ + *_test.go          #   mock 响应样本（含缺字段/畸形/未加群错误）

internal/source/telegram_user/     # MTProto 用户账号（独立可选模块，见 §9 ③）
├── register.go                    #   source.Register("telegram_user", {Pull:true, Push:true}, New, extraFlag)
├── connector.go                   #   Connector：登录态载入、Run 推送、回填
├── session.go                     #   会话持久化（gotd tdsession）
└── updates.go                     #   Updates → Item 映射
```

- `extraFlag` 各注册专属 flag：`telegram` 注册 `--chat`/`--mode`/`--token-env`；`telegram_user` 注册 `--dialogs`/`--session-file`。
- 两个类型 `Pull` 均置 `true`（都支持历史拉取/回填），`Push` 均置 `true`（都支持实时）。

### 7.2 前端（`NewsGlean-web`）

| 文件 | 改动 |
|---|---|
| `src/components/SourceForm.jsx` | `TYPES` 增加 `telegram`、`telegram_user`；按类型动态渲染配置字段（`chats` 列表、`mode`、token 环境变量名 / `dialogs` + 会话）；`telegram_user` 显示封号风险横幅 |
| `src/api.js` | 复用现有 `createSource`/`probeSource`（已通用），无需新方法 |
| `src/components/SourceList.jsx` | 推送源展示「连接状态」徽标（替代/并列「下次刷新」） |

> 注：`SourceForm.jsx` 当前 `TYPES` 硬编码且仅渲染 `url` 字段（`form.type === 'feed'` 分支），需把「类型 → 字段组」改成数据驱动，本功能是首个逼出这一重构的渠道。

## 8. 分阶段实施（每阶段有 DoD）

| 阶段 | 内容 | DoD |
|---|---|---|
| **T0 决策** | 拍板 §9 五个问题，更新 `PLAN.md` 渠道清单表（`telegram_user` 从「暂不支持」改为「计划中/独立模块」） | 决策落文档 |
| **T1 推送调度接线** | `app.pushRunner` + `scheduler` 分流 + 生命周期/重连 + `fetch_log` 语义 + mock 推送源单测 | 推送型 mock 源经 `Run` 实时入库；`build`/`vet`/`test` 全绿；现有 `feed` 拉取不受影响 |
| **T2 `telegram` poll 版** | `internal/source/telegram/`：getUpdates + `update_id` 游标 + `Item` 映射 + `Probe`；token 走环境变量名 | 加 bot → 拉进频道 → 定时采到消息 → 前端可读；`app`/`database`/`filter` 零改动 |
| **T3 `telegram` webhook 版（可选）** | `setWebhook` + 入站回调端点（依赖 T1 + `base_url`） | bot 消息真实时推送入库 |
| **T4 `telegram_user`** | gotd 登录 + 会话持久化 + `Run` 实时 + 历史回填；独立可选模块 | 账号登录后实时收到消息、重启后回填离线漏消息 |
| **T5 前端 + 凭据收尾** | `SourceForm` 类型扩展 + 动态字段 + 风险横幅 + 连接状态徽标 | 纯浏览器完成「选类型 → 配置 → 加源 → 实时/定时采到消息 → 读」 |

**依赖关系**：T1 是 T3/T4 的前置；T2 不依赖 T1（走现有拉取链路）可最先交付；T4 另需**联网拉 gotd**（§2 环境阻塞）。

## 9. 风险与待拍板决策

1. **类型拆分**：确认 `telegram` 与 `telegram_user` 两个独立类型（而非一个类型带 mode 开关）？—— 推荐前者，与 `PLAN.md` 枚举、`registry` 的 `Pull/Push` 元数据、`database.SourceType*` 常量对齐。
2. **Bot 实时范围**：`telegram` 首版只做 `mode=poll`（长轮询，零推送依赖），webhook 留作可选？—— 推荐首版 poll、webhook 按需再做。
3. **`telegram_user` 隔离方式**：直接内置主进程 vs 独立可选模块（build tag / 单独构建）？—— 推荐后者，符合 PLAN.md「账号风险渠道不并主线」约定；内置则把高风险代码并进主线。
4. **媒体消息首版策略**：首版只采「文本 + caption + 媒体元数据（`media_file_id` 等）」，媒体文件本体下载推迟到图片本地缓存阶段？—— 推荐是，避免首版引入大文件下载与存储复杂度。
5. **`telegram_user` 的默认数据范围**：`dialogs` 缺省是「账号可见全部」还是「必须显式列出对话」？—— 推荐缺省「必须显式列出」，降低误采与封号面。

### 当前推荐默认项（未拍板前按此推进）

- 两个独立类型 `telegram` + `telegram_user`
- `telegram` 首版 `mode=poll`，webhook 可选
- `telegram_user` 独立可选模块（build tag / 单独构建）
- 媒体首版只采文本 + caption + 元数据，不下载媒体本体
- `telegram_user` 必须显式列出对话，不做「默认全量」

### 环境阻塞提示（实施前必须解决）

- `telegram_user` 依赖 gotd，`.gomodcache` 内无此包，且本机沙箱网络被拦；开工 T4 前需**在可联网环境下 `go get github.com/gotd/td` 并 `go mod vendor`**，否则无法构建。
