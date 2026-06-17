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
     - Measure first item: process first item, record time T (minutes)
     - Estimate total: `estimated_total = T * item_count`
     - Calculate N = min(ceil(estimated_total / 20), 5) where 20min is target window
     - If T < 0.5 min: N = 1 (items too fast for parallel overhead)
     - If T > 5 min: N = 5 (items complex enough to always max out)
     - Create empty insights-1.md through insights-N.md
     - Ask merge strategy: "How should findings be organized in final insights.md?"
       - A) By category (Frustration, Confusion, etc.) - groups similar findings
       - B) Chronologically (by file order in todos.md) - preserves sequence
       - C) By frequency (most mentioned first) - highlights common patterns
     - Store choice in context.md for merge step
     - Spawn N subagents to process first 10 items
     - Merge to insights.md
     - Show insights.md to user
     - Inform user: "First item took [T] min, estimated [estimated_total] min total, using [N] subagents for ~[estimated_total/N] min completion"
     - Ask: "Does output format look correct? Adjust extraction rules or continue?"
     - If adjust: update context.md, reset first 10 items to `[ ]`, clear insights-*.md, reprocess
     - If continue: remove # DRY-RUN markers, spawn remaining subagents, process all items
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
   - Calculate N = min(ceil(remaining / 10), 5)
   - Inform user: "Resuming parallel batch - [N] subagents for [remaining] remaining items"
   - Read existing context.md (preserve original extraction rules)
   - Clean up stale `[>]` items (if any subagent IDs no longer active)
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
1. Coordinator spawns N subagents (1 per 10 items, max 5)
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

When parallel mode is chosen, the workflow differs from sequential:

**Coordinator Role:**
- Measures first item processing time T
- Calculates N adaptively: `N = min(ceil(T * item_count / 20), 5)`
- Special cases: T < 0.5 → N=1, T > 5 → N=5
- Spawns N subagents
- Monitors progress with exponential backoff (30s, 60s, 120s, 240s max)
- Checks at milestones (25%, 50%, 75%) regardless of backoff
- Reports aggregate completion: "X/Y complete (Z% - P in-progress)"
- Detects subagent failures and recovers work
- Merges insights when all items complete

**Subagent Role:**
- Reads context.md once at start (extraction rules)
- Executes work-stealing loop until no items remain
- Writes findings to isolated insights-N.md
- Reports completion when done
- insights-N.md visible to user immediately (no wait for merge)
- User can monitor quality by reading any insights-N.md at any time
- Allows early error detection without stopping processing

**Visibility:** insights-N.md files are immediately readable during processing. User can monitor quality at any time without waiting for merge completion.

**Work-Stealing Protocol:**

To prevent duplicate work, todos.md items have three states:

Note: `[>]` is custom syntax for this skill (not standard markdown checkbox).

```markdown
- [ ] transcript_01.txt              # Available - unclaimed
- [>] transcript_02.txt (agent-2)    # In-progress by subagent 2
- [x] transcript_03.txt              # Complete
```

**Claiming process:**
1. Subagent reads todos.md
2. Finds first `[ ]` item (skips `[>]` and `[x]`)
3. Immediately changes to `[>] item.txt (agent-N)` - atomic claim
4. Processes item per context.md rules
5. Appends findings to insights-N.md (includes source filename + context)
6. Changes to `[x]` - marks complete
7. Repeats until no `[ ]` items remain

This eliminates race conditions - only one subagent can claim each item.

**Progress Monitoring:**

Coordinator polls todos.md with exponential backoff (reduces token waste):
- Initial: 30s after spawn
- Then: 60s, 120s, 240s (doubles each time, max 240s = 4min)
- Milestone checks: 25%, 50%, 75% completion (always report regardless of backoff)
- Final: When all items `[x]`

Backoff timer and milestone checks are independent - coordinator checks occur at BOTH scheduled backoff intervals AND whenever a milestone percentage is reached, resulting in additional checks beyond the backoff schedule if milestones don't align with those intervals.

Example for 50-item batch (~20min total):
- 30s: "5/50 complete (10%)" - first check
- 90s: "15/50 complete (30%)" - 25% milestone
- 210s: "25/50 complete (50%)" - 50% milestone  
- 450s: "38/50 complete (76%)" - 75% milestone
- 690s: "50/50 complete (100%)" - completion

