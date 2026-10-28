# Architecture

How the orchestrator workflow is put together, and why.

## Two-layer design

OpenCode V2 separates *behavior* from *binding*:

| Layer | Where | Contains |
| --- | --- | --- |
| **Behavior** | `agents/*.md` (this repo) | System prompt (Markdown body) + `description` and `mode` (frontmatter) |
| **Binding** | `opencode.json` / `opencode.jsonc` (your config; see `opencode.example.json`) | `model` per agent, `permissions` per agent, `default_agent`, optional `mcp` |

The Markdown files are intentionally free of model and permission settings, so the same prompts work on any provider. A filename becomes the agent ID (`reviewer-critical.md` → `reviewer-critical`); OpenCode discovers them in `~/.config/opencode/agents/` (global) or `.opencode/agents/` (project; nested paths become slash-separated IDs — `.opencode/agents/team/reviewer.md` → `team/reviewer`).

Configuration values merge in order: later scalar values replace earlier ones and permission rules append, so your JSON can refine — not rewrite — the reference binding.

## Built-ins vs. custom agents

OpenCode V2 ships visible built-ins `build` (primary), `plan` (primary), `general` (subagent), `explore` (subagent), plus hidden maintenance agents (`compaction`, `title`, `summary`). This setup **overrides built-ins by reusing their IDs** in JSON (models for `build`/`plan`/`general`/`explore`) and adds 8 custom agents as Markdown files. There is no built-in `scout` agent in V2.

## Delegation graph

The orchestrator is the only primary agent in this setup that routinely delegates. Its `subagent` permissions **deny `*` and then allowlist** nine subagents — the built-ins `general` and `explore`, plus the seven custom subagents — an explicit closed list (V2 permission rules are ordered and the last matching rule wins, so the deny-first/allow-after pattern is what makes the allowlist exhaustive).

```text
orchestrator (primary)
├── explore            no edits (built-in)      code search / tracing
├── general            unrestricted             scoped non-UI implementation
├── architect          no edit/shell            system-level decisions
├── tester             no edit tools            reproduce / run checks (shell unrestricted)
├── reviewer           no edit/shell            routine independent review
├── reviewer-critical  no edit/shell            high-stakes audit
└── UI track (sequential)
    ├── ui-designer    no edit/shell            design brief (only if direction unclear)
    ├── ui-builder     unrestricted             implements the approved brief (scope is prompt-level)
    └── visual-qa      no edit/shell            inspects rendered evidence, if any
```

Rules encoded in the orchestrator prompt (policy, on top of config):

- Small, localized work is handled directly — delegation is not reflexive.
- `tester` and reviews are **optional**, chosen by impact and uncertainty; the two reviewers are never both launched automatically for routine edits.
- The UI track is **sequential**: `visual-qa` judges the *result* of `ui-builder`, so it is not run in parallel with implementation. Independent read-only investigations may run concurrently; concurrent edits to overlapping files are forbidden.
- Escalation after two failed attempts with the same strategy; visual correction loops are capped at two before diagnosing the direction/tooling instead of looping.

Non-delegation is **prompt policy, not an enforced boundary**: the prompts of `architect`, `reviewer`, `reviewer-critical`, and `ui-designer` explicitly forbid delegating further, while `tester`, `ui-builder`, and `visual-qa` don't mention delegation at all. The V2 docs additionally state that the built-in `general` cannot launch subagents. Do not rely on any of this as a security guarantee — only the orchestrator's `subagent` allowlist is config-enforced.

## Permission semantics (V2)

Permissions are an **ordered list** of rules; the last matching rule wins, so broad rules go first:

```json
{ "action": "edit", "resource": "*", "effect": "deny" }
```

- `action` — tool/permission action (`edit`, `shell`, `subagent`, `read`, `webfetch`, …); wildcards supported.
- `resource` — path, command text, agent ID, or other value matched against the action.
- `effect` — `allow`, `ask`, or `deny`.

**Edit tools and shell are independent axes.** `edit` gates the write/edit/patch tools; `shell` gates bash. Denying `edit` does **not** stop an agent that retains shell from changing files (output redirection, `sed -i`, package installs, …). Agents that deny both still keep their other tools (read, glob, grep, webfetch, …) — this is least-privilege scoping, not a read-only sandbox.

The capability split in this setup:

| Agent | Configured restrictions | Notes |
| --- | --- | --- |
| `build`, `orchestrator`, `general` | None | Unrestricted shell; full tool access |
| `ui-builder` | None in config | Its "UI files only" scope comes from the **prompt**, not enforced permissions |
| `tester` | `edit` denied | Shell is **unrestricted** — it can still modify files via shell commands |
| `architect`, `reviewer`, `reviewer-critical`, `ui-designer`, `visual-qa` | `edit` + `shell` denied | Read/search/web tools remain available |
| `plan`, `explore` | Built-in restrictions | `plan` cannot edit normal project files; `explore` cannot edit at all |

Per the V2 docs, `hidden` only affects visibility — permissions are the enforcement mechanism. Subagents run in child sessions with fresh context, and each child uses **its own** configured permissions (the parent's `subagent` rules only control which agents the parent may launch).

## Model binding

Each agent except `orchestrator` pins a concrete `provider/model` (optionally with a variant) in JSON — see [models.md](models.md). The orchestrator has **no model override**: the user selects the primary session model manually, and the prompt forbids the orchestrator from claiming it switched models on its own. Subagents without a model would inherit the invoking session's model.

## Optional MCP layer

MCP is deliberately **not** part of this repository. The prompts reference two optional capabilities by name only:

- **Code graph** (codebase-memory): when present, agents prefer graph tools over grep/glob; when absent, they fall back and say so.
- **Browser tooling** (e.g. chrome-devtools): when connected *and* a running app exists, `visual-qa` can capture real screenshots; when disabled or unreachable, it must report **VISUAL REVIEW NOT VERIFIED** instead of guessing.

The reference setup shipped the browser server **disabled**, so "honest unavailable evidence" is the normal state, not an edge case. Machine-specific server paths and credentials were omitted from this repo on purpose.

## Defaults

`default_agent` stays **`build`**: new sessions begin in the built-in coding agent, and the orchestrator is selected per session with the `/agents` selector or by cycling agents (Shift+Tab, keybind `agent_cycle_reverse`; see the OpenCode keybinds docs for current defaults). Changing the default is a one-line, user-owned decision that affects only future sessions without an explicit agent choice.

## Non-goals

- Not a sandbox: permission config + prompts bind the agents; the write-capable agents run with your user account's real capabilities.
- Not model-agnostic in *quality*: the procedure is model-agnostic, but output quality depends on the models you bind.
- Not version-frozen: the schema was verified against the V2 docs on 2026-10-28 (runtime v2.2.0); re-verify on upgrades.
