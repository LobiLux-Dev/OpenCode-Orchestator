# Adding an agent

The roster is intentionally closed: the orchestrator delegates only to allowlisted agents, and every new agent changes routing, permissions, and often models. Add one only when an existing role genuinely doesn't fit.

This guide uses a hypothetical `docs-reviewer` (configured without edit or shell tools) as its running example. **It is an example, not part of this repository.**

## 1. Write the prompt file

Create `agents/<name>.md`. The file name is the agent ID. Keep the frontmatter minimal — `description` and `mode` — and put the system prompt in the body:

```md
---
description: Reviews documentation for accuracy, broken references, and unclear structure; does not edit files
mode: subagent
---

# Docs Reviewer

You are an independent documentation reviewer. Verify claims against the actual
code and configuration. Report findings with file and line references using the
BLOCKING / IMPORTANT / OPTIONAL rubric. Do not edit files or run shell commands.
Never assert behavior you have not verified.
```

Follow the house style of the existing agents:

- `description` matters: OpenCode shows it to the model that decides whether to launch the agent.
- `mode: subagent` for specialists; `mode: primary` only for session-facing agents (a new custom agent defaults to `primary` if omitted — set it explicitly).
- Be **model-agnostic**: never request a specific model inside the prompt.
- Do **not** put models or permissions in Markdown frontmatter for this setup — the JSON binding layer owns them (see [architecture.md](architecture.md)).
- Read-only agents must state their boundaries plainly ("do not edit", "do not delegate", "do not fabricate evidence").

## 2. Bind model and permissions in JSON

Add the agent to the `agents` map in your OpenCode config (merged from `opencode.example.json`):

```jsonc
"docs-reviewer": {
  "model": {
    "providerID": "opencode-go",
    "model": "mimo-v2.6-flash"
  },
  "permissions": [
    { "action": "edit",  "resource": "*", "effect": "deny" },
    { "action": "shell", "resource": "*", "effect": "deny" }
  ]
}
```

Default to **least privilege**: deny `edit` and deny `shell` unless the agent genuinely needs to run commands. Remember that `edit` denial only disables the write/edit/patch tools — an agent that keeps `shell` can still change files through it, so granting shell is a real capability grant. Only grant write capability when the agent's entire purpose is implementation, and then scope the prompt to specific areas (prompt scope is policy, not enforcement). Pick a real model your providers actually offer (`opencode models`) — see [models.md](models.md).

## 3. Allowlist it on the orchestrator

The orchestrator denies `subagent *` and then allowlists each agent by ID. Add yours **after** the deny rule, alongside the others:

```jsonc
{ "action": "subagent", "resource": "docs-reviewer", "effect": "allow" }
```

(Permission rules are ordered and the last match wins; placing the allow before the deny would silently revoke it.)

## 4. Wire the routing policy into the orchestrator prompt

The config only makes delegation *possible*; the orchestrator prompt decides *when*. Add a bullet to its routing policy in `agents/orchestrator.md` describing:

- When to delegate (bounded trigger — e.g., "documentation changes touching more than one file").
- When **not** to (the small-work-direct rule still applies).
- How the result integrates into the verification workflow and final report.

Keep the "never launch everything by default" philosophy intact: one precise trigger beats a vague "use it when useful".

## 5. Verify

```sh
opencode agent list            # the new agent appears
opencode                      # switch to it with /agents, or ask the orchestrator to use it
```

Check that the orchestrator can launch it, that the configured restrictions hold (attempted file edits must be denied; shell commands too, where denied), and that the report format is what the workflow expects.

## Checklist

- [ ] Prompt file with `description` + explicit `mode`, model-agnostic, boundaries stated
- [ ] JSON binding: real model, least-privilege permissions
- [ ] Orchestrator `subagent` allowlist entry, ordered after the `deny *` rule
- [ ] Orchestrator routing policy updated with a bounded trigger
- [ ] `docs/models.md` roster/mapping updated if a model is introduced
- [ ] README roster and workflow diagram updated (this is a deliberate roster change)
- [ ] No secrets, API keys, or machine-specific absolute paths anywhere

## When *not* to add an agent

- The task is occasional and unstructured — the orchestrator plus `general` may be enough.
- It duplicates an existing specialist with a different name (splitting `reviewer` duties by file type rarely pays).
- It would need write access "sometimes" — prefer a reporter without edit or shell tools whose findings the orchestrator routes to an implementing agent.
- You only want a different model for an existing role — change that agent's `model` binding instead of cloning the agent.
