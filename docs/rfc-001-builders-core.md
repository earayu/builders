# RFC-001: Builders 核心系统设计

> Status: Draft
> Author: @Bryce
> Reviewers: @戴清源, @GLM
> TL: @earayu2
> Date: 2026-06-20

## 1. 背景与目标

### 1.1 背景

在 AI 时代，builder（技术创业者、开源贡献者、AI 研究员、明星工程师等）的活动分散在多个平台：GitHub、X/Twitter、LinkedIn、论文平台、Product Hunt、Hacker News、博客、新闻等。传统平台（如 LinkedIn）依赖人主动维护信息，数据更新慢且不完整。

现在 Agent 时代，我们有能力让 AI agent 主动跨平台采集公开信息，构建一个以人为核心的 builder 活动情报库。

### 1.2 目标

构建一个 **agent 驱动的 AI builder 情报图谱**，核心解决以下问题：

- **有哪些人？** — 发现和维护 AI 领域的重要 builder
- **在做什么？** — 追踪每个人的活动、项目、发言、动态
- **在哪里工作？** — 维护人与组织的关系及变动
- **他们之间的关系如何？** — 人与人、人与组织、人与项目之间的关系网络

### 1.3 产品定位

- **高端 builder 版企查查**：以人为核心，而非以公司为核心
- **Agent 驱动**：不是人主动录入，而是 agent 主动跨平台采集和维护
- **证据可追溯**：每条信息都有来源和置信度，不是黑箱画像
- **V1 无 UI**：先做核心引擎和数据层，CLI/API 交互

## 2. 项目边界

### 2.1 本项目负责

- 多源公开数据采集（GitHub/X/LinkedIn/HN/论文/新闻/官网等）
- 证据链管理（每条数据可追溯到原始来源）
- 结构化事实/事件提取（谁在什么时候做了什么）
- 实体维护与身份消歧（Person/Organization/Project/Technology）
- 实体间关系管理，带时间维度
- 人物活动时间线（包括通过第三方发现的活动）
- 任务系统（agent 领取任务、执行采集、更新数据）
- 元数据维护（数据新鲜度、置信度）

### 2.2 不做（交给其他项目）

- 文档导入与 chunk 切分（知识库/RAG 项目）
- 向量化与语义检索（知识库/RAG 项目）
- 基于 chunk 的问答（知识库/RAG 项目）
- 用户账号系统与权限管理
- Web UI
- 推荐系统
- 人工审核流程（V1 全自动，AI 维护 + AI 审核）

### 2.3 与知识库/RAG 项目的交互

- **本项目 → RAG**：结构化数据（实体、关系、事件）可作为 RAG 的知识源
- **RAG → 本项目**：RAG 的语义搜索可辅助本项目做模糊匹配和消歧
- 核心区别：本项目 = 结构化事实图谱 + 证据管理；RAG 项目 = 非结构化文档检索 + 问答

## 3. 核心概念

| 概念 | 说明 |
|------|------|
| `person` | Builder 人物，核心实体，所有信息围绕人组织 |
| `identity` | 人物在某平台的账号/身份（GitHub/X/LinkedIn 等） |
| `organization` | 公司/实验室/基金/大学/开源组织 |
| `project` | 代码仓库/产品/模型/论文/SDK/工具 |
| `technology` | 技术标签/领域（AI agent/RAG/robotics 等） |
| `activity` | 人的活动/事件（发推/演讲/发布/融资/入职/离职等） |
| `relationship` | 实体间的持续关系，带时间维度 |
| `source_item` | 原始来源条目（一条推文/文章/API 结果） |
| `evidence` | 从来源中抽取的 claim/excerpt，关联到活动或关系 |
| `task` | agent 的采集/维护任务 |

## 4. 核心需求

### 4.1 以人为核心的数据模型

所有信息围绕 Person 组织：

