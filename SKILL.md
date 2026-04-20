---
name: smart-reference
description: 统一参考检索入口。当用户说"看看大神怎么做""参考一下""有什么好方案""best practice""去GitHub找找""网上有没有""去网上查一下""上网查一下""搜一下""查一查""网上搜搜""有没有现成的""有轮子吗""官方文档怎么说""社区怎么看""有没有人踩过坑"时触发。自动选择检索通道，并行搜索，结果直接融入当前工作建议。不适用于：用户已有明确实现方案只需要执行、纯代码调试（无需外部参考）、用户明确说"不用查了直接做"。
---

# Smart Reference — 统一外部参考检索

<Use_When>
- 用户说"去网上查""上网查一下""搜一下""查一查"
- 用户说"参考一下""找参考""有什么参考""有什么好方案"
- 用户说"best practice""最佳实践""行业标准做法"
- 用户说"看看大神怎么做""Karpathy 怎么做"
- 用户说"去 GitHub 找找""有没有现成的""有轮子吗"
- 用户说"官方文档怎么说""官方推荐"
- 用户说"社区怎么看""有没有人踩过坑"
</Use_When>

<Do_Not_Use_When>
- 用户已有明确方案，只需执行（直接做，不用检索）
- 纯代码调试，无需外部参考
- 用户明确说"不用查了""直接做"
- 话题极其简单，官方文档一眼就能查到
</Do_Not_Use_When>

## 执行流程

### Step 0: 命中缓存检查（优先）

**立即用 Bash 工具执行以下命令，不等用户输入，不询问，直接运行：**

```bash
mkdir -p ~/.claude/reference-cache/reports && ls ~/.claude/reference-cache/index.json 2>/dev/null && cat ~/.claude/reference-cache/index.json || echo "NO_CACHE"
```

```
缓存路径: ~/.claude/reference-cache/
索引文件: ~/.claude/reference-cache/index.json
报告目录: ~/.claude/reference-cache/reports/
```

1. 从当前对话中提取 3-5 个核心关键词
2. 在 `index.json` 中做关键词匹配（命中 ≥ 70% 关键词视为相关）
3. 根据命中结果，**无需用户确认，直接执行**：

| 情况 | 行动 |
|------|------|
| 命中 + 未过期 | 直接读取报告，输出时标注"缓存报告，生成于 X 天前"，不询问用户 |
| 命中 + 已过期 | 自动进入 Step 1 重新检索；完成后按"增补逻辑"追加到旧报告末尾（不覆盖），并更新 index.json 中该条目的 `expires_at` 和 `version` 字段，在报告末尾注明"已增补，替换 X 天前旧版" |
| 未命中 / 缓存目录不存在 | 立即进入 Step 1，不等待用户，不询问 |

---

### Step 1: 意图解析 + 路由

分析当前任务上下文，确定：
1. **核心话题**: 用户在问什么（提取 3-5 个关键词）
2. **技术栈**: 涉及哪个框架/语言/工具
3. **意图类型**: 下表路由

| 意图类型 | 启用通道 |
|---------|---------|
| "怎么实现 X" / "有没有轮子" | 通道 1 (GitHub) + 通道 5 (代码范例) + 通道 2 (官方文档) |
| "best practice" / "最佳实践" | 通道 3 (大神博客) + 通道 4 (社区讨论) + 通道 2 (官方文档) |
| "大神怎么做" / "Karpathy/XXX 怎么做" | 通道 3 (大神博客) + 通道 1 (GitHub) |
| "有什么好方案" / "方案选型" | 通道 1 (GitHub) + 通道 4 (社区讨论) + 通道 5 (代码范例) |
| "文档怎么说" / "官方推荐" | 通道 2 (官方文档) [专注单通道] |
| 涉及协议 / 安全 / 规范 | 通道 6 (行业标准) + 通道 2 (官方文档) |

---

### Step 2: 并行检索（Agent 并行）

启用的通道用 Agent 并行执行，最多同时跑 4 个通道。每个通道内部用 `document-specialist` subagent 完成实际搜索。

