# Claude Code Sniffer — Grounding Document

*A development specification for a self-referential workflow discovery and skill generation tool*

---

## Overview

Claude Code frequently saves plans, task breakdowns, and structured workflows to the application data directory of the local installation. These files are a goldmine of implicit process knowledge — patterns of work that the user and Claude have already validated through real usage, but which have never been codified into reusable, automatable skills.

The **Claude Code Sniffer** (CCS) is a proposed tool that reads those stored plans, identifies the most frequently repeated action patterns, and surfaces the top ten as skill candidates. Each candidate is written up in enough detail to be handed directly to the skill-creator for development into a fully executable, batch-schedulable skill.

The goal is to close the loop: instead of skills being authored top-down from abstract intention, they emerge organically from observed practice.

---

## The Core Problem

When a developer works with Claude Code over time, they tend to repeat a set of actions without realising it:

- Scaffolding new modules in a consistent structure
- Running a specific sequence of linting, testing, and formatting steps
- Exporting documents in a standard format with consistent naming
- Updating version numbers across multiple config files
- Summarising a codebase before starting a new feature
- Cleaning and normalising datasets before analysis

These patterns exist implicitly in the plans Claude saves. But because they have never been formally extracted, each session either reinvents the process from scratch or relies on the user to remember and re-describe it. This is inefficient and introduces inconsistency.

The Claude Code Sniffer makes this tacit knowledge explicit.

---

## Where Plans Are Stored

Claude Code stores structured plans and task breakdowns in the application data directory. The exact location varies by OS:

- **macOS**: `~/Library/Application Support/Claude/` or within the project's `.claude/` directory
- **Linux**: `~/.config/claude/` or project-local `.claude/`
- **Windows**: `%APPDATA%\Claude\` or project-local `.claude\`

Within these directories, plans are typically stored as:

- Markdown files (`.md`) — natural language task breakdowns and multi-step instructions
- JSON files (`.json`) — structured task lists, tool call histories, and session logs
- Occasionally plain text files (`.txt`) — quick capture of ad-hoc steps

The sniffer must be capable of reading all three formats and normalising them into a common representation before analysis begins.

---

## The Sniffing Process

The sniffing workflow consists of five stages:

### Stage 1: Discovery

Scan the application data directory (and optionally a specified project directory) recursively. Collect all plan-like files based on extension and heuristic content checks — files that contain imperative language, numbered steps, tool invocations, or repeated structural patterns are strong candidates for analysis.

A configurable `--depth` parameter limits recursion. A `--since` parameter allows filtering by modification date so the sniffer can focus on recent work rather than the full history.

### Stage 2: Normalisation

Convert all collected files to a unified internal representation. For each file, extract:

- **Action atoms** — discrete, imperative steps ("run tests", "create file", "update config", "fetch from API")
- **Tool references** — mentions of specific tools, scripts, or commands used
- **Output formats** — what each task produces (a file, a report, a transformed dataset)
- **Trigger contexts** — what preceded this task in the same plan (the prior step or stated goal)

This stage does not interpret meaning — it tokenises behaviour.

### Stage 3: Frequency Analysis

Once normalised, action atoms are grouped by semantic similarity. The grouping should be fuzzy, not exact — "run the linter", "lint the code", and "execute eslint" should collapse into a single cluster.

Each cluster is scored by:

- **Raw frequency** — how many times this action appeared across all files
- **Co-occurrence** — which other actions it appears alongside (this identifies workflows, not just isolated commands)
- **Recency weight** — actions from recent plans are weighted more heavily than historical ones

The top clusters by combined score are retained. From these, coherent **workflow candidates** are assembled: a workflow is a cluster of two or more co-occurring action atoms that together constitute a repeatable, bounded process.

### Stage 4: Candidate Ranking

The top ten workflow candidates are ranked. Ranking criteria:

1. **Repetition score** — how many times this pattern appeared
2. **Completeness** — does the pattern have a clear start, body, and output?
3. **Automability** — does it consist of deterministic steps, or does it require significant human judgment at each stage?
4. **Breadth** — does it appear across multiple projects/contexts, or only within one?

Candidates with high repetition but low automability are flagged as **advisory candidates** — they are surfaced as insights but not written up as full skill specs, since they are not suitable for automation without significant further design work.

### Stage 5: Output Generation

The top ten candidates are written into a structured markdown file: `ccs-candidates.md`. Each entry includes all fields required for immediate handoff to the skill-creator workflow. Additionally, a summary of the sniffing run is appended as metadata — source file count, date range of material analysed, total action atoms found.

---

## Output Format: `ccs-candidates.md`

```markdown
# CCS Candidate Skills
*Sniffed from: [N] plan files across [date range]*
*[N] workflow candidates identified — [X] High / [Y] Medium / [Z] Low automability*

