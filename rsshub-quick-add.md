# 快速添加 RSSHub 渠道源（规划）

> 状态：**规划阶段（待决策）**，尚未实现。本文记录功能规划、调研结论与待拍板的开放问题。
> 关联文档：`PLAN.md`（主设计规格）。本功能不改动采集抽象层，见「定位与核心结论」。

## 1. 定位与核心结论

**一句话**：给「添加渠道」加一个 **RSSHub 向导**——用户填一个 RSSHub 服务器地址，从「全量站点/路由目录」里搜索、选路由、填参数，一键生成 feed 地址并作为普通 `feed` 渠道入库。

**关键结论：这不动采集抽象层，是纯「目录 + 向导」便捷层。**

理由：`PLAN.md` 已把 RSSHub 定性为「第三方桥接源按普通 `feed` 直接对接」（功能规划 → 采集渠道）。RSSHub 输出的就是标准 RSS 2.0 / Atom / JSON Feed，而 `internal/source/feed/feed.go` 已经能解析这三种格式（含 `.atom` / `.json` 后缀变体）。所以：

- ✅ 复用现有 `feed` Connector、`POST /api/source`、三层去重、调度——零契约改动。
- ✅ 生成出来的 source 就是 `type=feed`、`config={"url":"{server}{route}"}`，跟手填 feed URL 完全等价。
- ❌ 不新增 `rsshub` 渠道类型、不改 `Connector`/`Item` 接口、不碰 `internal/app`/`internal/database`/`internal/filter`。

「解析 RSSHub 支持的所有站点和格式」这句话里，「格式」对应 RSSHub 的 RSS/Atom/JSON 输出，现有解析器已覆盖；「所有站点」对应的**路由目录**才是本功能的实质工作。

## 2. 路由目录从哪来（本功能真正的难点）

调研结论（事实）：

