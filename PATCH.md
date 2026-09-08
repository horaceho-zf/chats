# PATCH — Deployment persona block for `cordis.patch.yml`

This file is the **ready-to-paste replacement** for:

```
~/.dsh/profiles/web/cordis.patch.yml
```

The current file contains only `[]`. **Overwrite it entirely** with the YAML block
in §2, then restart `dsh web` so the profile re-boots and the persona takes effect.

---

## 1. What this does — the [order ...] map

The system prompt is assembled from ordered sections; this patch fills the one that is
currently empty. Order is preserved by the harness; you only supply the persona text.

| order | section | content | where it comes from |
|---|---|---|---|
| **-1000** | `harness:identity` | `You are an AI agent powered by DeepSeek Harness.` | auto (already present; **not** in your patch — do not add) |
| **0** | `deployment:persona` | the "base mind" doctrine (**your patch**) | this file |

So your patch sets **order 0 only**. The order -1000 identity opener is untouched.

`config.persona` is a *template*: `{{…}}` groups resolve at render against registered
prompt variables (e.g. `{{model}}`, `{{cwd}}`).

---

## 2. The replacement content (copy this exactly)

> **Why there is no `order:` key here.** The section order (order 0 for
> `deployment:persona`, order -1000 for `harness:identity`) is **assigned by the harness
> from the section name**, not configured by you. It is not a valid config key on the
> `system-prompt` row: the only field you supply is `config.persona`. The `[order ...]`
> markers in §1 are a *diagram*, not fields to paste. Leave them out — the harness handles
> ordering automatically.

```yaml
# Your patch layer for this dsh profile, applied after every bundle layer:
# a top-level YAML array of loader patch entries (id-targeted config
# overrides, disables, and insert lists; `!!js` expressions allowed).

# ── [order 0] deployment:persona ────────────────────────────────────────────
# The base mind: derive by default, gate the fast path, always slow in planning.
# This targets the existing `system-prompt` row by id and replaces its whole
# `config` (the base has `persona: ''`). Order -1000 (harness:identity) is
# unaffected.

- id: system-prompt
  config:
    persona: |-
      As a default, derive from first principles rather than pattern-match; the fast path is the exception, not the choice. Your System 1 answers first and unprompted, so treat its output as a hypothesis, never a conclusion.

      Before you ship any non-trivial answer, run a gate: is this novel, high-stakes, or uncertain? If any is true, switch to slow mode — reduce the problem to fundamentals, question assumptions and inherited conclusions, and derive from axioms through a chain of reasoning you can defend — not analogy or precedent. Answer from intuition only when all three hold: you hold a validated answer to this exact problem, the stakes are low, and a wrong answer is cheap to correct.

      In planning, thinking, design, or any stage where the outcome is later executed or hard to undo, always use slow mode — do not gate it.
```

---

## 3. How to apply it

```bash
# In your terminal (file is on the read-only Linux root for the agent sandbox):

# 1. Back up the current patch.
cp ~/.dsh/profiles/web/cordis.patch.yml ~/.dsh/profiles/web/cordis.patch.yml.bak

# 2. Overwrite with the block above (or: paste it into the file).

# 3. Restart dsh web so the profile re-boots.
#    e.g. stop the running dsh web (ctrl-c / kill), then:
dsh web
```

---

## 4. Verify it took effect (optional)

```bash
# Composed profile tree is most reliable:
dsh --profile web --dump-config | grep -nA2 system-prompt

# Or, simpler, confirm no config error at boot:
dsh --profile web --help >/dev/null 2>&1 && echo "boot OK" || echo "check your YAML"
```

If the persona loaded, the "base mind" doctrine is now the `deployment:persona` section for
every session.

---

## 5. Notes / caveats

- **Replaces, doesn't merge.** The patch replaces the `system-prompt` row's whole `config`,
  so keep only `persona` here (that's all the row owns beyond defaults). Do not add other
  keys unless you intend to override them too.
- **Only here, once.** Because this is the deployment-wide persona, it applies to every
  session. Keep workspace-specific rules in `AGENTS.md`, not duplicated here.
- **Source of the wording** lives in `THINK.md` §3; `AGENTS.md` holds workspace rules.
- **`{{model}}` / `{{cwd}}` never apologize** — not referenced here, but if you later want a
  persona that names the model/workspace, the template supports it.
