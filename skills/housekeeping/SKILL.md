---
name: housekeeping
description: Git + machine housekeeping. Use whenever the user says "housekeep", "housekeeping time", "clean up my branches / worktrees", "what can I delete", "which branches are merged", "free disk space from worktrees", "sync my branches with origin", "the machine is slow", "check my memory / CPU", "what is eating RAM" — any tidy-up of worktrees, local/remote branches, tags, stash, node_modules, or of the processes left behind by dev sessions (orphaned dev servers, idle agent sessions, browser/Docker memory). Inventories, proves what is merged or superseded, proposes a cleanup table, executes only what the user approves, then syncs everything with origin.
---

# Housekeeping — worktrees, branches, tags, stash, disk, memory

Housekeeping for one repository (default: the cwd) **and** the machine it runs on. Optional
focus: `$ARGUMENTS` (e.g. `worktrees only`, `remote too`, `disk`, `memory`).

**Posture:** inventory → evidence → **picture** → proposal → approval → execute → sync → memory → report.
Nothing is deleted before the proposal is approved. Remote deletes and pushes are a
**second, separate** approval. Local-only is the default.

**The first thing the user sees is the picture (§4), not a log.** Gather §1–§3 and the memory
snapshot of §7 silently (background the slow parts, delegate the sweeps), then render one
overview of the whole context — repo, disk, machine — with what is cleanable already marked.
No narration of commands run, no partial tables along the way.

## 0. Project defaults (detect, don't assume)

