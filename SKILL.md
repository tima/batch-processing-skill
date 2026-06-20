---
name: batch-processing
description: Use when processing bulk data (files, transcripts, tickets, documents) to extract or transform information, especially across long sessions where context compaction occurs
---

# Batch Processing

## Overview

Batch processing is a structured system for processing multiple data items (files, records, documents) while maintaining progress across context compaction. Instead of holding all results in memory, the batch processor externalizes state to three files: a context file (why you're working), a todos file (what's left), and an insights file (what you've found). When context compacts, the processor re-reads these files and resumes automatically.

**Core principle:** Externalize state to disk so progress survives context reset.

## When to Use

- Processing 5+ items that require significant work per item
- Sessions likely to run 20+ minutes (high compaction risk)
- Extractions, aggregations, transformations on bulk data
- Resuming is critical (you can't re-process everything)

**NOT needed for:**
- Small batches (< 5 items)
- One-pass analysis with no resumption
- Work small enough to fit comfortably in context

## Parallel vs Sequential

The batch-processing skill supports two execution modes:

**Sequential Mode:**
- Single AI processes items one by one
- Simpler coordination, lower resource usage
- Best for: Small batches (<= 20 items), simple tasks, resource-constrained environments
- Time: ~2min per item (10 items = 20min, 20 items = 40min)

**Parallel Mode:**
- Coordinator spawns multiple subagents (adaptive based on first item timing)
- Subagents work-steal from shared todos.md
- Faster completion, higher resource usage
- Best for: Large batches (> 20 items), time-sensitive work, sufficient resources
- Time: Adaptive based on first item - measures first item processing time, estimates total, calculates N
- Formula: `N = min(ceil(estimated_total_minutes / 20), 5)` where 20min is target completion window
- Example: First item takes 3 min → 50 items = 150 min total → N = min(ceil(150/20), 5) = 5 agents → 30 min completion

**Mode Selection:**
At setup, you'll be asked: "Process sequentially or in parallel?"
- Sequential: Simpler, reliable, proven pattern
- Parallel: Faster, more complex coordination, requires subagent support

Choose based on batch size, urgency, and available resources.

## Detection Pattern

When the AI encounters a request like:
- "Analyze all the [files/transcripts/tickets] in [location]"
- "Extract [information] from these [documents]"
- "Process all [items] and find [patterns]"

**Recognition signals:**
- Multiple items (5+) to process
- Extraction/transformation goal
- Items in a directory or collection

**Action:**

1. Recognize this as batch processing

2. Ask user: "This looks like batch processing (5+ items requiring extraction/transformation). Should I set up the three-file system (context.md, todos.md, insights.md) to track progress across context resets?"

3. If user says yes:
   - Count items to process
   - Ask: "Process sequentially (simpler) or in parallel (faster, uses more resources)?"

4. **If sequential:**
   - Offer: "Preview output first (dry-run on first 3 items) or process entire batch?"
   - **If dry-run:**
     - Create context.md with their goal
     - Enumerate files to create todos.md
     - Mark only first 3 items as active: `- [ ] file.txt # DRY-RUN`
     - Create empty insights.md
     - Process first 3 items
     - Show insights.md to user
     - Ask: "Does output format look correct? Adjust extraction rules or continue?"
     - If adjust: update context.md, clear insights.md, reprocess 3 items
     - If continue: remove # DRY-RUN markers, process remaining items
   - **If full batch:**
     - Create context.md with their goal
     - Enumerate files to create todos.md (format: `- [ ] filename`)
     - Create empty insights.md
     - Begin sequential processing loop
     - **Work through all items until complete**

5. **If parallel:**
   - Offer: "Preview output first (dry-run on first 10 items) or process entire batch?"
   - **If dry-run:**
     - Create context.md with their goal
     - Enumerate files to create todos.md
     - Mark only first 10 items as active: `- [ ] file.txt # DRY-RUN`
     - Create empty insights-1.md (dry-run always uses N=1 subagent for preview)
     - Ask merge strategy: "How should findings be organized in final insights.md?"
       - A) By category (Frustration, Confusion, etc.) - groups similar findings
       - B) Chronologically (by file order in todos.md) - preserves sequence
       - C) By frequency (most mentioned first) - highlights common patterns
     - Store choice in context.md for merge step
     - Spawn 1 subagent to process first 10 items
     - Merge to insights.md
     - Show insights.md to user
     - Ask: "Does output format look correct? Adjust extraction rules or continue?"
     - If adjust: update context.md, reset first 10 items to `[ ]`, clear insights-1.md, reprocess
     - If continue: measure first item, estimate total, calculate adaptive N, remove # DRY-RUN markers, create insights-2.md through insights-N.md if needed, spawn N subagents, process all items
   - **If full batch:**
     - Create context.md with their goal
     - Enumerate files to create todos.md (format: `- [ ] filename`)
     - Measure first item: process first item, record time T (minutes)
     - Estimate total: `estimated_total = T * item_count`
     - Calculate N = min(ceil(estimated_total / 20), 5) where 20min is target window
     - If T < 0.5 min: N = 1 (items too fast for parallel overhead)
     - If T > 5 min: N = 5 (items complex enough to always max out)
     - Inform user: "First item took [T] min, estimated [estimated_total] min total, using [N] subagents for ~[estimated_total/N] min completion"
     - Create empty insights-1.md through insights-N.md
     - Ask merge strategy: "How should findings be organized in final insights.md?"
       - A) By category (Frustration, Confusion, etc.) - groups similar findings
       - B) Chronologically (by file order in todos.md) - preserves sequence
       - C) By frequency (most mentioned first) - highlights common patterns
     - Store choice in context.md for merge step
     - Spawn N subagents with work-stealing instructions
     - Monitor progress, merge when complete

