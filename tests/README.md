# Batch Processing Skill Tests

## Test Suite Overview

These tests validate both sequential and parallel modes of the batch-processing skill.

## Sequential Mode Tests

- **sequential-regression-test.md** - Verifies sequential mode unchanged after adding parallel

## Parallel Mode Tests

- **parallel-basic-test.md** - Basic parallel execution with 25 items (3 subagents)
- **parallel-max-agents-test.md** - Max agent cap with 50 items (5 subagents)
- **race-condition-test.md** - Verifies claiming protocol prevents duplicate work
- **failure-recovery-test.md** - Subagent crash recovery and work resumption
- **merge-validation-test.md** - Category-based merge correctness

## Running Tests

All tests are manual validation tests that exercise the skill in real scenarios.

**Setup:** Each test creates sample data in `/tmp/batch-test-*` directories.

**Execution:** Follow test steps, observe AI behavior, verify expected results.

**Success Criteria:** Each test documents specific outcomes to verify.

## Test Coverage

- PASS Sequential mode regression (backward compatibility)
- PASS Parallel mode basic functionality
- PASS Subagent count calculation (1 per 10 items, max 5)
- PASS Work-stealing coordination
- PASS Race condition prevention
- PASS Failure recovery
- PASS Multi-category merge
- PASS Progress monitoring
- PASS Cleanup (intermediate files deleted)
