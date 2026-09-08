# INTRO — dsh reference & installed skills

A reference note on how `dsh` works and the skills installed in this workspace. It
covers (1) the mechanics that mattered while setting up — profiles, skills, the sandbox,
and how workspaces scope context and access — and (2) a tour of the 14 project-scoped
skills in `.dsh/skills/`: what each is for, how to trigger it, and how to pick the right
one at each stage of software development.

This is **not** the repo entry point — for what this repo is and how to use it, see
[`README.md`](README.md). For the agent's operating rules, see [`AGENTS.md`](AGENTS.md).

---

## 1. What we learned today

Starting from a Codex background, we mapped **how `dsh` (DeepSeek Harness) does things**
and installed a set of skills. The four ideas below are the ones that repeatedly
mattered.

### 1.1 `dsh` is a *profile*-based harness, not one app

`dsh` boots a **profile** — an ordered stack of plugin bundles under your own patch
layers. Key profiles:

| Command | What it runs |
|---|---|
| `dsh web` | the web GUI (what we used) — `http://127.0.0.1:3080` |
| `dsh --profile headless "job"` | one-shot task, prints the answer, exits |
| `dsh --profile sdk` / `--profile sdk-minimal` | JSON-RPC SDK server |
| `dsh --profile acp` | ACP automation server |

Sessions persist under `~/.dsh/sessions/`. Config lives in `~/.dsh/settings.yaml`,
and the *profile* config in `~/.dsh/profiles/<name>/cordis.patch.yml` (and `--patch`
overlays).

### 1.2 Skills are files, not plugins

A skill is a plain directory `<name>/SKILL.md` (or a flat `<name>.md`) dropped into a
**scanned root**. It has YAML frontmatter (`name`, `description`, plus optional
`disable-model-invocation`, `user-invocable`) and the body contains the actual
instructions. The harness **watches** these roots, so adding a skill needs no restart.

Discovery roots (priority order, lower wins on a name clash):

| Rank | Scope | Path |
|---|---|---|
| 100 | project | `<projectRoot>/.dsh/skills` |
| 200 | project | `<projectRoot>/.agents/skills` |
| 300 | custom | `Config.customSkillDirs` |
| 400 | global | `~/.dsh/skills` |
| 500 | global | `~/.agents/skills` |

The **project root** is the nearest ancestor containing `.git`; if there's no `.git`,
it falls back to the current cwd. (In our session, `/mnt/c/Users/horaceho/Chats` has no
`.git`, so project scope === session workspace.)

### 1.3 Two invocation surfaces

- **Model-invocable** — the agent can call the `skill` tool itself (you often don't even
  have to ask).
- **User-invocable only** (`disable-model-invocation: true`) — the model can't load it;
  *you* invoke it with a `/name` token, e.g. `/grill-me`.

The catalog you saw injected into the conversation lists only **model-invocable** skills
(the summary + "call the skill tool if the task matches"). User-only skills are invisible
to the model until you type `/name`.

### 1.4 The read-only sandbox (the "why can't dsh write to ~/.dsh" mystery)

Inside the agent's command shell, the harness mounts the **Linux root filesystem `/`
read-only** and only grants write access to the session's workspace mount:

```
/dev/sdd on / type ext4 (ro,...)      ← /home/ohho, ~/.dsh live here → read-only
C:\ on /mnt/c/Users/horaceho/Chats    ← session workspace → writable
```

So the agent **can** create/edit skills in **project** scope (`/mnt/c/.../.dsh/skills`,
on the writable C: mount), but **cannot** write global skills under `~/.dsh/skills`.
That is not a permissions bug and not caused by `ctrl-z`/`bg`; the ownership (`ohho`,
`755`) is correct. It's a per-process mount boundary. Reads work everywhere; only writes
to the Linux root are fenced.

Consequence / recommended split:

