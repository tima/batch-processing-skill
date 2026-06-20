# batch-processing

A Claude Code skill for processing bulk data (files, transcripts, tickets, documents) that survives context compaction.

## What It Does

The skill externalizes batch state to three auto-generated files: `context.md` (your goal and extraction rules), `todos.md` (checklist of all items), and `insights.md` (running output). When Claude's context resets mid-batch, it re-reads these files and resumes from the next unchecked item automatically. This makes it possible to process hundreds of documents across long sessions without losing progress.

## When to Use

Good fit:
- 5+ items requiring significant per-item work
- Sessions likely to run 20+ minutes (high compaction risk)
- Extraction, aggregation, or transformation on bulk data
- Resumption is critical (can't re-process from scratch)

Not needed:
- Small batches (fewer than 5 items)
- One-pass analysis that fits comfortably in context
- No resumption requirement

## Modes

| Mode | How It Works | Best For |
|------|-------------|----------|
| Sequential | Single agent processes items one by one | <= 20 items, simple tasks |
| Parallel | Coordinator spawns N subagents (adaptive, max 5) using work-stealing | > 20 items, time-sensitive work |

Parallel mode calculates N from first-item timing: `N = min(ceil(total_estimated_min / 20), 5)`.

## Usage

Describe your batch task naturally:

> "Analyze all the transcripts in /data/interviews and extract customer pain points."

The skill detects the batch pattern, confirms the three-file setup, asks sequential or parallel, and optionally offers a dry-run preview on the first 3-10 items before committing to the full batch.

## Templates

Ready-to-paste prompts live in the `templates/` directory. Pick the one matching your use case, copy the Prompt section, and paste it into your session.

Available templates:
- `templates/customer-language-extraction.md` — extract emotional language for marketing
- `templates/feature-request-aggregation.md` — aggregate and rank feature requests
- `templates/support-ticket-analysis.md` — identify recurring technical issues
- `templates/generic-document-processing.md` — flexible template for any document type

See `templates/README.md` for usage instructions.

## Installation

```bash
# User scope — available in all sessions (recommended)
npx skills add tima/batch-processing -g

# Project scope — available in this project only
npx skills add tima/batch-processing
```

Target a specific agent:
```bash
npx skills add tima/batch-processing -g -a claude-code
```

Local development install:
```bash
git clone https://github.com/tima/batch-processing.git ~/projects/batch-processing
ln -sf ~/projects/batch-processing ~/.claude/skills/batch-processing
```

### Uninstall

```bash
npx skills remove batch-processing           # project scope
npx skills remove batch-processing --global  # user scope
```

## License

MIT — see [LICENSE](LICENSE).
