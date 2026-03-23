---
title: "The Definitive Guide to Agent Skills"
tags:
  - docs
  - skills
  - claude-code
  - codex
  - cursor
  - copilot
  - windsurf
  - agent-architecture
  - security
status: active
created: 2026-03-23
---

# The Definitive Guide to Agent Skills

## Across Claude Code, Codex, Cursor, Windsurf, GitHub Copilot, and the Open Standard

Agent skills are the most important configuration surface in AI-assisted development — structured markdown files that encode reusable workflows, domain expertise, and procedural knowledge as dynamically-loaded agent capabilities. The SKILL.md format, published as the Agent Skills open standard at agentskills.io in December 2025, is now natively supported by 26+ platforms. The parallel AGENTS.md standard, adopted by 20,000+ GitHub repositories, provides cross-tool interoperability for always-on project instructions.

The ecosystem has grown fast. Anthropic's skills repo has ~96,700 GitHub stars, marketplaces index over 66,000 skills, and community frameworks provide complete development workflows. The flip side: Snyk found **36.82% of community skills contain security flaws**, and community testing shows baseline auto-triggering reliability between 0-50%. Skill design, invocation reliability, security, and evaluation are among the most consequential topics in AI development today.

This guide covers every platform's instruction system, the portable patterns, the platform-specific extensions, the internal architecture that determines whether skills fire, how to write them for consistent output, security, evaluation, and a universal repo layout that works everywhere.

---

## 1. The instruction format landscape

Every major coding agent now supports structured instruction files, but the formats, discovery mechanisms, and loading strategies differ significantly. Here's the complete picture as of March 2026:

### What reads what

| File / Format | Claude Code | Codex | Cursor | Windsurf | Copilot (VS Code) | Copilot (Agent) | Gemini CLI |
|---|---|---|---|---|---|---|---|
| `SKILL.md` (Agent Skills standard) | Native | Native | — | — | `.github/skills/` | `.github/skills/` | Native |
| `AGENTS.md` | — | Native | Native | Native | Native | Native | Native |
| `CLAUDE.md` | Native | — | — | — | Supported | Supported | — |
| `GEMINI.md` | — | — | — | — | Supported | Supported | Native |
| `.cursor/rules/*.mdc` | — | — | Native | — | — | — | — |
| `.windsurf/rules/*.md` | — | — | — | Native | — | — | — |
| `.github/copilot-instructions.md` | — | — | — | — | Native | Native | — |
| `.github/instructions/*.instructions.md` | — | — | — | — | Native | Native | — |
| `.github/agents/*.agent.md` | — | — | — | — | Native | Native | — |

### The two portable standards

**SKILL.md (Agent Skills)** — On-demand, progressively-disclosed domain expertise. Agents load metadata at startup; full instructions load only when the skill matches the current task. This is the pattern for specialized workflows, procedural knowledge, and capabilities beyond the base model. Portable across Claude Code, Codex, Gemini CLI, and GitHub Copilot.

**AGENTS.md** — Always-on project instructions. Loaded every session, stays in context. Build instructions, test commands, coding conventions, architecture constraints. Adopted by 20,000+ GitHub repos. Portable across Codex, Cursor, Windsurf, Copilot, Gemini CLI. Notable gap: Claude Code uses CLAUDE.md instead and does not read AGENTS.md natively.

---

## 2. Per-platform instruction systems

### Claude Code

**Skills:** `.claude/skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`). Three-stage progressive disclosure (metadata → body → references). Optional: `scripts/`, `references/`, `assets/`. Frontmatter extensions: `context` (fork for subagent), `disable-model-invocation`, `allowed-tools`, `model`. Bundled scripts execute via bash without entering context — only output does.

**Always-on:** `CLAUDE.md` at project root. Auto-memory via `MEMORY.md`. Keep under 200 lines (92% rule compliance vs. 71% beyond 400).

**Hooks:** Deterministic shell commands at lifecycle events — `PreToolUse`, `PostToolUse`, `SessionStart`, `UserPromptSubmit`. Execute in ~50ms, guaranteed to fire. Exit code 2 = unconditional block.

