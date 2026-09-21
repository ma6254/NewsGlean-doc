# bilibili-cli 渠道集成（规划 + B1 字段实测）

> 状态：**实现中 —— B2 Connector / B3 环境检测+登录+用户信息 / B5 前端 已落地；B4 字幕回填、B6 上游补丁待做。**
> 关联文档：`PLAN.md`（主设计规格）、`rsshub-quick-add.md`（RSSHub 向导）。
> 本文不改动采集抽象层，见「定位与核心结论」。B1 实测结论均来自本机真实账号环境，非源码推断。

## 1. 定位与核心结论

**一句话**：新增一个 `bilibili` 渠道类型（`SourceType`），把 [bilibili-cli](https://github.com/public-clis/bilibili-cli)（Python CLI）作为**拉取型 Connector 的取数后端**，把 B 站内容规范化成 `Item` 进入统一阅读流。

**关键结论：**

- bilibili-cli 与已规划的 RSSHub 是**互补**关系，不是重复造轮子。RSSHub 覆盖「公开路由」（UP 主投稿、热门、排行、搜索），bilibili-cli 覆盖 RSSHub 给不了的「**个人数据 + 字幕全文**」。
- bilibili-cli 的独特价值恰好落在 NewsGlean 最在意的地方：把正文抓下来、可检索、在应用里读。对视频而言**字幕就是正文**。
- 核心冲突：NewsGlean 是「单二进制、本地优先」，bilibili-cli 需要 Python 3.10+。推荐**方案 A（子进程封装）**，把 Python 定位成「可选运行时」（对标 LLM 可选增强），`bili` 未装/未登录时渠道优雅降级。

### 三种拓扑（决策点 ①）

| 方案 | 做法 | 优点 | 代价 |
|---|---|---|---|
| **A. 子进程封装（推荐）** | `internal/source/bilibili/` 用 `os/exec` 调 `bili <cmd> --json`，解析 envelope 映射 `Item` | 完整复用智能认证、412 规避、字幕/音频；升级即修 API 漂移 | Python 成为可选运行时，破坏纯单二进制 |
| B. Go 重写端点 | Go 里实现登录/cookie/接口 | 保持单二进制 | 重造 bilibili-cli 最值钱的部分，维护翻倍 |
| C. 旁路服务 | bilibili-cli 包 HTTP 吐 JSON Feed，用 `feed` 渠道订阅 | NewsGlean 零改动 | 多常驻进程；`feed` 游标（ETag）对不上 B 站 offset/page |

## 2. 实测环境（B1）

| 项 | 值 |
|---|---|
| bilibili-cli 版本 | 0.6.2（`uv tool install`） |
| 可执行文件 | `C:\Users\mjc\.local\bin\bili.exe` |
| bilibili-api-python | 17.4.2 |
| 登录账号 | UID 11824232（LV6，年度大会员） |
| 编码约束 | **必须 `PYTHONIOENCODING=utf-8` 且按 UTF-8 读 stdout**，否则中文变 GBK 乱码（已实测复现） |
| 认证注意 | 二维码登录存在「假成功」缺陷，见 §5 补丁 5 |

> 已实测复现：`bili status --yaml` 不加 `PYTHONIOENCODING=utf-8` 时昵称显示为 `����ħ��ʦ`，加后正常显示「吔人魔法师」。这是 Connector 层 `exec.Cmd` 必须显式按 UTF-8 处理 stdout 的实锤依据。

## 3. 命令 → 字段实测结果

图例：✅ 可用 · ⚠️ 有缺口 · ❌ 有 bug

### 3.1 `bili history`（观看历史）✅ 最干净

```yaml
data:
  page: 1 / count: 3
  items:
    - id: BV1AbtJ6AErE / bvid: BV1AbtJ6AErE
      title: 点进一个厨师招聘网，他们的照片看起来都好诡异...【厨得快】
      author: 睡意躁動
      viewed_at: '2026-09-19T22:37:01'   # 标准 ISO 时间戳
```

- 分页：`--page`；时间：`viewed_at` 为标准 ISO，可直接映射 `Item.PublishedAt`。
- **首版优先候选 mode**。

### 3.2 `bili favorites`（收藏夹列表）✅

```yaml
- id: 72800232 / title: 默认收藏夹 / media_count: 2760
```

### 3.3 `bili favorites <id>`（收藏夹内容）✅ 基本干净

```yaml
data:
  folder_id: 72800232 / page: 1 / has_more: true
  items:
    - id: BV12AeU6NEej / bvid / title
      duration_seconds: 11 / duration: 00:11   # 正确
      upper: { name: 六牙_yaya }
```

- ⚠️ 唯一缺口：**无 `fav_time`（收藏时间）**，映射 `PublishedAt` 会缺失（需 InferredTime）。

### 3.4 `bili video <bvid>`（单视频详情）✅ 字段全对

```yaml
video:
  id / bvid / aid / title / description
  duration_seconds: 1632 / duration: '27:12'          # 正确
  owner: { id: '946974', name: 影视飓风 }              # 正确
  stats: { view / danmaku / like / coin / favorite / share }  # 全部有值
```

- ⚠️ 唯一缺口：**无 `pubdate`（发布时间）**（`normalize_video_summary` 未取该字段）。

### 3.5 `bili video <bvid> --subtitle`（字幕全文）✅ 杀手锏

- 实测 27 分钟视频返回完整字幕：`subtitle.text`（纯文本）+ `subtitle.items`（时间轴，1600+ 条 `from/to/content`）。
- ⚠️ 输出体量大（几十 KB），且 `--subtitle` 与 `--subtitle-timeline` 都会带上完整 `items`。Connector 做「按需回填」时应**只取 `subtitle.text` 丢弃 `items`**，不适合定时全量。

### 3.6 `bili feed`（关注动态）❌ 三个问题

```yaml
data:
  items:
    - id: '1249793156289921089'      # 动态 ID，不是 bvid
      author: { name: 黑纹白斑马 }
      published_at: ''                # ❌ 全空（相对时间 published_label: '1分钟前' 有值）
      title: 环世界器官农场 第一集...   # 视频动态只有 title，无 bvid/url
      text: ''
  next_offset: '1249783952150888457'  # ✅ 游标正常
```

| 现象 | 根因 |
|---|---|
| `published_at` 全空 | `normalize_dynamic_item` 读错字段路径，绝对时间戳未取到 |
| 视频动态丢 bvid/URL | 未取 `major.archive.bvid` |
| 部分条目 title/text 双空 | 转发/OPUS/纯图等动态类型未处理 |

- 唯一可用：`next_offset` 游标 + `published_label`（相对时间，不能用于去重/排序）。

### 3.7 `bili user-videos <uid>`（UP 主投稿）❌ 严重

```yaml
- id: BV1cSec6tEux / bvid / aid / title / description / url
  duration_seconds: 0 / duration: 00:00          # ❌ 错（实为 27 分钟）
  owner: { id: '', name: '' }                     # ❌ 空
  stats: { view: 8715953, danmaku: 0, like: 0, coin: 0, favorite: 0, share: 0 }  # ❌ 仅播放量对
```

- 根因：`user-videos` 把按「单视频 `get_info`」结构写的 `normalize_video_summary` 套到了「视频列表 `vlist`」结构上。vlist 里 UP 主字段是 `mid`/`author`（非 `owner` dict）、时长是 `"27:12"` 字符串（非秒 int）、统计是 `play`（无 `stat` dict）。
- **该命令输出基本不可用，需上游修复。**

### 3.8 `bili watch-later`（稍后再看）❌

```yaml
data:
  count: 46 / items: []     # ❌ 有数量但列表为空
```

- 根因：`get_toview` 从 `homepage.get_favorite_list_and_toview` 拿到了 `count`，但该响应的 `mediaListResponse.list` 为空，真正列表需另一次 API 调用。

## 4. 契约映射（bili 输出 → NewsGlean `Item`）

| `Item` 字段 | 来源（bilibili-cli 规范化字段） | 备注 |
|---|---|---|
| `GUID` | `bvid` | 稳定、全局唯一 ✅ |
| `URL` | `https://www.bilibili.com/video/{bvid}` | ✅ |
| `Title` | `video.title` / `item.title` | ✅ |
| `Author` | `video.owner.name` / `item.author` / `upper.name` | 视 mode 而定 |
| `PublishedAt` | ⚠️ 见 §5 补丁 1 | 目前多数命令缺失 |
| `Summary` | `video.description` / `item.text` | ✅ |
| `Content` | `subtitle.text`（按需回填） | 杀手锏：视频「全文」 |
| `ContentStatus` | 有字幕→`full`；仅简介→`summary` | 接阶段 11 `ContentStatus` |
| `ContentType` | `text/plain` | ✅ |
| `Tags` | 空（可后续补分区） | |
| `Extra` | `stats.*`、`duration`、`aid`、封面、`viewed_at`、`next_offset` | 逃生口 |

## 5. 上游补丁清单（需对 bilibili-cli 提 PR / fork 修复）

> 关键约束：`--yaml`/`--json` 输出的就是命令层规范化结果，**没有「原始模式」开关**。因此下列缺口 NewsGlean 侧无法优雅绕过，要么上游修，要么 Connector 放弃规范化层、直接调 bilibili-api-python 原始接口（牺牲解耦，不推荐）。

| # | 缺口 | 位置 | 修复建议 |
|---|---|---|---|
| 1 | 规范化视频无 `published_at` | `payloads.normalize_video_summary` | 单视频取 `pubdate`；`user-videos` 的 vlist 取 `created`；favorite media 取 `fav_time` |
| 2 | `user-videos` owner/duration/stats 全错 | `payloads` + `commands/user_search.py` | 为 vlist 单独写 `normalize_video_list_item`（读 `mid`/`author`/`length`/`play`） |
| 3 | `feed` 的 `published_at` 空 + 视频动态丢 bvid + 部分条目空标题 | `payloads.normalize_dynamic_item` | 修绝对时间戳路径；补 `major.archive.bvid`；处理转发/OPUS/纯图类型 |
| 4 | `watch-later` 列表为空 | `client.get_toview` | 用 `video.get_toview`（或对应 SDK 方法）单独拉列表，而非从 `get_favorite_list_and_toview` 取 |
| 5 | 二维码登录「假成功」 | 上游 `bilibili-api-python 17.4.2` `login_v2.check_state` | 从回调 `url` 拆 `SESSDATA` 失败仍返回 `DONE`；已用「手动注入 SESSDATA」绕过。非 Connector 必需，但影响开箱体验 |
| 6 | `normalize_user` 无头像 `face` | `payloads.normalize_user` | 加 `avatar: info.get("face","")`。**非阻塞（可选）**：首版头像用前端占位（见 §7.4），后续要真头像再提 |

## 6. 已拍板决策（本轮定稿）

| # | 决策项 | 结论 |
|---|---|---|
| 1 | 拓扑 | **方案 A（子进程封装）**，Python 作为可选运行时 |
| 2 | 首版 mode 范围 | **`history` + `favorites` 起步**（B1 字段干净）；`video` 详情 + 字幕按需回填随后；`user-videos`/`feed`/`watch-later` 挂到 §5 上游补丁后 |
| 3 | 上游缺口处理 | 首版**不 fork 不 PR**，只做 B1 干净能力；坏的三类（§5 补丁 2/3/4）记 backlog，待上游修复再启用 |
| 4 | 公开 B 站内容 | **只走 RSSHub**，`bilibili` 渠道不做 hot/rank/search/番剧 |
| 5 | 字幕回填策略 | **按需回填**（打开条目时拉 `subtitle.text` 写 `Content`），不做定时全量 |
| 6 | 环境检测 | **通用 `EnvChecker` 可选接口** + `GET /api/source/env-check`（详见 §7） |
| 7 | 登录方式 | 扫码走**终端引导**；手动输入 SESSDATA/bili_jct 由后端 `POST /api/bilibili/login` 写 `~/.bilibili-cli/credential.json`（详见 §7） |
| 8 | 用户信息展示 | 已登录经 `whoami --json` 展示用户名/等级/签名/粉丝/关注；头像用前端占位（`face` 缺失，见 §5 补丁 6） |

## 7. 环境检测、登录与用户信息（本轮新增，设计定稿）

新增「运行环境可发现性」：检测 `bili` 是否安装、是否登录，未装给安装教程、未登录给登录引导，已登录展示用户信息。

### 7.1 通用契约 `EnvChecker`（可选接口，向后兼容）

在 `internal/source` 增可选接口（与 `Prober` 同款、纯增量）：

```go
// EnvChecker 是 Connector 的可选能力：检测运行环境（如外部 CLI 是否安装/登录）。
type EnvChecker interface {
    CheckEnv(ctx context.Context) (EnvCheck, error)
}
type EnvCheck struct {
    Ready   bool      `json:"ready"`    // 是否可正常采集
    Version string    `json:"version"`  // 外部依赖版本（空=未装）
    Path    string    `json:"path"`     // 命中的可执行文件路径
    Authed  bool      `json:"authed"`   // 登录态（仅登录类 mode 有意义）
    User    *EnvUser  `json:"user"`     // 登录用户信息（未登录为 nil）
    Missing []string  `json:"missing"`  // 缺失项的人话描述
    Hints   []string  `json:"hints"`    // 安装/修复提示（供前端渲染教程）
}
type EnvUser struct {
    ID        string `json:"id"`
    Name      string `json:"name"`
    Avatar    string `json:"avatar"`    // ⚠️ 首版为空（占位），见 §5 补丁 6
    Level     int    `json:"level"`
    Sign      string `json:"sign"`
    Coins     int    `json:"coins"`
    Following int    `json:"following"`
    Follower  int    `json:"follower"`
}
```

- 后端：`GET /api/source/env-check?type=bilibili`（照 `/api/source/probe` 套路）；`internal/app` 加 `CheckEnv` 编排；`feed`/`webpage` 默认「就绪」或由核心层兜底。
- 检测序列：四级二进制探测 + `bili --version` → 已安装；再 `bili whoami --json` → 登录态 + 用户信息（`whoami` 返回 `user{id,name,level,coins,sign,vip}` + `relation{following,follower}`，`face` 被 `normalize_user` 丢弃 → 头像占位）。

### 7.2 四级二进制探测

1. 源配置 `bili_path` 覆盖（或全局 `fetch.bili_path`，镜像 `ChromePath` 模式）；
2. `exec.LookPath("bili")`；
3. `uv tool run bilibili-cli`（uv 用户）；
4. 已知路径：`~/.local/bin/bili.exe`、`%APPDATA%\uv\tools\bilibili-cli\Scripts\bili.exe`。

> 本机实测：entry point 装到 `C:\Users\mjc\.local\bin\bili.exe`，但该目录不在 PATH（`Get-Command bili` 为空），故第 2 级会落空、靠第 1/4 级兜底。

### 7.3 登录（两种方式）

| 方式 | 实现 | 说明 |
|---|---|---|
| 扫码 | Web 引导：提示到终端跑一次 `bili login`，扫完回 Web「重新检测」 | 零代码；QR 存在「假成功」缺陷（§5 补丁 5），引导文案注明 |
| 手动 | `POST /api/bilibili/login` 收 `{sessdata, bili_jct, buvid3?}` → 后端**合并**写入 `~/.bilibili-cli/credential.json`（CLI 同款格式、0600、不落日志；已存在的 buvid3/4/dedeuserid 保留）→ 回读 `whoami --json` 校验并返回最新用户信息 | 复用 CLI 私有凭证文件；仅 SESSDATA+bili_jct（缺 buvid3/4）可能触发 412，表单建议附可选 buvid3 |

安全：SESSDATA/bili_jct 只在请求体、绝不进日志/接口返回；凭证文件沿用 CLI 的 0600；仍「不落 B 站 cookie 到数据库」（写入的是 CLI 的 credential.json，非 NewsGlean 库）。

### 7.4 用户信息展示

`CheckEnv.User` 由前端渲染三态：
- **已登录** → 首字母占位头像 + 用户名 + Level + 签名 + 粉丝/关注；
- **已装未登录** → `bili login` 终端引导 + 手动输入表单；
- **未装** → 安装教程面板（`uv tool install bilibili-cli` 优先，附 pipx/pip 备选 + PATH 提示 + 「重新检测」按钮）。

## 8. 分阶段实施（对齐 PLAN.md v0.3 渠道扩展）

| 阶段 | 内容 | 状态 |
|---|---|---|
| **B0 决策** | §6 八项定稿 | ✅ 完成 |
| **B1 实测** | 本文档 §2–§5 字段映射 + 补丁清单 | ✅ 完成 |
| **B2 Connector** | `internal/source/bilibili/`：subprocess 调 `bili` + UTF-8 + JSON 解析 + `Item` 映射 + 四级探测 + **限速/重试** + hermetic 单测；mode：`history`+`favorites` | ✅ 完成（build/vet/test 全绿） |
| **B3 环境检测 + 登录 + 用户信息** | `EnvChecker` + `env-check` + `bilibili/login` + 前端三态面板 | ✅ 完成 |
| **B4 字幕回填** | 按需拉 `subtitle.text` 写 `Content`（丢弃 `items`） | ⏳ 待做 |
| **B5 前端** | 类型下拉 + 表单（mode/fav_id/bili_path/fetch_detail）+ 环境/登录面板 + 收藏夹下拉 + 回填按钮 + 封面缩略图 | ✅ 完成 |
| **B6 上游补丁（可选后续）** | fork bilibili-cli 提 PR 修 §5 补丁 1–4、6 | ⏳ 待做 |

## 9. 实现现状（已落地）

### 9.1 后端包 `internal/source/bilibili/`

| 文件 | 职责 |
|---|---|
| `config.go` | 配置：`mode`(history\|favorites) / `fav_id` / `bili_path` / `fetch_detail` |
| `runner.go` | 子进程执行：UTF-8 环境、JSON envelope 解析、错误码映射、四级探测 |
| `mapping.go` | history / favorites 条目 → `Item` 映射 |
| `connector.go` | `Connector` 实现 + `Enrich`（单条回填简介/封面） |
| `register.go` | 注册 `bilibili` 渠道类型 |
| `env.go` | `CheckEnv`（版本 + 登录态 + 用户信息） |
| `credential.go` | 写 `~/.bilibili-cli/credential.json`（手动登录） |
| `favorites.go` | `ListFavorites`（收藏夹列表） |
| `detail.go` | `GetVideoDetail`（简介 + 统计 + 可选字幕，回填原语） |
| `cover.go` | `GetVideoCover`（直连公开 view 接口拿封面） |
| `rate.go` | 限速（500ms 最小间隔）+ 重试（指数退避） |

### 9.2 契约与核心层增量

- `internal/source/enrich.go`：可选 `Enricher` 接口——核心层判定条目「新」后、写库前调用一次，实现「首次入库才回填」，避免 N+1 重复。
- `internal/source/env.go`：可选 `EnvChecker` 接口 + `EnvCheck`/`EnvUser`（与 `Prober` 同款、纯增量）。
- `internal/app/app.go`：`CheckEnv` 编排；`refreshOne` 里对新条目调用 `Enricher.Enrich`。
- `internal/database/entry.go`：`ListEntriesForBackfill` / `UpdateEntryEnrich`。

### 9.3 HTTP 端点

| 端点 | 说明 |
|---|---|
| `GET /api/source/env-check?type=bilibili[&bili_path=]` | 环境检测（安装/登录/用户信息） |
| `POST /api/bilibili/login` | 手动登录（写 credential.json，回读校验） |
| `GET /api/bilibili/favorites?bili_path=` | 收藏夹列表 |
| `POST /api/bilibili/backfill` | 回填已有空简介条目的简介+封面 |

### 9.4 前端

- `SourceForm.tsx`：`bilibili` 类型 + mode/fav_id 表单 + `bili_path` 覆盖 + `fetch_detail` 开关（**默认开**）+ 环境检测面板 + 收藏夹下拉 + 「自动获取」显示名。
- `BiliEnvPanel.tsx`：三态面板（未装教程 / 登录引导 / 已登录用户信息）。
- `EntryRow.tsx`：列表显示封面缩略图（`extra.cover`）。
- `SourceList/index.tsx`：bilibili 源卡片「回填简介」按钮。

### 9.5 行为要点

- 简介/封面回填仅对**新条目**（经 `Enricher`），`fetch_detail` 默认开；已有条目用「回填简介」按钮补。
- 封面走直连公开 view 接口（仅展示、无鉴权、失败静默），统一 https。
- 限速 500ms 最小间隔 + 可重试错误（`rate_limited`/`network_error`/412/5xx）指数退避重试 3 次。
- 发布时间仍为 `InferredTime`（视频/收藏夹无 pubdate，待 §5 补丁 1）。

## 10. 风险与合规

- **ToS 灰区**：用自己账号 cookie 轮询关注动态/收藏/历史本质是网页客户端同款行为，风险较低；但高频自动化可能触发 412。复用 NewsGlean 既有「礼貌限速 + 健康度降频」（阶段 2）即可。
- **412/反爬**：bilibili-cli 的浏览器 cookie + UA 是主要缓解；Connector 侧已加 500ms 限速 + 对 `rate_limited`/412/5xx 的指数退避重试（`rate.go`）。
- **凭据安全**：凭据仍存于 bilibili-cli 的 `~/.bilibili-cli/credential.json`（0600）。手动登录由后端写该文件、不落 B 站 cookie 到 NewsGlean 数据库、SESSDATA/bili_jct 不进日志；代价是与 CLI 私有文件格式耦合，CLI 改格式需跟随。
- **API 漂移**：bilibili-cli 更新频繁，Connector `Validate` 做版本门槛 + 升级提示。
