# Webnovel Writer 完全使用指南

## 一、系统是什么

Webnovel Writer 是基于 Claude Code 的长篇网文创作系统（v5.5.4）。它解决 AI 写长篇小说时的两个致命问题：

- **遗忘**：写到第 50 章时忘了第 3 章埋的伏笔
- **幻觉**：主角明明是斗者，突然用出斗皇的招式

核心机制：双 Agent 架构 + 六维并行审查 + SQLite 索引 + RAG 语义检索。

---

## 二、安装

### 2.1 安装插件

```bash
claude plugin marketplace add lingfengQAQ/webnovel-writer --scope user
claude plugin install webnovel-writer@webnovel-writer-marketplace --scope user
```

### 2.2 安装 Python 依赖

```bash
python -m pip install -r requirements.txt
```

### 2.3 配置 RAG（必做）

在项目根目录创建 `.env`：

```bash
EMBED_BASE_URL=https://api-inference.modelscope.cn/v1
EMBED_MODEL=Qwen/Qwen3-Embedding-8B
EMBED_API_KEY=你的key

RERANK_BASE_URL=https://api.jina.ai/v1
RERANK_MODEL=jina-reranker-v3
RERANK_API_KEY=你的key
```

Embedding 用于向量检索（"语义上相关的段落"），Reranking 用于结果重排序。两者缺失时系统降级为 BM25 关键词检索，精度下降但仍可用。

---

## 三、从零开始写一本新书

### 3.1 初始化项目

```bash
/webnovel-init
```

系统会分 6 步交互式收集信息：

| 步骤 | 收集内容 |
|------|----------|
| Step 1 | 书名、流派、目标字数、一句话梗概、核心冲突、目标读者 |
| Step 2 | 主角设定（姓名/性格/能力/情绪配置）、反派层级、关系冲突 |
| Step 3 | 金手指设计（类型/名称/成长节奏/不可逆代价） |
| Step 4 | 世界观（规模/势力/力量体系/社会等级/货币） |
| Step 5 | 创作约束（反套路规则/硬约束/卖点/开篇钩子） |
| Step 6 | 一致性确认 |

产出：
```
PROJECT_ROOT/
├── .webnovel/state.json       # 项目状态
├── .webnovel/index.db         # 空索引
├── .webnovel/vectors.db       # 空向量库
├── 设定集/主角卡.md
├── 设定集/世界观.md
├── 设定集/力量体系.md
├── 大纲/总纲.md
└── .webnovel/idea_bank.json   # 创意约束库
```

### 3.2 生成大纲

```bash
/webnovel-plan 1    # 生成第1卷大纲
```

流程：
1. 加载总纲和设定集
2. 生成卷节拍表（催化→危机链→中点反转→低谷→大高潮→新钩子）
3. 生成卷时间线（确保时序一致）
4. 生成章节大纲（每章含目标/阻力/代价/钩子/Strand 标签）

产出：
- `大纲/第1卷-节拍表.md`
- `大纲/第1卷-时间线.md`
- `大纲/第1卷-详细大纲.md`

每章大纲包含的关键字段：
```
目标 / 阻力 / 代价 / 时间锚点 / 爽点类型
Strand(Quest/Fire/Constellation) / 反派层级
本章变化 / 章末未闭合问题 / 钩子类型
```

### 3.3 写章节

```bash
/webnovel-write 1           # 标准模式（完整流水线）
/webnovel-write 1 --fast    # 跳过风格适配
/webnovel-write 1 --minimal # 最小化审查
```

**完整流水线（6 步）：**

```
Step 1  Context Agent 组装上下文
        ↓ 读取大纲/设定/前章摘要/实体状态/伏笔/追读力数据
        ↓ 输出"创作执行包"（8板块任务书 + Context Contract + 直写提示词）

Step 2A 正文起草
        ↓ 基于执行包写 2000-2500 字正文
        ↓ 遵循"大纲即法律/设定即物理/发明需识别"

Step 2B 风格适配（--fast 跳过）
        ↓ 表达层转译，消除模板腔/说明腔

Step 3  六维并行审查
        ↓ 核心3个（一致性/连贯性/OOC）始终执行
        ↓ 条件3个（爽点/节奏/追读力）按需触发
        ↓ 产出 overall_score + 各维度分 + 问题清单

Step 4  润色修复
        ↓ 按 critical > high > medium > low 优先级修复
        ↓ Anti-AI 终检（消除 AI 味）

Step 5  Data Agent 数据回写
        ↓ AI 提取实体/关系/状态变更 → index.db
        ↓ 生成章节摘要 → summaries/
        ↓ 场景切片 → scenes 表
        ↓ 向量嵌入 → vectors.db

Step 6  更新 state.json 进度
```

