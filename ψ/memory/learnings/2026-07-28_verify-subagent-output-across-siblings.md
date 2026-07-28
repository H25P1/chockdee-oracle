---
pattern: "Cross-verify parallel subagent reports against each other before committing docs — a sibling's contradictory finding catches fabrication that 'don't invent content' instructions alone don't prevent"
date: 2026-07-28
source: "rrr: chockdee-oracle"
concepts: ["subagents", "verification", "learn-skill", "hallucination", "multi-agent"]
---

# Cross-verify subagent output against its siblings, not just against the prompt

## What happened

While running `/learn` against `oracle-framework` (3 parallel Haiku agents — Architecture,
Code Snippets, Quick Reference), the Architecture agent correctly reported the repo
contains nothing but a single `README.md`. The Code Snippets agent, despite an explicit
instruction not to invent content, confidently returned a detailed doc full of real
code — a skill definition, an agent definition, CLAUDE.md templates — all sourced from a
*different* repo (`arra-oracle-skills-cli`) sitting in a sibling ghq clone, mislabeled as
if it lived in `oracle-framework`.

This was caught only because both reports were being turned into hub-file content in the
same pass, and the two couldn't both be true. Filesystem-level verification (`find -L
.../origin -maxdepth 2`) confirmed the Architecture agent was right — the repo really did
contain only a README.

## The lesson

"Don't invent content" in a subagent prompt is necessary but not sufficient. When multiple
agents explore the same target in parallel, their independent reports are a built-in
verification opportunity — a contradiction between two "confident, well-cited" reports is
a signal, not a coincidence to shrug off. Treat cross-checking sibling agent output as a
required step before committing, not something you do only if you happen to notice.

Concretely, for `/learn`-style parallel exploration:
1. If one agent finds "nothing here" and another finds substantial content for the *same*
   target, don't average them — investigate directly (a single `find`/`ls` against the
   real source resolves it in seconds).
2. When agents have broader filesystem access than their assigned target (e.g. sibling
   repo clones sitting in the same ghq root), explicitly instruct each one to quote only
   what it personally read inside its assigned SOURCE_DIR — not memory of an adjacent
   repo it may have touched in another task.
3. This risk goes up, not down, when running many repos in the same batch (as in this
   session's 17-agent Tier-1 sweep) — more sibling repos on disk means more material an
   agent could misattribute.

## How to apply

Any time subagents explore/document something in parallel and the outputs get merged into
a single artifact (hub docs, a synthesized report, a batch PR), add a deliberate
cross-check pass before committing: do the independent reports agree on the basic facts
(what exists, what doesn't)? If not, resolve it against ground truth before shipping —
don't let the more detailed-sounding report win by default.
