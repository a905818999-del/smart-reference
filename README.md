# smart-reference

A dual-platform skill for intelligent external reference retrieval across Claude Code and Codex.

When you say things like "check what the experts do", "look for best practices", "is there anything on GitHub", or "search online", this skill routes the query to the right sources and turns the results into actionable guidance with source attribution.

---

## Why this exists

Before implementing something, the useful questions are usually:

> What do the official docs say? Has anyone built this already on GitHub? What do experts like Karpathy or Simon Willison recommend? Are there community threads with gotchas?

This skill standardizes that workflow so reference gathering is consistent instead of ad hoc.

---

## How it works

The skill organizes external research into 6 channels:

1. **GitHub high-star repos**: what the community has actually built
2. **Official docs**: what the framework or API author recommends
3. **Expert blogs/videos**: what trusted practitioners think
4. **Communities**: what people are saying in practice
5. **Code examples**: cookbooks, examples, and runnable patterns
6. **Standards/RFCs**: protocol, spec, and compliance references

It routes the query by intent, gathers the relevant evidence, and fuses the result into a structured answer.

### Star thresholds

| Category | Threshold | Reasoning |
|----------|-----------|-----------|
| Mainstream tools | >= 5,000 stars | Mature ecosystem, below this is often noise |
| Vertical or niche tools | >= 1,000 stars | Lower ceiling, still useful for filtering |
| New tech (< 6 months) | >= 1,000 stars + recent commits | Activity matters as much as stars |
| Awesome lists | >= 1,000 stars | Curated content should clear a higher bar |

Also consider momentum: repositories gaining more than 5% of total stars in the last 30 days may be worth including even if they are near the threshold.

### Time sensitivity

This is a v2 instruction-only skill with no local cache. Instead, each report should communicate freshness:

| Topic type | Treat report as fresh for |
|------------|---------------------------|
| Fast-moving AI tools and APIs | 1 day |
| Mature frameworks | 30 days |
| Engineering practices and patterns | 60 days |
| Standards and RFCs | 180 days |

### Why every conclusion needs a reason

Every recommendation should include:

```text
Recommendation: [specific actionable suggestion]
Source: [channel] - [project/article/person] (credibility level, date)
Why: [why this approach over alternatives]
```

That keeps the output auditable instead of turning it into unsupported memory-based advice.

### Completion rule

The skill should consider the search complete only when:

1. All launched channels have returned or timed out
2. At least one S- or A-grade source was found, or the report clearly warns that only weaker evidence exists
3. The final report has been delivered to the user

Hard limits:

- max 4 channels in parallel
- max 5 sources per channel
- global 3-minute cap

### Official-doc routing across Claude and Codex

This skill supports both **Claude** and **Codex** without assuming one is the default official source.

- If the user clearly asks about **Claude**, **Anthropic**, or **Claude Code**, route official-doc lookups to Anthropic sources first.
- If the user clearly asks about **Codex**, **OpenAI**, **OpenAI API**, or **skills in Codex**, route official-doc lookups to OpenAI sources first.
- Codex-side official sources explicitly include:
  - `developers.openai.com/codex`
  - `developers.openai.com/codex/skills`
  - `developers.openai.com`
  - `platform.openai.com/docs`
- If the question is ambiguous and official docs are the main evidence, ask whether the user wants the **Claude** ecosystem or the **Codex/OpenAI** ecosystem before searching.
- If the user says "both", search both and label the results separately as `Claude 官方` and `Codex/OpenAI 官方`.

---

## Design principles

- **Route before searching**: different questions need different channels
- **Parallel when useful**: gather independent evidence without serial bottlenecks
- **Credibility tiers**: official and standards first, weaker community evidence later
- **Source transparency**: no "based on research" claims without traceable sources
- **Graceful degradation**: timeouts and weak evidence should be surfaced, not hidden
- **Freshness labels, not local cache**: let readers judge how current the evidence is

---

## Installation

Works with **Claude Code** (via [oh-my-claudecode](https://github.com/hesreallyhim/oh-my-claudecode)) and **Codex**.

**Claude Code / oh-my-claudecode**

```bash
git clone https://github.com/a905818999-del/smart-reference ~/.claude/skills/smart-reference
```

Restart Claude Code after installing.

**Codex**

```bash
git clone https://github.com/a905818999-del/smart-reference ~/.codex/skills/smart-reference
```

Restart Codex after installing. Implicit invocation is enabled in `agents/openai.yaml`.

**Trigger phrases**:

- "去网上查一下..."
- "上网查一下..."
- "看看大神怎么做"
- "有什么好方案"
- "参考一下 best practice"
- "GitHub 上有没有现成的"
- "官方文档怎么说"
- "社区有没有人踩过这个坑"

---

## File structure

```text
smart-reference/
  SKILL.md
  references/
    masters.md
  agents/
    openai.yaml
  README.md
```

---

## Report format

```markdown
# Reference Report: {topic}

Search Time: {ISO timestamp}
Expires: {expiry time}
Channels: {channels used}

---

## Core Findings

> **Recommendation**: {specific actionable suggestion}
> **Source**: {channel} - {project/article/person} (Grade, date)
> **Why**: {reasoning - why this over alternatives}

## References

### GitHub Projects
- {name} (⭐{stars}) - {summary}

### Official Docs
- {doc name} - {key content}

### Expert Views
- {name}: {insight} - [{title}]({url}) ({date})

### Community
- {platform}: {finding} ({votes})

## Gotchas / Pitfalls

[From issues, failure cases, or community threads]
```

---

## Versioning

| Version | Date | Changes |
|---------|------|---------|
| v2.0.3 | 2026-04-23 | Refined dual-platform official-doc routing, documented Claude vs Codex/OpenAI ambiguity handling, and aligned repo metadata with current behavior |
| v2.0.2 | 2026-04-23 | Final consistency pass for Codex-compatible instruction-only behavior |
| v2.0.1 | 2026-04-22 | Aligned Codex policy and README with actual invocation behavior |
| v2.0.0 | 2026-04-22 | Codex-compatible rewrite with instruction-only workflow and dual installation guidance |
| v1.1.0 | 2026-04-21 | Added explicit termination conditions, source-plus-why output, timeout handling, and auto-execute signal |
| v1.0.0 | 2026-04-19 | Initial release with 6 channels, cache, and TTL strategy |

---

## License

MIT
