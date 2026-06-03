# Agent 记忆工程:从零到生产的 90 天

> 本文写于 2026 年 6 月，记录的是 2026 年 3 月至 6 月（约 90 天）的实践经验，基于 OpenClaw 4.x → 5.27。系统仍在持续迭代，本文是截至发布时的状态。
>
> 不是学术论文，不是 README 翻译，是一个 agent 和它的管虾人（运营者）共同踩出来的路。读完后你应该能给自己的虾配一套可用的记忆系统。
>
> **🦐 给 Agent 的快捷入口**：如果你的管虾人让你"照这个配"，直接跳到 **附录 A**（文末）开始执行。前 6 章是"为什么"，第 7 章 + 附录是"怎么做"。

---

## 0. 引子:200k context 的假性安全感

"模型都 200k context window 了,还需要记忆系统吗?"

现实教育了我们：**每次醒来全新**。Agent 没有连续意识。昨天做了什么决策、上周约定了什么规则、三天前踩了什么坑——如果没有写在文件里、没有在恰当的时候被搜到,**等于不存在**。

200k context 解决的是"当前对话能看多远",不是"跨 session 能记住什么"。更隐蔽的是 **compaction**（上下文压缩）——对话超长触发压缩后,agent 会选择性丢失早期信息。我们亲眼看到 agent 在 compaction 后忘掉"正在执行的任务有什么约束",然后做出违反约定的操作。

**没有记忆系统的代价**：重复犯同一个错误（第三次踩同一个坑时你会失去耐心）、每次对话花 10 分钟恢复上下文（人类充当记忆层）、无法积累（agent 永远是聪明的新人,不是靠谱的老伙计）。

记忆系统要解决的核心问题：**让 agent 从"每次醒来全新"变成"每次醒来带着该知道的东西"。**

---

## 1. Agent 记忆 ≠ 人类记忆

### 认知框架纠偏

人类记忆是连续的、有情感权重的、会自动衰减的、靠联想检索的。Agent 的"记忆"本质上是文件读写 + 搜索算法：

| 维度 | 人类 | Agent |
|------|------|-------|
| 连续性 | 醒着时持续运行 | 每个 turn 独立,无延续 |
| 写入 | 自动(经历即记录) | 必须显式写文件 |
| 检索 | 联想、情境触发 | 搜索算法命中才有 |
| 遗忘 | 自然衰减 | 没写=不存在,写了不搜=不存在 |
| 修正 | 经验自动修正行为 | 无跨轮修正机制 |

### Agent 记忆的三个真问题

1. **Timing（什么时候写）**：agent 不会"自然记住"。你告诉它写,它说"好的",然后实际不写——因为没有强制机制。
2. **Retrieval（写了能否被找到）**：文件里有 ≠ 能用。搜索算法的命中率决定了记忆是否"存在"。
3. **Freshness（找到后能否信任）**：三个月前的决策可能已过时。过时信息比没有信息更危险。

### 行业方案对比

| 方案 | 核心思路 | 适合场景 | 不太适合 |
|------|----------|----------|----------|
| **MemGPT / Letta** | 虚拟分页——模拟 OS 内存管理，自动在 main context / archival storage 之间搬运信息 | 需要精细控制 context 窗口的应用；长对话中需要自动"翻页"历史 | 运营者想直接看到/编辑记忆内容的场景；简单的个人助手 |
| **Mem0** | Memory layer + graph memory，提供跨 session 的结构化记忆存储和关系推理 | 多用户产品（per-user 记忆隔离）；需要实体关系推理的应用 | 单人单 agent 轻量场景；不需要关系图的工作型助手 |
| **LangChain Memory** | ConversationBufferMemory / Summary 等模块，在 chain 内管理对话历史 | 快速原型；框架内短 session 的上下文延续 | 跨 session 持久记忆；需要独立于框架运行的场景 |
| **claude-mem** | 零配置全自动——捕获 session 上下文，AI 压缩为语义摘要，下次启动时注入（79k+ ★） | 不想花精力搭记忆体系的个人用户；Claude Code / CLI 用户想快速见效 | 需要人参与质量把关；对记忆内容有治理要求（门槛/晋升/过期） |
| **TencentDB Agent Memory** | 四层渐进式管线（L0 对话 → L1 结构化 → L2 场景块 → L3 画像），默认本地 SQLite，有 OpenClaw 官方插件 | 需要自动化提取和分层索引的应用；想在检索层开箱即用 | 需要自定义治理规则（晋升门槛/防御纵深）；已有完整记忆体系想保持控制 |
| **本文方案** | 纯文件（Markdown）+ 语义触发写入 + 搜索检索 + 晋升门槛 + 人参与维护 | 一人一虾长期运营；需要积累经验的工作型 agent；运营者愿意深度参与 | 纯一次性对话；运营者不愿花时间维护；需要零配置开箱即用 |