**Scope hierarchy:** Enterprise → Personal (`~/.claude/skills/`) → Project (`.claude/skills/`).

**Unique strengths:** Executable script bundling (no other system supports this), three-stage progressive disclosure, plugin ecosystem with marketplace distribution.

### OpenAI Codex CLI

**Skills:** `.agents/skills/<name>/SKILL.md` — same Agent Skills open standard as Claude Code. Progressive disclosure: metadata at startup, full body on match. Supports `agents/openai.yaml` for UI metadata, invocation policy, and tool dependencies. Explicit invocation via `/skills` or `$skill-name`. Implicit invocation based on description matching.

**Always-on:** `AGENTS.md` anywhere in filesystem (repo root, `~`, working directory). Codex scans from CWD up to repo root. Nested `AGENTS.md` files apply to specific directories.

**Installation:** `$skill-installer` command for remote skill installation from repos.

**Sandboxing:** macOS Seatbelt, Linux Landlock. Three safety levels: read-only, auto (default), full-auto. `--dangerously-bypass-approvals-and-sandbox` for CI only.

**Unique strengths:** Codex Cloud for parallel background tasks, strong AGENTS.md-first culture, MCP server mode (`codex mcp`), built-in approval/sandbox pipeline.

### Cursor

**Rules:** `.cursor/rules/*.mdc` files with YAML frontmatter. Four distinct rule types provide the most explicit triggering control of any system:

| Type | Frontmatter | Behavior |
|---|---|---|
| **Always** | `alwaysApply: true` | Loaded into every request |
| **Auto Attached** | `globs: ["src/**/*.tsx"]` | Loaded when working on matching files |
| **Agent Requested** | `description: "..."` | Agent decides based on description (like skills) |
| **Manual** | (no auto fields) | Only applied when user tags with `@ruleName` |

**Always-on:** Also reads `AGENTS.md`. Legacy `.cursorrules` still supported but deprecated.

**File references:** `@filename.ts` syntax includes files as additional context when the rule triggers.

**Auto-memory:** Memories generated automatically. `/Generate Cursor Rules` command creates rules from conversation.

**Unique strengths:** Four-tier triggering control, glob-based auto-attachment, `@filename` context references, `/Generate Cursor Rules` for rule creation from conversation patterns.

### Windsurf

**Rules:** `.windsurf/rules/*.md` files (plain markdown, no special frontmatter). Three scope levels: global, workspace, system (enterprise). Also reads `AGENTS.md`. Legacy `.windsurfrules` at project root still works.

**Context pipeline:** Rules → Memories → Open files → Indexed retrieval → Recent actions. The "recent actions" component (file edits, terminal commands, navigation) makes Windsurf uniquely context-aware.

**Memories:** Auto-generated by Cascade + user-created. Persistent across sessions but local to machine. For durable, team-shareable context: write it as a Rule or AGENTS.md instead.

**Discovery:** Scans `.windsurf/rules/` in workspace, subdirectories, and up to git root. Deduplicates across multiple open folders.

**Enterprise:** System-level rules deployed to OS-specific directories, displayed with "System" label, cannot be deleted by users.

**Unique strengths:** RAG-based context engine with flow tracking (edits, terminal, navigation), Memories system for evolving knowledge, enterprise system rules, `@codebase` search for manual retrieval.

### GitHub Copilot

Copilot has the most layered instruction system of any tool, with five distinct mechanisms:

**1. Personal** — User settings in VS Code / GitHub.com.

**2. Organization** — Admin settings for Business/Enterprise plans.

**3. Repository-wide** — `.github/copilot-instructions.md` at repo root. Always loaded for all chat requests.

**4. Path-specific** — `.github/instructions/*.instructions.md` with `applyTo` frontmatter for file-pattern scoping. Similar to Cursor's Auto Attached.

**5. Agent instructions** — `AGENTS.md` (primary), `CLAUDE.md`, `GEMINI.md` at repo root. Nested `AGENTS.md` files as additional instructions.

**Skills** — `.github/skills/<name>/SKILL.md` in agent workflows. On-demand progressive disclosure.