**三种模式对比：**

| | 标准 | --fast | --minimal |
|--|------|--------|-----------|
| Context Agent | ✅ | ✅ | ✅ |
| 正文起草 | ✅ | ✅ | ✅ |
| 风格适配 | ✅ | ❌ | ❌ |
| 审查 | 核心+条件 | 核心+条件 | 仅核心3个 |
| 润色 | ✅ | ✅ | ✅ |
| Data Agent | ✅ | ✅ | ✅ |

---

## 四、已有小说导入

如果你已有一部写了数万字的小说，按以下步骤接入系统。

### 4.1 准备文件

1. **章节文件**：命名为 `第0001章-标题.md`，放入 `正文/`
2. **大纲文件**：至少写一个简版总纲 + 已写章节的一句话章纲
3. **设定集**：填充主角卡、世界观、力量体系

### 4.2 初始化 + 逐章构建索引

```bash
/webnovel-init           # 创建项目骨架
# 将准备好的文件放入对应目录

/webnovel-write 1 --fast  # 从第1章开始，按顺序
/webnovel-write 2 --fast
...
/webnovel-write N --fast
```

**必须从第 1 章按顺序执行**，因为：
- 实体消歧依赖前 N-1 章累积的别名表
- 状态变更链有先后依赖
- 章节摘要是后续 Context Agent 的输入

时间估算：每章 2-3 分钟，30 章约 1-1.5 小时。

### 4.3 验证

```bash
/webnovel-query 状态
/webnovel-query 主角名
/webnovel-dashboard
```

---

## 五、日常创作命令速查

| 命令 | 用途 | 典型场景 |
|------|------|----------|
| `/webnovel-write N` | 写第 N 章 | 日常创作 |
| `/webnovel-write N --fast` | 快速写（跳过风格适配） | 赶进度 |
| `/webnovel-write N --minimal` | 最小审查 | 快速迭代 |
| `/webnovel-plan N` | 生成第 N 卷大纲 | 开新卷前 |
| `/webnovel-review N-M` | 审查第 N 到 M 章 | 阶段性质检 |
| `/webnovel-query 关键词` | 查询角色/伏笔/状态 | 随时查阅 |
| `/webnovel-query 紧急` | 查看紧急问题 | 矛盾预警 |
| `/webnovel-resume` | 断点恢复 | 中断后继续 |
| `/webnovel-dashboard` | 可视化面板 | 全局概览 |
| `/webnovel-learn "描述"` | 提取写作模式 | 记录成功经验 |

---

## 六、核心概念

### 6.1 防幻觉三定律

| 定律 | 含义 | 执行者 |
|------|------|--------|
| 大纲即法律 | 严格遵循大纲，不擅自发挥 | Context Agent 强制加载 |
| 设定即物理 | 不违背已有设定 | Consistency Checker 校验 |
| 发明需识别 | 新实体必须入库管理 | Data Agent 自动提取 |

### 6.2 Strand Weave 三线交织

| Strand | 含义 | 理想占比 | 最大断档 |
|--------|------|----------|----------|
| Quest | 主线剧情 | 55-65% | 5 章 |
| Fire | 感情线 | 20-30% | 10 章 |
| Constellation | 世界观扩展 | 10-20% | 15 章 |

系统通过 `strand_tracker` 自动追踪每章的 Strand 标签，超过断档阈值时 Pacing Checker 会报警。

### 6.3 追读力体系

**钩子类型**（章末悬念）：
- 危机钩：敌人/危险出现
- 悬念钩：信息缺口/未解之谜
- 欲望钩：预期的奖励/成就
- 情绪钩：愤怒/同情/崇拜
- 抉择钩：高代价的两难选择