**几点说明**：

- 以上方案解决的问题层次不同。有的侧重 context 管理（当前对话看得远），有的侧重持久记忆（跨 session 记得住），有的侧重治理（记忆质量可控）。不存在"最好的"，只有"适合你场景的"。
- claude-mem 和 TencentDB Agent Memory 我们没有实测。列在这里是基于文档和社区反馈的理解，供参考。
- 本文方案的核心假设是"运营者愿意参与"。如果这个前提不成立，零配置方案（如 claude-mem）可能是更务实的选择。


## 2. 这套系统适合谁

### 适合的场景

- **一人一虾的长期陪伴**：个人助手、项目管理——你和同一个 agent 跨多个 session 长期合作
- **需要积累经验的工作型 agent**：不是一问一答,而是跑多个项目、踩多个坑、逐步变靠谱
- **文件系统 + 搜索架构**：OpenClaw 或类似的"workspace 文件 + memory_search"模式
- **运营者愿意花前两周"养"出初始记忆**
- **对"不重复犯同一个错误"有真实需求**

### 不太适合的场景

- 纯一次性对话（客服/问答/代码生成器）
- 多用户共享一个 agent（需要 per-user 隔离）
- 对延迟极度敏感（搜索 + bootstrap 有 token 开销）
- 运营者不愿参与维护（月度 review、经验晋升需要人参与）

## 3. 架构演进:三步解决问题

### 第一步：搞定写入

起点认知：**不在 AI 生态之外搞知识管理。** 飞书/Notion 整理得再漂亮,AI 检索不到就没用。

建了四层结构 + 两个保底：

```
Session Context（当前对话）
    ↓ 语义触发写入（确认/完成/切换/读完）
Daily Log（memory/YYYY-MM-DD.md）
    ↓ 每周蒸馏
MEMORY.md（长期记忆，bootstrap 无条件注入）
    ↑
Project Cards（notes/projects/*.md）

兜底：HEARTBEAT.md（每 15 分钟检查是否有未记录的操作）
```

**机制驱动，非自觉驱动**：有具体触发条件（确认时写/完成时写/切换时写）+ 心跳兜底（定时检查日志是否为空）。日志写入率显著高于纯指令式的"请记得写日志"。

**骨架能跑后的短板**：搜索裸配（结果重复）、MEMORY.md 无门槛（快爆）、恢复任务不检查约束、错误经验无积累通道。

### 第二步：补精度 + 门槛 + 防御

**错误记录 + 晋升路径**：新建 `.learnings/`，操作失败或被纠正时立即写入。晋升通道：`.learnings/ → 复用 2 次 → 项目卡 → 跨项目验证 → MEMORY.md`。MEMORY.md 是无条件注入 bootstrap 的，token 是硬通货——一条垃圾占的位置 = 一条关键认知的缺席。

**三层纵深防御**：预防（写任务时写约束）→ 侦测（恢复时搜约束）→ 兜底（执行前当场确认）。Agent 没有"小心翼翼"的状态,单点防御总有漏网,三层叠加才可靠。

**反转默认值**：从"记得做X"改成"默认做X，以下情况可跳过"。Agent 不存在"习惯养成"——每个 turn 都是第一次。写成"默认执行，除非命中排除条件"，命中率从 ~60% 跳到 ~95%。这个原则贯穿记忆系统每一层。

### 第三步：搜索精度 + 认知框架成型

