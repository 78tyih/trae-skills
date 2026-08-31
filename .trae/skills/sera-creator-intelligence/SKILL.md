---
name: "sera-creator-intelligence"
description: "分析 Creator/YouTube 博主/频道/视频库：建立目录与 Transcript，拆解 Main Thesis、Claims、Evidence、Reasoning、Assumptions，区分 Fact/Interpretation/Prediction，计算 Knowledge Score 与 Must Watch/Worth/Skim/Skip，生成跨视频主题分布、重复观点、思想演变、矛盾、预测追踪和 Creator Intelligence Report，并沉淀到结构化 JSON + Markdown/Notion/Obsidian。用户提到分析博主、频道分析、视频总结、视频拆解、视频文字稿、论点论据、哪些视频值得看、博主知识库、监控博主或同步到 Notion 时调用；不要用于视频制作/剪辑。"
---

# Sera Creator Intelligence — Trae Entry

Canonical implementation:

`78tyih/sera-opc-os/business/sera-creator-intelligence/`

本文件是 Trae 的原生触发入口。执行时必须优先读取 canonical `SKILL.md`、Schema、Template、Validator；若用户要求 Notion/云端发布，再读取 `NOTION_PUBLISHER.md`。不要在 Trae 内另造一套不兼容协议。

## Core Pipeline

```text
Source
→ Inventory
→ Transcript + Provenance
→ Normalize
→ Single Content Intelligence
→ Claim / Evidence / Reasoning / Assumption
→ Fact vs Interpretation vs Prediction
→ Knowledge Score + Watch Verdict
→ Cross-Video Synthesis
→ Canonical Knowledge
→ Creator Report
→ Notion / Obsidian / Search / Incremental Monitor
```

## Modes

- `inventory`
- `video`
- `triage`
- `channel`
- `refresh`
- `ask`
- `rebuild-report`

## Hard Requirements

1. JSON/JSONL 是 Source of Truth，Markdown/Notion 是阅读层。
2. Raw Transcript 不得混入 AI 推断。
3. 重要 Claim 尽量回指 Timestamp/Source。
4. Fact / Interpretation / Prediction 必须分开。
5. 每条视频输出 Main Thesis + Claims + Evidence + Reasoning + Assumptions。
6. Knowledge Score / 100：Insight 20 + Evidence 15 + Evergreen 20 + Novelty 15 + Argument Quality 15 + Density 15。
7. Personal Relevance 单独 0–10。
8. Watch Verdict 只能是 `must_watch / worth_watching / skim / note_only / skip`，并包含 confidence + reason。
9. Cross-video 知识按 Idea canonicalize，不为重复概念无限新建页面。
10. 冲突观点显式标注，不静默调和。
11. 默认不永久保存整频道视频/音频。
12. 单条失败记录后继续，不得中断全批次。
13. 完成前运行 canonical validator。
14. 指定 Notion 时，validator PASS 后才发布；Video ID / Channel ID 幂等更新，不重复创建。
15. 没有 Notion 连接时生成 `publish/notion_publish_queue.jsonl`，不得因此判定整个分析失败。

## Notion Publish

当用户要求“写入 Notion / 同步到云端笔记 / 发布 Creator Report”时：

1. 读取 canonical `NOTION_PUBLISHER.md`。
2. 通过 Notion MCP/API 按精确标题发现：
   - `Sera Creator Intelligence｜博主视频学习系统`
   - `Creator Intelligence Index`
   - `Creator Video Knowledge Base`
3. 数据库 ID 只缓存到本机 runtime state，不写入公共 Git 仓库。
4. 单视频以 `Video ID` 去重；Creator 以 `Channel ID` 或 canonical Channel URL 去重。
5. 已存在则 update，不存在才 create。
6. 批次顺序：视频页 → 重算 Creator 聚合 → 更新 Creator 页 → publish audit log。

## First Run

若用于“一个狠人”，读取 canonical `SMOKE_TEST.md`，只处理其中 10 条样本；通过 validator 后同步这 10 条到 Notion（或生成 publish queue），然后停止。不要直接启动全量 Backfill。
