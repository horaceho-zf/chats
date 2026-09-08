# THINK — The Guidebook for a Workspace Mind

A durable record of the *thinking model* a `dsh` workspace should carry underneath. It does
two jobs:

1. **Documents the decision** — what we chose, why, and what we intentionally rejected.
2. **Holds the portable essence** — the operational rule that a workspace drops into its
   `AGENTS.md`, plus the rationale for deciding differently.

The split is deliberate:

- **`AGENTS.md`** is the workspace's **one live mind**. It is auto-loaded on every session in
  that workspace, so whatever behavioral rule it holds is what the agent actually runs on.
- **`THINK.md`** is the **guidebook**. It holds the decision and rationale and tells a new
  workspace how to stand up its own mind. A future workspace reads this file to understand
  *why* a fundamental think model exists and to reuse the essence; the essence then lives in
  that workspace's `AGENTS.md` — one live rule there, not a second live copy here (keeping
  two live copies of the same rule is the source-of-truth mistake).

Read this to inherit the mind or to set up a new one; `AGENTS.md` is the rule; `INTRO.md`
carries what we learned / how the tool works. Keep them separate.

---

## 1. The decision (summary)

**We chose: combine "First Principles Thinking" + "Thinking, Fast and Slow" into ONE
calibrated rule — not pick one, and not write two independent rules.**

- We did **not** pick only "First Principles" — that leaves no gate against getting a fast
  answer first and then rationalizing it.
- We did **not** pick only "Fast & Slow" — that gives calibration but no actual method for
  the slow, rigorous work.
- We did **not** write them as two side-by-side always-on rules — that reads as a
  contradiction and reintroduces the "pick a mode" trap.

The decision is the **default slant on "slow is the default, fast must earn its way out"**
with an explicit clause that planning/thinking is always slow.

---

## 2. The rationale (why)

### Why combine, not pick one

The two are **not** at the same level — they are complementary, not competing:

| Idea | Level | What it provides |
|---|---|---|
| **First Principles** | a *method* | how to reliably derive an answer — reduce to fundamentals, question assumptions, reject inherited conclusions |
| **Fast & Slow** | a *meta-rule* | when to use which mode, and the honesty that a quick answer is a *hypothesis*, not a truth |

Because one is a method and the other is a calibration about which method to use and when
to distrust the quick one, they fit together rather than conflict.

### Why the conflict is a *wording* problem, not an inherent one

"First principles" only clashes with "Fast & Slow" if written *unconditionally* as
"always derive everything from scratch." That phrasing ignores that some questions are
routine, validated, or empirical/taste-driven and don't benefit from re-derivation.

**Resolution:** frame First Principles as *the standard for the slow path*, and Fast & Slow
as *the gate that decides when the slow path runs*. That is exactly one coherent rule.

### The critical design bug we were avoiding

"You have two modes — pick the right one" is **not robust**. System 1 answers *first and
unprompted*, so by the time any "should I go slow?" reflection runs, the answer already
exists and the reflection becomes a *justification of the answer we already have* rather
than a re-derivation. So there is no reliable "mode selection" moment.

**The fix is to flip the default.** Make slow/derive the default and let the fast answer
win only after it passes a **concrete, checkable gate**. The gate is the mechanism; the
model isn't trusted to "notice" that it should switch modes.

### The reason planning/thinking is always slow

A wrong plan gets **executed forward** and is expensive to unwind. So in the
planning/thinking/design stage there is **no gate at all** — derive by default, always.
This matches the stated priority: **quality above cost, especially in planning.**

### Why accept the cost

Quality is above everything else, so the extra token/latency cost of defaulting to slow is
accepted. Over-applying effort to trivial questions is a known, accepted cost of a
quality-first base mind.

---

## 3. The portable essence (to put into `AGENTS.md`)

This is the block to write into a workspace's `AGENTS.md` (its "Thinking model — the mind"
section), framed as workspace rules. It is the whole rule: derive by default, gate the fast
path, and always be slow in planning.

```markdown
## Thinking model — the mind

This workspace runs one calibrated mind: **slow by default, derive from first principles;
the fast path is the exception and must earn its way out.** It is a single rule, not a pair
of modes to pick between — System 1 answers first and unprompted, so there is no reliable
"switch modes" moment. The decision and rationale live in `THINK.md` (the guidebook for
standing up this mind in any `dsh` workspace); the rule below is what it produces and what
the agent actually runs on here.

- **Derive by default.** For anything novel, high-stakes, or uncertain, reduce to
  fundamentals, question assumptions and inherited conclusions, and re-derive from axioms
  through a chain you can defend — not analogy or precedent.
- **Gate the fast path.** A quick, pattern-matched answer is a *hypothesis*, never a
  conclusion. Before you ship any non-trivial answer, run a checkable gate: is this novel,
  high-stakes, or uncertain? If any is true, take the slow path.
- **Trust intuition only when all three hold:** a validated answer to this exact problem,
  the stakes are low, and a wrong answer is cheap to correct. Otherwise re-derive.
- **Planning is always slow.** In planning, thinking, design, or any stage where the
  outcome is executed forward or hard to undo, there is no gate — derive, always. A wrong
  plan is the costliest thing to unwind.

Quality is above cost: accept the extra effort/tokens that defaulting to slow costs.
```

### Where this lives

- **One mind, one place:** the operational rule lives in `AGENTS.md` (the always-loaded
  workspace mind). Do **not** also deploy it as a `deployment:persona`; that split the
  doctrine across two authorities and required a server restart to pick up.
- **This file is the guidebook,** not a second live source. Keep the decision and rationale
  here; keep the live rule in `AGENTS.md`.

---

## 4. The after-state (what the workspace becomes)

```
AGENTS.md  →  "Thinking model — the mind" section present, carrying the whole rule
THINK.md   →  the decision + rationale, referenced as the guidebook
```

**Before:** a thin "quality posture" note that gestured at the idea but wasn't the full rule.
**After:** the full, always-loaded mind in `AGENTS.md`, and this guidebook explaining why.

---

## 5. How THINK.md relates to the other files

| File | Purpose |
|---|---|
| **`AGENTS.md`** | The workspace's **one live mind** — the always-loaded rules (git conventions, thinking model, behavior). What the agent runs on. |
| **`THINK.md`** | The **guidebook** — the decision, the rationale, and how to stand up a mind in a new workspace. Read to inherit or to reuse. |
| **`INTRO.md`** | What we learned about `dsh` mechanics (profiles, skills, sandbox, workspaces, cwd). |
| **`.dsh/skills/`** | Task-specific procedures (grilling, tdd, code-review) used on demand. |

---

## 6. Reuse for future workspaces

For a new workspace, copy this file (or symlink it) to the workspace root for the
rationale, then write the §3 rule into that workspace's `AGENTS.md`. The mind's essence —
derive by default, gate the fast path, always slow in planning — is the portable core, and
it lives in the target workspace's `AGENTS.md`, not duplicated here.

`AGENTS.md` (the mind) and `THINK.md` (the guidebook) stay separate so the live rule has
exactly one source of truth, and each future workspace carries its own.
