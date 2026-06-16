# Merge Validation Test

**Purpose:** Verify merge correctly combines findings from multiple subagents by category.

**Setup:**
1. Create test directory: `mkdir -p /tmp/batch-test-merge`
2. Create 15 items with mixed categories:

```bash
cd /tmp/batch-test-merge

# Frustration items (1-5)
for i in $(seq 1 5); do
  cat > "item_${i}.txt" <<EOF
Customer: "This is frustrating problem ${i}"
EOF
done

# Confusion items (6-10)
for i in $(seq 6 10); do
  cat > "item_${i}.txt" <<EOF
Customer: "I'm confused about issue ${i}"
EOF
done

# Stress items (11-15)
for i in $(seq 11 15); do
  cat > "item_${i}.txt" <<EOF
Customer: "This is causing stress around problem ${i}"
EOF
done
```

**Execution:**
1. Request: "Analyze all items. Extract quotes and categorize by: Frustration, Confusion, Stress."
2. Choose parallel mode
3. Observe: Spawns 2 subagents (ceil(15/10) = 2)
4. Wait for completion

**Expected Merge Behavior:**

**Before merge:**
- insights-1.md has findings from ~7-8 items (mixed categories)
- insights-2.md has findings from ~7-8 items (mixed categories)

**After merge:**

insights.md has all findings grouped by category:
```markdown
## Frustration
- "This is frustrating problem 1" (item_1.txt)
- "This is frustrating problem 2" (item_2.txt)
- "This is frustrating problem 3" (item_3.txt)
- "This is frustrating problem 4" (item_4.txt)
- "This is frustrating problem 5" (item_5.txt)

## Confusion
- "I'm confused about issue 6" (item_6.txt)
- "I'm confused about issue 7" (item_7.txt)
...
- "I'm confused about issue 10" (item_10.txt)

## Stress
- "This is causing stress around problem 11" (item_11.txt)
...
- "This is causing stress around problem 15" (item_15.txt)
```

**Validation Checks:**
- Count findings per category: Frustration = 5, Confusion = 5, Stress = 5
- Verify no category duplication (each category appears exactly once)
- Verify all findings from insights-1.md and insights-2.md are present
- Verify source filenames preserved for all findings
- Verify intermediate files deleted

**Success Criteria:**
- All 15 findings present in insights.md
- Correct category grouping (5 per category)
- No duplicates within categories
- Source attribution intact
