# CLAUDE.md

## 项目概述

Webnovel Writer 是基于 Claude Code 的长篇网文创作系统（v5.5.4），解决 AI 写作中的"遗忘"和"幻觉"问题。采用双 Agent 架构 + 六维并行审查，支持长周期连载创作。

## 技术栈

- **后端**: Python 3.10+, FastAPI, SQLite, aiohttp, Pydantic 2.0+
- **前端 (Dashboard)**: React 19, Vite 6.2, React Force Graph 3D
- **外部 API**: Qwen3-Embedding-8B (ModelScope), Jina Reranker v3
- **运行环境**: Claude Code 插件（Skills + Agents 体系）

## 项目结构

```
webnovel-writer/                  # 插件主目录
├── agents/                       # 8 个 Agent 定义 (.md)
│   ├── context-agent.md          # 读取型：写前上下文组装
│   ├── data-agent.md             # 写入型：章节数据提取入库
│   └── {6 checkers}.md           # 六维质量审查
├── skills/                       # 8 个 Skill（斜杠命令）
│   ├── webnovel-init/            # 项目初始化
│   ├── webnovel-plan/            # 卷/章大纲生成
│   ├── webnovel-write/           # 章节写作流水线（核心）
│   ├── webnovel-review/          # 多维质量审查
│   ├── webnovel-query/           # 运行时查询
│   ├── webnovel-resume/          # 断点恢复
│   ├── webnovel-dashboard/       # 可视化面板
│   └── webnovel-learn/           # 写作模式提取
├── scripts/                      # Python 后端（46 个业务模块）
│   ├── data_modules/             # 核心数据层
│   │   ├── index_manager.py      # SQLite index.db 总管（15 张表）
│   │   ├── index_entity_mixin.py # 实体 CRUD + 别名 + 状态变更 + 关系图谱
│   │   ├── index_chapter_mixin.py# 章节/场景/出场记录
│   │   ├── index_debt_mixin.py   # 追读力债务（Override Contract + 利息）
│   │   ├── index_reading_mixin.py# 章节追读力元数据 + 审查指标
│   │   ├── index_observability_mixin.py # 无效事实 + 工具统计
│   │   ├── rag_adapter.py        # 向量 + BM25 混合检索 + Jina 重排序
│   │   ├── context_manager.py    # 加权上下文包组装
│   │   ├── context_ranker.py     # 上下文排序（时近性/频率/钩子加分）
│   │   ├── entity_linker.py      # 实体消歧 + 置信度评估
│   │   ├── state_manager.py      # state.json 读写
│   │   ├── sql_state_manager.py  # SQLite 扩展写入
│   │   ├── query_router.py       # RAG 意图识别 + 查询路由
│   │   ├── schemas.py            # Pydantic 数据模型（Data Agent 输出）
│   │   ├── config.py             # 配置加载（.env 层级）
│   │   └── api_client.py         # Embedding/Reranking API 客户端
│   ├── workflow_manager.py       # 工作流状态追踪 + 断点恢复
│   ├── backup_manager.py         # Git 备份 + 原子回滚
│   ├── status_reporter.py        # 项目健康度报告
│   └── webnovel.py               # 统一 CLI 入口
├── dashboard/                    # FastAPI 可视化面板
│   ├── app.py                    # FastAPI 应用
│   └── frontend/                 # React + Vite（预构建 dist/）
├── genres/                       # 流派模板
├── references/                   # 共享知识库（约束规范）
└── templates/                    # 输出模板
```

## 核心架构

### 防幻觉三定律

1. **大纲即法律** — Context Agent 强制加载章节大纲，不擅自发挥
2. **设定即物理** — Consistency Checker 实时校验，不自相矛盾
3. **发明需识别** — Data Agent 自动提取新实体并消歧入库

### 双 Agent + 六 Checker

- **Context Agent**（读）: 写前组装"创作任务书"（上下文包）
- **Data Agent**（写）: 写后提取实体/关系/状态变更 → index.db
- **6 Checkers**（并行审查）: 爽点 / 一致性 / 节奏 / OOC / 连贯性 / 追读力

### 写作流水线