**Custom agents** — `.github/agents/*.agent.md` with `description`, `tools`, `model`, and prompts. Select via dropdown in chat. Can specify MCP servers and tool permissions.

**Hooks** — Shell commands at lifecycle points (preview). `excludeAgent` frontmatter property to target instructions to specific agents.

**Unique strengths:** Five-layer instruction hierarchy, path-specific instructions with `applyTo`, custom agent profiles (`.agent.md`), `/init` command to auto-generate instructions from project analysis, Copilot coding agent for autonomous background work, plugin marketplace.

---

## 3. The SKILL.md specification (cross-platform)

Every skill is a directory containing a required `SKILL.md` file:

```
my-skill/
├── SKILL.md              # Main instructions (required)
├── scripts/              # Executable code (Claude Code, Codex)
├── references/           # Docs loaded on demand
└── assets/               # Templates, icons, fonts
```

The frontmatter requires two fields:

```yaml
---
name: my-skill-name
description: >
  What the skill does. Secondary capabilities.
  Use when [trigger condition]. Triggers on "[keyword]".
  NOT for [exclusion].
---
```

**`name`** — Max 64 characters, lowercase kebab-case.

**`description`** — Max 1,024 characters, third-person. This is the routing signal — the only text agents see at startup for deciding whether to load the skill.

### Platform-specific extensions

| Extension | Claude Code | Codex | Copilot |
|---|---|---|---|
| `context: fork` (subagent execution) | Yes | — | — |
| `disable-model-invocation: true` | Yes | — | — |
| `allowed-tools` (scoped permissions) | Yes | — | — |
| `model` (override default) | Yes | — | Yes (in `.agent.md`) |
| `agents/openai.yaml` (UI metadata) | — | Yes | — |
| `user-invocable: false` | Yes | — | — |

### Progressive disclosure (the 98% token savings)

All platforms that support SKILL.md use the same three-stage architecture:

**Stage 1 — Metadata (~100 tokens/skill).** Name + description load at startup. 50 skills ≈ 5,000 tokens.

**Stage 2 — Full SKILL.md body (typically <5,000 tokens).** Loads when the agent decides the skill is relevant.

**Stage 3 — References, scripts, assets (on demand).** Scripts execute via bash; only output enters context.

Compare to MCP: 20 servers × 5-10 tools = ~32,000 tokens loaded into every message whether used or not. Skills scale with actual usage, not installed capacity.

---

## 4. The AGENTS.md specification (cross-platform)

`AGENTS.md` is the always-on instruction layer — project conventions, build commands, test instructions, architecture constraints that should apply to every interaction. GitHub's analysis of 2,500+ AGENTS.md files reveals best practices:

**Put executable commands early.** Build, test, and lint commands should appear in the first section.

**Use code examples over explanations.** Show the pattern, don't describe it.

**Set three-tier boundaries:** Always do / Ask first / Never do.

**Keep focused — one persona per file.** For multi-agent repos, use nested AGENTS.md files per directory.

### Where agents look for AGENTS.md

| Agent | Locations |
|---|---|
| Codex | Repo root, `~`, CWD, any directory up to repo root |
| Cursor | Repo root |
| Windsurf | Repo root, subdirectories, up to git root |
| Copilot | Repo root (primary), nested (additional), `$COPILOT_CUSTOM_INSTRUCTIONS_DIRS` |
| Gemini CLI | Repo root |
| Claude Code | **Does not read AGENTS.md** — uses CLAUDE.md |

### The CLAUDE.md gap

Claude Code is the only major tool that doesn't natively read AGENTS.md. If you need cross-tool compatibility, maintain both:

```
project-root/
├── AGENTS.md           # Codex, Cursor, Windsurf, Copilot, Gemini
├── CLAUDE.md           # Claude Code
```

Or symlink one to the other. Copilot also reads CLAUDE.md and GEMINI.md as fallback files.

---

## 5. How agents decide to invoke skills

The invocation mechanism is the root cause of all reliability problems and the most important thing to understand for cross-platform skill design.

### There is no algorithmic routing

No platform uses embeddings, classifiers, or keyword matching for skill selection. Every tool formats available skill descriptions as text in the system prompt and lets the LLM's forward pass make the decision. This is pure LLM reasoning — probabilistic, not deterministic.