| Skill location | Writable by agent? | Who authors it |
|---|---|---|
| project `.dsh/skills` | ✅ yes | the agent (this is what we did) |
| global `~/.dsh/skills` | ❌ no | you, in your own terminal |

If you ever want agent-writable *global* skills, use the per-call **escalation** flow
(the sandbox marks a denial, and a human approves a one-shot wider mode) — there is no
config setting that extends the writable path set in this build.

### 1.5 Workspaces and context — the "switch a department" mental model

A workspace is a **named user directory**, and it is only a **host-side grouping** (a
sidebar of projects). We verified directly in `dsh-workspace`: it is host-side only, so
it **adds no tokens, prompts, or request context** to the model. Switching workspace in
the GUI just tells the host "show sessions that ran in this directory."

The unit of **context** is the **session log** — an event-sourced, append-only log per
session. The model-visible message history is **derived** from it (`deriveMessages()`),
never stored separately. On disk, each session is keyed to the directory it ran in
(normalized cwd), which we confirmed from a real log header:

```
~/.dsh/sessions/
  --mnt-c-Users-horaceho-Chats--/            ← normalized-cwd project directory
    session-092d8761-.../                    ← one session per folder
      session.jsonl.zstd                     ← the append-only event log
```

The header literally records the linkage, e.g. `"cwd":"/mnt/c/Users/horaceho/Chats"`.

**So "switching workspace" does not swap the model's memory — it's file-namespace
isolation.** Two consequences to remember:

- **"Forgetting" another workspace = scoping, not deletion.** A session's log lives only
  under *its own* directory's project folder. Working in workspace B, the agent's context
  is only B's current session log; A's logs live in a different project folder and are
  never consulted, and nothing is erased.
- **"Picking up" context = durability + re-derivation, not transfer.** Every event is
  appended to the log; compaction can *shadow* older surface entries but never deletes
  history. Reopening a session runs `deriveMessages()` on its log to rebuild the history.

**Key implication:** context does **not** automatically carry from one workspace to
another. Opening a new workspace starts a **fresh** session whose context is only what
is in that session. To carry understanding deliberately, use one of:

| Want | Use |
|---|---|
| Branch from this session | `ctx.sessions.fork()` — child session from a stable prefix, with lineage metadata |
| Hand off to a new session / another agent | the `handoff` skill (`/handoff`) — compacts the conversation into a doc the next session reads |

### 1.6 Cross-workspace access — read is open, write is fenced

The sandbox fence is **not** "the agent can only see the workspace." It is specifically:

| Operation | Outside the workspace (verified on `~/codes/spoke/...`) |
|---|---|
| **Read** a file (`read` tool) | ✅ works |
| **Search** / glob (`grep`, `glob`) | ✅ works |
| **Write** / edit | ❌ **denied** (`Read-only file system`) |

Reads and searches are **open** across the accessible mounts (an external folder
resolves onto the filesystem, e.g. `~/codes` → C:, `fs=69`). Only **mutations** are
confined to the workspace root.

**Best practice for reading files/documents outside the current workspace:**

1. **Just reference the absolute path** (or the `~/codes` symlink) — no special
   incantation. *"Read `~/codes/ish/hub/hubapi/README.md` and compare X."*
2. I can `read` specific files, or `glob`/`grep` a whole external tree, directly.
3. **For sustained cross-project work**, prefer making that project the session
   workspace (so writes + the session log are scoped there), or use `handoff` to carry
   understanding between workspaces.
4. **Never assume I can write there.** Modifying files in another project requires that
   project to be the session workspace, or a per-call sandbox **escalation** with your
   approval. Reading for information is always fine.

> **Rule of thumb:** `dsh` scopes **context** (the session log) and **writes** to the
> workspace; **reads/searches** are open across the mountable filesystem. So pulling
> reference material from `~/codes/...` into the current workspace's session is
> straightforward — just point me at the path.

---

## 2. The installed skills (project scope, `.dsh/skills/`)

