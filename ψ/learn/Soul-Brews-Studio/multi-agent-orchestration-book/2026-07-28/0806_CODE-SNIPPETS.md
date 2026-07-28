# Multi-Agent Orchestration: 6 Actionable Patterns

*Extracted from: Soul-Brews-Studio/multi-agent-orchestration-book*  
*Date: 2026-07-28*

---

## Pattern 1: The Three-Tier Decision Tree

**Source:** Chapter 2: The Three Tiers (sections 2.6–2.7)

The foundation of multi-agent system design. Three tiers exist; each solves a different problem. Using the wrong tier is not a bug—it is a category error.

### The Three Tiers

| Tier | Mechanism | Setup | Survives Session | Cross-Machine | Best For |
|------|-----------|-------|------------------|---------------|----------|
| **Tier 1: Agent tool (Arrows)** | In-process subagent | Instant | No | No | Research, debate, transform under 5 min |
| **Tier 2: TeamCreate (Squads)** | Named agents, task-tracked | ~30s | No | No | Coordinated implementation, 3-5 agents, under 30 min |
| **Tier 3: tmux + `claude -p` (Federation)** | OS processes, tmux sessions | ~60s | **Yes** | **Yes** | Long-running (hours+), needs human visibility, cross-node |

### The Decision Tree (Use This)

Ask these questions in order. Stop at the first "yes."

1. **Will the work outlive my session, or does it need to run on another machine?**
   → **Tier 3.** No further questions.

2. **Do multiple agents need to coordinate or report progress to a lead?**
   → **Tier 2.**

3. **Can I describe this as "two to five independent reads or transforms, under five minutes"?**
   → **Tier 1.**

4. **None of the above?**
   → **Do it yourself.** Most tasks fall here. The third agent costs more in coordination than it saves.

**Meta-rule:** Prefer the lowest tier that works. Tier 1 is cheaper to spawn, cheaper to debug, cheaper to clean up.

---

## Pattern 2: The Reporting Contract

**Source:** Chapter 3: The Message Bus (sections 3.5–3.6); Chapter 8: The Federation Agent (sections 8.3–8.4)

*An agent that finishes silently has not finished.*

This pattern is load-bearing. Every multi-agent system fails the same way: agents complete their work, produce beautiful output, and exit without telling anyone.

### The Contract (Verbatim)

When spawning a Tier 2 or Tier 3 agent, include this in the prompt, **exactly as written**:

```
When you are done, run: `maw hey <orchestrator-name> "<task-id> complete: <one-line summary>"`
Do not exit until that command has succeeded.
```

For Tier 2 (TeamCreate), use `SendMessage` instead:
```
When done, call: SendMessage({ to: "team-lead", message: "Task <id> complete: <summary>" })
```

### Why This Matters: The Four Attempts

The WASM team (Chapter 3, section 3.6; Chapter 8, section 8.4) required four attempts:

1. **Attempt 1:** Prompt said "report when complete." → Agents finished silently; work was discovered only via `maw peek`.
2. **Attempt 2:** Added "send a message via maw hey." → Messages sent to wrong targets; no federation name resolution.
3. **Attempt 3:** Added explicit recipient. → Two of three agents reported; third assumed it had "already reported" from an earlier confidence test and exited.
4. **Attempt 4 — the one that worked:** 
   - Specified the exact command (not just "report back")
   - Specified the timing explicitly ("WHEN DONE ... and not before")
   - All three agents reported within minutes of completion.

**Key lesson:** The contract must specify *channel*, *recipient*, and *when to send*. Two out of three is silent agents.

---

## Pattern 3: The Lead-Compiles Pattern

**Source:** Chapter 4: Task Tracking (section 4.3)

*If two agents both think they own the file, one of them is about to lose work.*

The single most important convention for coordinated teams. Prevents merge conflicts and keeps work atomic.

### The Rule

**Only the lead writes to the canonical branch (main or integration).**