---

## 01. [Skill Name]
**Rank:** #1 (frequency score: [N])
**What it does:** [One sentence]
**Trigger phrase:** "[example invocation]"
**Input:** [what goes in]
**Output:** [what comes out]
**Automability:** High / Medium / Low
**Why it's a skill:** [One sentence on encoding value]
**Source files:** [list of plan files where this pattern appeared]
**Workflow steps:**
1. [Step one]
2. [Step two]
3. ...
**Batch/schedule notes:** [Notes on whether this is suitable for batch execution, scheduled runs, or both. Include any preconditions that must be true before scheduling.]

---

## 02. [Skill Name]
[Same structure]

...
```

---

## Batch and Scheduled Execution

A key design principle of the CCS is that every skill candidate should be assessed not just for manual invocation but for **unattended execution** — either as part of a batch pipeline or as a scheduled task.

Each candidate entry therefore includes a `Batch/schedule notes` field that answers:

- Can this run without human confirmation at each step?
- What preconditions must be met (e.g., a test suite must pass before a deployment step runs)?
- Is there a natural cadence for this task (nightly, on commit, weekly)?
- What should happen on failure — abort the batch, log and continue, or alert?

Skills that score High on automability and have an obvious cadence are flagged with a **⏱ Schedule candidate** marker. Skills that are best run on-demand as part of a pipeline are flagged with **⚙ Batch candidate**. Skills that require a human in the loop before proceeding are flagged with **👤 Advisory only**.

This three-tier classification guides the user's decisions after reading the report — they know immediately which candidates are ready for scheduling, which need to be composed into a pipeline, and which are insights rather than automation targets.

---

## Integration with Existing Skills

The CCS is designed to slot naturally into the existing skill ecosystem:

**Upstream — feeds into:**
- `skill-skimmer` — the CCS output (`ccs-candidates.md`) is valid input for the skill-skimmer, which can enrich the candidate entries with additional analysis before handing off to skill-creator
- `skill-creator` — each candidate entry contains all four fields the skill-creator needs (what it does, when it triggers, output format, test case basis)

**Downstream — builds on:**
- Existing user skills — the sniffer is aware of the skills already installed in `/mnt/skills/user/`. A workflow candidate that is already covered by an existing skill is suppressed from the report and instead noted as a confirmation that the existing skill is being used.

**Peer — complements:**
- `zero-agency` — before any scheduled or batch execution, the zero-agency framework should be applied to confirm the action scope and prevent runaway automation
- `unfold` — complex skill candidates from the CCS can be handed to unfold for stage-based implementation planning before being sent to skill-creator

---

## Development Architecture

### Components

```
claude-code-sniffer/
├── SKILL.md                        # Skill definition and trigger logic
├── scripts/
│   ├── discover.py                 # Stage 1: file discovery and filtering
│   ├── normalise.py                # Stage 2: extraction of action atoms
│   ├── frequency.py                # Stage 3: clustering and scoring
│   ├── rank.py                     # Stage 4: candidate ranking
│   └── generate_report.py          # Stage 5: output generation
├── references/
│   ├── action-atom-taxonomy.md     # Reference vocabulary for normalisation
│   ├── scheduling-patterns.md      # Guidance on batch/schedule classification
│   └── output-template.md          # The ccs-candidates.md template
└── assets/
    └── ccs-candidates-example.md   # A worked example for documentation
