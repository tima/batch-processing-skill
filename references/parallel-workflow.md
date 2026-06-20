# Parallel Processing Workflow

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