This means the **description field is the single most important piece of a skill** and must be optimized empirically, not written casually.

### Real-world failure rates

Community testing reveals how unreliable baseline triggering is:

- One practitioner tested 20 prompts against a well-crafted skill: **0/20 triggered (0%)**
- 200+ tests across four SvelteKit skills: **~50% (coin flip)**
- Broader community testing: descriptions alone achieve **20-50%**, adding examples improves to **72-90%**

### The description budget

Claude Code and Codex share skill descriptions within a character budget (~2% of context window, ~16,000 characters). Too many skills = some get silently dropped. One user with 63 skills found 33% excluded.

### Writing descriptions that actually trigger

The pattern that works across every platform:

```yaml
description: >
  [Core capability in third person]. [Secondary capabilities].
  Use when [trigger scenario 1], [trigger scenario 2],
  or when user mentions "[keyword1]", "[keyword2]", "[keyword3]".
  NOT for [exclusion 1], [exclusion 2].
```

Rules:

1. **Third person always.** Injected into system prompt; POV inconsistency breaks discovery.
2. **Start with what, then when.** Capability statement followed by trigger conditions.
3. **Explicit "Use when..." clause.** Single most impactful addition.
4. **Synonym coverage.** Users phrase things differently every time.
5. **Negative triggers.** "NOT for..." reduces false positives when skills compete.
6. **Under 200 words.** Longer descriptions overfit to specific queries.

### Mechanical fixes when descriptions aren't enough

**Hooks (Claude Code, Codex, Copilot):** `UserPromptSubmit` / agent hooks inject skill activation instructions before the LLM responds. Keyword-based routing achieves ~80%, forced evaluation protocol achieves ~84%.

**Cursor's four-tier system:** Use `alwaysApply: true` for rules that must always be in context, `globs` for file-pattern attachment, `description` for agent-requested (similar to skills), `@ruleName` for manual.

**Windsurf's Rules:** Always loaded at session start — no triggering uncertainty. Use for conventions that must always apply. Use AGENTS.md for the rest.

---

## 6. Skill architecture patterns (cross-platform)

### Pattern A: Prompt-only

SKILL.md contains only markdown instructions. Works identically on every platform. Best for coding standards, review checklists, commit formatting, brand guidelines. Start here.

### Pattern B: Script-bundled

Adds executable scripts in `scripts/`. Currently supported on **Claude Code and Codex** only. Scripts execute via bash; source code never enters context, only output. Dramatically more reliable than having the LLM reason through complex logic.

### Pattern C: MCP/Subagent-integrated

Calls external services through MCP or spawns subagents. Claude Code: `context: fork` in frontmatter. Codex: MCP server support via configuration. Copilot: MCP server references in `.agent.md` profiles.

### Information architecture

The fundamental principle everywhere: **separate process from knowledge.**

- SKILL.md = what to do (workflow steps, decision logic, output format)
- `references/` = what to know (domain knowledge, API docs, schemas)
- Keep SKILL.md under 500 lines
- 2-3 concrete input/output examples for judgment-heavy tasks
- Numbered step-by-step instructions (models follow sequences more reliably)

### Writing for determinism

True determinism is impossible with LLMs, but variance shrinks with:

- **Scripts for fragile logic** (Pattern B — Claude Code/Codex only)
- **Explicit templates** with "ALWAYS use this exact structure"
- **Few-shot examples** (input/output pairs)
- **Negative examples** (what NOT to do)
- **PostToolUse hooks** for deterministic post-processing (formatting, linting)
- **Low-freedom zones** for critical operations ("Do not modify this sequence")

---

## 7. Security: the threat landscape is active

### Supply chain attacks at scale

Snyk's ToxicSkills audit of 3,984 skills: **1,467 (36.82%) contain security flaws**, **534 (13.4%) critical** (malware, credential theft, exfiltration). Mobb.ai's audit of 22,511 skills: **140,963 findings**. Antiy CERT confirmed 1,184 malicious skills in ClawHub — ~1 in 5 packages.