Reports: "X/Y items complete (Z% - P in-progress)"

**Why backoff:** Early phase has high activity (many claims), late phase is predictable. Checking every 30s for 20min = 40 checks. Backoff = ~8 checks with milestone coverage.

**Merge Step:**

When all items are `[x]`:
1. Coordinator waits for all subagents to finish
2. Reads context.md for merge strategy choice
3. Reads insights-1.md through insights-N.md
4. **Applies chosen merge strategy:**

   **A) Category-based (default):**
   - Groups findings by category (Frustration, Confusion, etc.)
   - Writes insights.md with category headers
   - Findings organized under relevant categories from all subagents

   **B) Chronological:**
   - Groups findings by source file order (from todos.md)
   - Writes insights.md with file-based headers: "## Findings from file_01.txt"
   - Preserves processing sequence across all subagents

   **C) Frequency-based:**
   - Counts mentions of each finding across insights-*.md files
   - Ranks by frequency (most mentioned first)
   - Writes insights.md with frequency headers: "## High Frequency (5+ mentions)"
   - Includes source files for each finding

5. Deletes intermediate insights-*.md files
6. Reports completion: "Processed Y items, output in insights.md (organized by [strategy])"

**Example merge (category-based):**

```markdown
## Frustration (from insights-1.md, insights-3.md, insights-5.md)
- "We've been doing this manually for months" (file_01.txt - workflow pain)
- "System times out every time" (file_15.txt - export issue)

## Confusion (from insights-2.md, insights-4.md)
- "Why does it work this way?" (file_07.txt - questioning logic)
```

**Example merge (chronological):**

```markdown
## Findings from file_01.txt
- "We've been doing this manually for months" (insights-1.md - Frustration)

## Findings from file_07.txt
- "Why does it work this way?" (insights-2.md - Confusion)

## Findings from file_15.txt
- "System times out every time" (insights-3.md - Frustration)
```

**Example merge (frequency-based):**

```markdown
## High Frequency (3+ mentions)
- "System times out" (file_15.txt, file_23.txt, file_41.txt - export issue mentioned by 3 customers)

## Medium Frequency (2 mentions)
- "Confusing navigation" (file_07.txt, file_19.txt - UI issue)
```

**Failure Recovery:**

If a subagent crashes:
1. Coordinator detects failure
2. Finds all `[>] item.txt (agent-N)` entries for that subagent
3. Changes back to `[ ]` (available for retry)
4. Other running subagents claim and process these items
5. No data lost, minimal delay

**Incremental Results Visibility:**

Unlike sequential mode (where insights.md updates after each item), parallel mode creates isolated insights-N.md files per subagent. These files are immediately readable - no need to wait for merge.

**Monitoring during processing:**
1. List insights files: `ls insights-*.md`
2. Read any subagent's progress: `cat insights-3.md`
3. Check quality of findings
4. If errors found in insights-N.md:
   - Identify which subagent (N)
   - Check todos.md for items claimed by agent-N
   - Stop that subagent if possible
   - Update context.md with clearer rules
   - Reset agent-N's items to `[ ]` for retry