**执行细节**:
- 每个通道返回结果格式：`{通道名, 来源列表, 核心摘要, 可信度等级}`
- 单通道超时上限：60 秒。超时视为"无结果"，不阻塞其他通道
- 某通道返回 0 结果：记录"通道 X 无结果"，继续融合其余通道
- 所有通道均无结果：告知用户"未找到有效参考，建议调整关键词后重试"，不凭空生成建议
- 结果收齐后进入融合，不等超时通道

---

## 6 条检索通道详细规范

### 通道 1: GitHub 高星项目 + 源码

**星数阈值（按话题类型）**:

| 话题类型 | 最低 stars |
|---------|-----------|
| 主流框架/工具（React、Pandas、LangChain） | ≥ 5000 |
| 垂直领域/细分工具（MCP、Agent 框架、游戏插件） | ≥ 1000 |
| 新技术（近 6 个月出现） | ≥ 1000，且近期有 commit |
| Awesome 列表 | ≥ 1000 |

增速过滤：过去 30 天 new stars / total stars > 5% 的项目，即便绝对值接近下限也值得纳入。

**搜索策略**:
```
gh search repos "{关键词}" --sort=stars --limit=10
→ 过滤：stars 达标 + 12 个月内有更新 + 有 README
→ 读取：README（方案思路）、核心源码结构、Issues/Discussions（踩坑经验）
```

**输出**: 项目名 + stars + 方案概要 + 关键实现细节 + 可借鉴的模式

---

### 通道 2: 官方文档 + API Reference

按当前任务锁定对应官方域名：

| 领域 | 官方站点 |
|------|---------|
| Claude / Anthropic | docs.anthropic.com, github.com/anthropics |
| OpenAI | platform.openai.com/docs |
| Python 标准库 | docs.python.org |
| Pandas | pandas.pydata.org/docs |
| Web (MDN) | developer.mozilla.org |
| Node.js | nodejs.org/docs |
| 框架文档 | 根据上下文动态识别 |

**搜索策略**:
```
WebSearch: site:{官方域名} {关键词}
→ WebFetch 读取命中页面的正文
→ 优先取版本说明、API 参考、官方示例
```

**输出**: 官方推荐做法 + 最新 API 用法 + 版本差异注意事项

---

### 通道 3: 知名开发者博客 / 视频

> 完整信息源列表见 `references/masters.md`（按领域分类，30+ 人）。**仅在 Step 2 启动通道 3 Agent 时读取该文件**，不在 Step 1 路由阶段读取（避免未启用通道 3 时的无效 IO）。
> 若 `references/masters.md` 读取失败，使用以下降级域名直接搜索：
> - AI/LLM：`site:karpathy.github.io`、`site:lilianweng.github.io`、`site:simonwillison.net`
> - 前端：`site:overreacted.io`、`site:joshwcomeau.com`
> - 系统：`site:jvns.ca`、`site:martinfowler.com`

**搜索策略**:
```
WebSearch: "{大神名}" "{关键词}" site:{大神博客域名}
OR: "{大神名}" "{关键词}" 近12个月
→ 优先取原创文章/视频，过滤转载和摘要
```

**输出**: 大神的具体做法/观点 + 原始链接 + 发布时间

---

### 通道 4: 社区讨论

**英文信息源**:
- Hacker News — `site:news.ycombinator.com {关键词}`（高质量技术讨论）
- Reddit:
  - `site:reddit.com/r/MachineLearning {关键词}`
  - `site:reddit.com/r/LocalLLaMA {关键词}`（本地模型实战）
  - `site:reddit.com/r/ClaudeAI {关键词}`
  - `site:reddit.com/r/programming {关键词}`
  - `site:reddit.com/r/webdev {关键词}`
  - `site:reddit.com/r/gamedev {关键词}`（游戏相关）
- Stack Overflow — `site:stackoverflow.com {关键词}`（具体技术问题高票解答）