```
/webnovel-write N →
  Step 1: Context Agent 组装上下文
  Step 2A: 起草（大纲驱动）
  Step 2B: 风格适配（可选）
  Step 3: 六维并行审查
  Step 4: 润色修复
  Step 5: 债务追踪（利息计算）
  Step 6: Data Agent 提取 → index.db 持久化
```

## 数据层（index.db）

SQLite 数据库，15 张表，按版本演进：

| 代 | 表 | 用途 |
|----|-----|------|
| 基础 | chapters, scenes, appearances | 章节/场景索引，实体出场 |
| v5.1 | entities, aliases, state_changes, relationships | 实体图谱，别名一对多，状态变更链 |
| v5.3 | override_contracts, chase_debt, debt_events, chapter_reading_power | 追读力债务金融模型 |
| v5.4-5.5 | invalid_facts, review_metrics, tool_call_stats, writing_checklist_scores, relationship_events, rag_query_log | 可观测性 + 关系事件流 |

关键设计:
- **实体 Upsert**: 字段级智能合并 `{**old, **new}`，不覆盖已有状态
- **关系双层存储**: relationships（快照）+ relationship_events（事件流），查询时事件优先、快照补齐
- **债务金融模型**: 创建 → 每章计息 → 偿还 → 原子结清关联 Override Contract
- **28 个索引**: 覆盖实体类型/重要度/章节/关系/债务状态等高频查询路径

## 常用命令

```bash
# Python 依赖安装
python -m pip install -r requirements.txt

# 运行测试
cd webnovel-writer/scripts && python -m pytest data_modules/tests/ -v

# 统一 CLI
python webnovel-writer/scripts/webnovel.py <command> --project-root <path>

# Dashboard
python -m webnovel-writer.dashboard.server --project-root <path>
```

## Skill 命令（Claude Code 内使用）

| 命令 | 用途 |
|------|------|
| `/webnovel-init` | 初始化新小说项目 |
| `/webnovel-plan N` | 生成第 N 卷大纲 |
| `/webnovel-write N` | 写第 N 章（完整流水线） |
| `/webnovel-write N --fast` | 跳过风格适配 |
| `/webnovel-write N --minimal` | 最小化审查 |
| `/webnovel-review N-M` | 批量质量审查 |
| `/webnovel-query 关键词` | 运行时查询（角色/伏笔/状态） |
| `/webnovel-resume` | 自动检测断点并恢复 |
| `/webnovel-dashboard` | 启动可视化面板 |
| `/webnovel-learn "描述"` | 提取写作模式到项目记忆 |

## 小说项目目录结构

```
PROJECT_ROOT/
├── .webnovel/
│   ├── state.json          # 元数据 + 进度 + 主角状态 + 节奏追踪
│   ├── index.db            # SQLite 索引（15 张表）
│   ├── vectors.db          # RAG 向量嵌入
│   └── summaries/ch{NNNN}.md  # 章节摘要
├── 正文/第{NNNN}章-{标题}.md   # 章节正文
├── 大纲/
│   ├── 总纲.md              # 总大纲
│   ├── 第{N}卷-节拍表.md     # 卷节拍表
│   └── 第{N}卷-时间线.md     # 时间线
└── 设定集/                   # 世界观/角色卡/力量体系
```

## 环境变量（.env）

```bash
EMBED_BASE_URL=https://api-inference.modelscope.cn/v1
EMBED_MODEL=Qwen/Qwen3-Embedding-8B
EMBED_API_KEY=<key>
RERANK_BASE_URL=https://api.jina.ai/v1
RERANK_MODEL=jina-reranker-v3
RERANK_API_KEY=<key>
```

## 开发注意事项

- IndexManager 使用 Mixin 模式拆分为 5 个 Mixin，修改时注意继承关系
- `_init_db()` 中的建表顺序有依赖（chase_debt 外键引用 override_contracts）
- 实体消歧置信度阈值: >0.8 自动采纳, 0.5-0.8 标记待确认, <0.5 人工处理
- RAG 降级策略: 向量 API 失败(401) → 自动回退 BM25 关键词检索
- Override Contract 终态冻结: fulfilled/cancelled 状态的合约所有字段不可修改
- 债务计息防重复: 通过 debt_events 表检查本章是否已计过息
- 关系事件流兼容旧数据: 事件缺失时回退 relationships 快照补边