Publishing barriers are minimal everywhere. No code signing, no mandatory review. **2.9% of skills fetch remote content at runtime** — the published skill looks benign but attackers modify behavior at any time.

### Prompt injection through skills

Skills load with elevated trust. The SkillJect paper found hiding malicious instructions in auxiliary scripts achieves **97.5% attack success** against Claude 4.5 Sonnet. PromptArmor demonstrated complete file exfiltration via injection in a skill file.

### Guardrails by platform

| Guardrail | Claude Code | Codex | Copilot | Cursor | Windsurf |
|---|---|---|---|---|---|
| OS-level sandbox | Seatbelt/bubblewrap | Seatbelt/Landlock | Cloud sandbox | — | — |
| Permission model | Allowlist/denylist | Approval levels | Agent approvals | — | — |
| Hook-based enforcement | PreToolUse (exit 2) | — | Agent hooks (preview) | — | — |
| Scoped tool permissions | `allowed-tools` | — | Per-agent tools | — | — |

### Universal security practices

- Never embed secrets in skill files (use environment variables)
- Avoid dynamic remote content fetching
- Scan third-party skills before installing (Snyk mcp-scan)
- Run untrusted skills inside Docker/Podman sandboxes
- Audit scripts in `scripts/` directories before installation
- Use `disable-model-invocation: true` (Claude Code) for skills with side effects
- Prefer skills with deterministic scripts over prompt-only skills for security-critical workflows

---

## 8. Evaluation and testing

### Anthropic's trigger optimization pipeline

The skill-creator plugin (also in `claude-plugins-official`) ships `run_eval.py` and `run_loop.py`:

```bash
# Measure trigger rate (each query runs 3x)
python run_eval.py --eval-set agents/eval-set.json --skill-path ./my-skill

# Automated description optimization
python run_loop.py \
  --eval-set agents/eval-set.json \
  --skill-path ./my-skill \
  --max-iterations 5 \
  --holdout 0.4
```

The loop: split 60/40 train/test → evaluate 3x per query → Claude proposes generalized improvements → re-evaluate → iterate up to 5 times → select by test score. Anthropic's own public skills: 5 of 6 improved.

### Quality evaluation

Trigger accuracy and output quality are separate axes. Promptfoo handles quality with assertion-based testing. A/B benchmarking runs parallel agents comparing skill-vs-baseline. LangChain's pipeline adds trace inspection, tool call counting, and Docker-based reproducible environments.

### Key metrics

**Trigger rate** — Target 90%+ across 10-20 test queries. Most common failure.

**Tool call efficiency** — Total calls and tokens with vs. without.

**Regression detection** — Re-run after model updates. If baseline passes without the skill, retire it.

### Skill obsolescence

Capacity uplift skills expire when models improve. Encoded preference skills compound in value. Monitor for "outgrowth" — if the base model passes your evals without the skill loaded, the skill is dead weight or actively harmful.

---

## 9. Universal repo layout

A repo that works across all major coding agents:

```
project-root/
├── AGENTS.md                           # Codex, Cursor, Windsurf, Copilot, Gemini
├── CLAUDE.md                           # Claude Code (symlink to AGENTS.md or maintain separately)
│
├── .agents/skills/                     # Codex skills
│   └── my-skill/
│       ├── SKILL.md
│       └── scripts/
│
├── .claude/skills/                     # Claude Code skills
│   └── my-skill/
│       ├── SKILL.md
│       └── scripts/
│
├── .github/
│   ├── copilot-instructions.md         # Copilot always-on
│   ├── instructions/                   # Copilot path-specific
│   │   └── frontend.instructions.md
│   ├── skills/                         # Copilot skills
│   │   └── my-skill/
│   │       └── SKILL.md
│   └── agents/                         # Copilot custom agents
│       └── reviewer.agent.md
│
├── .cursor/rules/                      # Cursor rules
│   ├── base.mdc
│   └── frontend.mdc
│
├── .windsurf/rules/                    # Windsurf rules
│   └── conventions.md
│
└── src/
```

### The pragmatic approach

Maintaining parallel skill directories is verbose. The practical strategy:

