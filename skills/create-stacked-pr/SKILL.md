---
name: create-stacked-pr
description: Break a large pile of working changes into a stack of small, independently-mergeable pull requests. Inspects the diff, proposes an ordered decomposition with rationale, confirms with the user, then builds the stack with the gh stack CLI. Use when the user has a big/hard-to-review change and wants it split into a reviewable stack of PRs.
allowed-tools: Bash(git:*), Bash(gh:*)
---

# Create Stacked PR

Turn one unwieldy change into a chain of small PRs that each stand on their own.

This skill owns both the **judgement** — reading the diff, deciding where to cut, in
what order, and what is independently mergeable — and the **mechanics** of building the
stack with the [`gh stack`](https://github.com/github/gh-stack) CLI. A stack is an
ordered chain of branches rooted on a trunk, where each branch has one PR based on the
branch below it, so a reviewer sees only that layer's diff.

`gh stack` prints a stack trunk-first, left to right:

```
(main) <- auth <- api <- frontend
```

Left is the **bottom** (merges first, closest to trunk); right is the **top**. `up`
moves toward the top, `down` toward trunk. Foundational work belongs at the bottom, code
that depends on it above.

## Golden rules

- **Never create branches, rewrite history, or push before the user has confirmed a plan.** The proposal step (section 2) is a hard gate.
- **Submitting opens/updates real PRs on GitHub — that is publishing.** Confirm explicitly right before submitting, even if the plan was already approved. Approving the *slicing* is not approving the *push*.
- **Never lose work.** Before rewriting anything, capture a backup ref so the original state is recoverable.
- Each slice must build on the one below it and be mergeable on its own — order matters (bottom = closest to trunk).

## Setup

Ensure the extension and git config are in place (safe to re-run):

```bash
gh extension install github/gh-stack   # once per machine; no-op if already installed
git config rerere.enabled true         # remember conflict resolutions across rebases
git config remote.pushDefault origin   # required if the repo has more than one remote
```

## Non-interactive use of `gh stack`

`gh stack` branches on whether **stdout is a TTY**. Piped, most commands error cleanly or
print static text; under a PTY the same commands open a prompt or a full-screen TUI and
block forever. Agent harnesses differ, so always pass the flags below instead of relying
on that detection.

**Multiple remotes:** never run `push`, `submit`, `sync`, `rebase`, or `link` without
`--remote <name>` unless `remote.pushDefault` is configured. `checkout` and `trunk` have
no `--remote` flag and require the config.

| Always run | Never run bare | Why |
|---|---|---|
| `gh stack view --json` | `gh stack view` | opens a TUI under a PTY |
| `gh stack submit --auto` | `gh stack submit` | prompts for a title per new PR |
| `gh stack merge <target> --yes` | `gh pr merge` | `gh pr merge` cannot merge a stack |
| `gh stack init <branch>...` | `gh stack init` | prompts for branch names |
| `gh stack add <branch>` | `gh stack add` | prompts for a name, and fails even when piped |
| `gh stack checkout <target>` | `gh stack checkout` | opens a selection menu |
| `gh stack up` / `down` / `top` / `bottom` | `gh stack switch` | `switch` is menu-only |
| — | `gh stack modify` | TUI-only, no non-interactive path |

- `view --short` is safe in both modes, but it is formatted for humans. Use `--json` to parse.

## 1. Assess the current state

Run these and read them before proposing anything:

```
git branch --show-current
git status --short
git log --oneline --no-decorate <trunk>..HEAD    # trunk is usually main
git diff <trunk>...HEAD                            # committed work on this branch
git diff                                           # unstaged working changes
git diff --staged                                  # staged working changes
git status --porcelain --untracked-files=all       # untracked files
```

Determine which situation you're in:
- **Uncommitted working changes** — everything is in the working tree / staged, few or no commits. You'll be staging subsets into fresh commits.
- **A pile of commits on one branch** — the work is already committed but not yet split. You'll be regrouping commits into stacked branches.
- **A mix** — commit the loose changes into the right slices as you go.

Note the trunk/base branch (default branch of the repo, usually `main`) — every proposal is relative to it.

## 2. Propose the decomposition — then STOP

Decide the cuts from the *actual diff*, not a fixed rule. Aim for slices that are:
- **Independently mergeable** — each could ship to trunk on its own without breaking the build (as far as the diff shows).
- **Review-sized** — one coherent idea per PR; a reviewer can hold it in their head.
- **Ordered** — foundational/low-level changes at the bottom, features that depend on them on top.

Present the plan as an ordered list, **bottom (nearest trunk) first**, each slice with:

| # | Branch name | Title | What's in it (files/areas) | Depends on | ~size | Independently mergeable? |

For each slice add a one-line rationale for *why* it's a unit and why it sits where it does. Call out anything you're unsure about (a hunk that could go either way, a change that spans slices, a test that must travel with its code).

Then **stop and ask the user to confirm, edit, reorder, or merge slices.** Do not proceed until they respond. If they revise, re-present the updated plan.

## 3. Build the stack (only after confirmation)

First, back up the current state so nothing is unrecoverable:

```
git branch backup/<current-branch>-<timestamp>    # or: git tag backup-<timestamp>
```

Then build the stack bottom to top with `gh stack`. Create each layer's branch, stage
exactly that slice's changes, and commit it before moving up. Use `git add <paths>` (or
`git add -p` for hunk-level splits) and verify with `git status --short` that only the
intended paths are staged — do **not** use `add -Am`, which sacrifices that control.