**Benefits:**
- Early quality detection (don't wait for all 50 items to finish)
- Per-subagent quality check (identify problematic extraction patterns)
- User can track progress by checking file sizes: `ls -lh insights-*.md`

**Resuming Interrupted Parallel Batch:**

If coordinator crashes or user stops processing mid-batch:

1. **Check state:**
   - todos.md exists with mix of states
   - context.md exists (original extraction rules)
   - Some insights-N.md files exist (partial work)

2. **Calculate remaining work:**
   ```bash
   # Count unclaimed and in-progress items
   remaining=$(grep -c '^\- \[ \]' todos.md)
   in_progress=$(grep -c '^\- \[>\]' todos.md)
   total_remaining=$((remaining + in_progress))
   ```

3. **Clean stale claims:**
   - Any `[>] item.txt (agent-N)` where agent N is not running
   - Change back to `[ ]` for retry

4. **Resume with adaptive calculation:**
   - If insights-*.md files exist, estimate T from previous work:
     - `avg_time_per_item = total_elapsed / items_completed`
     - `estimated_remaining = avg_time_per_item * total_remaining`
   - Else measure first remaining item for T
   - Calculate N = min(ceil(estimated_remaining / 20), 5)
   - If T < 0.5: N=1, if T > 5: N=5
   - Identify max existing insights-N.md number
   - Create additional insights-M.md files if N > max (empty)
   - Spawn N subagents
   - Continue work-stealing from current todos.md state
   - Merge all insights-*.md at completion (including partial files from first run)

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

**Completion checks (automatic):**
- All items in todos.md checked off
- insights.md has entries
- Each insight includes source filename

**Validation Sampling Strategy:**

For batches > 20 items, spot-checking 2-3 items is insufficient. Use stratified sampling:

**Small batches (5-20 items):**
- Verify: 3 items (15-60% coverage)
- Select: First, middle, last items
- Check: Extraction matches context.md rules, categorization correct, source attribution present

**Medium batches (21-50 items):**
- Verify: 10% random sample (min 5 items)
- Select: Random items across batch (use `shuf` or random.org)
- Check: Same as small batches + category distribution matches expectations

**Large batches (50+ items):**
- Verify: 5% random sample (min 10 items)
- Select: Stratified random (2 items per category minimum)
- Check: Same as medium + no systematic errors within categories

**Early Detection (recommended):**

After first 10 items processed in any batch:
1. **Pause and validate:** Check 3 random items from first 10
2. **Look for systematic errors:**
   - Wrong category assignments
   - Missed extractions (too narrow)
   - Over-extraction (too broad)
   - Source attribution missing
3. **If errors found:**
   - STOP processing
   - Update context.md with clearer rules
   - Reset all items to `[ ]` in todos.md
   - Clear insights.md (or insights-*.md for parallel)
   - Restart with corrected rules
4. **If validation passes:** Continue processing remaining items

**Parallel mode early detection:**

After first 10 items complete (check `grep -c '^\- \[x\]' todos.md`):
1. List insights files: `ls insights-*.md`
2. Read 2-3 insights-N.md files randomly
3. Validate findings per standard checks
4. If errors found: stop processing, update context.md, reset todos.md, restart

**Why early detection matters:** 5% error rate on 100 items = 5 wasted items if caught early, 95 wasted items if caught at end.

**Reported at completion:**
- Items processed: X/Y
- Output location: insights.md
- Time elapsed (if available)
- Validation sample: "Checked N items ([percentage]%), [errors] errors found"

**Parallel Mode Additional Checks:**

**Before merge cleanup:**
- Verify insights-1.md through insights-N.md all exist
- Sample 1 finding from each insights-N.md (check quality per subagent)
- Verify each insights-N.md includes source filenames

**After merge:**
- Verify insights.md exists and has content from all subagents
- Count findings per category - should match sum of all insights-N.md
- Verify no category duplication (each category appears once)
- Verify intermediate files (insights-1.md through insights-N.md) are deleted

**If merge validation fails:**
- Intermediate files should still exist (not deleted prematurely)
- Re-run merge manually or with coordinator

## Troubleshooting

### Parallel Mode Issues

**Subagent Stuck in [>] State**

**Symptom:** Item shows `[>] item.txt (agent-N)` but subagent N is dead.

**Solution:**
1. Manually change `[>]` back to `[ ]` in todos.md
2. Running subagents will claim and process it
3. If no subagents running, restart parallel batch processing (it will skip `[x]` items)

**Duplicate Findings After Merge**

**Symptom:** Same quote appears twice in insights.md from different sources.

**Cause:** Two items contained the same quote (expected) OR race condition (rare).

**Solution:**
1. Check source filenames - if different files, duplicates are legitimate
2. If same file, race condition occurred - manually deduplicate
3. Re-run batch for affected items if needed

**Merge Failed Mid-Process**

**Symptom:** All todos.md items are `[x]` but insights.md missing or incomplete.

**State:** insights-1.md through insights-N.md still exist.

**Recovery:**
1. Read each insights-N.md file
2. Manually combine by category into insights.md, OR
3. Create simple script to merge:

```python
import glob

findings = {}
for file in glob.glob("insights-*.md"):
    with open(file) as f:
        content = f.read()
    # Parse by category headers (##)
    # Append to findings[category]
    
with open("insights.md", "w") as out:
    for category, items in findings.items():
        out.write(f"## {category}\n")
        for item in items:
            out.write(f"{item}\n")
```

**Uneven Load Distribution**

**Symptom:** Some subagents finish quickly, others still processing.

**Cause:** Item complexity varies (some files longer/more complex).

**Expected:** Work-stealing handles this - fast subagents finish early, slow ones keep working.

**Not a problem** unless one subagent is stuck (see "Subagent Stuck" above).

**Progress Monitoring Shows Wrong Count**

**Symptom:** Coordinator reports "30/50 complete" but you see different counts.

**Cause:** todos.md read at different time, or subagent just updated.

**Solution:**
- Wait for next 30s update - counts will sync
- Manually count `[x]` items in todos.md to verify

**Resume After Coordinator Crash**

**Symptom:** Coordinator crashed mid-parallel batch, some items complete, some in-progress, some unclaimed.

**State:**
- todos.md has mix of `[ ]`, `[>]`, `[x]` items
- context.md exists
- Some insights-N.md files exist

**Recovery:**
1. Count remaining: `remaining = count([ ]) + count([>])`
2. Calculate N = min(ceil(remaining / 10), 5)
3. Clean stale `[>]` claims (change to `[ ]`)
4. Identify max existing insights-N.md number
5. Create additional insights files if needed (insights-M.md where M > max)
6. Tell AI: "Resume parallel batch processing - read context.md and todos.md, spawn [N] subagents"
7. AI will spawn subagents, work-steal remaining items, merge all insights-*.md

### Session Interrupted (You Stopped Mid-Batch)

**Symptom:** You stopped the AI mid-processing and want to resume later.

**Solution:**
1. Check todos.md - find the first unchecked item
2. Tell the AI: "Resume batch processing from todos.md - continue from the first unchecked item"
3. The AI will read context.md + todos.md and continue

**No data lost** - todos.md checkpoint shows exactly where to resume.

### AI Stopped Before Completion

**Symptom:** AI stopped processing but items remain unchecked in todos.md.

**Possible causes:**
- Permission prompt interrupted flow
- Error reading a file
- Context reset happened but AI didn't auto-resume

**Solution:**
1. Check todos.md - identify what's left
2. Check insights.md - verify previous work is intact
3. Tell the AI: "Continue batch processing - read context.md and todos.md, then resume"

### Partial File Processing

**Symptom:** You suspect a file was only partially processed before context reset.

**Check:**
1. Look in todos.md - is the file checked off?
   - Checked off = fully processed
   - Not checked off = not processed (safe to process now)

**If somehow partial extraction happened:**
1. Manually uncheck the item in todos.md: change `- [x]` to `- [ ]`
2. Tell the AI to reprocess that specific file
3. Review insights.md to remove duplicate/partial entries

### Files Added After Starting

**Symptom:** New files appeared in directory after batch processing started.

**Solution:**
1. Add new files to todos.md manually as unchecked items: `- [ ] new_file.txt`
2. Tell the AI: "Continue processing - new items added to todos.md"
3. The AI will process them in sequence

### Quality Issues in Output

**Symptom:** insights.md contains low-quality extractions or missed items.

**Check:**
1. Read context.md - are extraction rules clear and specific?
2. Spot-check 2-3 files against insights.md entries
3. Identify what's wrong (too broad, too narrow, wrong category)

**Solution:**
1. Update context.md with clearer extraction rules
2. Mark problematic files as unchecked in todos.md
3. Tell the AI: "Re-read context.md (updated rules) and reprocess unchecked items"

## Why This Works

**Externalization:** Each file represents one layer of state:
- `context.md` preserves *why* across resets
- `todos.md` preserves *where* across resets  
- `insights.md` preserves *what* across resets

**Resumption:** On compaction, reading these two files (context + todos) takes <100 tokens. Context window fills with new content, not old. You resume within 30 seconds.

**Checkpointing:** Updating todos.md after each item creates natural checkpoints. If you crash mid-item, the unchecked item tells you exactly where to retry.

**Scalability:** Works for 5 items or 500. Same pattern, same files. Only insights.md grows.