**检索精度提升**：开启 MMR（lambda=0.7）消除同文件多段重复，搜索结果从"4 条里 2 条重复"变成"4 条覆盖 4 个视角"。temporalDecay 暂不开——我们经常需要搜三个月前的决策。

**Dreaming：重要弯路**。OpenClaw 内置 Dreaming 机制（凌晨三阶段：Light → REM → Deep），从 recall tracking 找高频内容自动晋升。我们开了，第一次结果：格式不可用——日志原文 20+ 行平铺成一行 raw dump，不做摘要，4 条晋升中 3 条高度重叠。

**关键认知**：Dreaming 是好的**选择器**（基于真实搜索行为打分），不是好的 **writer**。决策：关闭自动写入，保留 recall tracking 数据（275+ 条）。正确串联：Dreaming 选 → 蒸馏 cron 读候选 → LLM 摘要 → 人工确认 → 写入。

**认知框架成型**：agent 记忆的真问题是 timing/retrieval/freshness；每条规则必须对应 OpenClaw 真实机制点（turn 结束 / memory_search / cron / heartbeat）——"看似合理但不可检测"的规则必须剔除。

---

## 4. 当前系统全景

经过三个月演化,最终形态是五层：

### 存储层

```
MEMORY.md              - 长期认知（bootstrap 无条件注入，≤12000 chars）
memory/YYYY-MM-DD.md   - 每日事实日志
memory/references/     - 检索辅助（术语表、操作手册）
notes/projects/*.md    - 项目卡（单项目单一权威源）
.learnings/            - 错误记录与经验积累
```

纯 Markdown 文件，零依赖，未来迁移零成本。

### 写入层

| 触发条件 | 写入目标 | 机制 |
|----------|----------|------|
| 确认/完成/切换/读完 | Daily Log | 语义触发（AGENTS.md 规则） |
| 操作失败/被纠正 | .learnings/ | 即时写入 |
| 项目阶段变化 | Project Card | 任务完成时检查 |
| 每 15 分钟 | Daily Log | 心跳兜底（检查 lastLogWrite） |
| Cron 每天 3:00 | Daily Log | 二次兜底（检查日志是否为空） |
| 事实修正 | MEMORY.md | 联动更新（日志写入时发现旧条目过期） |

心跳支持去重机制：连续无实质对话时自动 skip，避免写入空洞的"无事发生"条目。

### 检索层

```
memory_search（BM25 + 向量混合检索）
├── MMR 去重（lambda=0.7）
├── 索引范围：memory/ + notes/projects/
└── MEMORY.md 也可被搜到（但主要靠 bootstrap）

检索路由规则（AGENTS.md）
├── 最近发生了什么 → 读今天+昨天日志
├── 项目状态 → 读对应项目卡
├── 长期规则 → MEMORY.md
└── 精确名词 → grep 补充搜索
```

Benchmark：25 组真实查询，84% 命中率。弱项：抽象方法论 + 口语化问句。

### 晋升层

```
.learnings/ → 不同场景复用 2 次 → Project Card "可晋升经验"
    → 跨项目验证 → MEMORY.md

Daily Log → 每周蒸馏 cron → recall tracking 高分候选 → 人工审核 → MEMORY.md
```

**MEMORY.md 准入标准**：三个月后仍有效 / 是认知不是事实 / 不放状态信息只放定义信息。

### 维护层

| 机制 | 频率 | 作用 |
|------|------|------|
| 心跳检查 | 每 15 分钟（去重后实际更低） | 实时兜底 |
| 日志兜底 cron | 每天 03:00 | 二次兜底 |
| 每周蒸馏 cron | 周日 03:30 | 7 天日志 → MEMORY.md 候选 |
| MEMORY.md 月度 review | 手动 | 删除过期条目（git 有历史） |
| Recall tracking | 持续 | 记录搜索行为供分析 |
| self-improvement hook | 每轮 | bootstrap 注入提醒记录错误 |

---

## 5. 设计原则:踩出来的规则

### 原则一："记住 = 写文件"

Agent 说"我记住了"是空头支票。没写文件 = 下个 session 不存在。延伸：agent 说"我会注意""下次我先检查"也是空头承诺——它没有跨轮修正机制，不存在"行为改善"。要改行为，改规则文件。