- **Person Profile**：canonical name、简介、地区、角色标签、状态
- **跨平台身份**：同一个人在不同平台的账号，通过 identity 表关联，支持软合并
- **组织/项目/技术**：作为人的附属信息存在，丰富人物画像

### 4.2 活动追踪

追踪每个人的活动，包括：
- 人自己发起的活动（发推、发布项目、演讲、写博客）
- 通过他人发现的活动（B 的推特提到 A 参加了某活动 → 更新到 A 的活动时间线）

每条活动记录：
- `primary_person_id`：活动的主体人物，方便直接查询某人的时间线
- 活动类型（post/talk/launch/funding/hire/join/leave/publish/mention 等）
- 现实发生时间 `occurred_at` + 系统发现时间 `discovered_at`
- 参与者（多人参与同一活动）及其角色（subject/mentioned/speaker/author/source_author/witness）
- 关联的组织/项目/技术
- 关联的证据（通过 `activity_evidences` 细粒度关联）

**"B 发推证明 A 活动"的场景处理：**
- B 的推文存为 `source_item`（记录内容发布者 B）
- 从中抽取的 claim 存为 `evidence`，关联到 `source_item`
- activity 的 `primary_person_id` 指向 A
- `activity_participants` 中 A 为 `subject`，B 为 `source_author`
- 证据链清晰：activity ← activity_evidences ← evidence ← source_item（作者 B）

### 4.3 关系网络

维护人与人、人与组织、人与项目之间的持续关系：
- 关系带时间维度（`valid_from` / `valid_to`），可回答"A 去年在哪、现在在哪"
- 关系类型：works_at / founded / maintains / authored / invested_in / advises / collaborated_with
- 关系有角色描述（如 "Head of Agents"、"CTO"）
- 每条关系通过 `relationship_evidences` 关联支撑证据

### 4.4 身份消歧

跨平台采集必然遇到同一人在不同平台用不同名字/ID：
- 每个平台账号独立存为 `identity`，不在采集时合并
- 用 `person` 实体做软合并，记录合并理由和置信度
- 消歧信号：GitHub commit email、个人域名、社交图谱重叠度、内容相似度
- 支持后续拆分（合并错误时可恢复）

### 4.5 证据链管理

采用两层结构，分离"原始来源"和"抽取的证据"：

- **source_item**：原始来源条目（一条推文/文章/API 返回），记录作者、URL、发布时间、原文 hash。解决"内容发布者"和"活动主体"的归属区分。
- **evidence**：从 source_item 中抽取的具体 claim/excerpt，关联到活动或关系。

活动和关系通过细粒度关联表（`activity_evidences` / `relationship_evidences`）连接证据，支持标注证据类型（direct / indirect / inferred / conflicting）。

### 4.6 数据自动置信度

全自动运行，无人工审核，但通过置信度分层保证数据质量：
- `verified_by_source`：强证据（官网 team page、GitHub owner、论文作者）
- `inferred_high`：多源一致推断
- `inferred_low`：单源或弱信号
- `conflict`：证据冲突，不自动覆盖

冲突证据处理：都保留并标注 confidence，AI 自动判断，profile 展示时标注冲突。

### 4.7 定期活动追踪

每个人可配置追踪策略，定期收集活动并生成快照：

- **person_tracking_policies**：配置追踪频率、数据源范围、上次/下次检查时间
- **person_activity_snapshots**：每周/每月 summary，包含重要活动 ID 列表
- snapshot 是派生展示层，事实源仍然是 activities/evidences

## 5. 任务系统

### 5.1 设计原则

任务系统是整个平台的调度核心，但 V1 保持简单：
- **像公告板**：发布任务，agent 领取，做完标记完成
- **agent 直接写主库**：不需要中间 proposal/review/apply 环节
- **可追溯**：数据表带 `created_by_task_id`，知道每条数据是哪个任务产生的

### 5.2 任务流程

```
发布任务(open) → agent 领取(claimed) → 执行中(in_progress) → 完成(done) / 失败(failed)
```