**中文高质量信息源**:
- 美团技术团队 — `site:tech.meituan.com {关键词}`
- 字节跳动技术 — `site:blog.bytedance.com {关键词}` / 字节 ByteTech
- 阿里云开发者 — `site:developer.aliyun.com/article {关键词}`
- 知乎高赞 — `site:zhihu.com {关键词}`（**仅取赞数 > 500 的专业回答**）
- 陈皓博客 — `site:coolshell.cn {关键词}`（存档，永久有效）
- 量子位 / 机器之心 — 仅深度报道，不取资讯速报
- 游资网 — `site:youxi.com {关键词}`（游戏运营垂直）

**过滤条件**:
- 近 12 个月内容（快速迭代话题近 3 个月）
- 有实质内容（正文 > 150 字）
- 高票 / 高赞（量化指标优先）
- 过滤纯提问帖、无实质内容的短回复

**输出**: 社区共识 + 争议点 + 实战踩坑经验 + 高票建议

---

### 通道 5: 代码范例（Cookbook / Examples）

**目标资源**:
- Anthropic Cookbook — github.com/anthropics/anthropic-cookbook
- OpenAI Cookbook — github.com/openai/openai-cookbook
- 框架官方 `/examples` 目录
- `awesome-{主题}` 精选列表（stars ≥ 1000）
- Kaggle Notebooks（数据分析/ML 完整 notebook）
- CodeSandbox / CodePen（前端组件可运行示例）

**搜索策略**:
```
gh search repos "awesome-{主题}" --sort=stars
gh search code "{关键词}" --extension=.py/.js/.ts --limit=5
WebSearch: "{框架} cookbook site:github.com"
WebSearch: "{框架} examples tutorial"
```

**输出**: 可直接参考的代码片段 + 使用场景说明 + 完整示例链接

---

### 通道 6: 行业标准 / RFC / 规范（按需触发）

**触发条件**: 话题涉及协议、格式、安全、合规、Web 标准时触发。

| 类型 | 来源 |
|------|------|
| 网络协议 | IETF RFC — rfc-editor.org |
| Web 标准 | W3C — w3.org，WHATWG — whatwg.org |
| 安全 | OWASP — owasp.org，CWE — cwe.mitre.org |
| 数据格式 | JSON Schema，OpenAPI Spec |
| AI / ML 学术 | arxiv.org（仅高引用论文，> 500 citations 或被广泛讨论的） |

**输出**: 规范要求 + 版本说明 + 合规注意事项

---

## 结果融合策略

### 可信度评级

| 级别 | 来源 |
|------|------|
| S 级 | 官方文档 / 框架作者原话 / RFC 规范 |
| A 级 | 通道 3 大神博客 / GitHub stars ≥ 5000 / 大厂技术博客 |
| B 级 | GitHub stars ≥ 1000 / HN 高票 / 知乎 500+ 赞 |
| C 级 | 其余社区讨论（作为佐证，不作主要依据） |

### 融合输出原则

1. **结论必须说明来源 + 为什么** — 每个建议后明确：①来自哪个通道/具体项目/文章，②为什么选这个方案（相比其他选项的优势或适用原因）
2. **按可信度排序** — S/A 级结论优先
3. **去噪去重** — 合并指向同一方案的多个来源时，说明"X 个来源均指向同一结论"
4. **融入当前任务**:
   - 正在写代码 → 给出参考后的实现建议
   - 方案选型 → 给出对比表格 + 推荐理由
   - 调试问题 → 给出别人遇到同样问题的解法及来源
   - 学习场景 → 给出学习路径和推荐资源
5. **注明来源和时间** — 每个关键结论后括注来源名称、可信度等级、发布/更新时间

---

## 本地缓存协议

### 存储结构

```
~/.claude/reference-cache/
  index.json          ← 元数据索引
  reports/
    {sha1_hash}.md    ← 具体报告
```

`index.json` 单条格式：
```json
{
  "topic": "Claude streaming response 实现",
  "keywords": ["streaming", "claude", "SSE", "实现"],
  "file": "reports/a1b2c3.md",
  "created_at": "2026-04-19T10:00:00Z",
  "channels_used": ["github", "official_docs", "community"],
  "ttl_days": 1,
  "expires_at": "2026-04-20T10:00:00Z",
  "version": 1
}
```