```

### Key Design Decisions

**Why Python for the scripts?** The application data directory contains files in mixed formats across OSes. Python's standard library handles path resolution, file encoding detection, and JSON parsing reliably across platforms without additional dependencies.

**Why semantic clustering rather than exact string matching?** Exact matching would produce fragmented results — "run tests" and "execute the test suite" would be treated as distinct actions. The sniffer must surface the intent behind the wording, not the wording itself. A lightweight embedding-based similarity check (or simple synonym expansion with a curated vocabulary) handles this without requiring a network call.

**Why weight recency?** Older plans may reflect abandoned workflows or superseded tooling. The user's current practice is the most relevant signal. Recency weighting ensures the report reflects where the user is now, not where they were six months ago.

**Why ten candidates?** Ten is the threshold at which the report remains actionable without becoming overwhelming. Fewer than ten risks missing important patterns; more than ten dilutes focus. The number is configurable but defaults to ten.

---

## Trigger Conditions

The Claude Code Sniffer should trigger when the user says any of:

- `"use the Claude Code Sniffer"`
- `"sniff my plans"`
- `"what tasks am I repeating?"`
- `"find repeated workflows in my Claude data"`
- `"generate skill candidates from my history"`
- `"what could I automate from my Claude plans?"`
- `"build a candidate list from my stored plans"`

It should also trigger proactively if the user asks for skill candidates and no source material has been provided — at which point the sniffer treats the application data directory as the implicit input source.

---

## Open Questions for Development

1. **Scope of discovery** — Should the sniffer default to the current project's `.claude/` directory, the global application data directory, or both? A `--scope project|global|all` flag would give users control.

2. **Privacy handling** — Plans may contain sensitive information (API keys embedded in commands, internal project names, credentials). The sniffer should redact known sensitive patterns before analysis and never write them into the output report.

3. **Cross-session coherence** — If the same workflow appears across many sessions but was never completed (the user kept starting and abandoning it), should that count as high-frequency or be penalised? This is an open design question — partial completion may signal that the workflow is harder than it looks and therefore a higher-value candidate for codification.

4. **Human-in-loop annotation** — After generating the initial report, should the sniffer ask the user to confirm the top ten before writing them up in detail? This would prevent effort being wasted on candidates the user immediately recognises as irrelevant. Recommend: yes, with a `--no-confirm` flag for automated pipelines.

5. **Versioning** — Should successive sniffing runs be diffed against each other so the user can see which patterns are growing in frequency and which are fading? A simple `--diff previous-run` mode would make this tractable.

---

## Success Criteria

The Claude Code Sniffer is considered successful when:

- It correctly identifies at least seven of the user's top ten repeated workflows without manual guidance
- Every candidate in the report can be handed directly to the skill-creator without needing additional scoping or clarification
- At least three candidates per run are flagged as ready for scheduling or batch execution
- The full run completes in under sixty seconds on a typical application data directory (hundreds of plan files)
- The output report is readable in under three minutes

---

## Next Steps

1. Finalise the action-atom taxonomy — the reference vocabulary that normalise.py will use to cluster semantically similar steps
2. Build and test discover.py against a real Claude Code installation
3. Write the SKILL.md trigger definition and test it against the skill-creator eval framework
4. Create the example output file (ccs-candidates-example.md) to anchor documentation and testing
5. Review the batch/schedule classification logic with the zero-agency skill to ensure safe defaults

---

*This document is the grounding specification for the Claude Code Sniffer. It is intended to be living documentation — update it as design decisions are resolved and development progresses.*
