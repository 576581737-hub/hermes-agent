---
name: skill-surgeon
description: "审计 Hermes Agent skills 体系：重复能力、职责冲突、膨胀风险、覆盖缺口、优化建议。"
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [skills, audit, governance, quality, devops]
    related_skills: [hermes-agent-skill-authoring, kanban-orchestrator]
---

# 技能外科手术 (skill-surgeon)

## 概述

审计当前 Hermes Agent skills 体系的健康度。识别重复能力、职责冲突、类型覆盖缺口，输出可执行的重构路线图。

**审计对象**：`~/.hermes/skills/**/SKILL.md`（用户本地 skills）和 `~/.hermes/hermes-agent/skills/**/SKILL.md`（内置 skills）。

## 何时使用

- **"帮我看看这些 skill 有没有重复"**
- **"哪些 skill 的职责冲突了"**
- **"当前 skill 体系还缺什么"**
- **"帮我做一次 skill 审计"**
- **"帮我优化当前 skill 结构"**

## 审计方法

### 0. 前置检查（高可用）

在执行审计前，先做环境检查：

```bash
# 检查 skills 目录是否存在
ls ~/.hermes/skills/ 2>/dev/null || echo "WARN: 用户本地 skills 目录不存在"
ls ~/.hermes/hermes-agent/skills/ 2>/dev/null || echo "WARN: 内置 skills 目录不存在"

# 统计 skill 数量
find ~/.hermes/skills/ -name "SKILL.md" 2>/dev/null | wc -l
find ~/.hermes/hermes-agent/skills/ -name "SKILL.md" 2>/dev/null | wc -l
```

**降级策略**：
- 若某个 SKILL.md 无法解析 frontmatter → 记录路径，跳过，继续审计其余
- 若 skills 目录为空 → 直接输出"当前无 skill 可审计"，不报错
- 若 skill 数量 > 50 → 分批审计，每批 20 个，避免超 context 限制

### 1. 读取 skill 基础信息

使用 `skill_view(name)` 逐一读取 skill，**不依赖猜测**。至少分析：

- **name** — 技能名称
- **description** — 技能描述
- **metadata.hermes.tags** — 标签分类
- **metadata.hermes.related_skills** — 关联技能
- **适用场景** — 从 Overview / When to Use 提取
- **工作流程** — 核心步骤

**错误隔离**：单个 skill_view 调用失败（网络/IO 错误），记录后跳过，不影响整体审计。

### 2. 判定重复能力

两个或以上 skill 同时具备以下特征，判为 **可能重复**：

- 服务同一类用户意图
- 产出高度相似的结果
- 工作流程差异很小
- 缺少明确主次职责区分

输出时区分：

- **完全重复** — 几乎可以合并
- **部分重叠** — 可以共存，但需明确边界

### 3. 判定职责冲突

两个 skill 都声称自己是同类任务的主处理 skill，且没有明确协作分工，判为 **职责冲突**。

常见冲突模式：
- 都想做代码审查主入口
- 都想做调试主入口
- 都想做计划生成主入口

### 4. 判定膨胀风险（Bloat Audit）

Hermes Agent 的 skill 加载机制会扫描 `~/.hermes/skills/` 和内置 `skills/` 目录，将匹配的 SKILL.md 注入 system prompt。随着 skill 数量增长，system prompt 会不断膨胀，影响推理效率和 token 消耗。

**膨胀指标**：

| 指标 | 健康 | 警告 | 危险 |
|------|------|------|------|
| 总 skill 数量 | ≤30 | 31-50 | >50 |
| 单 skill 内容量 | ≤8KB | 8-15KB | >15KB |
| 总注入体积 | ≤200KB | 200-400KB | >400KB |
| 冗余/废弃 skill 占比 | ≤10% | 10-20% | >20% |

**膨胀来源判定**：

- **超大 skill** — 单个 SKILL.md > 15KB，应考虑拆分到 `references/` 目录
- **低价值 skill** — description 过于宽泛导致频繁误匹配注入，实际很少使用
- **废弃 skill** — 功能已被其他 skill 覆盖，或依赖的 API/工具已不可用
- **过度细分** — 一个功能拆成多个高度相似的 skill，可合并
- **可降级 skill** — 不常用但体积大的 skill，应移至 `optional-skills/`

**判定方法**：

```bash
# 按体积排序 skill
find ~/.hermes/skills/ -name "SKILL.md" -exec wc -c {} \; 2>/dev/null | sort -rn | head -10
find ~/.hermes/hermes-agent/skills/ -name "SKILL.md" -exec wc -c {} \; 2>/dev/null | sort -rn | head -10

# 统计总注入体积
find ~/.hermes/skills/ -name "SKILL.md" -exec cat {} \; 2>/dev/null | wc -c
find ~/.hermes/hermes-agent/skills/ -name "SKILL.md" -exec cat {} \; 2>/dev/null | wc -c
```

**输出格式**：

- **P1 [skill-name]** — 体积 NKB（超过 15KB 阈值），建议将详细步骤拆分到 `references/`
- **P2 [skill-name]** — 疑似废弃（description 引用已不存在的 API/工具）
- **P2 [category]** — 该分类下 N 个 skill 高度相似，建议合并为 1-2 个

