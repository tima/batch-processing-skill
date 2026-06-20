# Validation

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
