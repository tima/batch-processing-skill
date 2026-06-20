# Troubleshooting

## Parallel Mode Issues

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
2. Clean stale `[>]` claims (change to `[ ]`)
3. Calculate adaptive N:
   - If insights-*.md files exist, estimate T from previous work:
     - `avg_time_per_item = total_elapsed / items_completed`
     - `estimated_remaining = avg_time_per_item * remaining`
   - Else measure first remaining item for T
   - Calculate N = min(ceil(estimated_remaining / 20), 5)
   - If T < 0.5: N=1, if T > 5: N=5
4. Identify max existing insights-N.md number
5. Create additional insights files if needed (insights-M.md where M > max)
6. Tell AI: "Resume parallel batch processing - read context.md and todos.md, spawn [N] subagents"
7. AI will spawn subagents, work-steal remaining items, merge all insights-*.md

## Session Interrupted (You Stopped Mid-Batch)

**Symptom:** You stopped the AI mid-processing and want to resume later.

**Solution:**
1. Check todos.md - find the first unchecked item
2. Tell the AI: "Resume batch processing from todos.md - continue from the first unchecked item"
3. The AI will read context.md + todos.md and continue

**No data lost** - todos.md checkpoint shows exactly where to resume.

## AI Stopped Before Completion

**Symptom:** AI stopped processing but items remain unchecked in todos.md.

**Possible causes:**
- Permission prompt interrupted flow
- Error reading a file
- Context reset happened but AI didn't auto-resume

**Solution:**
1. Check todos.md - identify what's left
2. Check insights.md - verify previous work is intact
3. Tell the AI: "Continue batch processing - read context.md and todos.md, then resume"

## Partial File Processing

**Symptom:** You suspect a file was only partially processed before context reset.

**Check:**
1. Look in todos.md - is the file checked off?
   - Checked off = fully processed
   - Not checked off = not processed (safe to process now)

**If somehow partial extraction happened:**
1. Manually uncheck the item in todos.md: change `- [x]` to `- [ ]`
2. Tell the AI to reprocess that specific file
3. Review insights.md to remove duplicate/partial entries

## Files Added After Starting

**Symptom:** New files appeared in directory after batch processing started.

**Solution:**
1. Add new files to todos.md manually as unchecked items: `- [ ] new_file.txt`
2. Tell the AI: "Continue processing - new items added to todos.md"
3. The AI will process them in sequence

## Quality Issues in Output

**Symptom:** insights.md contains low-quality extractions or missed items.

**Check:**
1. Read context.md - are extraction rules clear and specific?
2. Spot-check 2-3 files against insights.md entries
3. Identify what's wrong (too broad, too narrow, wrong category)

**Solution:**
1. Update context.md with clearer extraction rules
2. Mark problematic files as unchecked in todos.md
3. Tell the AI: "Re-read context.md (updated rules) and reprocess unchecked items"
