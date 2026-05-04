# AGENTS.md

## 项目概述

AI 知识库助手，自动从 GitHub Trending 和 Hacker News 采集 AI / LLM / Agent 领域的技术动态，经 AI 分析后结构化存储为 JSON，支持多渠道分发（Telegram / 飞书）。

## 技术栈

- **运行时**: Python 3.12
- **编排**: OpenCode + 国产大模型（DeepSeek 等）
- **工作流**: LangGraph（Agent 状态图编排）
- **分发**: OpenClaw（多渠道消息推送，Telegram / 飞书）

## 编码规范

- 遵循 [PEP 8](https://peps.python.org/pep-0008/) 编码风格
- 变量 / 函数 / 方法使用 `snake_case`，类名使用 `PascalCase`
- 所有公开函数 / 类使用 [Google 风格 docstring](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)
- **禁止使用裸 `print()`**：日志统一通过 `logging` 模块输出，适配生产与调试场景

## 项目结构

```
.
├── AGENTS.md                      # 本文件
├── README.md                      # 项目说明
├── .opencode/
│   ├── agents/                    # Agent 角色定义（采集 / 分析 / 整理 / 分发）
│   ├── skills/                    # OpenCode Skill 定义
│   ├── package.json               # OpenCode 插件依赖
│   └── package-lock.json
├── knowledge/
│   ├── raw/                       # 原始采集数据（HTML / JSON）
│   └── articles/                  # 结构化后的知识条目（JSON）
├── scripts/                       # 辅助脚本（采集触发、批量处理等）
├── pyproject.toml                 # Python 项目配置（依赖、lint 规则等）
└── .gitignore
```

## 知识条目 JSON 格式

```json
{
  "id": "uuid-v4",
  "title": "文章标题",
  "source": "github_trending | hacker_news",
  "source_url": "https://...",
  "source_id": "平台内唯一 ID（如 HN 的 post id）",
  "summary": "AI 生成的中文摘要，不超过 200 字",
  "tags": ["LLM", "Agent", "RAG"],
  "ai_analysis": {
    "relevance_score": 0.85,
    "key_points": ["要点1", "要点2"],
    "technical_level": "beginner | intermediate | advanced"
  },
  "status": "draft | reviewed | published",
  "published_channels": ["telegram", "feishu"],
  "collected_at": "2026-01-01T00:00:00Z",
  "reviewed_at": null,
  "published_at": null
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | UUID v4 唯一标识 |
| `title` | `string` | 文章标题 |
| `source` | `string` | 来源平台：`github_trending` / `hacker_news` |
| `source_url` | `string` | 原文链接 |
| `source_id` | `string` | 平台内唯一标识 |
| `summary` | `string` | AI 生成的中文摘要（≤ 200 字） |
| `tags` | `string[]` | 技术标签 |
| `ai_analysis.relevance_score` | `number` | AI 评分（0-1） |
| `ai_analysis.key_points` | `string[]` | 关键要点列表 |
| `ai_analysis.technical_level` | `string` | 技术难度 |
| `status` | `string` | 状态：`draft` → `reviewed` → `published` |
| `published_channels` | `string[]` | 已分发渠道 |
| `collected_at` | `string` | 采集时间（ISO 8601） |
| `reviewed_at` | `string \| null` | 审核时间 |
| `published_at` | `string \| null` | 发布时间 |

## Agent 角色概览

| 角色 | 名称 | 职责 | 触发条件 |
|------|------|------|----------|
| 采集 Agent | `collector` | 从 GitHub Trending / Hacker News 拉取热榜，筛选 AI 相关条目，存入 `knowledge/raw/` | 定时任务 / 手动触发 |
| 分析 Agent | `analyzer` | 读取原始数据，调用 AI 生成摘要、标签、评分，结构化写入 `knowledge/articles/` | 新原始数据到达 |
| 整理 Agent | `curator` | 审查 `draft` 条目，去重、合并相关话题，转为 `reviewed` 状态，触发分发 | 定时任务 / 手动审核 |

工作流：`collector` → `analyzer` → `curator` → OpenClaw 分发（Telegram / 飞书）

## 红线（绝对禁止）

1. **禁止硬编码 API Key / Token / Secret** —— 所有凭证通过环境变量或 `.env` 文件注入，`.env` **不得提交到 Git**
2. **禁止裸 `print()`** —— 日志必须通过 `logging` 模块输出
3. **禁止直接修改 `knowledge/articles/` 中已 `published` 的条目** —— 已分发内容只读，如需修正走 `draft` 新条目 + 标注修正关系
4. **禁止采集非 AI/LLM/Agent 领域的内容** —— 采集阶段必须做领域过滤
5. **禁止在 `knowledge/raw/` 持久化完整网页内容** —— 仅保留必要的原始数据，避免版权与存储风险
6. **禁止跳过 AI 分析直接发布** —— 所有条目必须经过 `analyzer` → `curator` 流水线