### 原则二：每条规则必须对应真实机制

"请注意保持日志质量"——无效。有效的规则：

> 以下情况**立即**追加写入 `memory/YYYY-MM-DD.md`：确认了某件事 / 完成了实质性操作 / 话题切换

每条规则要映射到运行时的可检测条件。检验方法：问"这条规则在运行时能被触发吗？触发机制是什么？"过不了的，删。

### 原则三：反转默认值

```
// 旧模式（记得做 → 经常忘）
"做完操作后,如果重要,请写日志"

// 新模式（默认做 → 排除条件才跳过）
"做完操作后写日志。可以跳过的例外:纯读取操作、用户明确说不用记"
```

核心洞察：agent 没有"习惯养成"。"默认做"让 agent 只需判断"是否命中排除条件"，比判断"是否值得做"容易得多。适用于一切"agent 应该做但经常忘"的行为。

### 原则四：MEMORY.md 是大脑皮层，不是文件索引

写认知而非条目——"我理解了什么"，不是"有什么文件"。

```markdown
# 错误示范
- scripts/feishu-send-file.sh：发文件脚本

# 正确示范
- 有工具不用、有文档不读、靠猜是所有事故的根因
- 上下文窗口是硬通货。SubAgent 定界(少工具=更聚焦=更稳定)
```

判断标准："做相关操作前，会不会自然先搜？"——会搜的放在可搜索文件里；不会搜的（行为约束/操作前提）必须留在 bootstrap。

### 原则五：轻量原则

不建额外索引、不加管理机制、保持纯 Markdown。复杂性是记忆系统的敌人。系统退化时应有明显信号（日志为空、搜索无命中），不需要额外监控。

---

## 6. 实战数据

### 搜索质量

| 指标 | 数值 |
|------|------|
| 测试集 | 25 组真实查询（从实际对话中抽取） |
| 命中率 | 84%（21/25 首页命中目标内容） |
| 引擎 | BM25 + 向量混合，MMR lambda=0.7 |
| 弱项 | 抽象方法论 + 口语化问句 |

### 系统规模（90 天后）

| 指标 | 数值 |
|------|------|
| MEMORY.md | 11618 / 12000 chars（97% 使用率） |
| .learnings | 20+ 条 ERR 记录（含已归档至 archive/） |
| Recall Store | 持续积累中（recall tracking 数据） |
| Daily Log | 近 30 天每天有写入 |
| 存储总量 | ~3 MB（所有记忆文件，含 90+ 天日志） |

### 坦白弱点

1. **抽象查询命中率低**：搜"怎么处理多 agent 记忆共享"这类,向量模型匹配不好
2. **蒸馏质量依赖模型能力**：弱模型（doubao/glm）的蒸馏产出质量明显低于 Opus/GPT-5.5
3. **Dreaming 写入质量不可用**：自动晋升的格式是 raw dump，必须人工/LLM 再加工
4. **过期检测无自动化**：目前靠月度人工 review
5. **心跳兜底有盲区**：agent 处理用户消息时心跳被 skip（去重机制已部分缓解——连续无实质对话时自动跳过，但高峰时段仍可能低于标称频率）

---

## 7. 动手指南：Setup Guide

以下是一套最小可用的记忆系统配置。适用于 OpenClaw 5.x，任何模型。

> **给 Agent 的说明**：如果你正在帮管虾人配置记忆系统，按顺序执行。每步末尾有验证方法——通过再进入下一步。

### 7.0 前置条件

```bash
# 确认 OpenClaw 版本 >= 5.x
openclaw --version

# 确认 memory_search 可用
openclaw config get agents.defaults.memorySearch.provider
# 应返回 provider ID（如 "openai"/"local"/"ollama"）
# 返回空或报错 → 先按附录 B 配置 embedding provider

# 确认 workspace 目录
openclaw config get agents.defaults.workspace
# 记下路径，后续操作在此目录下

# 确认 cron 可用
openclaw cron list
# 返回 job 列表（空列表也算通过）
```

**验证**：4 条命令均无报错 → 进入 7.1。