**爽点模式**（章内高潮）：每章 1-2 个，关键章 2-3 个，高潮章 3-4 个。

**差异化规则**：同一钩子类型不得连续 3 章使用，同一爽点模式不得连续 5 章使用。

### 6.4 债务追踪（默认关闭）

当 AI 违背软约束时（如钩子强度不足），系统创建"Override Contract"——记录违背理由和偿还计划。未兑现的承诺被量化为"债务"，每章计息 10%，逾期升级告警。

开启方式：用户明确要求或在项目配置中启用。

---

## 七、数据层：index.db

系统的"记忆"存储在 SQLite 数据库中，15 张表：

| 表 | 存什么 | 谁写入 |
|---|--------|--------|
| entities | 实体（角色/地点/物品/势力/招式）的当前状态 | Data Agent |
| aliases | 别名映射（"萧炎"→xiaoyan，一对多） | Data Agent |
| state_changes | 状态变更链（斗者→斗师，含原因和章节） | Data Agent |
| relationships | 关系快照（A→B 的关系类型和描述） | Data Agent |
| relationship_events | 关系事件流（含极性/强度/证据） | Data Agent |
| chapters | 章节元数据（标题/地点/字数/出场角色） | Data Agent |
| scenes | 场景索引（起止行/地点/摘要） | Data Agent |
| appearances | 实体出场记录（章节+提及列表+置信度） | Data Agent |
| override_contracts | 超控契约（违背软约束的理由和偿还计划） | 写作流水线 |
| chase_debt | 追读力债务（本金/利息/状态） | 写作流水线 |
| debt_events | 债务事件日志 | 写作流水线 |
| chapter_reading_power | 章节追读力元数据（钩子/爽点/微兑现） | 审查流水线 |
| review_metrics | 审查指标（6维评分） | 审查流水线 |
| invalid_facts | 无效事实标记 | 审查/手动 |
| writing_checklist_scores | 写作清单评分 | Context Manager |

---

## 八、上下文获取机制

写新章时，Context Agent 从 7 个数据源拉取上下文：

```
state.json        → 进度、主角状态、伏笔、节奏追踪
index.db          → 核心实体、最近出场、追读力、债务
summaries/        → 前 N 章摘要（剧情+钩子+承接点）
大纲/             → 本章目标/阻力/代价、时间线
设定集/           → 世界观、力量体系、风格契约
vectors.db (RAG)  → 语义相关的历史段落（按需触发）
流派画像          → 钩子偏好、节奏阈值、写作指导
```

组装成 **8 板块任务书**：

1. 核心任务（目标/阻力/代价）
2. 接住上章（上章钩子+读者期待）
3. 出场角色（状态/动机/情绪/红线）
4. 场景约束（地点/可用能力/禁用能力）
5. 时间约束（时间锚点/跨度/倒计时）
6. 风格指导（本章类型/差异化建议）
7. 伏笔（按紧急度排序，必须处理/可选）
8. 追读力策略（钩子类型/微兑现/未闭合问题）

---

## 九、六维审查体系

| Checker | 检查什么 | 典型问题 |
|---------|----------|----------|
| consistency-checker | 设定一致性 | 战力越级、地点矛盾、时间线错乱 |
| continuity-checker | 叙事连贯性 | 场景跳切无过渡、情节断裂 |
| ooc-checker | 角色一致性 | 行为偏离人设、动机断层 |
| reader-pull-checker | 追读力 | 钩子弱、期待落空、节奏拖沓 |
| high-point-checker | 爽点密度 | 爽点不足、模式重复 |
| pacing-checker | 节奏控制 | Strand 断档、信息密度失衡 |

核心 3 个（一致性/连贯性/OOC）始终执行，其余按需触发。

审查产出 `overall_score`（0-100）和各维度分数，写入 `review_metrics` 表。

---

## 十、流派支持

系统内置 6 个流派模板目录，支持更多流派通过 `genre_aliases` 映射：

| 目录 | 覆盖流派 |
|------|----------|
| xuanhuan | 玄幻、仙侠、修仙 |
| realistic | 都市、现实、职场 |
| dog-blood-romance | 狗血言情、甜宠 |
| period-drama | 宫斗、年代、古言 |
| rules-mystery | 规则怪谈、无限流 |
| zhihu-short | 知乎短篇 |

