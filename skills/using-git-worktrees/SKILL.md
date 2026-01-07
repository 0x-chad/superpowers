---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees using workmux for agent-driven workflows
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching. This skill uses `workmux` to manage worktrees with agent-driven workflows.

**Core principle:** Systematic directory selection + safety verification + agent prompts = reliable isolation.

**Trigger phrases:** "worktree", "work tree", "isolated workspace", "work on this in isolation"

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Workflow

### Step 0 - Check Existing Worktrees First

```bash
workmux list
```

If a worktree already exists for this task, ask the user if they want to use it or create a new one with a different name.

### Step 1 - Verify Worktree Directory is Ignored

**MUST verify before creating any worktree:**

```bash
git check-ignore -q .worktrees 2>/dev/null && echo "ignored" || echo "NOT ignored"
```

**If NOT ignored:** Add to `.gitignore` and commit before proceeding:

```bash
echo ".worktrees" >> .gitignore
git add .gitignore && git commit -m "Ignore .worktrees directory"
```

**Why critical:** Prevents accidentally committing worktree contents to repository.

### Step 2 - Ensure .workmux.yaml Handles Gitignored Files

Worktrees don't include gitignored files. Proactively configure them BEFORE creating worktrees.

**Check if already configured:**
```bash
test -f .workmux.yaml && grep -q "files:" .workmux.yaml && echo "configured" || echo "needs setup"
```

**If needs setup, scan gitignored files:**
```bash
git ls-files --others --ignored --exclude-standard --directory | head -30
```

**Categorize and create .workmux.yaml:**

| If you see... | Action | Why |
|---------------|--------|-----|
| libs/, *.aar, *.so, *.dylib | `files.symlink` | Static binaries, safe to share |
| models/, *.gguf, *.tflite, *.bin | `files.symlink` | ML models, read-only |
| assets/, res-large/, dictionaries/ | `files.symlink` | Large read-only assets |
| local.properties, .env* | `files.copy` | Config files, may need unique values |
| node_modules/, vendor/, .gradle/ | `post_create` hook | Dependencies, must reinstall per worktree |
| build/, dist/, *.pyc | ignore | Build outputs, regenerate fresh |

**Example .workmux.yaml:**
```yaml
files:
  symlink:
    - libs
    - java/res-large
  copy:
    - local.properties

post_create:
  - npm install  # if Node.js project

# Resource hints for parallel worktrees (read by agents, not workmux)
# x-resource-hints:
#   - "Use a different emulator than other worktrees (check: adb devices)"
#   - "Change PORT in .env to avoid conflicts (check: lsof -i :3000)"
#   - "Use a different Redis DB number in config"
```

**Also check for resource conflicts:**
If the project uses ports, emulators, databases, or other shared resources, add `x-resource-hints` listing what agents should change per worktree.

### Step 3 - Write Prompt Files (in parallel if multiple tasks)

For each task, generate a short descriptive worktree name (2-4 words, kebab-case) and write a detailed implementation prompt:

**Before writing prompts, check for resource hints:**
```bash
grep -A10 "x-resource-hints:" .workmux.yaml 2>/dev/null || echo "no hints"
```

**Prompt template:**
```bash
tmpfile=$(mktemp).md
cat > "$tmpfile" << 'EOF'
[Task description here]

## Context
[Include relevant context from the conversation]

## Requirements
[Specific requirements and acceptance criteria]

## Parallel Worktree Isolation
Other worktrees may be running simultaneously. Before starting services or tests:
- Check for resource conflicts (ports, emulators, containers, databases)
- Common checks: `lsof -i :<port>`, `adb devices`, `docker ps`
- Use different resources than what's already in use
[If x-resource-hints exist in .workmux.yaml, list them here]

## Instructions
- Use RELATIVE paths only (each worktree has its own root)
- Be specific about what to accomplish
- If build fails due to missing gitignored files:
  1. Identify what's missing (compare to main repo)
  2. Update .workmux.yaml in the MAIN repo (not worktree) to add symlinks/copies
  3. Run `workmux remove <name> && workmux add <name>` to recreate with correct files
  4. Commit the .workmux.yaml update so future worktrees work
- Review changes with Claude after completing the task
EOF
echo "$tmpfile"  # Note the path for step 4
```