Subordinate agents:
- Work in separate `git worktree` instances
- Commit to their own branches
- Open branches or report "branch ready"
- **Never merge to main themselves**

The lead:
- Reads each agent's report and branch diff
- Decides which branches to keep and in what order
- Merges to main in sequence
- Runs final tests before pushing

### Why This Works

1. **No write contention:** Three agents touching the same file is fine if each is on its own branch.
2. **Lead arbitrates:** When two agents touched the same file, the lead reads both reports, resolves semantic conflicts, writes the final commit message.
3. **Atomic visibility:** Work appears as one clean merge, not a confusing zigzag.
4. **Failure is local:** If one agent's code is broken, the lead drops that branch and the others are unaffected.

### Anti-Pattern (Tried at Hour 33, Abandoned at Hour 36)

Three agents on three feature branches, all auto-merging to a shared integration branch. By the third merge, the integration branch was failing in ways no individual agent had introduced. Lead spent more time bisecting than agents spent implementing.

---

## Pattern 4: The Research Swarm with Wave Execution

**Source:** Chapter 5: The Research Swarm (sections 5.2–5.5)

*Reading is embarrassingly parallel. You are just the scheduler.*

The most reusable pattern for extracting information quickly without consuming the lead's context.

### The Canonical Pattern

Spawn 3–5 **Haiku** agents in parallel, each with one narrow question, each returning a bounded report.

```typescript
// Single message with multiple concurrent Agent calls
Agent({ subagent_type: "Explore", description: "API surface",
        prompt: "Map every public export of framework/lib..." })
Agent({ subagent_type: "Explore", description: "Routing internals",
        prompt: "How does request dispatch work..." })
Agent({ subagent_type: "Explore", description: "Validation path",
        prompt: "Trace how schemas become validators..." })
Agent({ subagent_type: "Explore", description: "Plugin composition",
        prompt: "How do plugins compose via .use()..." })
Agent({ subagent_type: "Explore", description: "Testing patterns",
        prompt: "What are canonical e2e test idioms..." })
```

### Wave Execution: Surface First, Deep Only If Needed

Do NOT spawn 15 agents up front "to be thorough."

**Wave 1 (always):** 3–5 agents with broad, surface-level briefs.  
→ "Map the landscape."

**Wave 2 (only if Wave 1 surfaces a specific unknown):** 1–3 agents with narrow, deep briefs.  
→ Target exactly the gap discovered in Wave 1.

**Example (Elysia migration):**
- Wave 1: 5 agents on `/learn --deep` → produced 123K of structured docs in 2 minutes.
- Wave 2: Did not exist. The five summaries gave enough to draft the migration plan.
- (Later, when a bug appeared): Wave 2 spawned: one agent targeted "find every reference to `error` as export in elysia v1.4.28" → returned none in seconds, saved hours of trial-and-error.

### Report Contracts (Three Parts)

Specify these three things in every research swarm prompt:

1. **Length bound:** "under 400 words" or "three paragraphs" or "one table." Bounds force compression.
2. **Structure:** Required section headers, required fields, output format. When five agents all return documents with identical structure, synthesis reduces from "interpret" to "concatenate and dedupe."
3. **Citation discipline:** File paths and line numbers for every factual claim. No citations, no claim.

**Example strong contract:**
```
Explain Elysia's validation pipeline in under 400 words.
Required sections:
  (1) Schema declaration
  (2) Request parsing
  (3) Error shape on failure
Cite file:line for every claim. If version-dependent, state the version.
```

### Metrics: Elysia Case Study