- **Trunk** = the branch PRs merge into. It is not always `main` — many repos merge into
  `development`/`develop` and keep `main` as the release lineage. Check the default branch
  (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`) and the base of recent
  merged PRs.
- **Prod** = latest release tag (`git tag -l 'v*' --sort=-creatordate | head -1`), if the repo
  tags releases.
- **Worktrees** — find where they live from `git worktree list` (commonly `.claude/worktrees/`,
  `.worktrees/`, or siblings of the repo).
- **Handle** = the user's GitHub login (`gh api user -q .login`), used to find their remote
  branches when they prefix them (`<handle>/…`).
- **Work docs home** = where loose HTML/MD docs go when they are not repo deliverables. Ask once
  if it isn't written down anywhere.
- **Machine** = cores and RAM: `sysctl -n hw.ncpu hw.memsize` (macOS) or `nproc; free -g` (Linux).
- Read the repo's `CLAUDE.md` / `AGENTS.md` and any memory notes first — trunk polarity flips,
  and a note may already say why a branch is parked.

## 1. Inventory (one pass, all of it)

```bash
git fetch --all --prune -q
git worktree list
git for-each-ref --sort=-committerdate refs/heads \
  --format='%(refname:short)|%(upstream:short)|%(upstream:track)|%(committerdate:short)'
git stash list
git tag -l 'backup/*' 'archive/*' 'tmp/*'         # local safety tags
git branch -r | grep "origin/<handle>/"            # the user's remote branches, if prefixed
```

## 2. Evidence per branch — enumerate, never assume

For **every** local branch (not trunk/main), collect all four; one alone lies:

| Signal | Command | What it proves |
|---|---|---|
| Merge-base merged | `git branch --merged origin/<trunk>` and `--merged <prod-tag>` | merge-commit merges |
| Unique patches | `git cherry origin/<trunk> <b> \| grep -c '^+'` | survives rebase/squash |
| PR state | `gh pr list --head <b> --state all --json number,state,baseRefName,mergedAt` | MERGED / CLOSED / OPEN / none |
| Closed-PR reason | `gh pr view <n> --json comments -q '.comments[-1].body'` | "superseded by #N" vs abandoned |

Then the two questions a grep can't answer:
- **Containment:** is branch A a subset of branch B? `git log --oneline B..A \| wc -l` = 0 → A ⊂ B.
- **Did the work land under another name?** `gh pr list --search "<ticket key>"` and grep trunk
  for a distinctive identifier/test file from the branch. `--head` misses a PR whose head was a
  renamed or re-cut branch.

## 3. Worktree dirt — what is actually at risk

For each worktree: `git -C <w> status --porcelain`. Ignore known noise (local status notes,
agent scratch dirs, `.env*` symlinks, local-only config hacks). For every other modified file,
check whether its **working-tree content** already exists on a known branch:

```bash
h=$(git -C $w hash-object "$f")
for ref in origin/<trunk> origin/main <other candidates>; do
  [ "$(git -C $w rev-parse -q --verify "$ref:$f")" = "$h" ] && echo "$ref" && break
done
```

"NOWHERE" = genuinely unique work → it goes in the proposal as a line item, never silently.
Untracked scripts that exist only in a worktree (e.g. `.claude/workflows/*.js`) get copied out
before removal.

## 4. The picture — first response, one screen

Render this in **markdown** (GitHub-flavored), filled from §1–§3 and the §7 snapshot. Short
tables, plain words, generous whitespace. No box-drawing, no abbreviations the user has to
decode, no more than ~40 lines. Numbers are real counts — except disk, which may read
`n × ~3.8 GB` while `du` is still running.

Example (illustrative values):

---

### Housekeeping · acme-app · 2026-01-15

| | Now | Cleanable |
|---|---|---|
| Worktrees | 36 | 27 |
| Branches | 46 | 34 local · 10 on origin |
| Disk in worktrees | 106 GB | ~75 GB |
| Memory | 23 / 24 GB, swap 12.9 GB ⚠ | ~3 GB (orphaned servers) |
| Load (10 cores) | 6.1 / 8.5 / 8.0 | fine |

Trunk is `development`; prod is `v4.2.0`. `main` and `development` are 3 / 105 commits behind origin.

**Merged — delete (13)**

| Branch | Evidence | Worktree |
|---|---|---|
| `hotfix/payment-limits` | #347, in prod | dirty: `feature-flags.ts` — **exists nowhere else** |
| `you/onboarding-v2` | #306, in prod | dirty: country-picker WIP, 5 files — **nowhere else** |
| `you/fix-store-links` | #437 | clean |
| … | | |

**Superseded or closed — delete (17, 3 need your call)**

| Branch | Why | Worktree |
|---|---|---|
| `you/flows` | closed #379 → replaced by #408 | — |
| `you/cta-polish` | closed #343, "landed in #380/#408" | 105 staged files, 38 match nowhere |
| `you/pricing-disclosure` | **your call** — no PR, unmerged, pricing model changed since | clean |
| … | | |

**Scratch — delete (7, 27 GB)**
`capture-checkout`, `capture-prod-v4.1.9`, `brand-before`, `pr344-onboarding`, `pr415-review`, `pr415-api`, `pr415-web` — detached review/capture checkouts.

**Keep (9)**
`flow-chrome` #408 (**6 unpushed**) · `sidebar-shell` #464 · `limit-request` #445 · `test-study` #461 · `tax-id-format` #439 (2 behind remote) · …

**Also cleanable**

| What | Detail | Needs |
|---|---|---|
| Stash | 3 entries: schema WIP (landed?), TICKET-635 parked, TICKET-412 fork | a look |
| Local tags | 9 `backup/*` from rebases already done | yes |
| Origin branches | 10 with closed PRs | **separate yes** |
| Untracked docs | 17 files in `docs/` → work docs home | yes |
| Idle `node_modules` | `ticket-485` (since 12-24), `ramp` (01-08) — ~8 GB | yes |
| Orphaned servers | 3 × `tsx watch` without a port, 2 stale `api dev` | yes |
| Agent sessions | 6 running, 2 older than a day | you close |
| Browser | Chrome renderer 26111 — 4 GB, 2 days, 85 % CPU | you close |

**If everything above goes:** 36 → 9 worktrees, 46 → 12 branches, about 75 GB of disk and 3 GB of RAM back.

Which groups? Nothing is deleted until you say.

---

Rules for the picture:
- **The summary table comes first and is always present**, even when nothing is cleanable — the
  point is that the user sees repo, disk and machine together.
- Every branch in the two delete groups gets its own row: the evidence column is what makes the
  delete safe to approve at a glance (PR number, "in prod", "replaced by #N").
- **"nowhere else"** on a dirty worktree is the one phrase that turns a routine delete into a
  decision — bold it, never bury it.
- "Your call" rows say in a few words what changed since the branch was cut.
- Long groups: show the rows that carry a warning or a decision in full, collapse the rest to
  `… and N more, all clean` — the full list is one question away.
- End with the net effect and one question. Then **stop** — the picture *is* the proposal; do
  not follow it with a prose repeat or a list of commands run.
- If the user asks for it as a page, write the same content as a self-contained HTML file next
  to the work.

Group definitions:
- **Merged** — in trunk (say "in prod" when also under the latest release tag); branch +
  worktree, with the worktree's dirt named.
- **Superseded / closed** — cite the closing PR comment or the containing branch; "your call"
  for no-PR, unmerged or unclear ones.
- **Scratch** — detached HEADs for captures/reviews (`capture-*`, `pr<N>-*`, `*-before`).
- **Keep** — open PRs, deliverables, research, with unpushed / behind-remote counts.

## 5. Execute (only what was approved)

Order matters:

1. **Safety first:** `git tag archive/<name>-<date> <branch>` for any deleted branch holding
   unmerged non-trivial code; copy worktree-only scripts to a scratch dir; say where.
2. **Worktrees** — `git worktree remove --force <path>`, one per line with `REMOVED/FAILED`
   logging, as a **background** script with a 10-minute timeout (node_modules make each one
   slow; a SIGTERM mid-delete leaves a `.git`-less husk git refuses to remove). Then
   `git worktree prune`.
3. **Branches** — `git branch -D` after their worktrees are gone.
4. **Tags / stash** — verify a stash landed before dropping it (sample its added lines with
   `git grep -F` on trunk); drop by explicit ref (`stash@{2}` deepest first). The stash is
   shared across worktrees and sessions — never bare `stash`/`pop`.
5. **Remote** (separate approval) — `git push origin --delete <b>…` only for closed-PR or
   confirmed-superseded branches; never the kept ones.
6. **gc** — `git gc --prune=now`. **Do not** `git reflog expire --expire=now --all` unless the
   user asked for a hard wipe: `--all` includes `refs/stash` and deletes every stash entry
   silently, and it closes the 90-day recovery window for everything just deleted.
7. **Disk** — `find <worktree> -type d -name node_modules -prune -exec rm -rf {} +` for idle
   worktrees (branch and source stay; the package manager's install restores).
8. **Docs** — move untracked work docs out of the repo to the work docs home; update any note
   that pointed at the old path.

## 6. Sync

- Fast-forward `main` and trunk (`git merge --ff-only` if checked out, else
  `git fetch origin <b>:<b>`); check dirty files don't collide with the incoming diff first.
- Per branch: `+ahead/−behind` vs its upstream **and** vs trunk; `--ff-only` the ones behind
  their upstream (inside their worktree); `git push -u origin <b>` for any branch with no remote
  copy (a branch tracking `origin/<trunk>` has none).

## 7. Memory & processes — what the sessions left running

The snapshot feeds the picture's machine line and process rows, so take it **before** the first
response; the actions run here, after the repo work. Use the cores/RAM detected in §0.
Snapshot first, then the four holders (macOS commands; Linux equivalents: `uptime`, `free -h`,
`vmstat 1 2`, `ps aux --sort=-%cpu | head`, `ps aux --sort=-rss | head`):

```bash
uptime; top -l 2 -n 0 -s 1 | grep -E "CPU usage|PhysMem|Load Avg" | tail -3
sysctl vm.swapusage; memory_pressure | tail -3
top -l 2 -s 2 -o cpu -n 10 -stats pid,cpu,mem,command | tail -11   # 2 samples: the 1st is always 0%
top -l 1 -o mem -n 12 -stats pid,mem,cpu,command | tail -13
```

Read it like this: **swap used > 8 GB or "unused" < 500 MB = act now**; load counts runnable
threads, so anything under the core count is not a CPU problem. Then classify:

| Holder | How to see it | Action |
|---|---|---|
| **Orphaned dev servers** | `ps -eo pid,ppid,etime,command \| grep -E '[t]sx watch\|[v]ite\|[n]ext dev'` + `lsof -nP -iTCP -sTCP:LISTEN` for the project's dev ports | Anything not holding a port, or older than the session that owns it, is a zombie: `kill <parent pids>`. Only one stack per port can be live. |
| **Idle agent sessions** (~0.5 GB each) | `ps -eo pid,rss,etime,command \| grep -E '[c]laude( \|$)'` | Name the ones older than a day; the user closes them (never kill the one you are in). |
| **Browser** (renderer > 2 GB, alive > 1 day = leaking tab) | `ps -o rss,etime -p <pid>`; `--type=renderer` in its args | Report PID + size; the user closes the tab (Activity Monitor / task manager → search the PID shows its title). Two browsers open doubles the baseline. |
| **Docker VM** (`com.apple.Virtualization` on macOS) | `docker ps` | `docker stop` idle containers; the VM memory cap lives in Docker Desktop → Resources. |

Kill only what is provably orphaned (no port, no living parent session); everything else is a
line in the proposal. **Swap does not drain on its own** — after freeing memory, say plainly when
a reboot is the honest reset (e.g. 13 GB swap + 5 days uptime).

Test runners: none should be alive between runs; a `vitest`/`jest` older than a few minutes is a
watch-mode leak. Cap workers (`--maxWorkers=4`) and never leave watch mode running.

## 8. Report

Before/after: worktrees, branches, tags, stash, `.git` size, worktrees dir size, freed GB,
swap used, processes killed. List what still needs the user (unpushed commits, "your call"
branches, stashes kept, tabs/sessions to close, whether to reboot).

## Measuring disk without stalling

- `df -h /System/Volumes/Data` (macOS) or `df -h <mount>` (Linux) is instant and is the honest
  "freed" number (on macOS `df /` is the sealed system volume).
- `du` must stat every file; 20+ worktrees with `node_modules` take minutes cold. Run it in the
  background, per directory, or count `node_modules` dirs × ~3.8 GB instead.
- Per-dir `du` sums over-count vs one deduped total only if pnpm hardlinks are working; check —
  if sum == total, hardlinks aren't working and every installed worktree is a full copy.

## Known traps

- **zsh does not word-split `$var`** — `git diff -- $files` with a multi-line variable is one
  bogus path and prints "IDENTICAL" for everything. Use `xargs -a file`, `${=files}`, or a loop.
- `git branch --merged` says nothing about squash/rebase merges; `git cherry` says nothing
  about a merge that was re-cut under another branch name. Use all four signals.
- `gh pr list --head` returns nothing when the PR's head is a different remote branch
  (e.g. `feature-x-review-fixes-v2` was merged as `feature-x-on-development`).
- Some permission classifiers block looping `git stash drop`; use explicit refs, or
  `git update-ref -d refs/stash` for the last entry.
- A worktree with 100+ *staged* files identical to trunk is a stale `git checkout trunk -- …`,
  not lost work — but the files that match nowhere still get listed.
- "No importers" evidence is not deletion evidence; neither is "not found" for uniqueness —
  enumerate.