每个流派有独立的钩子偏好、爽点密度、节奏阈值配置。支持复合流派（如"都市+修仙"）。

---

## 十一、断点恢复

任何步骤中断后：

```bash
/webnovel-resume
```

系统自动检测上次完成到哪一步，提供恢复选项：

| 中断步骤 | 恢复策略 |
|----------|----------|
| Step 1 (Context) | 直接重新执行 |
| Step 2A (起草) | 删除半成品，重新起草 |
| Step 2B (风格) | 继续适配或回退到 2A |
| Step 3 (审查) | 用户决定：重新审查或跳过 |
| Step 4 (润色) | 继续润色或删除重写 |
| Step 5 (Data Agent) | 重新执行（幂等操作） |
| Step 6 (提交) | 检查暂存区，决定提交或回滚 |

---

## 十二、查询与监控

### 常用查询

```bash
/webnovel-query 萧炎        # 查角色状态
/webnovel-query 伏笔        # 查未回收伏笔
/webnovel-query 紧急伏笔    # 查即将超期的伏笔
/webnovel-query 金手指      # 查金手指状态
/webnovel-query 节奏        # 查 Strand 分布
/webnovel-query 状态        # 查项目整体进度
```

### 伏笔紧急度

系统自动计算伏笔紧急度：

- 🔴 **紧急**：已超期，或核心伏笔超过 20 章未回收
- 🟡 **警告**：已过目标章节 80%
- 🟢 **正常**：在计划内

### Dashboard

```bash
/webnovel-dashboard
```

打开 `http://127.0.0.1:8765`，可视化查看：
- 创作进度和 Strand 分布
- 实体关系图谱（3D 力导向图）
- 章节内容浏览
- 追读力分析数据

纯只读，不修改任何数据。

---

## 十三、写作模式学习

```bash
/webnovel-learn "本章的危机钩设计很有效，悬念拉满"
```

系统自动分类（hook/pacing/dialogue/payoff/emotion），追加到 `.webnovel/project_memory.json`。后续 Context Agent 会读取这些模式作为风格参考。

---

## 十四、项目文件结构

```
PROJECT_ROOT/
├── .webnovel/
│   ├── state.json              # 项目状态（进度/主角/节奏/伏笔/chapter_meta）
│   ├── index.db                # SQLite 索引（15张表，实体/关系/债务/审查）
│   ├── vectors.db              # RAG 向量嵌入
│   ├── summaries/ch0001.md     # 章节摘要
│   ├── project_memory.json     # 学习到的写作模式
│   ├── preferences.json        # 用户偏好
│   ├── context_snapshots/      # 上下文快照缓存
│   └── observability/          # 性能日志
├── 正文/
│   └── 第0001章-标题.md
├── 大纲/
│   ├── 总纲.md
│   ├── 第1卷-节拍表.md
│   ├── 第1卷-时间线.md
│   └── 第1卷-详细大纲.md
├── 设定集/
│   ├── 主角卡.md
│   ├── 世界观.md
│   └── 力量体系.md
└── .env                        # API 配置
```

---

## 十五、常见问题

### Q: 写到一半 Claude 断了怎么办？
`/webnovel-resume`，系统自动从断点恢复。

### Q: 审查分数太低怎么办？
Step 4 会自动修复 critical 和 high 级别问题。也可以 `/webnovel-review N` 单独审查后手动修改。

### Q: 能不能跳过某些步骤？
`--fast` 跳过风格适配，`--minimal` 最小化审查。但 Context Agent 和 Data Agent 不可跳过——它们是系统记忆的读写入口。

### Q: 向量 API 挂了怎么办？
系统自动降级为 BM25 关键词检索，精度下降但不中断。修复 API 后自动恢复。

### Q: 能支持多少字？
备份系统支持 200 万字规模。index.db 有 28 个索引，查询性能不会随章节数量显著下降。

### Q: 能不能多人协作？
当前设计为单人创作。index.db 使用 SQLite 文件锁，不支持并发写入。

### Q: 如何备份？
```bash
# Git 备份（内置）
python webnovel.py backup --chapter N

# 回滚到某章
python webnovel.py backup --rollback --chapter N
```