### 5. 判定覆盖缺口

按 Hermes Agent 实际分类审查覆盖情况：

| 分类 | 说明 | 审计标准 |
|------|------|---------|
| `software-development` | 编码、调试、测试、审查 | 至少 5 个 |
| `github` | PR、Issue、代码审查 | 至少 3 个 |
| `devops` | 部署、编排、监控 | 至少 2 个 |
| `creative` | 设计、绘图、视频 | 至少 3 个 |
| `research` | 搜索、论文、情报 | 至少 2 个 |
| `mlops` | 训练、推理、评估 | 至少 3 个 |
| `productivity` | 文档、日历、邮件 | 至少 2 个 |
| `note-taking` | 笔记、知识库 | 至少 1 个 |
| `social-media` | 发帖、搜索、管理 | 至少 1 个 |
| `smart-home` | 智能家居控制 | 至少 1 个 |
| `media` | 音视频处理 | 至少 1 个 |
| `autonomous-ai-agents` | 子代理编排 | 至少 2 个 |
| `mcp` | MCP 工具集成 | 至少 1 个 |
| `gaming` | 游戏相关 | 至少 1 个 |
| `email` | 邮件收发 | 至少 1 个 |

**缺口判定**：某分类 skill 数量低于审计标准，或某分类完全缺失。

### 6. 输出优化建议

每个问题配一条可执行建议：

- **合并** — 完全重复的 skill 合并为一个
- **拆分** — 职责过重的 skill 拆分为多个
- **明确主次** — 冲突 skill 增加协作说明
- **新增** — 覆盖缺口需要新建 skill
- **重命名** — 名称不反映实际职责

### 7. 优先级判定

| 级别 | 定义 | 处理时限 |
|------|------|---------|
| **P0** | 已造成明显主责冲突，继续扩展会加剧混乱 | 立即 |
| **P1** | 重复能力较高，短期内应明确边界或合并 | 本轮 |
| **P2** | 存在优化空间，但不影响当前整体可用性 | 后续 |

### 8. 生成重构路线图

三段式：

1. **立即处理** — 本轮就应修改的职责冲突或命名问题
2. **下一轮处理** — 应在新增 skill 前完成的边界梳理
3. **后续增强** — 等业务稳定后再补的治理能力

## 输出格式

输出格式严格遵循 **USER.md §3 Feishu MD Specs**：
- **✅ 允许**：标题、代码块、表格、**加粗**、*斜体*、~~删除线~~、`行内代码`、Unicode Emoji
- **❌ 禁止**：外部图片直链、Emoji 短码、`- [ ]` 任务列表（用 ☐ 替代）、HTML 标签、扩展语法

### 一、重复能力审计

- **[skill-A] vs [skill-B]** — 完全重复 / 部分重叠
  - 重叠点：...
  - 建议：合并 / 保留边界

### 二、职责冲突审计

- **P0 [skill-A] vs [skill-B]** — 主责冲突
  - 冲突点：...
  - 建议：重新划分主责

### 三、膨胀风险审计

- **总注入体积** — NKB（健康 ≤200KB / 警告 200-400KB / 危险 >400KB）
- **超大 skill** — [skill-name] 体积 NKB，建议拆分到 references/
- **废弃 skill** — [skill-name] 依赖已不可用
- **过度细分** — [category] 下 N 个 skill 高度相似，建议合并

### 四、覆盖缺口审计

- **software-development** — 已覆盖 N 个（标准 ≥5），正常
- **mcp** — **缺口**：当前 0 个（标准 ≥1），建议新增

### 五、结构优化建议

- **保留** — [skill-list]
- **重命名** — [skill-list] 建议改为 [新名称]
- **新增** — [缺口类型] 建议新建

### 六、处理优先级

- **P0** — [问题描述]（理由：...）
- **P1** — [问题描述]（理由：...）
- **P2** — [问题描述]（理由：...）

### 七、重构路线图

- **立即处理**：...
- **下一轮处理**：...
- **后续增强**：...

## 审计约束

1. **不要强行说有冲突** — 没有就是没有
2. **部分重叠优先建议"明确边界"**，而不是直接合并
3. **缺口真实存在要直接指出**，不要绕开
4. **优先级必须拉开** — 不要所有问题都打 P0
5. **路线图要落到具体修改动作**，不能只写抽象方向
6. **审计结果可保存** — 输出到文件供后续对比

## 常见陷阱

1. **误判"相关"为"重复"** — 两个 skill 关联但不重叠，不算重复
2. **忽略 description 的精确性** — description 太宽泛会导致误判，应参考 tags 和实际内容
3. **只审本地不审内置** — 用户可能依赖内置 skill，必须同时审计
4. **数量多时超 context** — 超过 20 个 skill 必须分批

## 验证清单

- ☐ 环境检查通过（目录存在、可读）
- ☐ 所有 SKILL.md 已通过 skill_view 读取
- ☐ 重复/冲突判定有具体依据（引用 description 原文）
- ☐ 覆盖缺口对照实际分类
- ☐ 膨胀风险评估（体积、废弃、过度细分）
- ☐ 优先级拉开（至少 2 个级别）
- ☐ 路线图包含具体修改动作
- ☐ 输出格式遵循 Feishu MD Specs