| Metric | Value |
|--------|-------|
| Agents | 5 (all Haiku) |
| Wall time | <2 minutes |
| Output | ~123K markdown (5 files, structured) |
| Tokens in lead's context | Only summaries (~5K) |
| Equivalent sequential work | 40+ minutes |
| Token multiplier | ~5× (but in sub-agent windows, not lead's context) |

---

## Pattern 5: Task Tracking Lifecycle

**Source:** Chapter 4: Task Tracking (sections 4.1–4.2)

*If two agents both think they own the file, one of them is about to lose work.*

A simple, built-in runtime primitive that keeps three agents from stomping on each other.

### The State Machine

Four states: `pending` → `in_progress` → `completed` → (implicit: `deleted`)

Two critical fields:
- **`owner`:** The name of the agent currently responsible. Routing point.
- **`blockedBy`:** List of task IDs that must reach `completed` before this task can start.

### The Lifecycle (Protocol)

1. **Lead creates:** `TaskCreate({ subject, description, owner })`  
   → Created `pending`, with owner pre-assigned or empty for first-available claim.

2. **Agent claims:** `TaskUpdate({ taskId, owner: "<my-name>", status: "in_progress" })`  
   → Claim and start in one call.

3. **Agent works:** May call `TaskUpdate` to refine subject or note progress. Should NOT call it just to say "still working."

4. **Agent completes:** `TaskUpdate({ taskId, status: "completed" })` **before** `SendMessage`.  
   → Order matters. Lead will look at `TaskList` while reading the report.

5. **Lead compiles:** When all tasks are `completed`, lead reads reports and merges.

### Why Not a TODO List in Markdown?

The runtime primitive buys you:

- **Microsecond latency:** Every agent can call `TaskList` anytime without file I/O.
- **Structured status:** `completed` is an enum, not a string. Runtime can filter and render it.
- **`blockedBy` is enforceable:** Tasks with unsatisfied deps cannot be claimed. Runtime enforces; convention requires agent memory.
- **`owner` is a coordination point:** Two agents racing for an unassigned task is resolved by the runtime, not by hope.

### Example: Dependency Chain

```typescript
TaskCreate({ subject: "Compile",  owner: "builder" });                // id 1
TaskCreate({ subject: "Test",     owner: "tester", addBlockedBy: ["1"] });  // id 2
TaskCreate({ subject: "Package",  owner: "packager", addBlockedBy: ["2"] }); // id 3
```

`tester` cannot claim task 2 until task 1 is `completed`. Runtime enforces it.

---

## Pattern 6: Implementation Team with Worktree Isolation

**Source:** Chapter 7: The Implementation Team (sections 7.2–7.3)

*Two agents editing the same file is not parallelism. It is a merge conflict with extra steps.*

When you need coordinated, parallel code writing without merge conflicts.

### The Four Rules

**Rule 1: Named roles, not "worker 1" and "worker 2."**  
A role is a contract. "Safety auditor," "test author," "verifier." Roles constrain scope and shape reports. When roles are named, cross-talk gets coherent.

**Rule 2: Worktree isolation per agent.**  
Each agent works in a separate `git worktree` on its own branch. No two agents ever see the same working tree. The filesystem enforces what the prompt cannot: you cannot edit a file you cannot see.

**Rule 3: Only the lead writes to main.**  
Agents push branches. Agents report "branch ready." The lead is the only actor authorized to merge. This discipline is the single thing that distinguishes working teams from broken ones.

**Rule 4: Every agent reports back via TaskUpdate.**  
No status polling. Agents mark their task `completed` when done. Lead reads `TaskList` and merges. Silence means not-done.

### The Canonical Pattern

```typescript
// Step 1: Create the team container
TeamCreate({
  name: "wasm-hardening",
  members: [
    { name: "safety",   role: "WASM runtime safety auditor",   model: "opus"   },
    { name: "tester",   role: "Test author for host functions", model: "sonnet" },
    { name: "verifier", role: "End-to-end runtime verifier",    model: "sonnet" },
  ],
})

// Step 2: Create tasks, one per agent, with explicit deliverables
TaskCreate({
  id: "1",
  subject: "Audit memory protocol",
  description: "Review src/wasm/memory.ts for OOB reads. Confirm 16MB cap. Branch: chore/wasm-safety-audit.",
  owner: "safety"
})
TaskCreate({
  id: "2",
  subject: "Write host function tests",
  description: "Cover maw_print, maw_identity, maw_send, maw_fetch. Branch: test/wasm-host-functions.",
  owner: "tester"
})

// Step 3: Send initial messages to each agent
SendMessage({ to: "safety",   message: "Task #1 is yours. Work in wt-safety. Report via TaskUpdate." })
SendMessage({ to: "tester",   message: "Task #2 is yours. Work in wt-tester. Report via TaskUpdate." })
SendMessage({ to: "verifier", message: "Task #3 is yours. Work in wt-verifier. Report via TaskUpdate." })

// Step 4: Lead waits and polls TaskList
// Step 5: Lead merges in dependency order, runs tests, pushes main
```

### Drawing the Seams: How the Lead Decomposes Work

Three heuristics for splitting tasks so no two agents need the same file:

1. **By module:** If `src/wasm/memory.ts` and `src/wasm/host.ts` don't import each other, assign one to each agent.
2. **By concern:** "Audit" (read + small patch), "test" (new test file), "verify" (e2e harness) are naturally disjoint.
3. **By branch:** If two tasks both require editing `src/server.ts`, they are not two tasks—they are one task with two sub-steps. Collapse or sequence across waves.

**If you cannot find a clean seam, the task is wrong.** Reshape before spawning. A team cannot rescue a plan that is not decomposable.

### Model Choice Per Role

Not every role needs Opus:

| Role shape | Default model |
|------------|---------------|
| Auditor, designer, architect, reviewer | Opus |
| Implementer, test author, refactorer | Sonnet |
| Reader, summarizer, file-sweeper | Haiku |

Mixing models across the team is economical and correct. A three-Opus team costs 3× a three-Haiku team and is only right when all three roles need Opus-grade reasoning.

### Metrics: The wasm-hardening Team

| Measurement | Value |
|---|---|
| Team members | 3 (1 Opus, 2 Sonnet) |
| Wall time | ~4 minutes |
| Branches produced | 3 |
| **Merge conflicts** | **0** |
| Tasks completed | 3/3 |
| Files touched by >1 agent | 0 |

Zero merge conflicts is not luck. It is a consequence of worktree isolation + disjoint task assignment.

### When Teams Are the Wrong Tier

- **Should have been a swarm:** If all three tasks are "read X and report," use a research swarm, not a team.
- **Should have been tmux:** If tasks take >30 minutes or need to survive a session crash, use federation, not a team.

**Decision tree (compressed):** Under 30 minutes and writes code → team. Over 30 minutes or cross-session → tmux. Read-only at any scale → swarm.

---

## Bonus: When Comfort Is a Tell

**Source:** Chapter 8: The Federation Agent (section 8.9)

> "I kept defaulting to the Agent tool because it's comfortable. But his instinct was right: real processes are independently killable, peek-able, survive session death. My comfort with the Agent tool was making me avoid the harder, better path."

When you notice you are reaching for the same tier repeatedly, ask: *Am I using this because it fits the task, or because it fits my hands?*

The Agent tool is a hammer. Federation is a nail gun. They are not the same tool. Tasks that need the nail gun will feel wrong in the hammer—they should. That wrongness is the signal.

---

## Summary: Quick Reference

| Pattern | Problem It Solves | Key Insight |
|---------|------------------|-------------|
| Three-Tier Decision Tree | Choosing the right tool | Outlive session? → Tier 3. Coordinate? → Tier 2. Quick read? → Tier 1. |
| Reporting Contract | Silent agents | Must specify channel, recipient, and **when** to send. "Let me know" is not a contract. |
| Lead-Compiles | Merge conflicts | Only lead writes to main. Subordinates on own branches. Prevents write contention. |
| Research Swarm | Reading at scale | 3–5 parallel Haiku agents, wave execution (surface first, deep if needed), bounded reports. |
| Task Tracking Lifecycle | Coordination without polling | Runtime primitive: `pending` → `in_progress` → `completed`. `owner` and `blockedBy` do the routing. |
| Implementation Team | Parallel code writing | Worktree isolation + named roles + lead compiles = zero merge conflicts. |

