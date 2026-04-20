# smart-reference

A Claude Code skill for intelligent external reference retrieval.

When you say things like "check what the experts do", "look for best practices", "is there anything on GitHub", or "search online" — this skill automatically kicks in, searches the right sources in parallel, and gives you actionable recommendations with cited sources and reasoning.

---

## Why this exists

When building with AI agents, I kept repeating the same mental workflow:

> "Before I implement this, let me check — what does the official docs say? Has anyone built something like this on GitHub? What do people like Karpathy or Simon Willison think about this approach? Any community threads with gotchas?"

This was ad-hoc every time. Sometimes I'd search GitHub, sometimes I'd just ask Claude from memory. The quality of reference varied wildly depending on how much effort I put in.

The question became: **can this whole workflow be standardized into a skill that does it consistently and well?**

---

## How the design came together

### The core idea

Instead of asking Claude to "search the web" (which is vague), define **6 specific channels** that cover the information landscape:

1. **GitHub high-star repos** — what has the community actually built?
2. **Official docs** — what does the framework/API author say?
3. **Expert blogs** — what do people like Karpathy, Lilian Weng, Simon Willison think?
4. **Communities** — what are practitioners saying? (Stack Overflow, Reddit, Hacker News, Chinese forums)
5. **Code examples** — Cookbooks, official examples, Awesome lists
6. **Standards/RFC** — for protocol, security, or spec questions

Route the query to the right channels based on intent, run them in parallel, fuse the results.

### Star thresholds (why these numbers)

Not all GitHub repos are equal. We settled on:

| Category | Threshold | Reasoning |
|----------|-----------|-----------|
| Mainstream tools | ≥ 5,000 ⭐ | Mature ecosystem, below this is likely noise |
| Vertical/niche tools | ≥ 1,000 ⭐ | Niche ceilings are lower, but 1k still filters junk |
| New tech (< 6 months) | ≥ 1,000 ⭐ + recent commits | Stars accumulate slowly; activity matters more |
| Awesome lists | ≥ 1,000 ⭐ | Curated content, higher bar is reasonable |

Also track **momentum**: repos gaining > 5% of total stars in the past 30 days are worth watching even if absolute count is lower.

### TTL strategy (cache expiry)

AI tools move fast. We halved the initial TTL estimates mid-design:

| Content type | TTL | Reasoning |
|-------------|-----|-----------|
| Fast-moving (LLMs, new AI frameworks) | **1 day** | These update daily. Yesterday's report is stale. |
| Mature frameworks (React, Pandas) | **30 days** | Stable but monthly versioning |
| Engineering practices / patterns | **60 days** | Slow-moving, safe to cache longer |
| Standards / RFC | **180 days** | Almost never changes |

### The "why" requirement

Early versions of the skill output recommendations without explaining the reasoning. This was the same problem as asking Claude from memory — you get an answer but can't evaluate it.

The fix: every conclusion in the report must follow this structure:

```
Recommendation: [specific actionable suggestion]
Source: [channel] — [project/article/person] (credibility level, date)
Why: [why this approach over alternatives]
```

This makes the output auditable. If the reasoning doesn't hold, you can reject it.

### Explicit termination condition

LLM agent loops have a natural failure mode: they don't know when to stop. Without a clear "done" signal, the model either stops too early or keeps searching indefinitely.

We researched how production agent frameworks handle this (LangGraph, OpenAI Agents SDK, AutoGen, 12-factor-agents) and found four patterns:
- Structured completion field (`status: "complete"`)
- No-tool-call = natural completion
- Specific termination token (`Final Answer:`)
- Hard `max_turns` cap

For smart-reference, we use a **3-condition AND gate**:
1. All launched agents have returned (or timed out)
2. At least 1 S/A-grade source found
3. Report output + cache written

Plus hard limits: max 4 parallel agents, max 5 sources per channel, global 3-minute cap.

### Local cache

Repeated queries on the same topic shouldn't re-trigger full searches. The cache:
- Stores reports as markdown files with SHA1-hashed filenames
- Tracks TTL per topic type
- Appends new results to existing reports (versioned, never overwrites)
- Checks cache first, skips to output if fresh, re-searches if expired

```
~/.claude/reference-cache/
  index.json          ← metadata index
  reports/
    {sha1}.md         ← topic reports (v1 → v2 appended)
```

---

## Design principles we landed on

**Route before searching** — different questions need different channels. "How do I implement X" → GitHub + docs. "What's the consensus on X" → community + experts.

**Parallel, not sequential** — all relevant channels run simultaneously, results fuse after.

**Credibility tiers** — S (official/RFC) > A (experts/high-star) > B (high-vote community) > C (general discussion). Never present C-grade content as definitive.

**Source transparency** — every conclusion traces back to a channel and specific source. No "based on research" with nothing behind it.

**Degrade gracefully** — if a channel times out, skip it and note the gap. If only B/C sources found, flag it prominently. Never fail silently.

**Cache with TTL** — fast-moving topics expire in 1 day. Stable ones in months. Reports version-append, not overwrite.

---

## Installation

This is a [Claude Code](https://claude.ai/code) / [oh-my-claudecode](https://github.com/hesreallyhim/oh-my-claudecode) skill.

```bash
# Clone into your Claude skills directory
git clone https://github.com/a905818999-del/smart-reference ~/.claude/skills/smart-reference
```

Then restart Claude Code. The skill auto-triggers on natural language — no slash command needed.

**Trigger phrases** (examples):
- "去网上查一下..."
- "上网查一下..."
- "看看大神怎么做的"
- "有什么好方案"
- "参考一下 best practice"
- "GitHub 上有没有现成的"
- "官方文档怎么说"
- "社区有没有人踩过这个坑"

---

## File structure

```
smart-reference/
  SKILL.md                  ← skill prompt (the actual instructions)
  references/
    masters.md              ← curated expert sources by domain
  README.md                 ← this file
```

---

## Report format

Every search produces a structured report:

```markdown
# Reference Report: {topic}

검색 Time: {ISO timestamp}
Expires: {expiry time}
Channels: {channels used}

---

## Core Findings

> **Recommendation**: {specific actionable suggestion}
> **Source**: {channel} — {project/article/person} (Grade, date)
> **Why**: {reasoning — why this over alternatives}

## References

### GitHub Projects
- {name} (⭐{stars}) — {summary}

### Official Docs
- {doc name} — {key content}

### Expert Views
- {name}: {insight} — [{title}]({url}) ({date})

### Community
- {platform}: {finding} ({votes})

## Gotchas / Pitfalls

[From issues, failure cases, community threads]
```

---

## Versioning

| Version | Date | Changes |
|---------|------|---------|
| v1.0.0 | 2026-04-19 | Initial release — 6 channels, cache, TTL strategy |
| v1.1.0 | 2026-04-21 | Added explicit termination conditions, source+why format, timeout handling, AUTO-EXECUTE signal |

---

## License

MIT
