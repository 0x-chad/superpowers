---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

**Prefer workmux:** If `workmux` is available, use it instead of raw git commands - it handles merge, branch deletion, and worktree cleanup in one step.

**Running from inside a worktree:** If your current working directory is inside the worktree being finished, do NOT delete the worktree. Deleting it while Claude is running inside it will terminate the session. Only merge to main; skip worktree removal.

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 3: Present Options

Present exactly these 4 options:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 4: Execute Choice

#### Option 1: Merge Locally

**If running from inside the worktree:** Only merge, don't delete the worktree.
```bash
git checkout <base-branch>
git pull
git merge <feature-branch>
<test command>  # Verify tests on merged result
# Skip worktree removal - Claude is running inside it
```

**With workmux (when NOT inside the worktree):**
```bash
workmux merge <feature-branch>
```
This merges to base branch, deletes the branch, and removes the worktree in one command.

**Without workmux (when NOT inside the worktree):**
```bash
git checkout <base-branch>
git pull
git merge <feature-branch>
<test command>  # Verify tests on merged result
git worktree remove <worktree-path>
git branch -d <feature-branch>
```

#### Option 2: Push and Create PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

Then: Cleanup worktree (Step 5)

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

**If running from inside the worktree:** Only delete the branch, not the worktree.
```bash
git checkout <base-branch>
git branch -D <feature-branch>
# Skip worktree removal - Claude is running inside it
```

**With workmux (when NOT inside the worktree):**
```bash
workmux remove <feature-branch>
```

**Without workmux (when NOT inside the worktree):**
```bash
git checkout <base-branch>
git worktree remove <worktree-path>
git branch -D <feature-branch>
```

### Step 5: Cleanup Worktree (only if not using workmux)

**Skip this step if you used `workmux merge` or `workmux remove`** - they handle cleanup automatically.

**Skip this step if running from inside the worktree** - deleting it would terminate the session.

**For Options 1, 2, 4 (manual cleanup):**
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ | ✓ | - |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

## Common Mistakes

**Using raw git commands instead of workmux**
- **Problem:** Multiple manual steps, easy to forget cleanup
- **Fix:** Use `workmux merge` or `workmux remove` - one command handles everything

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" → ambiguous
- **Fix:** Present exactly 4 structured options

**Automatic worktree cleanup**
- **Problem:** Remove worktree when might need it (Option 2, 3)
- **Fix:** Only cleanup for Options 1 and 4

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request
- Delete a worktree while running inside it (terminates session)

**Always:**
- Use `workmux` commands when available
- Verify tests before offering options
- Present exactly 4 options
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill
