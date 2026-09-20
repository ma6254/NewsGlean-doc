# NewsGlean 设计方案

> 拾取、清洗、归档你关心的内容。把 RSS、网页、聊天机器人里的信息统一成同一个阅读流。

本文是**设计方案与实现规格**。项目简介、快速开始请看 [README.md](./README.md)。

> **当前进度：M1（v0.1）已完成，阶段 1–9（定时后台调度、礼貌限速 + 健康度自动降频、采集日志可观测性 + SSE 实时刷新进度、阅读状态已读/收藏/归档、FTS5 建表 + 触发器 + 写入同步、中文分词 + 搜索 API、前端搜索界面、导出 Markdown、导出 JSON + EPUB）已完成，后续按 21 个执行阶段推进。** 采集契约、`feed` 渠道、数据层、配置、HTTP API、手动采集链路、三层去重、拉取游标持久化均已实现，`build`/`vet`/`test` 全绿。跨渠道指纹去重、渠道健康度计数、「稍后再阅」、Swagger UI、定时后台调度（ticker + 按渠道 interval + 全局并发上限）、礼貌限速与健康度自动降频、采集日志可观测性（fetch_log + 成功率/耗时/最后成功时间 + 界面展示）、SSE 实时刷新进度与阅读状态（已读 / 收藏 / 归档 + 列表过滤）已提前完成；后续按「阶段性开发目标」的 21 个阶段推进，请勿按本文使用未实现的功能。
> 本文先作为实现规格（spec）使用，实现进度会回写到各章节的勾选标记中。请不要按本文去使用还没有的功能。

### 文档中的两类内容

本文混着两种性质的段落，阅读时请留意区分：

- **契约（已定）** —— 抽象接口、数据模型这类一旦定下就影响全局的设计。它们被单独标注，修改代价高，欢迎现在提意见。
- **草案（待定）** —— 具体库的选型、命令名的取舍、配置字段名。实现时才定稿。

### 目录

