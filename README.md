# 🐾 Code Sniffer Skill

> *Bottom-up skill discovery for Claude Code — let your own workflow history write your automation roadmap.*

The **Code Sniffer** is a Claude skill that reads the plans and task breakdowns Claude Code saves to your application data directory, identifies the most frequently repeated action patterns across your sessions, and produces a ranked list of ten skill candidates — each fully specified and ready to hand off to the skill-creator for development into a fully executable, batch-schedulable automation.

Instead of authoring skills from abstract intention, they emerge organically from observed practice.

---

## 🧠 How It Works

Claude Code silently saves structured plans to your application data folder every session. Over time these files accumulate a rich record of your real workflows — the exact sequences of steps you keep repeating. The Code Sniffer mines that record through a five-stage pipeline:

```
Discover → Normalise → Cluster → Rank → Report
```

1. **Discover** — scans your Claude application data directory (or a specific project's `.claude/` folder) for all plan-like files (`.md`, `.json`, `.txt`)
2. **Normalise** — extracts discrete action atoms from each file, redacting any sensitive content (API keys, tokens, credentials) before storage
3. **Cluster** — groups semantically similar actions using a synonym taxonomy, so "run the linter", "lint the code", and "execute eslint" all collapse into a single pattern
4. **Rank** — scores each workflow cluster by frequency, recency, source breadth, and automability, then assembles the top ten candidates
5. **Report** — writes `ccs-candidates.md` with every candidate fully specified, including a batch/schedule classification for each

---

## 📦 Installation

Download `code-sniffer.skill` from the [Releases](https://github.com/MushroomFleet/claude-sniffer-skill/releases) page and install it into your Claude skills directory:

- **macOS / Linux**: `~/.claude/skills/`
- **Windows**: `%APPDATA%\Claude\skills\`

Then restart Claude Code. The skill will appear in your available skills list as `code-sniffer`.

---

## 🚀 Usage

Trigger the sniffer by saying any of the following in a Claude Code session:

```
"sniff my plans"
"use the Code Sniffer"
"what tasks am I repeating?"
"find repeated workflows in my Claude data"
"generate skill candidates from my history"
"what could I automate from my Claude plans?"
"build a candidate list from my stored plans"
```

If you don't provide a source path, the sniffer automatically uses the Claude application data directory as its scan root.

### Optional flags (via script invocation)

```bash
python scripts/discover.py --root ~/.config/claude --since 2025-01-01 --depth 5
```

| Flag | Description |
|---|---|
| `--root` | Directory to scan (default: OS application data dir) |
| `--since` | Only include files modified after this date (`YYYY-MM-DD`) |
| `--depth` | Maximum recursion depth |
| `--no-confirm` | Skip the human confirmation step before writing the full report |

---

## 📄 Output: `ccs-candidates.md`

Each run produces a markdown report with ten ranked candidates. Every entry includes:

- **Rank and frequency score**
- **One-sentence description** of the workflow
- **Example trigger phrase** for invoking the skill once built
- **Input / output specification**
- **Automability rating**: High / Medium / Low
- **Batch/schedule classification**:
  - ⏱ **Schedule candidate** — deterministic, safe to run unattended on a timer
  - ⚙ **Batch candidate** — pipeline-ready with a human approval gate
  - 👤 **Advisory only** — insight rather than automation target; convert to a guided checklist
- **Draft workflow steps**
- **Source files** the pattern was extracted from

### Example candidate entry

```markdown
## 01. `lint-automation`

**Rank:** #1 (combined score: 14.5)
**What it does:** Automates the lint workflow — a repeated action pattern extracted from your Claude Code session history.
**Trigger phrase:** "use the lint automation"
**Automability:** High
**Classification:** ⏱ Schedule candidate
**Workflow steps:**
1. Run eslint on src/
2. Fix auto-fixable issues
3. Report remaining violations
**Batch/schedule notes:** Fully deterministic. Natural cadence: pre-commit hook or CI step.
```

---

## 🔗 Pipeline Integration

The Code Sniffer is designed as the first stage of a larger skill development pipeline:

```
code-sniffer → skill-skimmer → skill-creator → installed skill
```

- Hand `ccs-candidates.md` to **skill-skimmer** for additional enrichment before building
- Hand any single candidate to **skill-creator** to develop it into a full, installable `.skill` file
- Pass any ⏱ Schedule candidate through **zero-agency** for consequence review before enabling unattended execution
- Use **unfold** to stage complex candidates into multi-phase implementation plans

---

## 📁 Skill Structure

```
code-sniffer/
├── SKILL.md                          # Skill definition and stage instructions
├── scripts/
│   ├── discover.py                   # Stage 1: file discovery and filtering
│   ├── normalise.py                  # Stage 2: action atom extraction + redaction
│   ├── frequency.py                  # Stage 3: semantic clustering and scoring
│   ├── rank.py                       # Stage 4: candidate ranking and classification
│   └── generate_report.py            # Stage 5: ccs-candidates.md output
├── references/
│   ├── grounding.md                  # Full design specification
│   ├── action-atom-taxonomy.md       # Synonym vocabulary for clustering
│   └── scheduling-patterns.md       # Batch/schedule classification guide
└── assets/
    └── output-template.md            # Blank template for the candidates report
```

---

## 🔒 Privacy

All processing is **local and read-only**. The sniffer:

- Never modifies any plan files it reads
- Redacts API keys, tokens, bearer credentials, and other sensitive patterns before any atom is stored
- Produces no network requests
- Writes output only to the current working directory

---

## 🛠 Requirements

- Python 3.10+
- No external dependencies — uses only the Python standard library

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{claude_sniffer_skill,
  title = {Claude Sniffer Skill: Bottom-up workflow discovery and skill candidate generation for Claude Code},
  author = {[Drift Johnson]},
  year = {2025},
  url = {https://github.com/MushroomFleet/claude-sniffer-skill},
  version = {1.0.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