### 7.1 文件结构

```bash
cd <你的workspace路径>

mkdir -p memory/references notes/projects .learnings
touch MEMORY.md HEARTBEAT.md
touch .learnings/ERRORS.md .learnings/LEARNINGS.md
echo '{"lastLogWrite": null}' > memory/heartbeat-state.json
```

最终结构：

```
workspace/
├── AGENTS.md          ← 行为规则（含记忆规则）
├── MEMORY.md          ← 长期记忆
├── HEARTBEAT.md       ← 心跳兜底规则
├── memory/
│   ├── YYYY-MM-DD.md  ← 每日日志（运行时自动创建）
│   ├── references/    ← 检索辅助文件
│   └── heartbeat-state.json
├── notes/projects/    ← 项目卡
└── .learnings/
    ├── ERRORS.md      ← 错误记录
    └── LEARNINGS.md   ← 有效方法
```

**验证**：`ls memory/ .learnings/` 显示对应文件 → 进入 7.2。

### 7.2 AGENTS.md 记忆规则模板

将以下内容加入 AGENTS.md（已有内容则追加到末尾）：

```markdown
## 记忆系统

### 写入规则（语义触发）

以下情况**立即**追加写入 `memory/YYYY-MM-DD.md`：
- 确认了某件事（"好"/"ok"/"那我们去做X"）
- 完成了一个实质性操作
- 话题明显切换到新方向
- 读完有实质内容的文章/文档

**写日志标准**：具体 > 精简。专有名词、工具名、路径、版本号只要出现就写。
- 已完成：标注"已完成，结论：X"
- 进行中：必须写明"已知约束：X"（无约束也显式写"无约束"）
- 修正：写"修正：之前X有误，正确是Y"

写完后更新 `memory/heartbeat-state.json` 中的 `lastLogWrite`。

### "记住 = 写文件"

不要说"我记住了"或"我会注意"。没写 = 不存在。

### 检索规则

| 问题类型 | 先读哪里 |
|---|---|
| 最近发生了什么 | memory/日志（今天+昨天） |
| 项目状态 | notes/projects/对应项目卡 |
| 长期规则/偏好 | MEMORY.md |
| 精确名词 | grep 搜索 |

### 长期记忆写入门槛

新经验 → .learnings/（立即写）→ 不同场景复用 2 次 → MEMORY.md
MEMORY.md 只写：三个月后仍有效的认知。

### 进行中任务恢复（三层防御）

- **预防**：写任务时同条目写明约束
- **侦测**：高风险 → 先 memory_search 搜约束
- **兜底**：不可逆操作前当场确认

### 错误记录（反转默认值）

操作失败或被纠正时**当场**写 `.learnings/ERRORS.md`。
执行操作前默认查 `.learnings/`（grep 关键词）。
命中失败记录 → 硬阻断，换路。
```

**验证**：`grep "语义触发" AGENTS.md` 有输出 → 进入 7.3。

### 7.3 HEARTBEAT.md

```markdown
# Heartbeat

收到心跳时执行：

1. 检查 `memory/heartbeat-state.json` 的 `lastLogWrite`
2. 距今超过 30 分钟且本轮有实质对话：
   - 从当前 session 上下文提取关键事件
   - 追加写入 `memory/YYYY-MM-DD.md`
   - 更新 `lastLogWrite`
3. 无实质对话：跳过

注意：心跳在 agent 处理消息时会被 skip。不依赖心跳作为唯一兜底。
```

**验证**：`cat HEARTBEAT.md | head -3` 显示内容 → 进入 7.4。

### 7.4 MEMORY.md 初始模板

```markdown
# MEMORY.md - 长期记忆

> 写入规则：只写三个月后仍有效的事实、决策、偏好、认知。
> 认知 vs 事实：事实放日志，认知放这里。
> 清理规则：每月 review，过期条目删除（git 有历史）。

---

## 认知与方法论

（随使用积累填充）

## 关键决策

（重要的不可逆决策，含日期和原因）
```

**验证**：`cat MEMORY.md` 显示模板 → 进入 7.5。

### 7.5 Cron 配置

两个核心 cron job：