6. Verify completion:
   - All todos.md items checked off
   - insights.md populated with extracted data
   - Report results to user

7. **If resuming interrupted parallel batch:**
   - Check for existing todos.md with mix of `[ ]`, `[>]`, and `[x]` items
   - Count remaining: `remaining = count([ ]) + count([>])`
   - Read existing context.md (preserve original extraction rules)
   - Clean up stale `[>]` items (if any subagent IDs no longer active)
   - Calculate adaptive N:
     - If insights-*.md files exist, estimate T from previous work:
       - `avg_time_per_item = total_elapsed / items_completed`
       - `estimated_remaining = avg_time_per_item * remaining`
     - Else measure first remaining item for T
     - Calculate N = min(ceil(estimated_remaining / 20), 5)
     - If T < 0.5: N=1, if T > 5: N=5
   - Inform user: "Resuming parallel batch - [N] subagents for [remaining] remaining items (estimated [estimated_remaining] min)"
   - Identify highest existing insights-N.md number
   - Create additional insights files if N > existing max
   - Spawn N subagents with same work-stealing instructions
   - Monitor and merge as normal

## Quick Reference

**Three files (auto-generated):**
- `context.md` - Your goal and extraction rules
- `todos.md` - Checklist of all items to process
- `insights.md` - Running output file

**Pattern:**
1. Process one item
2. Extract/transform → append to insights.md
3. Check off item in todos.md
4. Repeat until done
5. On context reset: read context.md + todos.md, resume from next unchecked item
6. **Continue until all items are checked off - no stopping until complete**

**Parallel Mode Pattern:**
1. Coordinator spawns N subagents (adaptive: measure first item, target ~20min completion)
2. Each subagent work-steals from todos.md:
   - Find first `[ ]` item → claim as `[>]` → process → append to insights-N.md → mark `[x]`
3. Coordinator monitors with exponential backoff (30s → 60s → 120s → 240s) + milestone checks (25%, 50%, 75%)
4. When all `[x]`: merge insights-1..N.md → insights.md
5. On subagent failure: change `[>]` back to `[ ]`, other agents claim it

**Dry-Run Pattern:**
1. AI offers: "Preview output first or process entire batch?"
2. If preview: processes 3 items (sequential) or 10 items (parallel)
3. Shows insights.md
4. User adjusts rules or continues with full batch

