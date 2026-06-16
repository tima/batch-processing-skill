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

### Initial Setup

Invoke the skill with your goal and extraction rules:
```
/batch-process "Extract customer pain points from all transcripts. Only include language tied to genuine emotion (tone shifts, repeated mentions, explicit statements). Categorize by: Frustration, Confusion, Stress."
```

The skill auto-generates:
- `context.md` with your goal
- `todos.md` with enumerated list of all files to process
- `insights.md` (empty, ready for output)

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

## Example Prompt

This template works for any bulk processing task. Adjust the extraction rules for your use case.

```
GOAL:
Analyze all the customer support tickets in this folder.
Extract issues that caused customer frustration, confusion, or unmet needs.
Categorize by severity: Critical (product blocking), High (workflow impact), Medium (minor friction).

BEFORE YOU START:
Run: /batch-process

This generates:
- context.md: your goal and rules
- todos.md: enumerated list of all tickets
- insights.md: where extracted issues go

AS YOU WORK:
- For each unchecked item in todos.md:
  1. Read the file
  2. Extract issues matching the criteria
  3. Append to insights.md with: severity, issue description, source ticket name
  4. Check off the item in todos.md
  5. Move to next item

CRITICAL: Update todos.md BEFORE context resets.

Work through all items in todos.md until every line is checked off.
When done, open insights.md for the complete output.
```

## Validation

When done, verify:
- All items in todos.md are checked off
- insights.md has entries for every processed item
- Each insight includes source filename

(Optional: spot-check 2-3 items by re-reading the source to verify extraction quality)

## Why This Works

**Externalization:** Each file represents one layer of state:
- `context.md` preserves *why* across resets
- `todos.md` preserves *where* across resets  
- `insights.md` preserves *what* across resets

**Resumption:** On compaction, reading these two files (context + todos) takes <100 tokens. Context window fills with new content, not old. You resume within 30 seconds.

**Checkpointing:** Updating todos.md after each item creates natural checkpoints. If you crash mid-item, the unchecked item tells you exactly where to retry.

**Scalability:** Works for 5 items or 500. Same pattern, same files. Only insights.md grows.
