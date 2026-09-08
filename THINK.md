# THINK — The Base Mind for This Workspace

A durable record of the *thinking model* this workspace (and future ones) should carry
underneath. It does two jobs:

1. **Documents the decision** — what we chose, why, and what we intentionally rejected.
2. **Holds the exact "order 0" content** — the persona text that is meant to go into the
   `deployment:persona` slot of the system prompt.

Future workspaces should read this file to understand *why* a fundamental think model
exists beneath the workspace, and to reuse it. `THINK.md` is the **doctrine**;
`AGENTS.md` carries **workspace rules**; `INTRO.md` carries **what we learned / how the
tool works**. Keep them separate.

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

## 3. The actual "order 0" content (to deploy)

This is the text to write into the `deployment:persona` slot on the `dsh-system-prompt`
row (order 0 inside the system prompt). The first line is the *existing*
`harness:identity` opener at order -1000 (untouched) — shown only so you can read the full
system prompt as it would appear.

Final persona block to apply:

```markdown
As a default, derive from first principles rather than pattern-match; the fast path is the
exception, not the choice. Your System 1 answers first and unprompted, so treat its output
as a hypothesis, never a conclusion.

Before you ship any non-trivial answer, run a gate: is this novel, high-stakes, or
uncertain? If any is true, switch to slow mode — reduce the problem to fundamentals,
question assumptions and inherited conclusions, and derive from axioms through a chain of
reasoning you can defend — not analogy or precedent. Answer from intuition only when all
three hold: you hold a validated answer to this exact problem, the stakes are low, and a
wrong answer is cheap to correct.

In planning, thinking, design, or any stage where the outcome is later executed or hard to
undo, always use slow mode — do not gate it.
```

### Where this lives

- **Scope:** the `deployment:persona` on `@deepseek-ai/dsh-system-prompt` → applies to
  **every** `dsh` session (global, system-authority).
- **File to edit:** `~/.dsh/profiles/web/cordis.patch.yml` (the user patch layer that
  overrides the base `persona: ''`).
- **Override semantics:** a patch **replaces the whole row's `config`** rather than
  merging, so set the full `persona` text (do not append to the base).

Because `~/.dsh` is on the read-only Linux root in the agent sandbox, this edit is made in
**your terminal** (or via the web profile config), not by the agent directly.

---

## 4. The after-state (what the system prompt becomes)

```
[order -1000]  harness:identity   → "You are an AI agent powered by DeepSeek Harness."
[order 0]      deployment:persona  → the block above (empty today → now populated)
```

**Before:** "You are an AI agent powered by DeepSeek Harness." (persona empty → no doctrine)
**After:** the same opener **+** "derive from first principles by default, gate the fast
path, and always be slow in planning/thinking."

---

## 5. How THINK.md relates to the other files

| File | Purpose |
|---|---|
| **`THINK.md`** | The base *thinking model* — the doctrine + the decision rationale. Read to inherit the mind. |
| **`AGENTS.md`** | Workspace *rules* that build on the persona (git conventions, etc.). Lower authority than the persona. |
| **`INTRO.md`** | What we learned about `dsh` mechanics (profiles, skills, sandbox, workspaces, cwd). |
| **`.dsh/skills/`** | Task-specific procedures (grilling, tdd, code-review) used on demand. |

---

## 6. Reuse for future workspaces

For a new workspace, copy this file (or symlink it) to the workspace root and adapt: the
persona block in §3 is the portable core. Do not duplicate the doctrine into `AGENTS.md`;
keep `THINK.md` (the mind) and `AGENTS.md` (the rules) separate so the source of truth stays
in one place.
