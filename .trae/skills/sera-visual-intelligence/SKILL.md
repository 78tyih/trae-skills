---
name: "sera-visual-intelligence"
description: "把研究、项目、笔记、工作流、架构、时间线、数据与知识关系自动转换为最合适的可视化表达。Agent 先判断信息结构，再路由到 Flowchart、Gantt、Timeline、Mind Map、Infographic、Chart、React Flow、Knowledge Graph 或 Galaxy；适用于用户要求可视化、流程图、甘特图、知识图谱、项目总览、架构图、信息图，或当长文本明显可以通过视觉表达提升理解时。"
---

# Sera Visual Intelligence — Trae Entry

Canonical specification:

`78tyih/SeraContextHub/99_System/skills/sera-visual-intelligence/`

Trae 执行本 Skill 时必须先读取：

1. `SKILL.md`
2. `VISUAL_ROUTER.md`
3. `VISUAL_ASSET_SCHEMA.json`

不要在 Trae 内创建一套与 Canonical 不兼容的视觉路由协议。

## Core Pipeline

```text
Source Knowledge
→ Normalize
→ Visual Audit
→ Visual Intent Router
→ Visual Spec
→ Renderer
→ Validate
→ Register Visual Asset
→ Publish / Embed
```

## Default Renderer Order

```text
Mermaid / Markmap
→ AntV Infographic / GPT-Vis
→ React Flow / tldraw
→ G6
→ 3d-force-graph
```

优先选择可编辑、可版本管理、可移植的输出；不得为了视觉冲击直接使用 3D。

## Modes

- `auto`
- `audit`
- `explain`
- `workflow`
- `timeline`
- `architecture`
- `knowledge-map`
- `galaxy`
- `dashboard`

## Hard Requirements

1. 图必须忠于 Source，不得为了好看添加不存在的事实、数字或因果关系。
2. 至少保存一种可编辑 Source；不能只有 PNG / Screenshot。
3. 重要实体、数据与关系保留 source refs。
4. Galaxy 只在复杂关系探索时使用，不作为默认视图。
5. 同一事实只维护一个 Canonical Source，多张图只是不同 Views。
6. 对外展示优先 10 秒内看懂；内部 Explorer 才优先信息密度。
7. 每次完成研究、PRD、项目梳理、日报、复盘或 Workflow 后执行一次 Visual Audit。

## Context Hub Relationship

Context Hub 是 Canonical Knowledge Layer，本 Skill 是 Visual Projection Layer。

```text
Context / Memory / Wiki / Project
→ Visual Projection
→ Visual Asset
→ ASSET_INDEX / Notion / GitHub / Demo
```

旧 `knowledgestar-galaxy` 作为 Galaxy renderer / design reference 保留，不再成为独立事实源。
