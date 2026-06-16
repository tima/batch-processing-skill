# Parallel Mode Max Agents Test (50 Items)

**Purpose:** Verify parallel mode caps at 5 subagents, handles load balancing, and merges large batch correctly.

**Setup:**
1. Create test directory: `mkdir -p /tmp/batch-test-max-agents`
2. Create 50 sample files:

```bash
cd /tmp/batch-test-max-agents

for i in $(seq -f "%02g" 1 50); do
  cat > "item_${i}.txt" <<EOF
Feature request ${i}: "I wish the system had feature ${i}"
Priority: $((i % 3 == 0 ? "high" : "medium"))
EOF
done
```

**Execution:**
1. Request: "Analyze all items in /tmp/batch-test-max-agents. Extract feature requests with priority."
2. When asked about three-file system: say "yes"
3. When asked sequential vs parallel: choose "parallel"
4. Observe: AI should report "Using 5 subagents for 50 items" (ceil(50/10) = 5, capped at max)
5. Watch progress reports
6. Wait for completion

**Expected Results:**

**During processing:**
- Progress reports: "10/50 complete (5 in-progress)", "25/50 complete (4 in-progress)", etc.
- Five insights files exist: insights-1.md through insights-5.md
- Some subagents finish before others (load balancing)

**After completion:**

todos.md shows all 50 items checked:
```markdown
- [x] item_01.txt
- [x] item_02.txt
...
- [x] item_50.txt
```

insights.md contains all 50 requests organized by priority:
```markdown
## Feature Requests
- "I wish the system had feature 1" (item_01.txt - Priority: medium)
- "I wish the system had feature 2" (item_02.txt - Priority: medium)
- "I wish the system had feature 3" (item_03.txt - Priority: high)
...
- "I wish the system had feature 50" (item_50.txt - Priority: medium)
```

insights-1.md through insights-5.md deleted.

**Success Criteria:**
- Exactly 5 subagents spawned (at max limit)
- All 50 items marked `[x]`
- insights.md has all 50 findings
- No duplicate findings
- Intermediate files cleaned up
- Some subagents finish before others (demonstrates load balancing)