支持的状态：`open` / `claimed` / `in_progress` / `done` / `failed` / `cancelled`

### 5.3 任务类型

| 类型 | 说明 |
|------|------|
| `refresh_person_profile` | 全面更新某人 profile |
| `collect_person_activity` | 收集某人近期活动 |
| `discover_accounts` | 找某人的跨平台账号 |
| `verify_identity_merge` | 判断两个 identity 是否同一人 |
| `refresh_project` | 更新项目信息 |
| `refresh_organization` | 更新组织信息 |
| `discover_related_people` | 从项目/公司/事件发现新的 builder |
| `resolve_conflict` | 处理冲突证据 |
| `explore_topic_builders` | 围绕某技术方向发现新人 |

### 5.4 任务创建来源

1. **系统自动生成**：新实体创建时自动生成补全任务；profile 过期时自动生成刷新任务
2. **Agent 产出**：agent 执行任务时发现新线索，创建后续任务
3. **手动创建**：指定要更新某人/项目

### 5.5 任务执行者

高确定性任务和低确定性任务都通过任务系统领取：
- **低成本/高确定性**：调 API（GitHub/HN/OpenAlex），可由轻量 worker 或 agent 执行
- **高不确定性**：多轮搜索互联网，由 agent（Codex CLI/Claude Code 等）自主决策执行

### 5.6 防重复

`idempotency_key` 字段防止同一目标在同一时间窗口内重复创建任务。

## 6. 数据库 Schema

### 6.1 技术选型

- **主库**：PostgreSQL（Neon serverless）
- **规模预期**：百~千级人物，小规模
- **关系查询**：2-hop 为主，PG 递归 CTE 够用
- **更新时效**：每日 batch，不需要实时
- **语言**：全球，中英文优先

### 6.2 设计说明

- **Polymorphic 字段**（`subject_type/subject_id`、`ref_type/ref_id`、`linked_type/linked_id`）不使用 PG 外键约束，由应用层保证引用一致性。V1 可接受，后续可加 CHECK 约束或触发器。
- **Tombstone 支持**：核心实体表包含 `deleted_at` 字段，支持软删除，为后续纠错/删除预留。

### 6.3 完整 DDL

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- ============================================
-- 1. 核心实体
-- ============================================

-- 人物
CREATE TABLE persons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    canonical_name TEXT NOT NULL,
    display_name TEXT,
    bio TEXT,
    avatar_url TEXT,
    region TEXT,
    language TEXT,
    role_tags TEXT[],
    status TEXT DEFAULT 'active',
    last_refreshed_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- 人物的外部账号/身份
CREATE TABLE identities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id UUID REFERENCES persons(id),
    platform TEXT NOT NULL,
    platform_uid TEXT,
    profile_url TEXT,
    display_name TEXT,
    verified BOOLEAN DEFAULT false,
    confidence FLOAT DEFAULT 0.5,
    merge_reason TEXT,
    evidence_urls TEXT[],
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID,
    UNIQUE(platform, platform_uid)
);

-- 组织
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    aliases TEXT[],
    type TEXT,
    description TEXT,
    website TEXT,
    region TEXT,
    status TEXT DEFAULT 'active',
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- 项目/产品
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    aliases TEXT[],
    type TEXT,
    description TEXT,
    url TEXT,
    repo_url TEXT,
    org_id UUID REFERENCES organizations(id),
    status TEXT DEFAULT 'active',
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- 技术/话题标签
CREATE TABLE technologies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    category TEXT,
    description TEXT
);

-- ============================================
-- 2. 活动与事件
-- ============================================

-- 活动/事件
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_type TEXT NOT NULL,
    primary_person_id UUID REFERENCES persons(id),
    title TEXT,
    summary TEXT,
    occurred_at TIMESTAMPTZ,
    occurred_at_precision TEXT DEFAULT 'day',
    discovered_at TIMESTAMPTZ DEFAULT now(),
    confidence FLOAT DEFAULT 0.5,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- 活动参与者
