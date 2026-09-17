# skills

My [agent skills](https://code.claude.com/docs/en/skills), packaged as a Claude Code plugin.

## Install

This repo is its own single-plugin marketplace. From inside a Claude Code session:

```
/plugin marketplace add edrickleong/skills
```

```
/plugin install edrickleong-skills@edrickleong
```

Or from the shell:

```bash
claude plugin marketplace add edrickleong/skills
```

```bash
claude plugin install edrickleong-skills@edrickleong
```

The repo is **private**, so installing needs a git credential that can read it —
`gh auth login` as `edrickleong`, or an SSH key on the account.

Updating:

```bash
claude plugin marketplace update edrickleong
```

## Why These Skills Exist

### #1: The PR Nobody Wants To Review

**The Problem**. Agents produce large changes quickly. A good session ends with forty
files touched — a schema change, the API built on it, the UI on top, and a refactor
picked up along the way. That is one enormous pull request, and it has exactly two
outcomes. It gets rubber-stamped, because nobody can hold it in their head. Or it sits
for three days while a reviewer works up the energy.

The usual advice — "make smaller PRs" — is no help here, because it is advice about the
*past*. The work is already done. It is sitting in your working tree right now.

So why doesn't anyone split it afterwards? Because splitting by hand is miserable. You
are cherry-picking hunks into the right branch, rebasing each branch onto the one below
it, and re-pointing every PR base each time the bottom layer changes. It is fiddly,
and getting it wrong means losing work. Shipping the giant PR is the path of least
resistance, so that's what happens.

**The Fix** is a **stack**: an ordered chain of branches rooted on trunk, where each
branch has one PR based on the branch below it. The reviewer of any layer sees only
that layer's diff.

```
(main) <- billing/schema <- billing/api <- billing/ui
```

[`/create-stacked-pr`](./skills/create-stacked-pr/SKILL.md) does both halves of that
job. The **judgement** — reading the actual diff, deciding where the cuts go, in what
order, and what can genuinely merge on its own — and the **mechanics** of building it
with the [`gh stack`](https://github.com/github/gh-stack) CLI.

It is deliberately not a one-shot. The interesting decisions in a split are yours:

- **It proposes before it touches anything.** You get an ordered table — bottom slice
  first — with a rationale per slice for why it is a unit and why it sits at that
  height, plus an explicit list of the hunks it wasn't sure about. Then it stops. No
  branches, no rewritten history, no pushes until you've said yes.
- **Approving the slicing is not approving the push.** Submitting opens real PRs on
  GitHub, so that gets its own confirmation, immediately before it happens — with the
  option to open them as drafts first.
- **It backs up before it rewrites.** A backup ref is captured before any history is
  touched, so a bad split is one `git switch` away from undone.

The rest of the skill is the unglamorous part: which `gh stack` commands hang forever
when an agent runs them (several open a full-screen TUI), what each exit code means,
and how to recover from a rebase conflict or a squash-merged bottom layer.

> [!TIP]
> Reach for it *before* you start reviewing your own work, not after. The skill reads
> the diff cold, which means it will suggest cuts you've already talked yourself out of.

## Reference

- **[create-stacked-pr](./skills/create-stacked-pr/SKILL.md)** — Break a large pile of
  working changes into a stack of small, independently-mergeable PRs. Inspects the diff,
  proposes an ordered decomposition with a rationale per slice, gates on your
  confirmation, then builds and submits the stack with `gh stack`.

## Licence

MIT — see [LICENSE](./LICENSE).
