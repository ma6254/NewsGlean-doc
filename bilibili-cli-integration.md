# bilibili-cli 渠道集成（规划 + B1 字段实测）

> 状态：**规划阶段（B1 字段实测已完成，待决策）**，尚未实现代码。
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

## 6. 待拍板决策

1. **拓扑**：接受方案 A（Python 作为可选运行时）？还是坚持单二进制转方案 B/C？（前置决策）
2. **首版 mode 范围**：推荐先做 `history` + `favorites`（字段干净）；`feed`/`user-videos` 需等上游补丁。
3. **上游缺口处理**：fork bilibili-cli 提 PR 修复 §5 的 1–4？还是 Connector 读原始字段兜底？
4. **公开 B 站内容**：是否确认「只走 RSSHub，不重复造」（即 `bilibili` 渠道不做 hot/rank/search）？
5. **字幕回填策略**：按需拉字幕（打开条目时回填）而非定时全量，是否接受？

## 7. 分阶段实施（对齐 PLAN.md v0.3 渠道扩展）

| 阶段 | 内容 | DoD |
|---|---|---|
| **B0 决策** | 拍板 §6 五个问题，更新 `PLAN.md` 渠道清单表 | 决策落文档 |
| **B1 实测** | ✅ 已完成（本文档） | 字段映射 + 补丁清单定稿 |
| **B2 Connector** | `internal/source/bilibili/`：subprocess 调 `bili` + UTF-8 读 stdout + JSON 解析 + `Item` 映射 + mock 样本单测（含 `not_authenticated`/412/乱码样本） | `build`/`vet`/`test` 全绿，`internal/app`/`database`/`filter` 零改动 |
| **B3 降级与探活** | 未装 bili / 未登录 / 版本过低 → 界面「未就绪」+ 健康度计数 | 断网/未登录不崩 |
| **B4 字幕回填** | 打通 `extract` 的 bilibili 分支（按需拉 `subtitle.text` 写 `Content`） | 打开条目见字幕全文，FTS5 可搜 |
| **B5 前端** | 渠道类型下拉加 `bilibili` + 配置表单（mode/uid/fav_id） | 纯浏览器完成「加源 → 采集 → 读 → 搜字幕」 |

## 8. 风险与合规

- **ToS 灰区**：用自己账号 cookie 轮询关注动态/收藏/历史本质是网页客户端同款行为，风险较低；但高频自动化可能触发 412。复用 NewsGlean 既有「礼貌限速 + 健康度降频」（阶段 2）即可。
- **412/反爬**：bilibili-cli 的浏览器 cookie + UA 是主要缓解；Connector 把失败映射到 `RateLimitError` 触发降频。
- **凭据安全**：全委托 bilibili-cli（`~/.bilibili-cli/credential.json`），NewsGlean 不落 B 站 cookie 到数据库，符合 `PLAN.md` 安全默认值。
- **API 漂移**：bilibili-cli 更新频繁，Connector `Validate` 做版本门槛 + 升级提示。
