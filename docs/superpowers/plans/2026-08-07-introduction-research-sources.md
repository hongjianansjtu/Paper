# Introduction Research Sources Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task with review checkpoints.

**Goal:** Add a source-controlled research memo for the Introduction annotation on branch `source_hjk`.

**Architecture:** One self-contained Markdown memo under `sources/`, with reproducible formulas and links. Supporting design and plan files live under `docs/superpowers/`; the LaTeX manuscript remains untouched.

**Tech Stack:** Markdown, Git, Coin Metrics Community API, official protocol/project documentation.

## Global Constraints

- Snapshot date: 2026-08-07; fee window: 2026-08-01 through 2026-08-06.
- Every quantitative claim must state its unit, denominator, time window, and source URL.
- Do not present SEO/blog figures as primary evidence.
- Do not compare consortium infrastructure cost directly with public-chain transaction fees.
- Do not call Haveno a scriptless atomic-swap implementation.

### Task 1: Write the evidence memo

**Files:**
- Create: `sources/introduction-research.md`

- [ ] Add the annotation interpretation, evidence checklist, quantitative tables, formulas, source links, caveats, and candidate Introduction wording.
- [ ] Verify that all fee values and market-cap values are copied from the stated Coin Metrics snapshot and that no unsupported ratio is presented.

### Task 2: Review and commit

**Files:**
- Review: `docs/superpowers/specs/2026-08-07-introduction-research-sources-design.md`
- Review: `docs/superpowers/plans/2026-08-07-introduction-research-sources.md`
- Review: `sources/introduction-research.md`

- [ ] Run `git diff --check`.
- [ ] Confirm `git diff --name-only` contains only the three new Markdown files.
- [ ] Commit with `docs: add Introduction research source memo`.
- [ ] Push `source_hjk` to `origin` and verify the remote branch with `git ls-remote`.