- RSSHub 的路由定义在 GitHub 仓库 [`DIYgod/RSSHub`](https://github.com/DIYgod/RSSHub) 的 `lib/routes/<namespace>/<route>.ts`，每个文件里 `export const route: Route = {...}`。
- 元数据字段稳定：`path`（含 `:param` 占位符，`/xxx/:id`，可选参数带 `?`）、`name`、`categories`、`example`、`parameters`（参数名→说明）、`features`（`requireConfig`/`requirePuppeteer`/`antiCrawler` 等布尔）、`maintainers`、`radar`。
- `namespace` = `lib/routes` 下的一级目录名，也是路由 path 的第一段。
- **RSSHub 实例本身不提供「列出全部路由」的 API**；目录权威来源是源码仓库/文档站。
- 体量：路由在千级，序列化成 JSON 约 1–2 MB。

三种数据来源方案：

| 方案 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| A. 运行时拉取 | 前端/后端用 jsDelivr 枚举 `lib/routes/**/*.ts` 并解析 TS | 总是最新 | 依赖联网、首次要等、TS 解析脆弱，违背「本地优先」 |
| B. **打包快照（推荐）** | 维护者脚本联网生成 `routes.json`，提交进仓库，`go:embed` 打进二进制 | 本地优先、离线可用、构建可复现 | 需定期手动跑脚本更新 |
| C. 混合 | B 之上再加「从上游刷新目录」按钮/端点 | 兼顾 | 实现量最大 |

**推荐 B（首版），C 作为可选增强。** 这与本项目「本地优先、离线构建、`go:embed` 打进二进制」的既有风格一致（参考 `internal/webui` 的做法）。

生成脚本（维护者工具，联网时跑一次，不进正常构建）：`git clone --depth 1 DIYgod/RSSHub` 或下载 GitHub tarball → 遍历 `lib/routes/**/*.ts` → 提取 `route` 元数据（字段都是字面量，正则/轻量解析即可，不必完整解析 TS）→ 产出 `routes.json` 并记录生成时的 commit hash / 日期。

## 3. 数据模型 / 契约

### 3.1 目录快照 schema（`routes.json`）

```jsonc
{
  "version": "master@<commit-hash>",   // 来源版本
  "generated_at": "2026-…T…Z",
  "routes": [
    {
      "namespace": "bilibili",
      "name": "用户投稿视频",
      "path": "/bilibili/user/video/:uid",
      "example": "/bilibili/user/video/2267573",
      "categories": ["social-media"],
      "parameters": { "uid": "UP 主 id，主页可见" },
      "maintainers": ["DIYgod"],
      "require_config": false        // 来自 features.requireConfig 等，扁平化
    }
  ]
}
```

字段取舍：前端只需 `namespace`（分组）、`name`（显示名默认值）、`path`（渲染参数 + 拼 URL）、`example`（占位提示）、`parameters`（输入框与说明）、`categories`（分组/筛选）、`require_config`（标记「需实例配置，公共实例不可用」）。`features` 里与「能否直接用」相关的布尔扁平化，其余丢弃以缩小体积。

### 3.2 RSSHub 源入库形态（零契约改动）

```jsonc
// 就是普通 feed 源
{ "type": "feed", "config": { "url": "https://rsshub.app/bilibili/user/video/2267573" } }
```

首版**不加** `origin`/`rsshub` 之类特殊 config 字段——保持 `config` 对核心层不透明、RSSHub 与手填 feed 完全等价。溯源靠 URL 前缀即可。

### 3.3 服务器地址（base_url）

- 默认值 `https://rsshub.app`（官方公共实例），向导里始终可见可改（自建实例填自己的地址）。
- 首版用**前端 `localStorage` 记住上次填写的地址**（轻量、无后端改动）；持久化到后端（`config.yml` 的 `rsshub.base_url` 或 settings 表）作为可选增强，不在首版引入新的 settings 机制。

## 4. 后端设计

### 4.1 新包 `internal/rsshub/`

```
internal/rsshub/
├── routes.json      # go:embed 的目录快照
├── catalog.go       # Route / Catalog 结构体、go:embed 加载、解析
└── catalog_test.go
```

纯静态读取：`Load()` 解析内嵌 JSON，提供 `Routes()`、按 namespace/category 分组、简单过滤。**不引入联网逻辑**（联网留给生成脚本）。

### 4.2 新 API 端点

| 端点 | 作用 | 说明 |
|---|---|---|
| `GET /api/rsshub/routes` | 返回目录 | 一次性返回全量（含 `version`/`generated_at`），前端缓存后客户端搜索 |
| `GET /api/rsshub/meta`（可选） | 目录版本 + 默认 base_url | 前端显示「目录更新于何时」 |

- 在 `internal/server/route.go` 注册；新增 `internal/server/rsshub_api.go`（含 Swagger 注解）。
- 一次全量返回的理由：1–2 MB 本地读取、无网络往返，客户端过滤体验更顺；不做服务端 `?q=` 搜索。
- 目录不落数据库（它是对外部开源项目的快照，不是用户数据），走 `go:embed` 只读。

## 5. 前端设计

### 5.1 入口

`SourceList.jsx` 顶部「+ 添加渠道」旁新增「+ RSSHub」按钮，打开向导对话框（`RsshubWizard.jsx`）。不动现有 `SourceForm.jsx`（那是手填 feed 的表单）。

### 5.2 向导流程

```
① 填服务器地址（默认 https://rsshub.app，记住上次）
      ↓
② 搜索/浏览目录（按 namespace 或 categories 分组，模糊搜名称/namespace/描述）
      ↓
③ 选中一条路由 → 展示其参数 → 逐个填参数（example 作占位、parameters 作说明）
      ↓
④ 实时预览：完整 feed URL + 显示名（默认用 route.name，可改）+ 刷新间隔
      ↓
⑤ 「添加」→ 调 api.createSource({type:'feed', config:{url:预览URL}, name, interval, enabled})
      ↓
⑥ 停留在向导，可继续加下一条（快速连续添加）；关闭后刷新渠道列表
```

### 5.3 组件与改动

| 文件 | 改动 |
|---|---|
| `NewsGlean-web/src/components/RsshubWizard.jsx` | 新增：搜索框 + 分组列表 + 参数表单 + 预览 + 添加 |
| `NewsGlean-web/src/components/SourceList.jsx` | 加「+ RSSHub」入口、复用 `load()` 刷新 |
| `NewsGlean-web/src/api.js` | 新增 `listRsshubRoutes()`、`rsshubMeta()` |
| 复用 | `Dialog`/`Input`/`Select`/`Badge` 等现有 UI 组件 |

### 5.4 交互细节

- 参数占位符：`path` 里的 `:param` 必填项逐一生成输入框；`example` 里的对应片段作 placeholder；`parameters` 说明显示在输入框下方。
- `require_config=true` 的路由：标注「需实例配置（公共实例不可用）」，仍允许添加（用户可能自建），但不误导。
- 搜索：纯客户端对 `name`/`namespace`/`path` 做不区分大小写子串匹配；`categories` 提供侧栏筛选（如 `social-media`、`programming`、`new-media`…）。
- 显示名：默认取 `route.name`，可在预览区改；不再调 `probeSource`（路由名已足够，且省一次网络请求）。

## 6. 分阶段实施（每阶段有 DoD）

| 阶段 | 内容 | DoD |
|---|---|---|
| **R1 目录快照 + 生成脚本** | 定 schema；写生成脚本（联网拉仓库、提取元数据、产出 `routes.json`）；生成并提交初版快照 | `routes.json` 覆盖全部 namespace，字段齐全，能被 Go 结构体解析；脚本可重复运行 |
| **R2 后端 `internal/rsshub` + API** | `catalog.go`（go:embed + 解析）；`GET /api/rsshub/routes`（含版本/生成时间）+ Swagger 注解；`route.go` 注册 | 接口返回目录；`go build`/`vet`/`test` 全绿（注意 `GOCACHE`/`GOMODCACHE` 指工作区内的环境约束） |
| **R3 前端向导** | 入口按钮 + `RsshubWizard.jsx` + `api.js` 封装 + 参数表单 + URL 预览 + 调 `createSource` | 纯浏览器完成「选服务器→搜路由→填参数→添加 feed→列表出现→手动采集到条目」 |
| **R4 增强（可选，待定）** | 批量多选添加、base_url 后端持久化、「从上游刷新目录」、来源溯源标记 | 视需求再定 |

## 7. 风险与开放问题（待拍板）

1. **目录更新策略** —— 推荐「打包快照 + 手动跑脚本」（方案 B）。能否接受「目录可能滞后于 RSSHub 上游，需定期跑脚本刷新」？还是首版就要「运行时一键刷新」（方案 C）？
2. **批量添加** —— 首版做「单条添加、可停留连续加」够不够？还是要「购物车式多选批量入库」？
3. **服务器地址持久化** —— 首版用前端 `localStorage`（最简单）可以吗？还是要落到后端配置？
4. **生成脚本放哪/用什么语言** —— 建议放仓库根 `scripts/update-rsshub-routes.mjs`（前端已是 Node 环境，解析 TS 用正则/`@babel` 顺手）；还是要求纯 Go 生成器？
5. **是否要 `require_config` 路由的硬拦截** —— 倾向「标注但不拦截」，是否同意？

### 当前推荐默认项（未拍板前按此推进）

- 方案 B：打包快照 + 手动跑脚本
- 单条添加、可停留连续加
- 前端 `localStorage` 记住服务器地址
- Node 生成脚本，放 `scripts/`
- `require_config` 只标注不拦截
