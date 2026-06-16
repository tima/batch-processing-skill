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

## Detection Pattern

When you (Claude) encounter a request like:
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
3. If yes:
   - Create context.md with their goal
   - Enumerate files to create todos.md
   - Create empty insights.md
   - Begin processing loop

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

When you request bulk processing, Claude will:

1. **Detect** the batch pattern from your request
2. **Ask** if you want the three-file system
3. **Enumerate** files matching your criteria
4. **Create** three files:
   - `context.md` - your goal and extraction rules
   - `todos.md` - checklist of all items (unchecked)
   - `insights.md` - empty, ready for output

You can also use the ready-to-use templates below - paste one, answer Claude's confirmation, and processing begins.

### Processing Loop

**For each unchecked item in todos.md:**

1. **Read** the file/data completely
2. **Extract/transform** entirely based on rules in context.md (finish the whole item)
3. **Append to insights.md** (with source file name and brief context)
4. **Update todos.md** immediately - check off the item (`- [x] filename`)
5. **Repeat**

**Critical ordering:** Extract → Append → Check off. Never skip a step.

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

### Template 1: Customer Language Extraction

Use when: Extracting emotional language from sales calls, support transcripts, or customer conversations for marketing copy.

```
GOAL:
Analyze all transcripts in this directory. Extract phrases where customers describe problems, frustrations, stress, confusion, or pain points. Only include language showing genuine emotion (tone shifts, repeated mentions, explicit statements like "this is frustrating").

SETUP:
[Claude will ask if you want three-file system - say yes]

EXTRACTION RULES:
- Only extract direct quotes (no paraphrasing)
- Only statements tied to visible emotion
- Categorize by: Frustration, Fear, Confusion, Stress, Pain Points
- Include multiple categories if quote fits both
- Ignore competitor discussion unless it reveals product gaps
- Ignore feature requests

OUTPUT FORMAT (insights.md):
Organize by category:
## Frustration
- "We've been manually doing this for six months" (transcript_042.txt - data entry workflow)

Work until all todos.md items are checked off.
```

### Template 2: Feature Request Aggregation

Use when: Collecting product/service feature requests from customer conversations or feedback.

```
GOAL:
Analyze all transcripts/documents. Extract feature requests, suggestions, or "I wish..." statements from customers.

SETUP:
[Claude will ask if you want three-file system - say yes]

EXTRACTION RULES:
- Extract requests, suggestions, wishes, or improvement ideas
- Include exact quote and context
- Note if mentioned by multiple customers (track frequency)
- Categorize by product area if clear
- Distinguish "must-have" from "nice-to-have" based on customer language

OUTPUT FORMAT (insights.md):
Organize by frequency or product area:
## High Frequency (3+ mentions)
- "Bulk export feature" (ticket_023.txt, ticket_091.txt, transcript_15.txt - all mentioned CSV export)

Work until all todos.md items are checked off.
```

### Template 3: Support Ticket Analysis

Use when: Analyzing support tickets for recurring issues, root causes, or severity patterns.

```
GOAL:
Analyze all support tickets. Extract recurring issues, error patterns, and root causes. Categorize by severity and product area.

SETUP:
[Claude will ask if you want three-file system - say yes]

EXTRACTION RULES:
- Identify recurring error messages or failure patterns
- Extract root cause if mentioned
- Note workarounds used
- Categorize severity: Critical (blocking), High (workflow impact), Medium (friction)
- Track product area affected

OUTPUT FORMAT (insights.md):
Organize by severity and area:
## Critical - Authentication
- Login timeout after 2FA (ticket_034.txt - root cause: session timeout = 30s, 2FA takes 45s)

Work until all todos.md items are checked off.
```

### Template 4: Generic Document Processing

Use when: Processing any bulk documents for specific information extraction.

```
GOAL:
[Describe what you want to extract and from what type of documents]

SETUP:
[Claude will ask if you want three-file system - say yes]

EXTRACTION RULES:
[List your specific criteria - what to extract, what to ignore, how to categorize]

OUTPUT FORMAT (insights.md):
[Describe how findings should be organized]

Work until all todos.md items are checked off.
```

## Validation

**Completion checks (automatic):**
- All items in todos.md checked off
- insights.md has entries
- Each insight includes source filename

**Reported at completion:**
- Items processed: X/Y
- Output location: insights.md
- Time elapsed (if available)

**Optional quality checks:**
- Spot-check 2-3 items: re-read source, verify extraction matches rules
- Check insights.md format consistency
- Verify categorization is consistent

## Why This Works

**Externalization:** Each file represents one layer of state:
- `context.md` preserves *why* across resets
- `todos.md` preserves *where* across resets  
- `insights.md` preserves *what* across resets

**Resumption:** On compaction, reading these two files (context + todos) takes <100 tokens. Context window fills with new content, not old. You resume within 30 seconds.

**Checkpointing:** Updating todos.md after each item creates natural checkpoints. If you crash mid-item, the unchecked item tells you exactly where to retry.

**Scalability:** Works for 5 items or 500. Same pattern, same files. Only insights.md grows.