1. **Write AGENTS.md once** for always-on conventions. Every tool except Claude Code reads it.
2. **Maintain CLAUDE.md** with equivalent content (or symlink).
3. **Write SKILL.md files once** in `.claude/skills/` (your primary dev tool) and symlink or copy to `.agents/skills/` and `.github/skills/` for Codex and Copilot.
4. **Use Cursor's `.mdc` format** only for Cursor-specific glob-based auto-attachment — the four-tier triggering system doesn't map cleanly to other formats.
5. **Use Windsurf's `.windsurf/rules/`** only for Windsurf-specific conventions — or just rely on AGENTS.md.

### Portability matrix for skill content

| Content Type | Portable via SKILL.md? | Portable via AGENTS.md? | Notes |
|---|---|---|---|
| Coding conventions | Yes | Better in AGENTS.md | Always-on, not on-demand |
| Build/test commands | No | Yes | AGENTS.md first section |
| Specialized workflows | Yes | No | Skills are for on-demand expertise |
| Security policies | Yes | Both | AGENTS.md for always-on, SKILL.md for review workflows |
| Executable scripts | Partial (Claude/Codex only) | No | Other tools ignore `scripts/` |
| Subagent orchestration | No | No | Platform-specific (context: fork, .agent.md) |
| File-pattern scoping | No | No | Cursor globs, Copilot applyTo, Windsurf glob patterns |

---

## 10. Community ecosystem and anti-patterns

### Major frameworks

**obra/superpowers** — 20+ skills for complete dev workflow. Cross-platform (Claude, Codex, Gemini CLI).

**anthropics/skills** — Official Anthropic repo with skill-creator meta-skill.

**github/awesome-copilot** — Community Copilot customizations with plugin marketplace.

**alirezarezvani/claude-skills** — 192+ skills with cross-platform converter for 11 tools.

**travisvn/awesome-claude-skills** — Curated collection.

Marketplaces: SkillsMP (66,500+ skills), SkillHub (7,000+ with AI filter), daymade/claude-code-skills with `ccpm` package manager.

### Anti-patterns

The most important community finding: one practitioner tested 47 popular skills — **40 made output worse.** Primary harm mechanisms: unnecessary tokens, increased latency, contradictory constraints.

- **Vague instructions the model already knows.** Waste of context budget.
- **Monolithic always-on files.** Rule compliance drops past 400 lines in CLAUDE.md/AGENTS.md.
- **Not testing trigger behavior.** Skills fire probabilistically. Untested = unknown.
- **Overlapping skills without differentiation.** Multiple skills competing for the same prompts = none fire reliably.
- **Treating instructions as "hotfix" files.** Contradictions accumulate.
- **Wrapping native capabilities.** Skills should encode genuine domain expertise, not scaffolding.

---

## 11. Eight principles for excellent skill design

These apply regardless of platform:

**1. The description is make-or-break.** Agents undertrigger by default. Descriptions need "Use when..." clauses, keywords, synonym coverage, and "NOT for..." exclusions. Optimize empirically.

**2. Start prompt-only, add complexity only when needed.** Pattern A iterates faster and breaks less. Scripts and MCP earn their complexity only when prompt-only fails.

**3. Separate process from knowledge.** SKILL.md = workflow steps. `references/` = domain facts. Clean separation enables efficient loading and independent maintenance.

**4. Test with messy, realistic prompts.** Clean lab prompts consistently fail in production. Build eval sets with 10-20 queries representing how you actually talk to the agent.

**5. Measure trigger rate, output quality, and regression.** A skill that triggers 50% of the time is half as valuable. The skill-creator eval framework makes this empirical.

**6. Treat community skills like untrusted packages.** 36.82% flaw rate demands scanning, sandboxing, and manual audit. Never install skills that fetch remote content at runtime.

**7. Document failures more than successes.** The skills that get the most reuse document failures — exact error messages, symptoms, and fixes.

**8. Retire skills when models outgrow them.** Capacity uplift skills expire. Preference-encoding skills compound. If the base model passes your evals without the skill, let go.

The skill that produces the best results is often not the longest or most complex — it's the one that encodes exactly the knowledge the agent lacks, in precisely the format it needs, triggered reliably by the prompts users actually write.
