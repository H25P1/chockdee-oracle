---
pattern: "A broad 'proceed as recommended' covers the category of action, not the specific content of an edit to a shared/cross-team source-of-truth document — draft and show before writing"
date: 2026-07-28
source: "rrr: chockdee-oracle (cross-project work on kbn-portal / ERP_Frabbe)"
concepts: ["approval", "shared-docs", "multi-agent", "scope-of-consent"]
---

# A doc edit to a shared source-of-truth file is its own approval checkpoint

## What happened

H asked Chockdee to look at a separate, external, production project (an ERP system:
Frappe/ERPNext backend + Next.js frontend) and summarize its status, then said "proceed
as recommended" — which covered three things: git branch cleanup, a read-only backend
check, and updating the project's `KBN_ERP_CONTEXT.md` (a file that explicitly
self-describes as "source of truth สำหรับ AI Assistant อ่านก่อนเริ่มงานทุกครั้ง" — read by
at least one other AI agent already committing to that repo).

The branch cleanup and backend check were carried out (after calling `advisor()` for
repo-specific safety constraints, since this was an unfamiliar and high-stakes shared
system). For the doc update, a large append-only section was drafted — properly sourced,
citing commit hashes and command output, marking unverified claims as unverified — and
then written directly via the Edit tool. H rejected that Edit call before anything was
saved.

## The lesson

"Proceed as recommended" grants permission for the *category* of action listed, not
necessarily for the *exact content* of every artifact produced along the way — especially
when that artifact is a shared document other agents or teammates treat as authoritative.
Low-stakes, mechanically-verifiable actions (deleting confirmed-merged git branches with a
tool that refuses unmerged ones) can reasonably proceed straight through. A large edit to
a cross-team "read this before every task" doc is a different kind of action: its
*wording* is a decision, not just its category.

## How to apply

Before writing to a document that:
- other agents/teammates read as authoritative, or
- represents a team's shared understanding of project state, or
- would be awkward or costly to have gotten subtly wrong,

draft the content, then show it to the human for a quick look, then write it — even
under a broad "go ahead" that already covered the general action. This is a cheap,
one-extra-turn checkpoint that costs little and catches cases where the human wanted to
review wording, correct a claim, or simply hadn't meant to delegate that specific write.
Reserve straight-through execution for actions that are mechanically safe and easily
verified (like a `git branch -d` that already refuses to do the wrong thing).