**Important notes for prompts:**
- If tasks reference something discussed earlier (e.g., "do option 2", "implement the fix we discussed"), include all relevant context from the conversation
- If tasks reference a markdown file (e.g., a plan or spec), re-read the file to ensure you have the latest version before writing prompts
- Use RELATIVE paths only (never absolute paths, since each worktree has its own root directory)
- Include instruction to review changes with Claude after completing the task

### Step 4 - Create Worktrees

After ALL prompt files are written, run workmux commands (can run in parallel for multiple worktrees):

```bash
workmux add feature-x -b -P /tmp/tmp.abc123.md
workmux add feature-y -b -P /tmp/tmp.def456.md
```

### Step 5 - STOP (Work Happens in Tmux)

**CRITICAL:** After `workmux add` succeeds, your job is DONE. The work happens in the tmux session that workmux spawned, NOT in the current session.

1. Report which worktrees/branches were created
2. Tell the user the agent is working in the tmux session
3. **DO NOT continue working on the task yourself**

The user can switch to the tmux session to monitor progress, or wait for the agent to complete.

## Completing Work

When work in a worktree is done, use these commands to finish:

### Merge and Cleanup

```bash
# Merge branch into main, then remove worktree, tmux window, and branch
workmux merge <name>

# Options:
workmux merge <name> --rebase    # Rebase before merging (fast-forward)
workmux merge <name> --squash    # Squash all commits into one
workmux merge <name> --keep      # Merge but keep worktree/branch
```

### Discard Without Merging

```bash
# Remove worktree, tmux window, and branch without merging
workmux remove <name>

# Options:
workmux remove <name> --force        # Skip confirmation, ignore uncommitted changes
workmux remove <name> --keep-branch  # Keep the branch, just remove worktree
workmux remove --gone                # Clean up worktrees whose remote branch was deleted (after PR merge)
workmux remove --all                 # Remove all worktrees
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Check existing worktrees | `workmux list` |
| Create worktree with prompt | `workmux add <name> -b -P <prompt-file>` |
| Merge and cleanup | `workmux merge <name>` |
| Discard without merging | `workmux remove <name>` |
| Clean up after PR merge | `workmux remove --gone` |
| Directory not ignored | Add to .gitignore + commit |
| Missing gitignored files | Create/update .workmux.yaml with symlinks/copies |
| Resource conflicts (ports, emulators) | Add `x-resource-hints` to .workmux.yaml |

## Example Workflow

```
User: In a worktree, can you investigate why the bubble auto-shows incorrectly?

You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Step 0: workmux list - no existing worktrees]
[Step 1: git check-ignore .worktrees - confirmed ignored]
[Step 2: Check .workmux.yaml - needs setup, scan gitignored files, create config]
[Step 3: Write prompt file to /tmp/tmp.xyz.md]
[Step 4: workmux add bubble-autoshow-bug -b -P /tmp/tmp.xyz.md]

Worktree created on branch `bubble-autoshow-bug`.
An agent is now working on this in a tmux session.

You can switch to it with: tmux attach -t bubble-autoshow-bug
Or run `workmux list` to see status.

[Step 5: STOP - do not continue working on the task]
```

## Red Flags

**Never:**
- Create worktree without checking existing ones first (`workmux list`)
- Skip checking/creating .workmux.yaml for gitignored files
- Skip writing detailed prompt files
- Use absolute paths in prompts
- Proceed without verifying directory is ignored (project-local)
- **Continue working on the task after `workmux add` succeeds** (the tmux agent handles it)

**Always:**
- Run `workmux list` first
- Check/create .workmux.yaml with gitignored file handling
- Add `x-resource-hints` if project uses ports, emulators, or shared resources
- Include parallel isolation section in prompts with any resource hints
- Write comprehensive prompts with full context
- Use relative paths in prompts
- Inform user which branches were created

## Integration

**Called by:**
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
- **executing-plans** or **subagent-driven-development** - Work happens in this worktree