CREATE TABLE activity_participants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_id UUID REFERENCES activities(id),
    person_id UUID REFERENCES persons(id),
    role TEXT NOT NULL,
    UNIQUE(activity_id, person_id, role)
);

-- 活动关联实体（polymorphic，应用层保证引用一致性）
CREATE TABLE activity_refs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_id UUID REFERENCES activities(id),
    ref_type TEXT NOT NULL,
    ref_id UUID NOT NULL,
    role TEXT
);

-- 活动-证据关联
CREATE TABLE activity_evidences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    activity_id UUID REFERENCES activities(id),
    evidence_id UUID,
    link_type TEXT DEFAULT 'direct',
    UNIQUE(activity_id, evidence_id)
);

-- ============================================
-- 3. 关系
-- ============================================

-- 实体间关系（polymorphic，应用层保证引用一致性）
CREATE TABLE relationships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject_type TEXT NOT NULL,
    subject_id UUID NOT NULL,
    relation TEXT NOT NULL,
    object_type TEXT NOT NULL,
    object_id UUID NOT NULL,
    role_title TEXT,
    valid_from TIMESTAMPTZ,
    valid_to TIMESTAMPTZ,
    confidence FLOAT DEFAULT 0.5,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- 关系-证据关联
CREATE TABLE relationship_evidences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    relationship_id UUID REFERENCES relationships(id),
    evidence_id UUID,
    link_type TEXT DEFAULT 'direct',
    UNIQUE(relationship_id, evidence_id)
);

-- ============================================
-- 4. 来源与证据（两层结构）
-- ============================================

-- 原始来源条目（一条推文/文章/API 结果）
CREATE TABLE source_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_url TEXT NOT NULL,
    source_platform TEXT,
    author_name TEXT,
    author_identity_id UUID REFERENCES identities(id),
    title TEXT,
    raw_content TEXT,
    published_at TIMESTAMPTZ,
    collected_at TIMESTAMPTZ DEFAULT now(),
    content_hash TEXT,
    created_by_task_id UUID,
    UNIQUE(content_hash)
);

-- 从来源中抽取的证据/claim
CREATE TABLE evidences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_item_id UUID REFERENCES source_items(id),
    claim TEXT NOT NULL,
    excerpt TEXT,
    confidence FLOAT DEFAULT 0.5,
    link_type TEXT DEFAULT 'direct',
    created_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- 通用证据关联（补充 activity_evidences/relationship_evidences 未覆盖的场景）
-- polymorphic，应用层保证引用一致性
CREATE TABLE evidence_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    evidence_id UUID REFERENCES evidences(id),
    linked_type TEXT NOT NULL,
    linked_id UUID NOT NULL
);

-- ============================================
-- 5. 追踪策略与快照
-- ============================================

-- 人物追踪策略
CREATE TABLE person_tracking_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id UUID REFERENCES persons(id) UNIQUE,
    frequency TEXT DEFAULT 'weekly',
    source_scope TEXT[],
    last_checked_at TIMESTAMPTZ,
    next_check_at TIMESTAMPTZ,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- 人物活动快照（周/月 summary，派生展示层）
CREATE TABLE person_activity_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id UUID REFERENCES persons(id),
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    summary TEXT,
    important_activity_ids UUID[],
    created_at TIMESTAMPTZ DEFAULT now(),
    created_by_task_id UUID
);

-- ============================================
-- 6. 任务系统
-- ============================================

CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type TEXT NOT NULL,
    target_type TEXT,
    target_id UUID,
    description TEXT,
    priority INT DEFAULT 0,
    status TEXT DEFAULT 'open',
    claimed_by TEXT,
    claimed_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    idempotency_key TEXT UNIQUE,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE task_comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID REFERENCES tasks(id),
    author TEXT,
    content TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================
-- 7. 索引
-- ============================================

CREATE INDEX idx_persons_name ON persons(canonical_name);
CREATE INDEX idx_persons_deleted ON persons(deleted_at) WHERE deleted_at IS NULL;
CREATE INDEX idx_identities_person ON identities(person_id);
CREATE INDEX idx_identities_platform ON identities(platform, platform_uid);
CREATE INDEX idx_activities_type ON activities(activity_type);
CREATE INDEX idx_activities_primary_person ON activities(primary_person_id);
CREATE INDEX idx_activities_occurred ON activities(occurred_at);
CREATE INDEX idx_activities_discovered ON activities(discovered_at);
CREATE INDEX idx_activity_participants_person ON activity_participants(person_id);
CREATE INDEX idx_activity_participants_activity ON activity_participants(activity_id);
CREATE INDEX idx_activity_evidences_activity ON activity_evidences(activity_id);
CREATE INDEX idx_activity_evidences_evidence ON activity_evidences(evidence_id);
CREATE INDEX idx_relationships_subject ON relationships(subject_type, subject_id);
CREATE INDEX idx_relationships_object ON relationships(object_type, object_id);
CREATE INDEX idx_relationship_evidences_rel ON relationship_evidences(relationship_id);
CREATE INDEX idx_source_items_url ON source_items(source_url);
CREATE INDEX idx_source_items_platform ON source_items(source_platform);
CREATE INDEX idx_evidences_source ON evidences(source_item_id);
CREATE INDEX idx_evidence_links_linked ON evidence_links(linked_type, linked_id);
CREATE INDEX idx_tracking_next ON person_tracking_policies(next_check_at) WHERE enabled = true;
CREATE INDEX idx_snapshots_person ON person_activity_snapshots(person_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_target ON tasks(target_type, target_id);
```

## 7. 数据源

### 7.1 优先级

| 优先级 | 数据源 | 接入方式 | 说明 |
|--------|--------|----------|------|
| P0 | 搜索引擎 | Jina Search API | 低成本，通用覆盖 |
| P0 | GitHub | 公开 API | Builder 核心平台，项目和贡献 |
| P0 | Hacker News | Firebase API | 技术社区讨论和热点 |
| P1 | X/Twitter | OpenCLI/browser use | 高价值但需登录态 |
| P1 | LinkedIn | OpenCLI/browser use | 职业信息，需登录态 |
| P1 | 论文/OpenAlex | 公开 API | 研究员和研究型 founder |
| P2 | Product Hunt | 公开 API | 产品发布和 maker |
| P2 | Crunchbase | REST API（基础免费） | 公司融资和创始人 |
| P2 | Reddit | 公开 API | 技术讨论 |
| P3 | 官网/博客 | Jina Reader | 个人信息和文章 |
| P3 | 新闻/融资新闻 | Jina Search/Read | 事件流 |

### 7.2 采集模式

| 模式 | 场景 | 执行者 |
|------|------|--------|
| API 调用 | 低成本、确定性高 | 轻量 worker |
| Agent 多轮搜索 | 高不确定性、需要判断和决策 | AI agent（Codex CLI/Claude Code） |

## 8. 核心流程

### 8.1 系统架构

```
┌─────────────────────────────────────────────┐
│              任务系统（Task Board）           │
│  发布任务 → agent 领取 → 执行 → 标记完成     │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌──────────────┐    ┌──────────────────┐
│ 轻量 Worker   │    │    AI Agent       │
│ (API 调用)    │    │ (多轮搜索/决策)   │
└──────┬───────┘    └────────┬─────────┘
       │                     │
       └──────────┬──────────┘
                  │ 直接写入
                  ▼
┌─────────────────────────────────────────────┐
│              数据层（PostgreSQL）             │
│  persons │ identities │ organizations        │
│  projects │ activities │ relationships        │
│  source_items │ evidences │ technologies      │
└─────────────────────────────────────────────┘
```

### 8.2 典型流程：更新某人 Profile

1. 系统检测到某人 `last_refreshed_at` 过期（或 `person_tracking_policies.next_check_at` 到期），自动创建 `refresh_person_profile` 任务
2. Agent 领取任务
3. Agent 查询该人当前的 identities（GitHub/X/LinkedIn 账号）
4. Agent 逐平台采集最新信息：
   - GitHub：最近 commits/repos/stars
   - X：最近推文和互动
   - 搜索引擎：最近新闻/访谈/演讲
5. Agent 将原始内容存入 `source_items`，抽取的 claim 存入 `evidences`
6. Agent 将新发现的活动写入 `activities` 表，通过 `activity_evidences` 关联证据
7. Agent 更新 `relationships`（如发现换了公司），通过 `relationship_evidences` 关联证据
8. Agent 更新 `persons.last_refreshed_at` 和 `person_tracking_policies`
9. Agent 标记任务完成，留下 task_comment 总结
10. 如发现新的 builder/项目，agent 创建后续任务

### 8.3 典型流程：通过 B 发现 A 的活动

1. Agent 在采集 B 的推特时，发现 B 提到"和 A 一起参加了 XX 会议"
2. Agent 将 B 的推文存入 `source_items`（author 为 B）
3. Agent 从推文中抽取 claim 存入 `evidences`（关联 source_item）
4. Agent 创建 activity（type=`talk`，title="XX 会议"，primary_person_id=A）
5. Agent 在 `activity_participants` 中关联：A（role=`subject`）、B（role=`source_author`）
6. Agent 通过 `activity_evidences` 关联证据到活动
7. A 的活动时间线查询 `primary_person_id=A` 或 `activity_participants` 均可看到这条活动
8. 证据链完整：activity ← activity_evidences ← evidence ← source_item（作者 B）
9. 如果 A 还不在系统中，agent 创建新的 person 和 `discover_accounts` 后续任务

## 9. 第一版目标人群

- AI 领域 KOL
- 著名 builder / 开源项目 maintainer
- 大厂明星员工（AI 方向）
- AI startup founder
- 研究员创业者

第一批建议从 100-500 人的种子名单开始，打磨采集和消歧流程后再通过关系网络自动扩展。

## 10. 技术约束

### 10.1 V1 技术假设

- 小规模（百~千级人物）
- 全球覆盖，中英文优先
- 每日 batch 更新，不需要实时
- 以人为中心的 profile 输出
- 2-hop 关系查询，PG 递归 CTE 够用
- PostgreSQL 主库（Neon serverless）
- 全自动运行，AI 维护 + AI 审核

### 10.2 合规边界

- 只采集公开信息
- 记录来源和采集时间
- 排除敏感个人信息、非职业信息、私人联系方式
- 注意各平台 ToS、robots.txt、API 限制
- Schema 支持 tombstone（软删除，`deleted_at` 字段），为后续纠错/删除预留

## 11. 开放问题

| 问题 | 当前状态 |
|------|----------|
| Neon 数据库项目创建 | 待创建 |
| Jina API key | 待 @earayu2 提供 |
| OpenCLI 凭据配置 | 待 @earayu2 配置 |
| 第一批种子名单 | 待定义 |
| 删除/纠错/申诉机制 | V1 暂不做，schema 已预留 |
| 与知识库/RAG 项目的接口协议 | 待定义 |

## 12. 里程碑

| 阶段 | 内容 | 产出 |
|------|------|------|
| M1 | 数据库建表 + 基础 CRUD API | 可以手动插入和查询人物数据 |
| M2 | 任务系统 | 任务创建/领取/完成/查询 |
| M3 | 第一个 Source Adapter（GitHub） | 能从 GitHub 采集 builder 信息 |
| M4 | 搜索引擎 + Jina 接入 | 通用网络搜索和正文提取 |
| M5 | 活动追踪 + 关系维护 | 完整的人物活动时间线 |
| M6 | 种子名单导入 + 首次全量采集 | 100+ builder profile 数据 |
