# housekeeping

A Claude Code skill for git + machine housekeeping. Say "housekeeping time" and it:

1. **Inventories** every worktree, local/remote branch, tag, stash entry, idle `node_modules`, and the processes your dev sessions left behind (orphaned dev servers, idle agent sessions, leaking browser tabs, Docker).
2. **Proves** what is safe to delete — four signals per branch (merge-base, `git cherry`, PR state, closing-PR reason), plus containment and "did it land under another name" checks. Dirty worktree files are hashed against known branches so unique work is never lost silently.
3. **Shows one picture** — a single-screen summary of repo, disk and machine with cleanable items grouped (Merged · Superseded · Scratch · Keep · Also cleanable) and the evidence for each row.
4. **Executes only what you approve.** Remote deletes are a separate approval. Unmerged work is tagged `archive/*` before deletion.
5. **Syncs** trunk and branches with origin, then reports before/after.

## Install

### As a standalone skill (one file, no plugin)

For all your projects:

```bash
mkdir -p ~/.claude/skills/housekeeping && curl -fsSL https://raw.githubusercontent.com/jubscodes/housekeeping/main/skills/housekeeping/SKILL.md -o ~/.claude/skills/housekeeping/SKILL.md
```

For one project only, run from the repo root:

```bash
mkdir -p .claude/skills/housekeeping && curl -fsSL https://raw.githubusercontent.com/jubscodes/housekeeping/main/skills/housekeeping/SKILL.md -o .claude/skills/housekeeping/SKILL.md
```

Start a new Claude Code session and it's available.

### As a plugin

```
/plugin marketplace add jubscodes/housekeeping
/plugin install housekeeping@housekeeping
```

## Use

In any git repo:

```
housekeeping time
clean up my worktrees
which branches are merged?
what is eating RAM?
```

Optional focus: `worktrees only`, `remote too`, `disk`, `memory`.

## Requirements

- `git`, and [`gh`](https://cli.github.com/) authenticated for PR evidence
- macOS commands by default for the machine snapshot; Linux equivalents are noted in the skill

## Safety

- Nothing is deleted before you approve the proposal.
- Remote branch deletes and pushes need a second, separate approval.
- Never runs `git reflog expire --all` (it silently drops every stash) unless you ask for a hard wipe.
- Never kills a process that still holds a port or has a living parent session.

## License

MIT
