# Sequential Mode Regression Test

**Purpose:** Verify sequential mode still works after adding parallel mode.

**Setup:**
1. Create test directory: `mkdir -p /tmp/batch-test-sequential`
2. Create 5 sample transcript files with customer language:

```bash
cd /tmp/batch-test-sequential

cat > transcript_01.txt <<'EOF'
Customer: "This export feature is incredibly frustrating. It times out every single time I try to use it."
Agent: "I understand your frustration. Let me help you with that."
EOF

cat > transcript_02.txt <<'EOF'
Customer: "I'm confused about how the workflow is supposed to work. The documentation doesn't make sense."
Agent: "Let me walk you through it step by step."
EOF

cat > transcript_03.txt <<'EOF'
Customer: "We've been doing this manually for six months and it's killing our productivity."
Agent: "I can see how that would be frustrating."
EOF

cat > transcript_04.txt <<'EOF'
Customer: "The interface is confusing. I can't find where to configure the settings."
Agent: "Let me show you where that is."
EOF

cat > transcript_05.txt <<'EOF'
Customer: "This is causing us a lot of stress. Our team is falling behind because of these issues."
Agent: "I understand how stressful that must be."
EOF
```

**Execution:**
1. Request: "Analyze all transcripts in /tmp/batch-test-sequential. Extract customer pain points showing genuine emotion. Categorize by: Frustration, Confusion, Stress."
2. When asked about three-file system: say "yes"
3. When asked sequential vs parallel: choose "sequential"
4. Observe: AI creates context.md, todos.md, insights.md
5. Wait for completion (should take ~5-10 minutes)

**Expected Results:**

todos.md shows all items checked:
```markdown
- [x] transcript_01.txt
- [x] transcript_02.txt
- [x] transcript_03.txt
- [x] transcript_04.txt
- [x] transcript_05.txt
```

insights.md contains categorized findings:
```markdown
## Frustration
- "This export feature is incredibly frustrating" (transcript_01.txt - export timeout issue)
- "We've been doing this manually for six months and it's killing our productivity" (transcript_03.txt - manual workflow pain)

## Confusion
- "I'm confused about how the workflow is supposed to work" (transcript_02.txt - workflow understanding)
- "The interface is confusing" (transcript_04.txt - settings location)

## Stress
- "This is causing us a lot of stress" (transcript_05.txt - team falling behind)
```

**Success Criteria:**
- All 5 items marked `[x]`
- insights.md has findings organized by category
- Each finding includes source filename and context
- No insights-N.md files created (sequential mode only uses insights.md)