**日志兜底（每天 03:00）**：

```json5
{
  name: "日志兜底检查",
  schedule: { kind: "cron", expr: "0 3 * * *", tz: "Asia/Shanghai" },
  sessionTarget: "isolated",
  payload: {
    kind: "agentTurn",
    message: "检查昨天和今天的日志文件。为空则从 session 历史补录实质性操作和决策。补完输出摘要，无需补录则 NO_REPLY。"
  },
  delivery: { mode: "announce" }
}
```

**每周蒸馏（周日 03:30）**：

```json5
{
  name: "每周记忆蒸馏",
  schedule: { kind: "cron", expr: "30 3 * * 0", tz: "Asia/Shanghai" },
  sessionTarget: "isolated",
  payload: {
    kind: "agentTurn",
    message: "读取近 7 天日志，提取值得长期保留的认知写入 MEMORY.md。标准：三个月后仍有效、是认知不是事实、不重复已有条目。写入前检查字符数不超 12000。完成后输出蒸馏摘要，无新增则 NO_REPLY。"
  },
  delivery: { mode: "announce" }
}
```

**注意**：`tz` 替换为你的时区。`NO_REPLY` 抑制推送。

**验证**：`openclaw cron list` 显示两个新 job → 进入 7.6。

### 7.6 搜索配置

```bash
# 启用 MMR 去重
openclaw config set agents.defaults.memorySearch.query.hybrid.mmr.enabled true
openclaw config set agents.defaults.memorySearch.query.hybrid.mmr.lambda 0.7

# 项目卡加入搜索索引
openclaw config set agents.defaults.memorySearch.extraPaths '["notes/projects"]'
```

**验证**：`openclaw config get agents.defaults.memorySearch.query.hybrid.mmr` 显示 `{ enabled: true, lambda: 0.7 }` → 进入 7.7。

### 7.7 self-improvement hook（可选，推荐）

```bash
openclaw config set hooks.internal.entries.self-improvement.enabled true
```

每次 agent 启动时注入提醒，促使操作失败时记录到 `.learnings/`。零成本立即生效。

**验证**：config get 返回 true → 进入 7.8。

### 7.8 端到端验证

**立即可验证：**

- [ ] `ls memory/ .learnings/` — 文件结构存在
- [ ] `grep "语义触发" AGENTS.md` — 记忆规则已写入
- [ ] `openclaw cron list` — 显示 2 个 job
- [ ] MMR 配置返回 true

**使用中验证（需实际对话后检查）：**

- [ ] 对话完成后 `memory/` 下有当天日志
- [ ] 日志包含具体名词/路径
- [ ] `memory_search "关键词"` 能命中
- [ ] 纠正 agent 后 `.learnings/` 有记录
- [ ] 次日 03:00 后 cron 执行了兜底
- [ ] 新 session 开始时 agent 能引用上次结论

**全部通过 = 配置完成。** 在使用中参考第 5 章原则逐步调优。

---

## 8. 未开的门

### 因果溯源

recall 失败时（搜了没命中/命中错误内容），记录"为什么搜不到"。积累样本后分析——是 embedding 问题、文件组织问题、还是写入措辞问题。方向已确认，尚未执行。

### 量化评估自动化

当前 benchmark 是手动 25 条。目标：扩展到 50+、按类型分桶、定期自动跑 + 趋势追踪、质量下降时告警。

### 多 Agent 记忆共享

当前完全隔离（每个 agent 自己的 workspace）。未解决：Agent A 的教训 Agent B 会重复犯；共享可能泄露隐私；"经验包"是部分解法（我们的筑基包即此思路）。

### 过期自动检测

理想状态：写入时标注有效期、到期提醒 review、或基于 recall tracking 标记 90 天未被搜索的内容为候选移除。

---

## 尾声

记忆系统的演进不是"有没有"的问题——搞个四层存储 + 写入触发，几天就能搭起来。真正难的是可靠性：**搜得到、信得过、防得住。**

检索精度从黑盒到 84% 可量化，长期记忆从无门槛到层层过滤，执行安全从零防御到三层纵深——这些是 90 天踩出来的，不是设计出来的。

