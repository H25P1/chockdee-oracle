---
pattern: "When a scope-of-consent lesson gets re-triggered in a different shape almost immediately, the first formulation was too narrow — restate it around the actual invariant (this specific high-stakes target), not the surface category (doc vs. code) of the artifact that happened to trigger it"
date: 2026-07-30
source: "rrr: chockdee-oracle (cross-project work on kbn-portal / ERP_Frabbe)"
concepts: ["approval", "scope-of-consent", "external-repo", "production-system"]
---

# Scope-of-consent lessons are about the target, not the artifact type

## What happened

Two retros ago ([[2026-07-28_doc-edit-is-its-own-checkpoint]]), an Edit call to
`KBN_ERP_CONTEXT.md` — a cross-team "source of truth for AI Assistant" doc on H's
external production ERP repo — was rejected before it wrote anything. The lesson drawn
was: doc edits to shared source-of-truth files need a preview step before writing, even
under a broad "proceed as recommended."

Two days later, in the same repo, H reported a concrete bug (a per-line "hide price"
checkbox on a Quotation edit form wasn't excluding hidden lines from the displayed
total). The fix was diagnosed precisely — a one-line change to a `reduce()` call,
duplicated across two files, with a known caveat (the backend-persisted total and PDF
wouldn't be affected by a client-only fix). This was judged to be a routine, low-risk
code fix — not a "shared doc" — and reasoned as fine to write directly without a preview
step. The Edit call was rejected again, in the same shape: right before the write, no
stated reason.

## The lesson

The first lesson was true but scoped too narrowly. It categorized the *artifact*
("shared doc" vs. "ordinary code") rather than naming the actual invariant: **this is a
specific external repo — a live, real-money production system belonging to someone
else's company — and any write to it deserves a preview before it lands**, regardless of
whether the write is to a doc, a config file, or application code. A one-line change to
a financial-total calculation is not lower-stakes than a doc edit just because it's
"code" — if anything, it's higher-stakes, since it silently changes what a customer is
billed.

When a scope-of-consent lesson gets re-triggered in a different shape almost
immediately, that recurrence is itself the signal that the first formulation missed the
real boundary. Re-examine what actually varies between "things I can just do" and
"things that need a preview" — it's more often about *which system/repo/stakes* than
*which kind of file*.

## How to apply

For any repo that is:
- external to the agent's own working project,
- a live production system (especially one handling money, customer data, or anything
  another human/team depends on), and
- not something the agent built or has full context on,

default to a propose-then-write pattern for **every** edit — doc, config, or code —
regardless of how routine or well-diagnosed the fix seems. Reserve direct writes for the
agent's own working repo/vault, or for changes explicitly pre-authorized at the specific-content
level (not just a general "proceed"). If a lesson from one incident gets hit again in a
superficially different form, treat that as a prompt to broaden the rule, not to write a
second, parallel lesson for the new surface shape.
