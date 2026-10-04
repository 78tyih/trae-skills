# Trae Skills — Agent 技能库

> 给用 Trae（及任意支持 skills 目录的 Agent）的人：四个即装即用的技能包——从模糊想法到执行的决策树、反诈骗查链核验、博主知识拆解、可视化路由。每个技能是一个目录 + 一份 SKILL.md，复制进 `.trae/skills/` 即生效。

## 技能清单

| 技能 | 干什么 | 什么时候触发 |
|---|---|---|
| **grill-decision-tree** | 模糊想法 → 苏格拉底式 5 轮提问 → 共享理解摘要 → 自动路由到 3 条执行路径 | 用户提出模糊想法/需求，不直接动手，先澄清再执行 |
| **onchain-transaction-verifier** | 核验「已转账未到账 / 假成功页 / 共享屏幕同步付款」：从公开账本反推并交叉核验 TxHash / From / To / Block / Time | 对方声称已转账但存疑；TRON/TRC20 优先，可扩展 EVM/Solana |
| **sera-creator-intelligence** | 频道/视频库 → Transcript → Main Thesis / Claims / Evidence 拆解 → Knowledge Score + Must Watch/Worth/Skim/Skip → Creator Report | 分析博主、频道、视频值得不值得看；沉淀 JSON + Notion/Obsidian |
| **sera-visual-intelligence** | 先判断信息结构，再路由到 Flowchart / Gantt / Timeline / Mind Map / Chart / Knowledge Graph 等最合适的可视化 | 要求画图/总览/架构图，或长文本明显可以视觉化时 |

## 快速开始

```bash
git clone https://github.com/78tyih/trae-skills.git
cp -r trae-skills/.trae/skills/<技能名> <你的项目>/.trae/skills/
```

之后对 Trae 说「我有个模糊想法……」即自动触发 grill-decision-tree；「核验这笔 TRC20 转账」触发查链技能。

## 决策树怎么工作（grill-decision-tree）

```
模糊想法 → grill-me 5 轮提问 → 共享理解摘要 → 三选一分支
```

1. **5 轮提问**（每轮只问一个问题）：目标澄清 → 边界与成功标准 → 约束 → 倾向方案 → 决策确认
2. **共享理解摘要**：目标 / 范围 / 约束 / 倾向方案 / 任务类型（Agent 判断：简单 / 技术实现 / 复杂项目）
3. **分支路由**：

| 任务类型 | 路由到 | 流程 |
|---|---|---|
| 边界清晰，答案即解决 | 分支 1 · 单独结束 | 直接给答案 → 确认理解 → 闭环 |
| 技术实现，需代码质量 | 分支 2 · mattpocock/skills | Spec → Implement → Test → Code Review |
| 跨职能、长周期、需验证 ROI | 分支 3 · Superpowers | Design → Plan → Execute → Verify |

硬规则：5 轮不可跳过、每轮只问一个问题、分支选定不混用、发现选错回阶段二重新确认。

**交互演示：** [决策树可点选演示](https://78tyih.github.io/trae-skills/showcase.html)（在线版，逐轮走完整个流程）

## 反诈骗查链怎么工作（onchain-transaction-verifier）

核心思想一句话：**截图、共享屏幕、绿色 Success 都只是「对方展示的信息」；公开账本才是事实源。**

五个核心字段：TxHash · From · To · Block Height · Timestamp（辅助：Amount / Token / Chain）。拿到一个强锚点（尤其 TxHash）或两项可组合线索，即可反推其余字段交叉核验；组合不唯一时加时间窗口 / Token / 金额消歧。

最短核验路径示例：`TxHash → 全字段` · `Block + Time → 自洽验证` · `Time → 反推真实 Block` · `To + Time → 窗口内入账扫描` · `Address + Amount → 筛候选再消歧`。

## 四问速览

| 问 | 答 |
|---|---|
| **解决什么问题** | Agent 拿到模糊需求就闷头开干，返工率高；诈骗场景里用户拿「转账截图」当事实。四个技能分别把「先澄清再执行」「先查账再下结论」「先拆解再判断价值」「先选图型再画图」变成硬流程 |
| **什么场景 → 什么结果** | 「我有个模糊想法」→ 5 轮澄清 + 明确分支而不是半成品；「对方说转了 USDT」→ 链上字段核验报告；「这个博主值得追吗」→ 带证据链的 Knowledge Score |
| **什么结构** | 每技能 = 目录 + SKILL.md（frontmatter 触发描述 + 流程 + 硬规则）；creator/visual 两个技能是 canonical 实现的 Trae 触发入口，指向 `sera-opc-os` / `SeraContextHub` 的规范源，不在本仓库复刻协议 |
| **能复用什么** | ① grill 5 轮问题模板 + 三分支路由表，任何 Agent 可直接抄；② 查链五字段交叉核验方法论（TRON 起步）；③ Knowledge Score 六维权重（Insight 20 / Evidence 15 / Evergreen 20 / Novelty 15 / Argument 15 / Density 15） |

## 说明

- sera-creator-intelligence 与 sera-visual-intelligence 的规范本体分别在 `78tyih/sera-opc-os` 与 `78tyih/SeraContextHub`（私有），本仓库是其 Trae 原生触发入口
- onchain-transaction-verifier 只做「交易是否真实存在且自洽」的技术核验，不做人身诈骗定性；「两项足够」非数学保证，需继续消歧