14 skills, from [mattpocock/skills](https://github.com/mattpocock/skills). Each bundle
kept its referenced resources (e.g. `tdd/mocking.md`, `codebase-design/DEEPENING.md`,
`teach/*-FORMAT.md`), so none are truncated.

Legend: **M** = model-invocable (agent can load it); **U** = user-invocable (also
invoke with `/name`).

| Skill | Surface | Use it for |
|---|---|---|
| `grilling` | M | **Core primitive.** Relentlessly interview the user to turn a loose idea into decisions. |
| `grill-me` | U (`/grill-me`) | Router → `grilling`. A no-docs, stateless grilling session (subject needn't be code). |
| `grill-with-docs` | U | Router → `grilling` + `domain-modeling`. Same interview, and also writes `CONTEXT.md` + ADRs as it goes. |
| `domain-modeling` | M | Build/sharpen a domain model: challenge terms, write `CONTEXT.md` and ADRs. |
| `codebase-design` | M | Deep-module design vocabulary: where a seam goes, making code testable and AI-navigable. |
| `tdd` | M | Red-green-refactor loop; build features or fix bugs test-first. |
| `diagnosing-bugs` | M | Disciplined hard-bug loop: build a red feedback loop → minimise → hypothesise → instrument → fix → regression test. |
| `prototype` | M | Throwaway prototype to answer a design question (state/logic or UI look-and-feel). |
| `code-review` | M | Two-axis diff review (Standards + Spec) via parallel sub-agents. |
| `teach` | U (`/teach <topic>`) | Teach you a new skill/concept over sessions, using the workspace as a stateful teaching area. |
| `handoff` | U (`/handoff`) | Compact the current conversation into a handoff doc for another agent to continue. |
| `to-questionnaire` | U | Turn a decision you can't fully answer into a questionnaire for someone else to fill in. |
| `wait-what` | U (`/wait-what`) | "Stop, that didn't land" — re-pitch a confusing message with the missing context. |
| `writing-for-agents` | M | Write/edit skills, `AGENTS.md`, `CLAUDE.md` well (context pointers, information hierarchy, pruning). |

> Note: `grill-with-docs` depends on `domain-modeling`. We installed `domain-modeling`
> so it's complete. Other engineering skills (`to-spec`, `to-tickets`, `triage`,
> `wayfinder`, `implement`, `improve-codebase-architecture`) are **not installed** — they
> assume a per-repo scaffold via `setup-matt-pocock-skills` (issue tracker, triage labels,
> `CONTEXT.md` layout), and a real repo, so they'd be added later per project.

---

## 3. How to use each — quick triggers

### Grilling & planning (start of a piece of work)

- **`/grill-me`** — "I have a rough idea, interrogate me." Stateless, no repo needed.
  Start this in a *fresh conversation*, not on top of a plan you already had written.
  You answer in rounds; the agent finds facts itself (it won't ask you for anything it
  could look up). Stop when the frontier is empty and you've confirmed shared understanding
  before acting.
- **`/grill-with-docs`** — same, but in a codebase you want to align against: it reads your
  code and keeps what it learns in `CONTEXT.md` + ADRs.
- **`domain-modeling`** — use it directly when you're clarifying terminology or recording
  a decision (`CONTEXT.md` / ADRs). It bolsters both grill paths.

### Building

- **`tdd`** — the default for implementing a feature or fixing a bug with tests as the
  driver. Leaf skills for "what makes a good test" and "mocking" live in its bundle
  (`tests.md`, `mocking.md`).
- **`prototype`** — when a question can't be answered by talking (e.g. "does this state
  model feel right?" or "what should the UI look like?"). Build a throwaway, react to it,
  answer in one line, then go back to `tdd`.
- **`codebase-design`** — consult alongside when you're placing a seam, designing an
  interface, or deciding whether a module should grow or split.

### Debugging

- **`diagnosing-bugs`** — reach for it the moment something is broken/failing/slow. It
  enforces a loop, so it *goes red before you change anything*, which prevents shotgun
  fixes.

### Reviewing & finishing

- **`code-review`** — review a branch/PR/WIP against a fixed point (commit, tag, merge-base)
  along two **separate** axes: Standards (repo conventions + Fowler smell baseline) and
  Spec (does it implement the originating issue). Runs both as **parallel sub-agents** so
  they don't pollute each other; reports side-by-side and never reranks across axes.
  Needs an issue-tracker reference (`docs/agents/issue-tracker.md`) — if that's missing it
  will tell you to run `/setup-matt-pocock-skills`.

### Communication & knowledge (around the work)

- **`teach`** — "teach me X"; stateful, uses the workspace to build fluency.
- **`to-questionnaire`** — "I can't decide this; get the person who can to answer it."
- **`wait-what`** — "that didn't land", fire it immediately.
- **`handoff`** — pass the conversation to another agent (or session) cleanly.
- **`writing-for-agents`** — whenever you write a skill or an `AGENTS.md`/`CLAUDE.md`.

---

## 4. Recommended sequence during a software-development cycle

One way to chain them so each covers a phase without overlap:

```
/incoming idea
     │  (rough, not committed)
     ▼
 /grill-me        ──►  turn idea into decisions (stateless). If a real codebase:
 /grill-with-docs ──►  same + build CONTEXT.md / ADRs (fires domain-modeling)
     │
     ▼  you now have a spec / shared understanding
     │
 /domain-modeling ──►  sharpen terminology + record decisions
 /to-questionnaire ─►  if a decision needs someone else's answer, send it the questionnaire
     │
     ▼  choose an approach, then build
     │
 /codebase-design ──►  decide the seam / interface (consult, don't run-and-done)
 /prototype       ──►  if the design question needs something to react to
 /tdd             ──►  implement test-first (feature or bug-fix)
     │
     ▼  when something breaks or is slow
     │
 /diagnosing-bugs ──►  find the root cause with a loop (before/around tdd fixes)
     │
     ▼  changes are in; verify before you walk away
     │
 /code-review     ──►  Standards + Spec, as parallel sub-agents
     │
     ▼  hand off or keep going
     │
 /handoff         ──►  another agent continues
 /teach / /wait-what ─►  if you want to learn, or a message didn't land
```

**Selection tips (the "which one" decision):**

- Start with a **grill** if you can't yet commit to the thing. It's cheap and catches
  assumptions before they cost you.
- **Stats vs. conversation**: if the question can only be answered by seeing something,
  use `prototype` — don't keep grilling (ungrillable questions balloon the session).
- Choose `grill-me` vs `grill-with-docs` by whether the subject is tied to a codebase you
  want to keep aligned and documented.
- `code-review` is deliberately **two axes, never merged** — a change can pass Standards
  but fail Spec (or vice-versa), and merging them hides that.
- Reach for `diagnosing-bugs` over intuition the moment you say "it's broken/slow."

---

## 5. Notes / gotchas

- **Invoke user-only skills with `/name`** (and possibly an argument, e.g. `/teach
  vector-databases`); the agent cannot fetch them on its own. Model-invocable skills get
  picked up automatically when the task matches.
- **Skill bodies are loaded fresh each call**; the catalog is a summary only. The agent is
  told never to act on a summary without loading the skill.
- **Project vs global.** This install is project-scoped (`.dsh/skills`). It travels with
  the repo once this directory is under `.git`. Global installs go to `~/.dsh/skills` and
  are authored by you (the agent sandbox can't write the read-only Linux root).
- **`domain-modeling` was added to satisfy `grill-with-docs`'s dependency.** If you later
  adopt the engineering suite, run `setup-matt-pocock-skills` per repo first so `code-review`
  and friends have their issue-tracker/docs conventions.
- **This is a handoff, not a spec.** The 14 skills and the sequence above are a sensible
  default; adjust to your real workflow as you go.