### TTL 策略（时效性）

| 话题类型 | TTL | 判断依据 |
|---------|-----|---------|
| 快速迭代（LLM、新兴 AI 工具、近期框架） | **1 天** | AI 工具日更，隔天报告大概率过时 |
| 成熟框架（React、Pandas、Django） | **30 天** | 稳定但每月有版本迭代 |
| 工程实践 / 设计模式 / 架构 | **60 天** | 基本稳定，可长期参考 |
| 行业标准 / RFC / 规范 | **180 天** | 标准几乎不变，半年一次检查即可 |

**如何判断话题类型**: 
- 标题或关键词含 Claude/GPT/Gemini/LLM/Agent/MCP → 快速迭代
- 含具体版本号或"latest/new" → 快速迭代
- 其余 AI/ML 话题 → 成熟框架
- 协议/规范类 → 行业标准

### 增补逻辑（版本追加）

旧报告不覆盖，新内容追加到末尾：

```markdown
# {话题名}

---
## v1 — 2026-01-10 首次检索
{原始内容}

---
## v2 — 2026-04-19 增补
**增补原因**: 报告已过期，重新检索更新
{新增内容，重点标注与 v1 的变化}
```

检索完成后自动写入/更新缓存，无需用户操作。

---

## 报告生成格式

每次检索完成，输出一份可读报告，同时写入本地缓存：

```markdown
# 参考报告：{话题名}

**检索时间**: {ISO 时间}  
**有效期至**: {过期时间}  
**启用通道**: {通道列表}  
**关键词**: {提取的关键词}

---

## 核心结论

[每条建议格式如下]
> **建议**: {具体可操作的方案}
> **来源**: {通道名} — {项目名/文章名/人名}（{可信度等级}，{日期}）
> **为什么**: {选这个方案的理由，或相比其他选项的优势}

[多条建议按可信度 S→A→B 排列]

## 参考来源

### GitHub 项目
- {项目名} (⭐{stars}) — {方案概要}

### 官方文档
- {文档名} — {关键内容}

### 大神观点
- {人名}: {核心观点} — [{文章标题}]({url}) ({日期})

### 社区讨论
- {平台}: {核心结论} ({票数/赞数})

## 注意事项 / 踩坑

[从 Issues、失败案例、社区讨论中提炼]

---
*来源可信度: S={官方/规范} A={大神/高星项目} B={高票社区} C={一般讨论}*
```

---

## 与现有 Skill 的关系

| 现有 Skill | 与 smart-reference 的关系 |
|-----------|--------------------------|
| `external-context` | 底层工具，smart-reference 调用它做通道 2/4 的搜索 |
| `skill-from-github` | 定位不同（学习后创建 skill），保留 |
| `skill-from-masters` | 定位不同（从案例归纳创建 skill），保留 |

smart-reference 是上层路由，不替代，而是统一入口。

---

<Escalation_And_Stop_Conditions>

## 停止与上报条件

**立即停止，告知用户**:
- 所有启用通道均返回 **0 条内容**（含 B/C 级） → 告知用户并建议调整关键词，不凭空生成建议
- index.json 格式损坏无法解析 → 告知用户，跳过缓存直接走检索
- 用户在检索过程中说"停""算了""不用查了" → 立即终止
- 全局总时长超过 **3 分钟** → 停止新检索，用已有结果出报告，标注"部分通道超时未完成"

**降级处理，不停止**:
- 单个通道超时（>60秒）→ 标记该通道"无结果"，继续其他通道
- 某通道返回内容质量极低（全是广告/无关内容）→ 丢弃，不纳入融合
- 仅有 B/C 级来源，无 S/A 级 → 继续出报告，但在报告顶部标注"⚠️ 未找到高质量参考，以下为社区讨论级结果，仅供参考"
- index.json 不存在（首次使用）→ 创建空索引，直接走检索

**禁止行为**:
- 禁止在所有通道 0 结果时凭印象生成"参考建议"——这是幻觉，不是参考
- 禁止因单通道失败就声称"检索完成"
- 禁止跳过缓存写入步骤（每次检索完必须写缓存）