- [它解决什么问题](#它解决什么问题)
- [内容来源抽象](#内容来源抽象) ← 核心设计决策
- [功能规划](#功能规划)
- [快速开始](#快速开始)
- [命令设计](#命令设计)
- [架构概览](#架构概览)
- [数据模型](#数据模型)
- [配置](#配置)
- [阶段性开发目标](#阶段性开发目标)
- [路线图](#路线图)
- [技术选型（草案）](#技术选型草案)
- [开发](#开发)
- [贡献](#贡献)
- [License](#license)

---

## 它解决什么问题

RSS 只给你「条目」：标题、链接、一小段摘要，正文留在原站，读起来还得一条条点开。而且很多你想看的内容根本没有 feed —— 一个只发在公众号里的号、一个没有订阅入口的新闻列表页、一个只在群里转发的消息流。

现有工具的两难：

- **RSS 阅读器** 优雅，但只认 feed，墙外的内容进不来。
- **爬虫/机器人脚本** 抓得到，但抓完就散：没有统一的已读状态、没有去重、没有归档、没有搜索，通常就是往数据库一写或者发个通知了事。
- **RSSHub 之类的桥接方案** 很实用（本项目也打算直接支持对接），但每个源都要自己搭一套路由，且桥挂了你就断了。

NewsGlean 的思路：**把「内容怎么来」和「内容怎么读」彻底分开。**

1. **取** —— 内容来源是可插拔的适配器：RSS/Atom、网页列表页、Telegram/QQ 机器人、Webhook……接口统一。
2. **净** —— 抓取原文正文，剥离广告与导航，正文可读可检索。
3. **筛** —— 按关键词/正则/来源规则过滤，跨源去重，只留你真正要看的。
4. **读** —— 浏览器界面 + REST API，同一份数据多种入口，不管内容原本来自哪里。
5. **存** —— 导出 Markdown / JSON / EPUB，数据永远在本地。

两条硬约束：

- **本地优先**。默认单进程、单文件数据库，不依赖任何外部服务；LLM 能力是可选增强项，不配置也完整可用。
- **采集方式可插拔**。核心流程只认「规范化条目」，不认识任何具体来源。新增一个渠道不应该改动去重、过滤、存储、阅读这些模块的任何一行代码。

---

## 内容来源抽象

这是本项目的核心设计决策，因此单独成章。**接口一旦定下就属于「契约」，改动代价高。**

### 三种接入模式

抽象的关键在于：不同渠道的**到达方式**根本不同，不能只写一个 `Fetch()` 假装它们一样。

| 模式 | 说明 | 典型渠道 |
| --- | --- | --- |
| **拉取（pull）** | 定时主动去问「有新的吗」，用游标标记进度 | RSS/Atom、网页列表页、Telegram `getUpdates` / 频道历史 |
| **推送（push）** | 由外部把消息送到我们暴露的 HTTP 端点，实时写入 | Telegram webhook、QQ 官方 Bot 回调、通用 Webhook、邮件转发 |
| **本地（local）** | 不联网，从文件系统或标准输入读入 | 手工 OPML/Markdown/JSON 导入、目录监控 |

因此接口不叫 `Feed`，而叫 `Source` / `Connector`：一个渠道实例。`SourceType`（渠道类型）与 `Source`（渠道实例）是两回事 —— 同一个 `telegram` 类型可以实例化出多个机器人源。

### 契约：Connector 接口

```go
package source

// Connector 是一个已配置好的渠道实例。
// 实现方只负责「拿到原始内容并规范化」，不负责去重、过滤、存储。
type Connector interface {
	// Type 返回渠道类型标识，如 "feed"、"webpage"、"telegram"。
	Type() string

	// Validate 在保存配置前校验参数与凭据，尽早报错而不是等到定时任务静默失败。
	Validate(ctx context.Context) error

	// Init 建立长连接或载入游标（例如 Telegram 的 bot 信息、网页源的会话）。
	// 必须可重入：进程重启会再次调用。
	Init(ctx context.Context, state State) error

	// Fetch 拉取一批新条目。游标由本适配器自行解释，
	// 核心层只负责持久化与去重，不理解游标含义。
	// 返回 io.EOF 或空切片表示「暂无新内容」，不算错误。
	Fetch(ctx context.Context, cursor Cursor, limit int) ([]Item, Cursor, error)

	// Run 以推送模式运行，阻塞直到 ctx 取消。
	// 返回 ErrPushUnsupported 表示本渠道只支持拉取，核心层会改为定时调用 Fetch。
	Run(ctx context.Context, emit func(Item) error) error

	Close() error
}
```

设计取舍，逐条说明理由：

- **游标由适配器自己解释**（`Cursor` 是不透明字节串）。RSS 存 `ETag`/`Last-Modified`，Telegram 存 `update_id` 或最后一条 `message_id`，网页爬虫存上次见到的最新链接与发布时间。硬要在核心层统一这些概念只会得到一个什么都不是的抽象。
- **去重不在适配器里做**。适配器只管产出，判定「这条我见过没有」是核心层的事，这样三层去重逻辑只实现一次。
- **`Run` 与 `Fetch` 并存**。推送型渠道必须能自己持有长连接或 HTTP 回调，否则 Telegram webhook 这类渠道根本没法实现。
- **`Validate` 单独存在**。凭据写错是最常见的故障，必须在 `source add` 当场报错，而不是等到凌晨的定时任务里连续失败。

### 契约：Prober（可选能力）

渠道可额外实现 `Prober`，用于保存前探测元信息（如 feed 标题），支撑前端「自动获取显示名」。未实现的渠道由核心层返回 `ErrProbeUnsupported`：

```go
// Prober 是 Connector 的可选能力：探测渠道元信息。
type Prober interface {
	// Probe 探测渠道元信息。实现应复用与 Fetch 一致的抓取逻辑与超时约束。
	Probe(ctx context.Context) (ProbeInfo, error)
}

// ProbeInfo 是渠道在保存前可探测到的元信息。
type ProbeInfo struct {
	Title string `json:"title"` // 渠道标题（如 feed 的 <title>），可能为空
}
```

当前 `feed` 已实现（探测 RSS/Atom/JSON Feed 的标题），HTTP 层对应 `POST /api/source/probe`。

### 契约：规范化条目

所有渠道都必须收敛到同一个结构，这是「内容怎么来」与「内容怎么读」之间唯一的接口：

```go
// ContentStatus 表示 Content 相对原文的完整程度。
type ContentStatus string

const (
	ContentStatusEmpty     ContentStatus = "empty"     // Content 为空，无任何正文
	ContentStatusSummary   ContentStatus = "summary"   // 只有摘要，正文缺失
	ContentStatusTruncated ContentStatus = "truncated" // 有正文但被截断
	ContentStatusFull      ContentStatus = "full"      // 判定为全文
	ContentStatusUnknown   ContentStatus = "unknown"   // 无法判定
)

type Item struct {
	// 身份（按优先级去重：GUID → URL → ContentHash，三者都缺则该条目被丢弃并告警）
	GUID string // 渠道自带的稳定 ID（RSS 的 guid、TG 的 chat_id+message_id）
	URL  string

	Title       string
	Author      string
	PublishedAt time.Time // 缺失时用抓取时间，并标记 InferredTime=true
	Summary     string    // 渠道自带的摘要（如 RSS description），可能为空
	Content       string        // 渠道自带正文；为空则交给正文提取模块去原文取
	ContentStatus ContentStatus // 正文完整度判定：适配器给基准值，核心层启发式可降级
	ContentType   string        // "text/html" / "text/plain" / "text/markdown"

	SourceID  string            // 归属渠道实例
	Tags      []string          // 渠道自带标签（如 Telegram 话题、网页栏目标签）
	Extra     map[string]string // 渠道特有字段，原样透传，不做规范化
	FetchedAt time.Time
}
```

`ContentStatus` 的语义与来源：适配器只报「我读到了哪个字段」这一客观事实并赋基准值（RSS 有 `content:encoded` → `full`，只有 `description` → `summary`；Atom 有 `content` → `full`；JSON Feed 有 `content_html`/`content_text` → `full`；正文全空 → `empty`）；核心层用一个纯函数启发式对 `full`/`unknown` 扫描截断特征（结尾 `…`/`...`/`阅读全文`/`Read more` 等标记、正文/摘要长度比异常），命中则降级为 `truncated`。RSS 的 `content:encoded` 只是「惯例=全文」而非规范保证，故其 `full` 保留降级空间。该字段属契约变更：`feed` 适配器需补赋值、`entries` 经 `AutoMigrate` 增加 `content_status` 列、存量行按 `unknown` 解释。

`Extra` 是刻意保留的逃生口：QQ/TG 消息有回复链、转发来源、群号之类 RSS 完全没有的概念。与其为每个渠道往 `Item` 上加字段，不如让它们塞进 `Extra`，Web 界面按需展示。

### 渠道实现清单

| 渠道 | 类型标识 | 接入方式 | 凭据 | 稳定性 | 状态 |
| --- | --- | --- | --- | --- | --- |
| RSS / Atom / JSON Feed | `feed` | 拉取 | 无 | 高（开放标准） | 已实现（v0.1），支持 `http` / `chromedp` / `chromedp_headed` 三种抓取方式 |
| OPML / 本地文件导入 | `import` | 本地 | 无 | 高 | 计划中 |
| 网页列表页爬取 | `webpage` | 拉取 | 无（部分站点需 Cookie） | 中（页面改版即失效） | 计划中 |
| 通用 Webhook / HTTP 入站 | `webhook` | 推送 | 自定义共享密钥 | 高 | 计划中 |
| Telegram 频道 / 群 | `telegram` | 拉取（`getUpdates`）或推送（webhook） | Bot Token | 高（官方 API 稳定） | 计划中 |
| Telegram 用户会话 | `telegram_user` | 拉取 | 账号登录态 | 低（违反 ToS，有封号风险） | 暂不支持 |
| QQ 官方机器人 | `qq_bot` | 推送（官方回调） | AppID + Token + 公钥 | 中（需平台审核，能力受官方限制） | 计划中 |
| QQ 非官方协议 | `qq_unofficial` | 拉取/推送 | 账号登录态 | 极低（协议逆向，随时失效，有封号风险） | 需单独讨论 |
| 微信公众号 | `wechat_mp` | 见下方专门说明 | — | — | 见下方 |

### 关于微信公众号：需要单独决策

必须先说清楚：**微信公众号没有官方开放的内容读取 API。** 想拿到公众号文章，现实里只有这几条路，没有一条是干净的：

1. **RSSHub 之类的第三方桥接** —— 目前最省事的选择。它在外面替你处理了抓取问题，NewsGlean 只需把它当普通 `feed` 源订阅。**推荐做法**，零额外风险。
2. **搜狗微信搜索抓取** —— 反爬严格、验证码频繁、结果不全，需要维护 Cookie 池。属于高维护成本、低可靠性。
3. **微信读书 / 公众号后台接口** —— 依赖私有接口与登录态，随时可能失效。
4. **PC 客户端 Hook / 自动化** —— 逆向客户端，违反用户协议，可能涉及账号封禁与法律风险。**本项目不打算实现。**

因此本项目的态度是：**把公众号当作「通过桥接进来的普通 feed」处理，不自己造第 2–4 条路。** 如果你明确需要某条具体路线，请先开 Issue 讨论，尤其是涉及账号风险的方案需要你自行评估并承担后果 —— 我不会把它默认做进主线。

### 关于 QQ 与微信机器人：先确认用途

「机器人」这个词有两种完全不同的含义，实现方向差别很大：

- **A. 只读采集** —— 机器人作为观察者，把群/频道里的消息读进来，进入阅读流。这是本文档上面描述的场景。
- **B. 双向交互** —— 你还能通过机器人发指令、做搜索、收推送、标记已读。这需要额外一套「出站」抽象。

本文档只覆盖 **A**。**B 需要单独设计**，因为出站动作的语义（发消息、改状态）跟入库完全不同。若你要 B，请说明，我会补一节「动作/出站接口」。

### 新增一个渠道的步骤

理想情况下，接入新渠道**只写一个包、不改核心**：

1. 在 `internal/source/<name>/` 下实现 `Connector`，把原始数据映射成 `Item`。
2. 在 `internal/source/registry.go` 注册类型标识与构造函数。
3. 在 `internal/source/<name>/testdata/` 放样本（含畸形响应、缺字段、编码异常），补齐单元测试。
4. 在本文档的渠道清单表里加一行。

如果第 1 步发现「不改核心就做不到」，说明抽象漏了东西 —— 这时应该改抽象，而不是往核心层打补丁。

---

## 功能规划

图例：`[x]` 已实现 · `[ ]` 计划中 · `[~]` 部分实现

### 采集渠道
- [x] 渠道抽象层：`Connector` 接口 + 类型注册表（见「内容来源抽象」）
- [x] 渠道实例管理：添加 / 删除 / 启停，每个实例独立配置与刷新间隔
- [~] `feed`：RSS 2.0、Atom、JSON Feed 解析（已完成），OPML 导入与导出（阶段 17）
- [ ] `import`：从本地 Markdown / JSON / OPML 批量导入
- [ ] `webpage`：通用网页列表页爬取（CSS 选择器配置，无需改代码）
- [ ] `webhook`：入站 HTTP 端点，供任意脚本 push 内容
- [ ] `telegram`：频道/群消息采集（拉取与 webhook 两种模式）
- [ ] `qq_bot`：QQ 官方机器人消息采集（待评估，需平台审核）
- [x] RSSHub 等第三方桥接源按普通 `feed` 直接对接（`feed` 已实现，无需额外开发）
- [~] 渠道健康度：连续失败计数、最近错误、自动降频（指数退避）已实现；UI 告警待做

### 抓取与提取
- [~] 定时后台刷新（按渠道间隔 + 全局并发上限 + 礼貌限速已完成）
- [~] HTTP 条件请求（`ETag` / `Last-Modified`，含游标持久化）已实现；指数退避重试待做
- [ ] 内容完整度判定（`full`/`summary`/`truncated`/`unknown`/`empty` 五态，独立小阶段，先于正文提取）
- [ ] 全文抓取与正文提取（剥离广告、导航、评论）
- [ ] 编码自动识别（含 GBK/GB18030 等中文站点常见编码）
- [ ] 图片可选本地缓存，避免原文失效后图片丢失
- [ ] 渠道级凭据加密存储（Bot Token 等不落明文到配置文件）

### 去重与过滤
- [x] 三层去重：条目 GUID/ID → 规范化链接 → 正文内容指纹
- [ ] 规则过滤：包含/排除关键词、正则、按来源与作者
- [x] 阅读状态（`entry_state` 表）：已读 / 收藏 / 归档 / 稍后再阅 全部接线

### 阅读
- [~] Web 界面（React + Vite + Tailwind，独立仓库 NewsGlean-web）：列表 / 详情 / 渠道管理 / 稍后再阅 / 收藏 / 归档 / 已读标记与过滤已完成；搜索、快捷键待做
- [~] REST API `/api`：条目、渠道、手动采集、稍后再阅、已读 / 收藏 / 归档状态、检索、Swagger UI、Markdown / JSON / EPUB 导出已完成
- [x] 全文检索（SQLite FTS5 bigram 中文分词 + `/api/search` 已完成，阶段 5–6；MySQL FULLTEXT 待阶段 20）
- [ ] 实时推送刷新进度（SSE）
- [ ] 一次性 CLI 子命令（`list` / `read` / `search`）作为 API 的轻客户端

### 导出与集成
- [x] 导出 Markdown（按源/日期分目录，含 YAML front matter）
- [x] 导出 JSON（便于喂给其他脚本）
- [x] 导出 EPUB（离线整期阅读）
- [x] 导出下载端点 + 前端导出按钮（`GET /api/export/download?format=markdown|json|epub`，浏览器直接下载；v0.2 联调收尾）
- [ ] Webhook 通知（新条目推送到自建服务）

### 智能增强（可选）
- [ ] LLM 摘要：每条目的短摘要 + 定期「今日概览」
- [ ] LLM 分类与打标、相关性打分
- [ ] 兼容 OpenAI 风格 API，可指向任意自建/第三方端点；未配置密钥时整块功能静默关闭

---

## 快速开始

构建步骤与环境要求见 [README.md 的「快速开始」](./README.md#快速开始)，此处不重复。

本节只保留**各渠道的配置参考**。

### 期望的首次使用流程（蓝图，尚未实现）

```bash
# 启动服务（数据库与目录首次运行时自动创建，无需单独的 init 命令）
news-glean -c ./config.yml -d ./release

# 以下子命令通过 server.http_addr 调用上面的服务
news-glean source add feed --url https://example.com/atom.xml
news-glean refresh                     # 手动触发一轮采集（推送型渠道无需此步）
news-glean list --unread               # 终端看列表
news-glean search 关键词                # 全文检索
# 交互式阅读走 Web 界面：http://127.0.0.1:28080
```

### 各渠道的配置示例（蓝图）

拉取型渠道用 `source add` 当场配置并校验：

```bash
# RSS / Atom / JSON Feed
news-glean source add feed --url https://example.com/feed.xml --interval 30m

# 网页列表页：选择器决定「什么算一条」，在浏览器开发者工具里复制即可
news-glean source add webpage \
  --url https://example.com/news \
  --selector "div.article-list > a" \
  --full-text

# Telegram 频道（只需 Bot Token，无需登录你的个人账号）
news-glean source add telegram --chat @some_channel --token-env NEWSGLEAN_TG_TOKEN

# 通用入站 Webhook：拿到的地址交给任何脚本去 POST
news-glean source add webhook --name my-scripts
# → http://127.0.0.1:28080/ingest/<随机密钥>
```

推送型渠道把消息 POST 到入站端点或由渠道回调驱动，例如：

```bash
curl -X POST http://127.0.0.1:28080/ingest/<随机密钥> \
  -H 'Content-Type: application/json' \
  -d '{"title":"构建失败","url":"https://ci.example.com/1234","tags":["ci"]}'
```

> 凭据一律推荐用 `--token-env` 读取环境变量，而不是写进命令行或配置文件 —— 命令行会进 shell history，配置文件容易误提交。具体做法见「配置」一节。

---

## 命令设计

单一可执行文件 `news-glean`。主流程是 `serve` 常驻进程，其余子命令是**基于 HTTP API 的一次性客户端**（照参考项目的思路：服务持有状态，客户端只是入口）。下表是目标形态，右列为当前状态。

| 命令 | 作用 | 状态 |
| --- | --- | --- |
| `news-glean` | 等价于 `serve`（根命令即启动服务，照参考项目写法） | 已实现 |
| `news-glean source types` | 列出可用渠道类型及其参数说明 | 计划中 |
| `news-glean source add <type>` | 添加渠道实例（URL/Token 等由各渠道自己的 flag 提供） | 计划中 |
| `news-glean source ls [--type telegram]` | 列出渠道实例，含健康度与最近错误 | 计划中 |
| `news-glean source rm\|enable\|disable\|test <id>` | 删除 / 启停 / 连通性自检 | 计划中 |
| `news-glean refresh [source...]` / `--type feed` | 手动触发拉取；推送型渠道会被跳过并提示 | 计划中 |
| `news-glean list` | 列出条目，支持 `--unread` `--source` `--type` `--since` `--json` | 计划中 |
| `news-glean read <id>` | 在终端渲染单条（正文 + 元数据） | 计划中 |
| `news-glean search <keyword>` | 全文检索 | 计划中 |
| `news-glean ingest <source-id>` | 从 stdin 读入一条内容（`webhook`/`import` 的手工等价物） | 计划中 |
| `news-glean opml import\|export` | OPML 互迁（仅涉及 `feed` 渠道） | 计划中 |
| `news-glean export --format md\|json\|epub` | 导出条目 | 计划中 |
| `news-glean version` | 打印版本、commit、构建时间（读 `internal/build`） | 已实现 |
| `news-glean --help` | Cobra 帮助 | 已实现 |

全局 flag（照参考项目）：

| flag | 默认值 | 作用 |
| --- | --- | --- |
| `-c, --config` | `./config.yml` | 配置文件路径 |
| `-d, --dir` | `./` | 工作目录，启动前 `os.Chdir` |

全局约定：

- **服务是唯一权威**。`list` / `read` / `refresh` 等子命令通过 `-c` 读到的 `server.http_addr` 去调用 API，不直接开数据库 —— 避免「命令行改的」和「网页看到的」出现不一致。
- 所有列表型命令支持 `--json`，便于脚本消费。
- 破坏性操作（`source rm`、`opml import` 覆盖）默认二次确认，`--yes` 跳过。
- 渠道参数统一走 `--token-env VAR_NAME` 形式读环境变量，避免凭据进 shell history。
- **推送型渠道必须靠 `serve` 常驻**才能接收回调。纯 CLI 一次性调用无法接收 webhook，这个限制会在文档和 `source add` 输出里明确提示。

### 为什么是 `source` 而不是 `feed`

早期草稿用的是 `feed`。改动原因是：一旦渠道可插拔，`feed add` 这个命令名会误导 —— Telegram 频道不是 feed，入站 webhook 更不是。`source` 覆盖「一个内容来源实例」的全部含义，代价是失去一点行业惯用词。如果你更偏好 `feed`（迁移成本更低、更熟悉），现在说还来得及。

---

## 架构概览

### 分层

```
  外部世界                采集层（可插拔）              核心层                   HTTP 层
┌───────────────┐
│ RSS / Atom    │──┐
│ 网页列表页    │  │   ┌────────────────────┐
│ Telegram      │──┼──>│  Connector 适配器  │
│ QQ 官方机器人 │  │   │  feed / webpage /  │
│ Webhook 推送  │  │   │  telegram / ...    │
│ 本地文件      │──┘   └─────────┬──────────┘
└───────────────┘                │  Item（唯一契约）
                                 v
                  ┌──────────────────────────────┐
                  │        应用服务层            │
                  │  去重 / 过滤 / 正文提取 /     │
                  │  检索 / 导出 的统一编排       │
                  └──────────────┬───────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              v                  v                  v
   存储 (GORM: sqlite/mysql)  调度 (scheduler)   LLM 客户端 (可选)
                                 │
                                 v
                  ┌──────────────────────────────┐
                  │   server 包（HTTP 层）        │
                  │   路由 / 会话鉴权 /           │
                  │   Swagger / 入站端点          │
                  └──────────────┬───────────────┘
                                 v
                  REST API（/api）──> Web 前端（React SPA，静态资源）
```

三个关键点：

1. **采集层是唯一的可变部分**。核心层与 HTTP 层只认识 `Item`，不认识任何具体渠道。新增渠道 = 新增一个适配器包，其余模块零改动。
2. **HTTP 层是唯一对外的门**。Web 界面、外部脚本、将来的移动端都走同一套 `/api`，不存在「只有网页能用」的能力。
3. **采集与调度跑在 `serve` 进程内**，不另开进程；推送型渠道的入站端点也挂在同一个 `Mux` 上。

### 数据流

```
┌── 采集（按渠道类型分流）────────────────────────────────────────────┐
│                                                                     │
│  拉取型: 调度器 ──> Connector.Fetch(cursor) ──> 原始响应            │
│  推送型: HTTP 入站 / 长连接 ──> Connector.Run(emit) ──> 原始消息    │
│                                                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               v
                   规范化 ──> Item（统一结构，见上文章节）
                               │
                               v
                 去重（GUID → URL → ContentHash）
                               │
                               v
                    规则过滤（保留 / 丢弃 / 标记）
                               │
              ┌────────────────┴─────────────────┐
              v                                  v
     内容为空 ──> 正文提取（回原站取全文）   内容已足够，直接用
              │                                  │
              └────────────────┬─────────────────┘
                               v
                   写入数据库（GORM）+ 检索索引
                               │
                               ├──> 可选：LLM 摘要 / 分类
                               ├──> 可选：Webhook 通知
                               └──> 通过 /api 暴露给 Web 前端
```

注意正文提取的位置：它在规范化**之后**，因为很多渠道（Telegram、Webhook 推送）本身就带完整内容，不需要回原站再抓一次。是否有正文是 `Item.Content` 是否为空决定的，与渠道类型无关。

### 项目结构

结构参考同组织的 `bookcocoon-server`：**cobra 入口 + 顶层包（用 `internal/` 包裹防外部 import）+ `server` 包集中放 HTTP 层 + 每实体一个文件的数据层**。

```
.
├── main.go                    # 入口：Swagger 注解 + cmd.Execute()
├── go.mod / go.sum
├── vendor/                    # go mod vendor，保证可复现构建
├── cmd/
│   └── root.go                # 根命令：解析 flag、启服务、信号处理
├── internal/                  # 业务代码，禁止外部 import
│   ├── build/                 # BuildTime / BuildVersion，由 ldflags 注入
│   ├── config/                # YAML 配置结构与加载（含默认值）
│   ├── database/              # GORM 数据层，每实体一个文件
│   │   ├── database.go        #   连接、AutoMigrate、Install()
│   │   ├── source.go          #   渠道实例
│   │   ├── credential.go      #   渠道凭据（与 source 分表）
│   │   ├── entry.go           #   条目
│   │   ├── entry_state.go     #   已读 / 收藏 / 归档 / 稍后再阅
│   │   ├── source_cursor.go   #   拉取游标持久化
│   │   ├── rule.go            #   过滤规则
│   │   └── fetch_log.go       #   采集历史
│   ├── source/                # ★ 采集抽象层
│   │   ├── source.go          #   Connector / Item / Cursor / State 契约
│   │   ├── probe.go           #   Prober 可选能力：探测渠道元信息
│   │   ├── registry.go        #   类型标识 → 构造函数 注册表
│   │   ├── feed/              #   RSS / Atom / JSON Feed
│   │   ├── webpage/           #   网页列表页爬取
│   │   ├── webhook/           #   通用入站
│   │   ├── telegram/          #   Telegram Bot API
│   │   └── opml/              #   OPML 导入导出
│   ├── server/                # HTTP 层（照搬参考项目的组织方式）
│   │   ├── server.go          #   Server 结构体、Run/Stop、会话、响应工具
│   │   ├── session.go         #   会话与 token 生命周期
│   │   ├── route.go           #   路由注册（方法 + 路径模式）
│   │   ├── source_api.go      #   渠道 CRUD
│   │   ├── entry_api.go       #   条目列表 / 详情 / 已读标记
│   │   ├── search_api.go      #   全文检索
│   │   ├── export_api.go      #   导出
│   │   ├── ingest_api.go      #   入站推送端点
│   │   ├── install.go         #   首次安装（建表、生成入站密钥）
│   │   └── tokens/            #   token 格式校验
│   ├── app/                   # 应用服务层：编排采集、去重、过滤、写库
│   ├── fetch/                 # 共享 HTTP 客户端、条件请求、重试、限速
│   ├── extract/               # 正文抽取与清洗
│   ├── filter/                # 去重、关键词/正则规则
│   ├── scheduler/             # 定时调度（拉取型渠道）
│   ├── llm/                   # LLM 客户端与提示词（可选）
│   ├── export/                # Markdown / JSON / EPUB 导出
│   ├── webui/                 # 内嵌前端构建产物（go:embed NewsGlean-web 的 dist）
│   ├── utils/                 # 通用工具
│   └── validator/             # 输入校验（正则 + 长度，错误信息可读）
├── docs/
│   ├── default.yml            # 默认配置，首次构建时复制到 release/
│   └── swagger.json|yaml|docs.go   # swag init 生成，加入 .gitignore 可选项
├── build.ps1                  # swag init + go build + 注入版本 + 复制默认配置
├── build_run.ps1              # tidy → vendor → build → 清理数据 → 运行
├── run.ps1                    # ./release/app.exe -d release
├── release/                   # 构建产物（已在 .gitignore）
└── .agent/skills/             # 项目内的 agent 技能（可选，见下）
```

约定说明：

- `cmd/root.go` 保持薄：解析 flag → 读配置 → `server.NewServer(c)` → `Run()` → 等待信号 → `Stop()`。命令逻辑不写在这里。
- **不设 TUI**。终端用户通过 `list` / `read` 这类一次性子命令访问 HTTP API，交互式阅读走 Web 界面。
- `internal/source/` 是最值得先定稿的目录 —— 它的接口改了，上层全都要跟着改。
- 前端是独立仓库 NewsGlean-web（React + Vite + Tailwind），构建产物经 `internal/webui` 用 `go:embed` 打进二进制；发布流程 = 先 `npm run build` 再 `go build`。
- 参考项目另有 `.agent/skills/`（cobra-cli-design、database-design、http-api-design、yaml-config-design 等 7 项）。要不要在 NewsGlean 也放一套，由你决定；放了的好处是后续 agent 协作时约定的落点一致。

---

## 数据模型

采用 **GORM**（与 `bookcocoon-server` 一致），每张表一个文件放在 `internal/database/`，建表走 `AutoMigrate`，首次安装由 `Install()` 统一调用。

| 文件 → 表 | 说明 |
| --- | --- |
| `source.go` → `sources` | 渠道实例：`type`（`feed`/`webpage`/`telegram`/…）、显示名、启用状态、刷新间隔、配置 JSON、健康度 |
| `credential.go` → `source_credentials` | 渠道凭据（Bot Token 等）。**与 `sources` 分表**，便于单独加密、单独备份排除、单独审计 |
| `source_cursor.go` → `source_cursors` | 拉取游标（不透明字节串）与状态，如 RSS 的 ETag、Telegram 的 `update_id` |
| `entry.go` → `entries` | 条目：所属渠道、GUID、链接、标题、作者、发布时间、摘要、正文、内容完整度、内容指纹、提取状态 |
| `entry_state.go` → `entry_state` | 条目阅读状态：已读、收藏、归档、稍后读 |
| `entry_meta.go` → `entry_meta` | 渠道特有字段（对应 `Item.Extra`），键值对存储，核心逻辑不解析其含义 |
| `rule.go` → `rules` | 过滤规则：类型（关键词/正则）、字段、动作（保留/丢弃/标记）、优先级 |
| `tag.go` → `tags` / `entry_tags` | 标签与条目关联（含渠道自带标签） |
| `summary.go` → `summaries` | LLM 产出的摘要与分类结果（含模型名，便于区分来源） |
| `fetch_log.go` → `fetch_log` | 采集历史：时间、耗时、新增条目数、错误信息，用于排障 |

当前已落地的表：`sources`、`entries`、`source_cursors`、`entry_state`、`fetch_log`。其中：

- `source_cursors` 存拉取游标（不透明字节串），`feed` 的 `ETag`/`Last-Modified` 条件请求依赖它持久化。
- `entry_state` 按四状态建模（`read` / `favorite` / `archive` / `read_later`），阶段 4 起四状态全部接线：已读 / 收藏 / 归档均有对应的 `PUT /api/entry/{id}/<state>` 端点，`GET /api/entry/list` 支持 `read` / `favorite` / `archive` 布尔过滤参数。
- `fetch_log` 记采集历史（耗时、新增/跳过条数、成功、错误），聚合出每渠道成功率 / 最后成功时间 / 最近耗时，用于可观测性与排障。
- 其余表随对应阶段逐个落地。

### 结构体风格

照参考项目的写法：显式 `column` 标签、行尾中文注释、软删除用 `deleted` 字段而非物理删除。

```go
package database

const (
	SourceTypeFeed     = "feed"     // RSS / Atom / JSON Feed
	SourceTypeWebpage  = "webpage"  // 网页列表页爬取
	SourceTypeTelegram = "telegram" // Telegram 频道/群
	SourceTypeWebhook  = "webhook"  // 通用入站
)

var (
	ErrorSourceNotFound = errors.New("source not found")
)

type Source struct {
	ID        uint64 `gorm:"column:id;unique;primaryKey;autoIncrement"` // 渠道实例ID
	Name      string `gorm:"column:name"`                              // 显示名
	Type      string `gorm:"column:type"`                              // 渠道类型标识
	Config    string `gorm:"column:config;type:text"`                  // 渠道配置 JSON，核心层不解析
	Interval  int    `gorm:"column:interval"`                          // 刷新间隔（秒）
	Enabled   bool   `gorm:"column:enabled"`                           // 是否启用
	FailCount int    `gorm:"column:fail_count"`                        // 连续失败次数
	LastError string `gorm:"column:last_error;type:text"`              // 最近一次错误
	CreatedAt string `gorm:"column:created_at"`                        // 创建时间
	UpdatedAt string `gorm:"column:updated_at"`                        // 更新时间
	Deleted   bool   `gorm:"column:deleted"`                           // 删除标记，软删除
}
```

`sources.config` 的形态取决于类型，例如：

```jsonc
// type = "webpage"
{ "url": "https://example.com/news", "selector": "div.article-list > a", "full_text": true }

// type = "telegram"
{ "chat": "@some_channel", "mode": "poll" }   // token 在 source_credentials 表
```

核心层只把 `config` 当成不透明 JSON 转交给对应的适配器解析，**不为其建索引、不做类型判断**。这是让新渠道不改核心的代价与前提。

### 两个要留意的取舍

1. **时间字段用 RFC3339 字符串**（照参考项目）。GORM 默认会用 `time.Time` 并用数据库原生类型存储，两种都可以，但**必须全项目统一** —— 混用会让比较与排序出错。参考项目选字符串的原因是 sqlite/mysql 一致性好读；代价是范围查询必须保证格式规范。
2. **全文搜索与 `AutoMigrate` 的冲突**。SQLite FTS5 需要虚拟表与触发器，`AutoMigrate` 不会创建。方案是在 `Install()` 里对 FTS 表单独执行 `db.Exec(...)` 建表语句，并明确写进迁移说明，不指望 ORM 代劳。

去重键的选择顺序：GUID → 规范化后的链接 → 正文内容指纹（标题 + 正文归一化后的哈希，用于识别改标题重发的稿件）。

采集层带来的两条去重补充规则：

- **无 URL 的渠道**（Telegram 消息、Webhook 推送）走 GUID 分支，GUID 由适配器保证稳定（如 `telegram:<chat_id>:<message_id>`）。这类条目导出 Markdown 时不写 `url` 字段。
- **同一内容多渠道重复**是常态（RSS 与 Telegram 频道常常推同一条新闻）。此时 GUID 与 URL 都不同，只有内容指纹能识别。默认行为是**保留最早入库的一条，其余标记为重复并隐藏**，不直接删除 —— 误判时还能找回来。

---

## 配置

配置优先级：**命令行参数 > 环境变量 > 配置文件 > 内置默认值**。

- 配置文件路径：默认 `./config.yml`，由 `-c, --config` 指定。发布包里的 `release/config.yml` 由 `build.ps1` 首次构建时从 `docs/default.yml` 复制。
- 工作目录：`-d, --dir` 指定，启动时 `os.Chdir` 过去，数据库等相对路径都基于它解析（照参考项目）。
- 环境变量前缀：`NEWSGLEAN_`，例如 `NEWSGLEAN_HTTP_PROXY`、`NEWSGLEAN_LLM_API_KEY`。
- 数据库默认路径：`database.sqlite.file`，默认 `./data.db`（相对 `-d` 工作目录）。

配置骨架（规划）：

```yaml
# 日志参数（照参考项目）
log:
  dir: "log"          # 日志文件路径
  level: "info"       # 日志级别：debug, info, warn, error
  max_size: 10        # 日志文件最大大小（MB）
  max_backups: 5      # 最大备份数量
  max_age: 30         # 最大保存天数

# HTTP 服务参数
server:
  http_addr: "127.0.0.1:28080"  # 默认仅监听本机
  base_url: ""                 # 反向代理/公网回调场景下的外部地址

# Web 前端参数
web:
  mode: embed                  # 前端加入方式：embed（默认，内嵌二进制）| proxy（反代外部服务）| dir（本地目录）| gz（打包文件）| off（仅 API）
  proxy_url: "http://127.0.0.1:38080"  # mode=proxy 时的前端服务地址（Vite dev server）
  dir: "./web"                 # mode=dir 时的静态资源目录
  archive: ""                  # mode=gz 时的打包文件路径（.tar.gz）

# 数据库参数（照参考项目）
database:
  driver: "sqlite"    # sqlite（默认） | mysql
  sqlite:
    file: "./data.db"
  mysql:
    host: "127.0.0.1"
    port: 3306
    user: ""
    password: ""
    database: "news_glean"

# 采集参数
fetch:
  interval: 30m                # 全局默认刷新间隔
  concurrency: 4               # 并发采集上限
  rate_limit: 1s               # 后台采集全局请求最小间隔（礼貌限速）；空或 0 表示不限速
  timeout: 20s
  user_agent: "NewsGlean/0.1 (+https://github.com/ma6254/news-glean)"
  proxy: ""                    # 留空则读环境变量
  chrome_path: ""              # Chrome 可执行文件路径，留空自动探测（chromedp 抓取用）
  full_text: auto              # 回源抓全文策略：auto（默认，按 ContentStatus 决定）| on（强制全抓）| off（永不回源）

# 凭据引用方式。适配器读取凭据时统一走这一层，不直接读环境变量。
credentials:
  store: env                   # env（默认，仅存变量名） | file（加密落盘）
  passphrase_env: NEWSGLEAN_MASTER_KEY

# 渠道实例的实际配置存数据库，不写这里，避免配置文件被误提交时泄露渠道参数

# 过滤
filter:
  dedup: true
  cross_source_dedup: true     # 跨渠道内容指纹去重
  rules: []                    # 见「去重与过滤」

# LLM（可选）
llm:
  enabled: false               # 关闭时以下字段全部忽略
  base_url: "https://api.openai.com/v1"
  api_key: ""
  model: ""
  summarize: false
  classify: false

# 导出
export:
  markdown_dir: "./export"
  epub: false
```

这份骨架同时要落成 `docs/default.yml`（首次构建时由 `build.ps1` 复制到 `release/config.yml`），两处内容必须一致 —— 配置项改了要同步改 `docs/default.yml`，否则发布出来的默认配置是旧的。

### 凭据处理

渠道凭据（Telegram Bot Token、QQ 机器人密钥、带 Cookie 的爬取会话）是本项目唯一真正的敏感数据。约定：

1. **默认只存环境变量名**。`credentials.store: env` 时，数据库里保存的是 `NEWSGLEAN_TG_TOKEN` 这个字符串，不是 token 本身。进程启动时解析。
2. **不写进配置文件**。渠道配置与凭据存放在数据库，不落 `config.yml`，降低误提交风险。
3. **`serve` 启动时打印一次凭据来源摘要**（变量名与是否解析成功），不打印内容，便于排障又不泄漏。
4. 需要加密落盘时（`store: file`）用 `passphrase_env` 指向的口令派生密钥，未提供口令则拒绝启动，**不做「静默降级为明文」**。
5. 日志与错误信息中，凭据一律替换为 `***`。

### 安全默认值

服务默认只绑定 `127.0.0.1:28080`（`server.http_addr`）。要暴露到局域网需显式改 `http_addr`，并**必须**同时配置认证，否则任何人都能读到你的订阅与正文。

入站端点（`/ingest/...`）额外注意：

- 每个 `webhook` 渠道生成独立随机密钥，路径即凭据，长度足够抵御枚举。
- 支持可选的 HMAC 签名校验（`X-NewsGlean-Signature`），供有能力的调用方使用。
- 收到校验失败或来源不明的入站请求时记日志但不入库，避免被当作免费存储。
- **若 `http_addr` 改为非本机地址且未配置认证，启动时直接拒绝**，而不是只打印警告 —— 这类误配置的后果（订阅列表与全文对外可读）太重。

---

## 阶段性开发目标

先以 M1 搭骨架，之后按 **21 个执行阶段**推进。每个阶段有**明确的完成定义（DoD）**，未达标不进入下一阶段；阶段按「产品可用 → 渠道扩展 → CLI/运维 → LLM」组织。

本节与下面的「路线图」互补：**路线图列「哪个版本交付哪些阶段」，本节列「每个阶段做到什么程度才算完成」。** 每个阶段开工时，把勾选项同步到会话的任务列表里逐项落地；达成 DoD 后，本节的勾选标记从 `[ ]` 改为 `[x]`。

状态：`[ ]` 计划中 · `[~]` 进行中 · `[x]` 已完成

---

### M1 — 搭骨架，跑通一条链路 `[x]`

**目标**：把参考 `bookcocoon-server` 的项目结构落成，证明「采集 → 去重 → 存储 → 通过 API 读出来」这条链路能跑通，且采集层可插拔。

**交付物**：

- [x] `internal/source`：`Connector` / `Item` / `Cursor` / `State` 契约定稿，导出标识符均有中文注释
- [x] Provider 注册：`Get(name)` / `Register(name, metadata, create, extraFlag)`
- [x] 第一个渠道 `feed`：RSS 2.0 / Atom / JSON Feed，配单元测试样本
- [x] `internal/database`：GORM 连接，`source.go` + `entry.go` 两表，`Install()` 首次建表
- [x] `internal/config`：YAML 结构 + 默认值 + `-c` 指定路径；`docs/default.yml` 落地
- [x] `internal/server`：`Server` 结构体、`Run`/`Stop`、`route.go` + `source_api.go` + `entry_api.go`
- [x] `internal/app` + `internal/scheduler`：手动触发一轮采集并写库，三层去重
- [x] `cmd/root.go` 启动服务、`version` 子命令、`internal/build` 版本注入
- [x] `build.ps1` / `build_run.ps1` / `run.ps1`

**验收标准（DoD）**：

1. `go build ./...`、`go vet ./...` 通过，`go test ./...` 全绿（含 feed 解析的畸形样本用例）。
2. `news-glean` 启动后：`source add feed` 成功入库 → 手动触发采集 → 数据库出现条目 → `GET /api/entry/list` 返回该条目。
3. 契约文件 `internal/source/source.go` 是接口唯一权威，其余模块只依赖它、不反向依赖任何渠道实现。

**退出条件**：新增第二个渠道时，`internal/app`、`internal/database`、`internal/filter` 零改动即可完成接入（此条在阶段 14 实测，但 M1 就要按这个方向设计）。

---

### 阶段 1–9 · 产品可用

把「能手动采集 + 能读」补成「挂上就能自动更新、能搜、能导出」的日常工具。

#### 阶段 1 — 定时后台调度 `[x]`
- ticker 调度 + 按渠道 `interval` + 全局并发上限
- 验收：挂上服务后自动按间隔刷新，无需手动触发
- 实现：`internal/scheduler` 后台循环（10s 扫描粒度）+ 每渠道 `nextRun` 到期判定 + 信号量限并发；`cmd/root.go` 启动即 `sched.Start(ctx)`、停机 `sched.Stop()` 等待在途采集收敛；启动时首轮立即调度，新加/改动的渠道随扫描自动纳管

#### 阶段 2 — 礼貌限速 + 健康度自动降频 `[x]`
- 请求间隔限速；连续失败自动降频、恢复后回升
- 验收：失败渠道降低频率，恢复后回升
- 实现：`internal/fetch` 新增 `Limiter`（相邻两次放行至少间隔 `fetch.rate_limit`，默认 1s，可配）；`internal/scheduler` 采集前先过限速，采集后按渠道 `FailCount` 做指数退避（`2^failCount` 封顶 `maxBackoff`=24h、不低于基础间隔），成功则 `FailCount` 归零即回升基准间隔；手动刷新不受后台限速影响

#### 阶段 3 — 采集日志可观测性 + SSE 实时刷新进度 `[x]`
- [x] 每渠道成功率 / 耗时 / 最后成功时间 + 界面展示
- [x] SSE 端点推送刷新进度，前端显示进度 / 新内容高亮
- 验收：界面可见每渠道健康状态；刷新时前端实时可见进度
- 实现：`internal/database` 新增 `fetch_log` 表（每次拉取记一条：耗时 / 新增 / 跳过 / 成功 / 错误）；`app.refreshOne` 结束时写日志；`FetchStatsBySource` 聚合成功率、最后成功时间、最近耗时；`SourceDTO` 扩展 `fetch_count`/`success_count`/`success_rate`/`last_success_at`/`last_fetch_at`/`last_elapsed_ms`；新增 `GET /api/source/{id}/logs` 排障端点；前端渠道列表展示「采集 N 次 · 成功 M 次（%）· 最近耗时 · 最后成功」；`internal/app` 新增 `progressHub` 发布-订阅进度中心，`refreshOne` 发布 `source_started`/`source_done`/`source_failed`，新增 `GET /api/refresh/stream`（SSE，含心跳），前端顶栏实时显示「正在刷新：渠道名…」，并联动阅读页高亮新条目（手动与后台刷新均可见）

#### 阶段 4 — 阅读状态：已读 / 收藏 / 归档 `[x]`
- [x] `entry_state` 三状态接线 + 前端交互 + 列表过滤
- [x] 验收：浏览器标记与过滤，重启后保留
- 实现：`internal/database` 新增 `SetRead` / `SetFavorite` / `SetArchive`（共用 `setStateFlag` 通用实现，更新 `updated_at`）；`ListEntries` 改为接收 `EntryFilter`，`read` / `favorite` / `archive` 通过 LEFT JOIN `entry_state` + `COALESCE` 过滤（无状态行按 false 处理）；`EntryDTO` 扩展 `read` / `favorite` / `archive` 字段；新增 `PUT /api/entry/{id}/read` / `favorite` / `archive` 三端点（共用 `handleEntrySetState`）；`GET /api/entry/list` 新增 `read` / `favorite` / `archive` 过滤参数。前端：`EntryRow` / `EntryDetail` 提供四状态切换按钮，详情页打开自动标记已读；新增收藏 / 归档页（复用 `StateList` 通用列表页）并接入路由与导航；阅读页（收件箱）默认排除已归档条目，支持「全部 / 未读 / 已读」过滤，归档或筛选失效的条目即时移出列表

#### 阶段 5 — FTS5 建表 + 触发器 + 写入同步 `[x]`
- [x] `Install()` 建虚拟表，条目写入/更新同步索引
- [x] 验收：索引随条目自动维护
- 实现：`internal/database/entry_fts.go` 建 `entries_fts` 外部内容表（`content='entries'`、`content_rowid='id'`，rowid 对应 `entries.id`，索引不存正文副本），索引列 `title`/`summary`/`content`/`author`；三个同步触发器 `AFTER INSERT`（写入）/`AFTER DELETE`（删除，回传旧值）/`AFTER UPDATE`（先删旧再插新）；`Install()` 末尾接 `installFTS()`——建表与存量回填仅在表不存在时执行、触发器每次启动重建，保证幂等。条目用 `deleted` 布尔软删，故 `AFTER DELETE` 实际不触发，软删由 `AFTER UPDATE` 的 `new.deleted=1` 分支只删不插、从索引移除；回填按 `deleted=0` 过滤。因 `mattn/go-sqlite3` 的 FTS5 需编译进构建，`build.ps1` 与测试均须加 `-tags sqlite_fts5`

#### 阶段 6 — 中文分词 + 搜索 API `[x]`
- [x] bigram 分词 + `/api/search`
- [x] 验收：中文关键词命中正文
- 实现：`internal/database/entry_fts.go` 注册自定义 sqlite 驱动 `sqlite3-bigram`（`ConnectHook` 在每条连接上经 `RegisterFunc` 注册确定性标量函数 `bigram`），`database.Open` 对 sqlite 改用该驱动；触发器/回填写入索引前对 `title`/`summary`/`content`/`author` 统一包 `bigram()`——连续中文串按 `unicode.Han` 切成相邻双字、其余原样保留，再交给默认 unicode61 分词器按空格切词，使每个双字成为独立 token。`installFTS()` 用 `PRAGMA user_version`（`ftsSchemaVersion=1`）标记 tokenization 版本，落后即 `DROP` 旧表重建并回填存量（幂等），触发器每次启动重建；`DB` 增 `driver` 字段，`installFTS()` 仅在 sqlite 下执行。新增 `SearchEntries(q, filter, page, pageSize)`（`internal/database/entry_search.go`）：`JOIN entries_fts` + `MATCH`，查询侧 `ftsQuery()` 用同一 `bigramTokenize` 切词后逐 token 加引号、空格连接（隐式 AND），复用从 `ListEntries` 抽出的 `applyEntryFilter` 叠加渠道/状态过滤，按发布时间倒序分页。接入层 `internal/server/search_api.go` 新增 `GET /api/search`（`q` 必填，`page`/`page_size`/`source_id`/`read`/`favorite`/`archive` 可选），复用 `EntryListResponse` 与 `entryDTOs`；mysql 待阶段 20 以 FULLTEXT + ngram 落地，届时走同一 `SearchEntries` 契约分流

#### 阶段 7 — 前端搜索界面 `[x]`
- [x] 搜索框 + 结果列表 + 命中高亮 + 详情跳转
- [x] 验收：搜索到详情可跳转
- 实现：纯前端，改动集中在 `NewsGlean-web`。`src/services/index.ts` 新增 `search(params)`（封装 `GET /api/search`：`q` 必填、trim 后拼接，`page`/`page_size`/`source_id`/`read`/`favorite`/`archive` 可选，复用现有 `request` 与统一错误处理），新增 `SearchParams` 类型（`q: string` + 复用 `EntryListParams` 其余字段），返回值复用 `EntryListResponse`。新建 `pages/Search/index.tsx`：顶部搜索框（受控输入，回车或「搜索」按钮触发，空关键词提示且不发请求），结果列表复用 `EntryRow`（同 `EntryList` 先 `listSources` 拉渠道表显示渠道名；点标题走 `EntryRow` 内置的 `Link to=/entries/{id}` 跳详情，即「搜索到详情可跳转」验收点），分页复用 `EntryList` 的「上一页 / 下一页 + 第 x/y 页」模式与 `PAGE_SIZE`，空态区分「未搜索 / 无结果 / 加载中 / 错误」四种。新增高亮工具（`src/utils/index.ts` 加 `highlight(text, keyword)`）：对 `stripHtml` 后的标题/摘要做不区分大小写的子串匹配，命中片段包 `<mark>`；摘要先 `stripHtml` 再高亮，避免 HTML 标签被切碎；关键词先转义正则特殊字符，空关键词直接返回原文。注意后端按 bigram 双字 token 做 AND 匹配、前端高亮按用户原始连续子串，两者命中范围不完全一致，属可接受差异（在代码注释标注）。可选：结果页叠加渠道/已读/收藏/归档筛选（复用 `EntryList` 的筛选组件与 `source_id`/`read`/`favorite`/`archive` 参数）。`src/router/index.tsx` 注册 `/search` 路由，`src/components/Layout.tsx` 导航栏加「搜索」链接

#### 阶段 8 — 导出 Markdown `[x]`
- YAML front matter + 按源/日期分目录
- 验收：导出结构正确
- 实现：新建 `internal/export` 包，入口 `ExportMarkdown(db, dir, opts)`（dir 取 `export.markdown_dir`，默认 `./export`）。目录结构 `markdown_dir/<源名>/<YYYY-MM>/<安全化标题>.md`：源名与标题经 `sanitizeName` 清洗 `/\:*?"<>|` 与控制字符、去首尾空格与点，空名兜底 `source-<id>` / `untitled-<id>`；月份取 `published_at` 前 7 位（RFC3339 的 `YYYY-MM` 前缀），空则退 `fetched_at`，再空用 `unknown`。front matter 用已引入的 `yaml.v3` 序列化，字段 `title`/`author`/`url`/`source`/`guid`/`published_at`/`fetched_at`/`content_type`/`tags`/`extra`，空字段 omitempty——无 URL 条目自然不含 `url`，呼应「无 URL 渠道导出 Markdown 不写 url 字段」；`tags` 为 YAML 列表、`extra` 为键值对 map。正文保留原始 HTML 原样写入，`Content` 为空时用 `Summary` 兜底。文件名同目录冲突追加 `-2`/`-3` 序号；条目遍历顺序确定（`source_id` 升序、`published_at` 降序、`id` 降序），故序号分配确定；重复导出覆盖同名文件（快照语义，幂等）。导出范围复用 `EntryFilter`（新增不分页查询 `ListAllEntries`），默认全量，可传 `source_id`/`read`/`favorite`/`archive` 过滤。接入层新增 `POST /api/export/markdown`（过滤参数走 query，复用 `parseEntryFilter`），返回 `{total, sources, dir}`；`route.go` 注册 + Swagger 注解同步。CLI `export` 子命令属阶段 19、JSON/EPUB 属阶段 9，均不在本阶段；本阶段不提前端导出按钮（留到 v0.2 联调）

#### 阶段 9 — 导出 JSON + EPUB（可选）`[x]`
- JSON 供脚本消费；EPUB 标准库 `archive/zip` 自写
- 验收：格式可读、无外部依赖
- 实现：在 `internal/export` 包扩展两种格式，复用阶段 8 的 `Options{Filter}`/`Result`/`ListAllEntries`/`sanitizeName`/`monthOf`/`parseTags`/`parseExtra`。**JSON**：新增 `ExportJSON(db, dir, opts)`（dir 取 `export.json_dir`，默认 `./export-json`），输出单文件数组 `entries.json`（`json.MarshalIndent` 两空格缩进），条目用导出专用 DTO `EntryJSON`（tags/extra 展开为原生结构、冗余 `source` 渠道显示名，空字段 omitempty——无 URL 条目自然不含 `url`），幂等覆盖。**EPUB**：新增 `ExportEPUB(db, dir, opts)`（dir 取 `export.epub_dir`，默认 `./export-epub`），每源一本 `<安全源名>.epub`（书名=源名，按 `monthOf` 月份分章、章内条目按时间排）。容器用标准库 `archive/zip` 自写、无外部依赖：`mimetype`（首项、`Method=Store` 不压缩）、`META-INF/container.xml`、`OEBPS/content.opf`（metadata/manifest/spine）、`OEBPS/toc.ncx`（EPUB2 兼容）、`OEBPS/nav.xhtml`（EPUB3 导航）、每月一个 `OEBPS/chapter-<YYYY-MM>.xhtml`。正文最小 XHTML 化：void 元素（br/img/hr 等）补自闭合、裸 `&` 转义、文本字段 `xml.EscapeText`，纯文本按 `<p>`/`<br/>` 包裹；属「尽力而为」不做完整良构校验，完整清洗留到阶段 12 正文提取后增强。幂等覆盖。配置 `ExportConfig` 新增 `json_dir`/`epub_dir`（`docs/default.yml` 同步）。接入层新增 `POST /api/export/json`、`POST /api/export/epub`（复用 `parseEntryFilter`），`route.go` 注册 + Swagger 注解同步。CLI `export` 子命令属阶段 19、前端导出按钮留到 v0.2，均不在本阶段

---

### 阶段 10–17 · 渠道扩展

落地三类新渠道，补齐正文提取与过滤规则。**M1 的「抽象判决」在阶段 14 实测**。

#### 阶段 10 — `webpage` 渠道 `[ ]`
- `goquery` 解析静态列表页 + CSS 选择器配置
- 验收：列表页可采

#### 阶段 11 — 内容完整度判定（`ContentStatus`）`[ ]`
- `source.Item` 增加 `ContentStatus` 五态枚举（`empty`/`summary`/`truncated`/`full`/`unknown`），`feed` 适配器按字段来源赋基准值
- 核心层纯函数 `ClassifyContentStatus`：对 `full`/`unknown` 扫截断特征（结尾标记 + 正文/摘要长度比）降级为 `truncated`，其余态透传
- `entries` 增加 `content_status` 列，`entryFromItem` 透传；`EntryDTO` 与前端列表/详情显示「全文 / 摘要 / 截断」徽标
- `fetch.full_text` 由 `bool` 改为三态 `auto|on|off`（默认 `auto`），供阶段 12 回源抓取使用
- 验收：RSS（仅 description）、RSS（content:encoded）、Atom、JSON Feed 四类样本判定正确；截断标记样本识别为 `truncated`；`build`/`vet`/`test` 全绿

#### 阶段 12 — 正文提取（`extract`）`[ ]`
- 剥离广告 / 导航 / 评论；仅在 `ContentStatus != full` 且 `fetch.full_text=auto` 时回源
- 验收：正文干净可读；已有全文的条目不回源

#### 阶段 13 — 编码识别（GBK/GB18030）`[ ]`
- `chardet` + `x/text`
- 验收：中文老站点不乱码

#### 阶段 14 — `webhook` 渠道 `[ ]`（抽象判决）
- 入站端点 + 独立密钥 + 可选 HMAC
- 验收：POST 入库；同一内容经 RSS 与 webhook 只留一条（M1 抽象在此实测）

#### 阶段 15 — `telegram` 渠道 `[ ]`
- `getUpdates` 拉取 + `update_id` 游标（token 走 `store=env`）
- 验收：频道/群消息可采

#### 阶段 16 — 过滤规则引擎 `[ ]`
- 关键词 / 正则 / 来源 / 作者
- 验收：规则命中后保留/丢弃/标记正确

#### 阶段 17 — OPML 导入/导出 + `import` + 图片本地缓存 + Webhook 通知 `[ ]`
- OPML 互迁、本地导入、图片落缓存、新条目推送到自建服务
- 验收：OPML 互迁正确，图片落缓存，通知可达

---

### 阶段 18–21 · CLI / 运维 / 增强

一次性子命令作为 API 轻客户端，加上部署/安全收尾与可选 LLM。

#### 阶段 18 — CLI：`source` 系列 `[ ]`
- `types` / `add` / `ls` / `rm` / `enable` / `disable` / `test`
- 验收：命令行管理渠道，状态与网页一致

#### 阶段 19 — CLI：其余子命令 + 环境变量 `[ ]`
- `refresh` / `list` / `read` / `search` / `ingest` / `opml` / `export`；`NEWSGLEAN_` 前缀覆盖
- 验收：单二进制 serve + 子命令同源

#### 阶段 20 — MySQL 驱动 + 凭据加密 `[ ]`
- `database.Open` 支持 mysql；`store=file` + passphrase 派生密钥
- 验收：MySQL 可跑、凭据不落明文

#### 阶段 21 — LLM 智能增强（可选）`[ ]`
- 摘要 + 「今日概览」+ 分类打标 + OpenAI 兼容端点
- 验收：不配置密钥时功能照常

---

### 执行约定

1. **一次只推进一个阶段**，阶段内按「契约 → 数据层 → 核心 → 接入层」的顺序落，避免依赖倒挂。
2. **阻塞即停**：遇到无法离线获取的依赖、或契约需要改设计这类阻塞，立即记录并停下等决策，不硬啃、不打补丁。
3. **验收先行**：每个阶段开工时，把 DoD 转成会话任务清单；DoD 未全绿，不声称阶段完成。
4. **文档同步**：契约或数据模型变了，`PLAN.md` 对应章节与本文状态标记同一步更新。

### 开发环境约束（已预检）

这些是本机当前环境的实测结果，直接影响「怎么做」，先记录以免开工时踩坑：

- **Go 构建默认失败**：沙箱下 `go build` 因 go-build 缓存在工作区外而报 `Access is denied`。必须把 `GOCACHE` 指到工作区内（如 `.gocache/`，已在 `.gitignore`），已验证 `build`/`vet`/`test` 全绿。
- **离线构建**：依赖经本地模块缓存 `.gomodcache/` 解析，已含 `gorm`、`sqlite`、`yaml.v3`、`swaggo`、`cobra`、`chromedp`、`coder/websocket`、`go-figure` 等；构建时把 `GOCACHE` 与 `GOMODCACHE` 都指到工作区内即可，无需联网。
- **已确定的选型**：SQLite 驱动用 CGO 版（`gorm.io/driver/sqlite`）；feed 解析用标准库自写（不引 `gofeed`）。

---

## 路线图

### v0.1 — 搭骨架，跑通一条链路（最小可用）
- [x] 项目结构初始化：`internal/` 各包、`main.go` Swagger 注解、`docs/default.yml`
- [x] `build.ps1` / `build_run.ps1` / `run.ps1` 与 `internal/build` 版本注入
- [x] `internal/config`：YAML 配置结构、默认值、`-c` 指定路径
- [x] `internal/database`：GORM 连接、`source.go` 与 `entry.go`、`Install()` 首次建表
- [x] **定稿 `Connector` / `Item` / `Cursor` / `State` 接口**（最先做，这是地基）
- [x] 轻量 Provider 注册：`Get(name)` / `Register(name, metadata, create, extraFlag)`（照参考项目 `web_novel_book` 的扩展写法）
- [x] 第一个渠道 `feed`（RSS 2.0 / Atom / JSON Feed）
- [x] `internal/server`：`Server` 结构体、`Run`/`Stop`、`route.go`、`source_api.go`、`entry_api.go`
- [x] `internal/app` + `internal/scheduler`：手动触发一轮采集并写库
- [x] 三层去重 + `cmd/root.go` 启动服务 + `version` 子命令

> v0.1 故意只做一个渠道，但**接口按多渠道设计**。验收标准：写第二个渠道时不改 `internal/app`、`internal/database`、`internal/filter` 的任何一行。

### v0.2 — 产品可用（阶段 1–9）
自动刷新、阅读状态、全文检索、导出、可观测性/SSE。发布标准：纯浏览器完成「加源 → 自动刷新 → 读 → 标记已读 → 搜索 → 导出」。

### v0.3 — 渠道扩展与抽象判决（阶段 10–17）
`webpage` / `webhook` / `telegram` + 正文提取 + 过滤规则 + OPML/import/图片缓存/通知。发布标准：`webpage` 与 `webhook` 不改核心层接入，跨渠道去重生效。

### v0.4 — CLI 与运维化（阶段 18–20）
CLI 子命令 + 环境变量 + MySQL + 凭据加密。发布标准：单二进制既能 serve 也能用子命令操作同一份状态。

### v0.5 — LLM 智能增强（阶段 21，可选）
摘要 / 分类 / 「今日概览」。发布标准：不配置密钥时功能照常。

### 待评估（需要先决策，不排期）
- [ ] `qq_bot`：QQ 官方机器人采集。**取决于你的具体需求**：官方 Bot 能力受限，能不能拿到群消息要看平台策略与审核结果，需要先调研确认可行性。
- [ ] 出站/动作接口（通过机器人搜索、推送、标记已读）。这是「双向交互」场景，见「关于 QQ 与微信机器人」一节的说明。
- [ ] `wechat_mp`：公众号采集。默认方案是走 RSSHub 等桥接，不自行实现高风险路径。
- [ ] 多用户与 API Token 鉴权（仅在打算做多设备/团队共享时才需要）
- [ ] 国际化

---

## 技术选型（草案）

选型原则：与 `bookcocoon-server` 保持一致，优先纯 Go、免 CGO、可复现构建。

| 领域 | 选型 | 说明 |
| --- | --- | --- |
| CLI | [spf13/cobra](https://github.com/spf13/cobra) | 已引入，与参考项目同版本 v1.10.2 |
| 配置 | `gopkg.in/yaml.v3` | 照参考项目直接 struct tag 映射，不引入 viper |
| ORM | [gorm.io/gorm](https://gorm.io) + `driver/sqlite` + `driver/mysql` | 与参考项目一致，sqlite 默认、mysql 可选；建表走 `AutoMigrate` |
| SQLite 驱动 | `gorm.io/driver/sqlite`（CGO，`mattn/go-sqlite3`） | **已确定**：离线缓存内仅有 CGO 版，纯 Go 的 `glebarez/sqlite` 缺包且无法联网获取。本机 gcc 可用，构建已验证通过 |
| HTTP 层 | 标准库 `net/http`（ServeMux 方法+路径模式）+ [swaggo](https://github.com/swaggo/swag) | 照参考项目：`Mux.HandleFunc("POST /api/...")`，Swagger 注解写在 handler 上 |
| 前端 | React 19 + Vite + Tailwind CSS（独立仓库 NewsGlean-web） | 构建产物静态托管，经 `internal/webui` 用 `go:embed` 打进二进制；开发时 Vite 代理 `/api` 到后端 |
| ID 生成 | `github.com/bwmarrin/snowflake` / `github.com/google/uuid` | 与参考项目一致，按需引用 |
| 中文编码识别 | `github.com/saintfish/chardet` + `golang.org/x/text` | 与参考项目一致，正好解决 GBK/GB18030 站点问题 |
| Feed 解析 | 自写解析层（`encoding/xml` + `encoding/json`） | **已确定**：`gofeed` 离线缺包，故自写，覆盖 RSS 2.0 / Atom / JSON Feed 与畸形样本 |
| 网页爬取 | `net/http` + `goquery`（CSS 选择器） | 只处理静态 HTML（`webpage` 渠道，未实现）；JS 渲染的 feed 页走 chromedp，见下行 |
| 无头浏览器 | [chromedp](https://github.com/chromedp/chromedp) v0.9.5 | feed 渠道的 `fetch_mode: chromedp` / `chromedp_headed` 抓取用（JS 渲染页）；通用网页列表页爬取尚未启用 |
| Telegram | 自写 HTTP 客户端（Bot API） | 只用 `getUpdates` / `setWebhook` / `getChat` 几个接口，引入 SDK 不划算 |
| QQ 官方机器人 | 待调研（官方 SDK 或自写签名校验） | 需要 WebSocket/回调、签名校验与平台审核，可行性确认后再定 |
| 正文提取 | [go-trafilatura](https://pkg.go.dev/github.com/savetoink/go-trafilatura/v2) 或 go-readability | 中文站点表现需实测对比后再定，样本放测试数据目录 |
| 调度 | robfig/cron 或简单 ticker 调度器 | 一期用 ticker 即可，避免过度设计 |
| LLM | OpenAI 兼容 HTTP 客户端（自写） | 只要 `chat/completions` 一个接口，无需 SDK |

**已移除的选型**：bubbletea（TUI），见「项目结构」一节说明。

> 选型在对应模块开工前定稿，届时本节改为「已确定」并写明版本号。

---

## 开发

### 构建与运行

照参考项目，提供三个 PowerShell 脚本：

| 脚本 | 作用 |
| --- | --- |
| `build.ps1` | `swag init --parseDependency` → `go build` 注入版本与时间 → 首次构建时从 `docs/default.yml` 复制默认配置到 `release/` |
| `build_run.ps1` | `go mod tidy` → `go mod vendor` → `build.ps1` → 清理旧数据 → `run.ps1` |
| `run.ps1` | `./release/app.exe -d release` |

版本号注入走 `internal/build` 包，与参考项目同款：

```go
package build

var BuildTime string
var BuildVersion string
```

```powershell
$LdFlags = "-s -w " +
    "-X ${Package}/internal/build.BuildTime=$BuildTime " +
    "-X ${Package}/internal/build.BuildVersion=$BuildVersion"
```

开发时的常用命令：

```bash
go build ./...          # 构建
go run . --help         # 本地运行
go test ./...           # 测试
go vet ./...            # 静态检查
gofmt -l .              # 格式检查（应无输出）
go mod tidy && go mod vendor
```

### 约定

- 提交信息用中文或英文均可，但需说明「为什么」而非仅「改了什么」。
- 新增外部依赖前先说明能否用标准库替代；依赖数量本身是维护成本。
- 代码注释用中文，导出标识符需有注释；错误信息用英文小写开头（照参考项目风格）。
- **业务逻辑不写在 `cmd/` 里**：`cmd` 只解析参数与装配依赖，逻辑下沉到 `internal/`。
- **一次性子命令不直接开数据库**，走 `server.http_addr` 调 API，保证状态只有一处权威。
- 新增渠道必须遵守抽象边界：适配器只产出 `Item`，不得直接写数据库、不得自己实现去重或过滤。若发现做不到，先改抽象再实现。
- **`internal/source/source.go` 属于契约**：修改接口需要同步更新本文档并说明迁移方式，不允许悄悄加参数。
- 渠道路由与 Swagger 注解同步更新；接口改了文档没改视为未完成。
- 抓取默认遵守 robots 与限速，不引入并发爆破。
- 涉及账号风险的渠道（非官方协议、客户端 Hook）默认不合并进主线；确有需要时以独立可选模块提供，并在文档中显著标注风险。

### CI / 发布（规划）

- `.github/workflows/ci.yml`：先跑 `build.ps1 -SkipSwag` 确保构建脚本可用，再 `go vet` + `go test` + 构建矩阵（linux/amd64、linux/arm64、darwin/arm64、windows/amd64）。
- 发布走 tag 触发 goreleaser，产物附 `checksums.txt`。
- 版本号经上面那条 `-ldflags` 注入，`news-glean version` 读取 `internal/build` 输出。

---

## 贡献

1. 开工前先在 Issue 里对齐范围，避免大改动返工。
2. Fork → 分支 → 提交 → PR，PR 描述写清动机与验证方式。
3. 涉及采集/提取行为的改动，请附上能复现的真实 URL 与前后对比。

---

## License

[MIT](./LICENSE) © 2026 ma6254

选择依据：目标是让工具被尽可能多的人使用与改造，不介意他人闭源使用，也不需要专利授权条款，因此选最短、限制最少的 MIT。

需要留意的后果：**他人可以闭源改造并商用而不回报代码**。若将来要开托管服务、或要阻止竞品直接套壳，需要重新考虑许可证 —— 可选方向是 AGPL-3.0（网络服务也触发开源义务）或 open-core 边界（核心开源、托管与团队功能闭源）。改证时注意：**已按 MIT 发布的版本无法收回**，换证只对后续版本生效。

另：同组织的 `bookcocoon-server` 的 Swagger 注解声明的是 Apache 2.0，但它的 `LICENSE` 文件同样是 0 字节。两个项目许可证不一致不会互相影响（各自独立），但如果希望统一，改这个项目或那个项目都可以 —— 需要在代码里同步声明的地方见 main.go 的 Swagger 注解。