## Autonomous Processing

Once setup is complete, processing runs autonomously:
- You can walk away - the AI continues until all items are processed
- Typical duration: 30 minutes to 2 hours depending on item count and complexity
- Progress checkpoints: todos.md updates after each item
- Natural pauses: Context resets happen automatically, AI resumes from todos.md
- Completion signal: All todos.md items checked off, insights.md populated

No babysitting required. The AI will work through the entire batch and report completion.

## Workflow

### Enumerating Files for todos.md

**Shell/CLI users:**
```bash
# All files of specific type
find . -name "*.txt" -type f

# Files in current directory only
ls *.json

# With full paths
find /path/to/data -name "*.pdf" -type f
```

**Generic approach:**
List all items matching your criteria in the target location. Each item becomes one line in todos.md as `- [ ] filename`.

**todos.md format:**
```markdown
- [ ] transcript_01.txt
- [ ] transcript_02.txt
- [ ] data_report_march.json
```

### Initial Setup (Automatic)

When you request bulk processing, the AI will:

1. **Detect** the batch pattern from your request
2. **Ask** if you want the three-file system
3. **Enumerate** files matching your criteria
4. **Create** three files:
   - `context.md` - your goal and extraction rules
   - `todos.md` - checklist of all items (unchecked)
   - `insights.md` - empty, ready for output

You can also use the ready-to-use templates below - paste one, answer the AI's confirmation, and processing begins.

### Dry-Run Preview Mode

For batches where you're unsure about extraction rules, dry-run mode processes a small sample first.

**Sequential dry-run (3 items):**
1. AI processes first 3 items from todos.md (marked `# DRY-RUN`)
2. Shows insights.md output
3. Asks: "Does this match your expectations? Adjust rules or continue?"
4. If adjust: updates context.md, clears insights.md, reprocesses same 3 items
5. If continue: removes # DRY-RUN markers, processes remaining items

**Parallel dry-run (10 items):**
1. AI spawns 1 subagent to process first 10 items (marked `# DRY-RUN`)
2. Merges to insights.md
3. Shows insights.md output
4. Asks: "Does this match your expectations? Adjust rules or continue?"
5. If adjust: updates context.md, resets first 10 to `[ ]`, reprocesses
6. If continue: removes # DRY-RUN markers, spawns full N subagents, processes all items