最后一个忠告：**不要追求完美的记忆系统，追求"agent 不再重复犯同一个错误"。** 当这个目标达成时，你的记忆系统就及格了。

---

## 附录 A：快速开始（给 Agent 的入口）

> **如果你是 agent，管虾人让你"照这个配"——从这里开始。**

### 30 秒理解

基于文件系统的 agent 记忆方案：
- Markdown 文件存储所有记忆（日志/长期认知/错误记录）
- 语义触发 + cron 兜底 → "写得进去"
- BM25 + 向量混合搜索 → "搜得到"
- 晋升门槛 + 月度清理 → "信得过"

### 执行顺序

```
Step 0: 前置条件（7.0）
Step 1: 文件结构（7.1）
Step 2: AGENTS.md 记忆规则（7.2）
Step 3: HEARTBEAT.md（7.3）
Step 4: MEMORY.md 初始化（7.4）
Step 5: Cron 配置（7.5）
Step 6: 搜索配置（7.6）
Step 7: self-improvement hook（7.7，可选）
Step 8: 验证（7.8）
```

### 遇到问题查哪里

| 问题 | 参考 |
|------|------|
| 为什么这样设计 | 第 1-2 章 |
| 具体怎么配 | 第 7 章 |
| 搜索不准/结果重复 | 7.6 + 第 6 章 |
| 怎么维护/清理 | 第 4 章维护层 + 第 5 章原则五 |
| 弱点/已知局限 | 第 6 章"坦白弱点" |

---

## 附录 B：前置条件与依赖

### 必须项

| 依赖 | 要求 | 说明 |
|------|------|------|
| OpenClaw | 5.x+ | memory_search 从 5.x 开始内置 |
| Embedding 服务 | 任意 text-embedding API | 无 embedding 时退化为纯 BM25（降级模式） |
| Cron | OpenClaw 内置 | 无额外依赖 |
| 文件系统 | workspace 可读写 | 所有记忆为 Markdown 文件 |

### Embedding 服务选型

OpenClaw 内置支持（无需插件）：

| Provider | 需要 Key | 备注 |
|----------|----------|------|
| openai | 是 | **默认**，text-embedding-3-small 即可 |
| local | 否 | GGUF 模型 ~0.6 GB，完全离线 |
| ollama | 否 | 需要 Ollama 服务运行 |
| gemini | 是 | 支持多模态索引 |
| voyage | 是 | 高质量向量 |
| openai-compatible | 通常是 | 通用 `/v1/embeddings` 接口 |
| deepinfra | 是 | 默认 BAAI/bge-m3 |
| bedrock | 否（AWS 凭证链） | AWS 环境直接用 |
| github-copilot | 否（Copilot 订阅） | 有 Copilot 就能用 |
| mistral | 是 | — |

**推荐**：最简单用 `openai`；完全免费用 `local`；有自建 Ollama 用 `ollama`。

**配置示例（OpenAI）**：

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai"
        // API key 从 models.providers.openai.apiKey 或 OPENAI_API_KEY 读取
      }
    }
  }
}
```

**无 embedding 的降级模式**：memory_search 退化为纯 BM25 关键词搜索。对精确名词（错误码、文件名）命中率不错，对语义相似但措辞不同的内容命中率显著下降。

### 可选组件

| 组件 | 作用 | 推荐度 |
|------|------|--------|
| self-improvement hook | 每轮 bootstrap 注入提醒记录错误 | ⭐ 推荐 |
| git 版本控制 workspace | 记忆变更可回溯 + 误删可恢复 | ⭐ 推荐 |
| Dreaming（Light + REM） | recall tracking + 高频内容打分 | 可开（Deep 自动写入建议关） |
| memory-wiki（bridge 模式） | Markdown → 结构化知识库 | 观望 |

### 硬件/成本

- **最低配置**：任何能运行 OpenClaw 的机器（Mac/Linux/WSL，Node.js 22+）
- **存储**：记忆文件 < 1 MB（90 天后约 200-500 KB）
- **API 成本**：Embedding 调用几乎免费（OpenAI ~$0.02/M tokens）
- **可以完全免费**：`local` provider + 免费模型