```bash
gh stack init auth              # create the stack; checks out its (bottom) branch
git add internal/auth/...       # stage only this slice
git commit -m "Add auth middleware"
gh stack add api                # next layer, branched from the current one
git add internal/api/...
git commit -m "Add API routes"
gh stack view --json            # confirm the shape
```

- Branch names are verbatim — `gh stack add refactor/foo` creates `refactor/foo`. Prefer a shared topic prefix plus the layer's concern (`billing/schema`, `billing/api`, `billing/ui`), unless the repo has its own convention.
- `gh stack add` must run from the **top** branch of the stack. It does not touch the working tree, so uncommitted changes carry over — commit or stash before adding a layer if you want it to start clean.

**Regrouping existing commits** instead of staging hunks: build the empty stack with
`gh stack init <bottom> <next> ... <top>` (adopts existing branches, creates missing
ones bottom to top), then cherry-pick each commit onto the layer that owns it and
`gh stack rebase --upstack` to replay the layers above.

**Editing a layer after the fact:** check out the owning branch, edit, then rebase the
layers above onto the change:

```bash
gh stack down                   # or: gh stack checkout api
git add ... && git commit -m "Fix get-user endpoint"
gh stack rebase --upstack       # replay every branch above onto the change
gh stack top                    # return to the top
```

When the stack is built, run `gh stack view --json` and show the user that it matches the
approved plan before touching the remote.

## 4. Submit (separate, explicit confirmation)

Submitting pushes every branch and creates/updates PRs on GitHub. **Confirm with the
user immediately before submitting** — this is the point of no easy return. Offer to
submit as **draft** first if they'd prefer to eyeball the PRs before requesting review.

```bash
gh stack submit --auto          # push every branch, open draft PRs, link them on GitHub
gh stack submit --auto --open   # same, but PRs ready for review instead of drafts
```

`submit` is **not atomic** — if a later push is rejected, earlier pushes and PR updates
stand; fix the rejection and rerun the same command. PR titles/bodies are auto-generated;
use `gh pr edit` afterwards to refine them. If stacked PRs are not enabled on the repo,
`submit` exits **9** — tell the user.

After submitting, run `gh stack view --json` and report the created PR links, in stack order.

## Reading state

`gh stack view --json` writes JSON to **stdout**; status messages go to **stderr** (don't
parse them — branch on exit codes).

```
trunk           string
currentBranch   string
branches[]      name, head, base, isCurrent, isMerged, isQueued, needsRebase
branches[].pr   number, url, state ("OPEN" | "MERGED" | "QUEUED"); absent when no PR exists
```

`needsRebase` is true when the current parent tip is no longer an ancestor of the branch.

## Exit codes

| Code | Meaning | Recovery |
|---|---|---|
| 0 | Success | — |
| 1 | Generic error | Read stderr |
| 2 | Not in a stack | `gh stack init`, or `gh stack checkout <target>` |
| 3 | Rebase conflict | Resolve files, `git add`, `gh stack rebase --continue` (or `--abort`) |
| 4 | GitHub API failure | Check `gh auth status`, retry |
| 5 | Invalid arguments | Fix the invocation; see `<command> --help` |
| 6 | Disambiguation required | Branch is in several stacks; check out a non-shared branch |
| 7 | Rebase already in progress | `gh stack rebase --continue` or `--abort` |
| 8 | Stack file locked | Another `gh stack` process is writing; retry after ~5s |
| 9 | Stacked PRs unavailable | Not enabled on the repository; tell the user |
| 10 | Modify recovery required | `gh stack modify --abort` |

## Recovering / bailing out

- If the split goes wrong, the backup branch/tag from step 3 restores the original state: `git switch backup/<...>` or `git reset --hard <tag>`.
- On a rebase conflict, after squash-merges, on local/remote divergence, or when restructuring a stack, read `references/troubleshooting.md`.

## More detail

`gh stack <command> --help` is authoritative for flags and arguments (note that
`gh stack help <command>` does **not** work — it prints the top-level help). Open the
reference whose trigger matches the task; no need to preload all three.

- `references/commands.md` — read when a command fails unexpectedly or you need its preconditions, side effects, atomicity, or ordering guarantees.
- `references/troubleshooting.md` — read on a rebase conflict, after a squash-merge, on local and remote divergence, when restructuring a stack, or when driving stacks from another tool.
</content>
</invoke>