</Escalation_And_Stop_Conditions>

---

<Examples>

## 示例：好与坏

<Good>
**场景**: 用户说"去网上查一下 MCP server 设计的 best practice"

执行:
1. **立即用 Bash 工具运行缓存检查命令**（不等用户，不询问）：
   ```bash
   mkdir -p ~/.claude/reference-cache/reports && ls ~/.claude/reference-cache/index.json 2>/dev/null && cat ~/.claude/reference-cache/index.json || echo "NO_CACHE"
   ```
2. 缓存未命中 → 意图路由 → "best practice" → 通道 3 + 4 + 2
3. 并行启动 3 个通道，读取 `references/masters.md` 获取 AI 领域博客列表
4. 通道 3 找到 Simon Willison 关于 MCP 的文章（A 级）
5. 通道 4 找到 HN 高票讨论（B 级）
6. 通道 2 找到 Anthropic 官方 MCP 文档（S 级）
7. 融合输出：先引官方文档结论，再引大神观点，附踩坑
8. 写入缓存，ttl_days=1（AI 工具类）

**为什么好**: 触发后立即用工具执行，不等用户；按来源可信度排序，结论可溯源，缓存写入，没有凭空发明内容。
</Good>

<Bad>
**场景**: 用户说"去网上查一下 React hooks 最佳实践"，通道 3 超时无结果

错误做法: "根据我的了解，React hooks 最佳实践包括：1. 只在顶层调用 hooks... 2. 只在 React 函数组件中调用..."（凭模型记忆生成，伪装成检索结果）

**为什么坏**: 没有实际检索，却以"参考结论"形式输出，是幻觉。正确做法是标注"通道 3 无结果，以下来自通道 2 官方文档"，或告知用户部分通道失败。
</Bad>

<Bad>
**场景**: 话题是"Claude streaming API"，命中缓存但已过期（2天前，TTL=1天）

错误做法: 直接读旧报告输出，不告知用户已过期。

**为什么坏**: AI 工具日更，2天前的报告极可能过时。正确做法是忽略旧报告，自动进入 Step 1 重新检索，完成后在报告末尾注明"已替换 2 天前旧报告"。
</Bad>

</Examples>

---

<Termination_Conditions>

## 显式终止条件（Complete Gate）

**检索完成的充要条件 — 3 条全部满足才可输出报告**:

| # | 条件 | 如何验证 |
|---|------|---------|
| 1 | 所有已启动的 Agent 均已返回结果（或超时标记为"无结果"） | 启动 N 个 Agent，收到 N 个返回 |
| 2 | 至少有 1 个 S 级或 A 级来源命中（有实质内容，非空） | 可信度评级表中至少 1 条 ≥ A |
| 3 | 报告已输出给用户，缓存已写入 | 输出 + 写入均完成 |

**硬性上限（防止无限检索）**:
- `max_agents`: 同时最多 4 个并行 Agent，超出则排队等待，不再新增
- `max_sources_per_channel`: 每条通道最多取 5 个来源
- `max_retries`: 单通道失败后最多重试 1 次，仍失败则记录"无结果"继续

**终止时的行为**:
- 3 条全满足 → 立即输出报告，不再继续检索
- 条件 2 不满足（0 个 S/A 级来源）→ 告知用户"未找到高质量参考，以下为 B/C 级结果，仅供参考"，不伪装成高质量结论
- 超过 max_agents 上限仍无 S/A 结果 → 停止，不再扩大检索，用已有结果出报告

</Termination_Conditions>

---

<Final_Checklist>

## 完成检查

每次检索结束前确认（按顺序）：

- [ ] 缓存已检查（命中/未命中均有明确处理）
- [ ] 启用的通道均已执行或记录失败原因
- [ ] 输出结论有来源标注（S/A/B/C 级 + 来源链接/名称），无裸结论
- [ ] 每条结论均可追溯到通道名称或具体来源（不可只写"根据检索"）
- [ ] 输出格式包含检索时间和有效期
- [ ] Termination_Conditions 3 条全部满足（含缓存写入）后才输出报告

</Final_Checklist>
