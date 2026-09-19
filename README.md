# NewsGlean 设计文档

> NewsGlean 的设计方案与实现规格仓库。把「怎么想、怎么做、为什么这么做」沉淀成文档，与代码仓库分离，让规格先行、实现跟进。

本仓库只放**文档**，不含任何代码。所有文档统一用中文书写，内容随实现进度持续回写更新。

---

## 文档索引

| 文档 | 类型 | 状态 | 内容 |
| ---- | ---- | ---- | ---- |
| [PLAN.md](./PLAN.md) | 主设计规格 | 持续更新（M1 已完成，后续按 21 个执行阶段推进） | 采集抽象、功能规划、命令设计、架构概览、数据模型、配置、路线图、技术选型 |
| [rsshub-quick-add.md](./rsshub-quick-add.md) | 渠道规划 | 规划阶段（待决策） | RSSHub 向导：填服务器地址 → 搜索路由 → 一键生成 `feed` 渠道 |
| [bilibili-cli-integration.md](./bilibili-cli-integration.md) | 渠道规划 | 规划阶段（B1 字段实测已完成，待决策） | 以 bilibili-cli 为取数后端，把 B 站内容（含字幕正文）纳入统一阅读流 |
| [telegram-integration.md](./telegram-integration.md) | 渠道规划 | 规划阶段（待决策） | `telegram`（Bot API）与 `telegram_user`（MTProto）两个渠道类型 |

---

## 文档之间的关系

```
PLAN.md（主设计规格，唯一权威）
   ├── 采集契约（Connector / Item / Cursor / State）—— 各渠道规划都锚定它
   ├── 数据模型、命令设计、路线图、技术选型
   │
   └── 渠道扩展（独立成篇，决策后回写 PLAN.md 的渠道清单表）
        ├── rsshub-quick-add.md    —— 零契约改动，纯「目录 + 向导」便捷层
        ├── bilibili-cli-integration.md —— 新增 bilibili 渠道类型（拉取型）
        └── telegram-integration.md     —— 新增 telegram / telegram_user，需补「推送模式」调度接线
```

- **PLAN.md 是主文档**：设计决策、契约定义、路线图都收敛在这里，是唯一权威来源。
- **渠道规划独立成篇**：每个渠道的调研、实测、开放问题单独成文，避免 PLAN.md 膨胀；拍板后再把结论回写 PLAN.md 的渠道清单表。
- **状态标记约定**：每篇文档开头标注状态（「已完成 / 进行中 / 规划阶段（待决策）」），PLAN.md 内各章节用 `[x]` / `[~]` / `[ ]` 勾选标记实现进度。

---

## 阅读约定

PLAN.md 中混着两类内容，阅读时请区分：

- **契约（已定）** —— 抽象接口、数据模型这类一旦定下就影响全局的设计，修改代价高。
- **草案（待定）** —— 具体库选型、命令名、配置字段名，实现时才定稿。

> ⚠️ 本文档描述的不少功能仍处于规划阶段，**请勿按文档去使用尚未实现的功能**；以代码仓库的实际实现为准。

---

## 关联仓库

| 仓库 | 说明 |
| ---- | ---- |
| [news-glean](https://github.com/ma6254/news-glean) | 服务端：Go + Cobra + GORM，采集、去重、存储、HTTP API |
| NewsGlean-web | 前端：React SPA，阅读流、渠道管理、系统信息 |

---

## 贡献

1. 涉及「契约」（接口、数据模型）的改动，先在 Issue 里对齐范围再动手。
2. 渠道规划类文档遵循「调研 → 实测 → 开放问题 → 待决策」的结构；决策后回写 PLAN.md 对应章节。
3. 文档改动与代码改动保持同步：契约或数据模型变了，PLAN.md 对应章节与状态标记同一步更新。

---

## License

[MIT](https://github.com/ma6254/news-glean) © 2026 ma6254