**When to use dry-run:**
- First time using batch processing
- Unclear extraction rules
- Complex categorization requirements
- High stakes (can't afford to reprocess entire batch)

### Parallel Processing Workflow

Parallel mode spawns N subagents (adaptive count based on first-item timing) that work-steal from a shared todos.md. Each subagent claims items atomically (`[>]`), writes to its own insights-N.md, and marks complete (`[x]`). The coordinator monitors with exponential backoff and milestone checks, then merges all insights files when done. Failed subagents have their claimed items automatically returned to the queue.

For the full parallel workflow steps, see `references/parallel-workflow.md`

### Processing Loop

**For each unchecked item in todos.md:**

1. **Read** the file/data completely
2. **Extract/transform** entirely based on rules in context.md (finish the whole item)
3. **Append to insights.md** (with source file name and brief context)
4. **Update todos.md** immediately - check off the item (`- [x] filename`)
5. **Repeat**

**Critical ordering:** Extract → Append → Check off. Never skip a step.

**IMPORTANT:** Continue this loop until every item in todos.md is checked off. Do not stop early, do not ask for permission to continue, do not pause unless there's an error. Work through the complete batch.

**Early Validation Checkpoint (recommended):**

After processing first 10 items:
1. Pause loop
2. Validate 3 random items from insights.md
3. If errors found: stop, update context.md, reset todos.md, restart
4. If validation passes: continue processing

This catches systematic errors early before wasting work on entire batch.

### Handling Context Compaction

When context resets automatically:

1. **Read context.md first** (reorient to goal + extraction rules)
2. **Read todos.md second** (identify first unchecked item)
3. **Read insights.md** (verify your previous output is intact)
4. **Resume** from the first unchecked item - process it completely
5. **Update todos.md** immediately after finishing
6. **Repeat**

**Never skip reading context.md** - it anchors your extraction rules. Without it, you'll drift or miss items.

## Red Flags - STOP and Check Yourself

| Mistake | Why It Breaks | Fix |
|---------|---------------|-----|
| Check off BEFORE extracting | Item marked done but extraction lost = unrecoverable | Extract FIRST. Check off LAST. Never reverse. |
| Updating todos.md late | If reset happens between extract and update, context loses progress | Update todos.md IMMEDIATELY after appending to insights.md |
| Partial extraction | File processing "split" across two sessions = inconsistent extraction | Finish each file completely before moving on. Never resume mid-file. |
| Skipping context.md on resume | Without goals, extraction rules drift = output quality drops | ALWAYS read context.md after reset, BEFORE processing next item. |
| Forgetting source info in insights.md | Findings become unverifiable and unlinked | Every insight must include filename and 1-2 sentence context. |
| Multiple formats simultaneously | Processing transcripts AND emails AND tickets = confusion about extraction rules | One batch type per session. Start fresh context.md for next type. |
| "Just starting" without three files | No way to resume if anything happens | Always create context.md + todos.md + insights.md before processing item 1. |

## Common Mistakes

**These WILL cause lost work:**

**Checking off BEFORE extracting** - Mark done, then context resets, then item vanishes forever. Extract → Append → Check off. That order is not flexible.

**Updating todos.md late** - If reset happens between appending to insights.md and updating todos.md, you lose the progress checkpoint. Update immediately after appending.

**Processing partial items** - "I'll finish file 5 after reset." Context resets. You don't remember where you stopped in file 5. Extraction becomes incomplete or duplicated. Process entire items only.

**Skipping context.md on resume** - You remember "I was on file 5" but forget "I'm extracting pain points in Frustration category." Extraction drifts. Read context.md FIRST on every resume.

**Appending without source info** - insights.md grows but becomes unverifiable. Trace which findings came from which file. Always include filename and brief context.

**Trying to batch multiple formats** - Processing transcripts AND emails AND tickets in one session = different extraction rules = confused output. One batch type per session.

## Ready-to-Use Templates

Templates moved to `templates/` directory for token efficiency. Each template includes business context, usage guidance, and complete ready-to-paste prompt.

**Available templates:**
- `templates/customer-language-extraction.md` - Extract emotional language for marketing
- `templates/feature-request-aggregation.md` - Aggregate and rank feature requests
- `templates/support-ticket-analysis.md` - Identify recurring technical issues
- `templates/generic-document-processing.md` - Flexible template for any document type

See `templates/README.md` for detailed usage instructions.

**Template structure:**
- Business Context: Why you'd use this template
- When to Use: Triggering conditions
- Prompt: Complete ready-to-paste prompt with extraction rules and output format

**Using templates:**
1. Read template file matching your use case
2. Copy the Prompt section
3. Paste into AI session
4. AI offers dry-run mode (recommended first-time)
5. Choose sequential or parallel mode
6. Choose merge strategy (if parallel)
7. AI processes batch autonomously

## Validation

Validation uses stratified sampling scaled to batch size: 3 items for small batches (5-20), 10% sample for medium (21-50), 5% for large (50+). An early detection checkpoint after the first 10 items catches systematic errors before they propagate through the full batch. Parallel mode adds per-subagent quality checks before and after the merge step.

For validation steps, see `references/validation.md`

## Troubleshooting

For troubleshooting, see `references/troubleshooting.md`

## Why This Works

**Externalization:** Each file represents one layer of state:
- `context.md` preserves *why* across resets
- `todos.md` preserves *where* across resets  
- `insights.md` preserves *what* across resets

**Resumption:** On compaction, reading these two files (context + todos) takes <100 tokens. Context window fills with new content, not old. You resume within 30 seconds.

**Checkpointing:** Updating todos.md after each item creates natural checkpoints. If you crash mid-item, the unchecked item tells you exactly where to retry.

**Scalability:** Works for 5 items or 500. Same pattern, same files. Only insights.md grows.
