# Models

Exact provider/model/variant mapping used by the reference configuration, captured **2026-10-28** from a live OpenCode v2.2.0 setup. Every ID below was confirmed present in `opencode models` on that date.

> These are real model IDs from a working configuration — **not placeholders and not a stable contract**. Model catalogs change, and availability depends on your account and providers. Run `opencode models` to see what you can use, and substitute equivalents where noted.

## Exact mapping

| Agent | `providerID` | `model` | `variant` | Notes |
| --- | --- | --- | --- | --- |
| `build` | `opencode-go` | `deepseek-v4.1-flash` | `low` | Built-in primary; default agent |
| `plan` | `opencode-go` | `deepseek-v4.1-flash` | `high` | Built-in primary |
| `orchestrator` | — | — | — | **No override configured.** Runs on the session model the user selects manually |
| `general` | `opencode-go` | `mimo-v2.6-flash` | — | Built-in subagent |
| `explore` | `opencode-go` | `mimo-v2.6-flash` | — | Built-in subagent |
| `architect` | `openai` | `gpt-6.1-sol` | `high` | Decision agent; edit + shell denied |
| `tester` | `opencode-go` | `mimo-v2.6-flash` | — | Runs checks via shell (shell unrestricted); edit tools denied |
| `reviewer` | `opencode-go` | `deepseek-v4.1-flash` | `high` | Routine review; edit + shell denied |
| `reviewer-critical` | `openai` | `gpt-6.1-terra` | `high` | High-stakes audit; edit + shell denied |
| `ui-designer` | `openai` | `gpt-6.1-terra` | `medium` | Design brief; edit + shell denied |
| `ui-builder` | `opencode-go` | `mimo-v2.6-flash` | — | UI implementation; no tool restrictions (UI scope is prompt-level) |
| `visual-qa` | `openai` | `gpt-6-luna` | `high` | Critique of rendered evidence; edit + shell denied |

In Markdown frontmatter or `provider/model` string form these are written, for example, as `openai/gpt-6.1-terra#high` — the `#variant` suffix is optional. JSON configuration uses the expanded object form shown in `opencode.example.json`:

```json
{ "providerID": "openai", "model": "gpt-6.1-terra", "variant": "high" }
```

## Conceptual tiers

The tiers below describe **how this configuration groups models by role**. They are a mental model for maintainers, **not** OpenCode concepts, and they are not model identifiers — never write something like `tier: high` or a made-up model ID into configuration.

| Tier (conceptual) | Members in this setup | Used for | Rationale in this workflow |
| --- | --- | --- | --- |
| Fast / high-volume | `opencode-go/mimo-v2.6-flash`, `opencode-go/deepseek-v4.1-flash#low` | `general`, `explore`, `tester`, `ui-builder`, `build` | Breadth work: search, bounded implementation, running checks |
| Deliberate reasoning | `opencode-go/deepseek-v4.1-flash#high`, `openai/gpt-6.1-sol#high` | `plan`, `reviewer`, `architect` | Planning, independent review, and system-level decisions benefit from deeper reasoning |
| Design & critical audit | `openai/gpt-6.1-terra` (`#high` / `#medium`) | `reviewer-critical`, `ui-designer` | High-consequence auditing and design direction |
| Visual critique | `openai/gpt-6-luna#high` | `visual-qa` | Judging rendered screenshots and interface evidence |

## Variants

A variant selects a named preset for a specific model (this setup uses `low`, `medium`, and `high`). **Variants are per-model**: a variant suffix is only valid where the model/provider defines it, so `#high` on one model says nothing about another. If you swap models, check which variants your replacement supports (or drop the variant and let defaults apply).

## Provider requirements

- `opencode-go/*` — requires an [OpenCode Go](https://opencode.ai/v2/docs/console/go/) subscription.
- `openai/*` — requires OpenAI credentials, typically configured with `/connect` in the TUI or via OpenCode Console.

Neither is bundled with OpenCode itself. Until a provider is configured, sessions using its models will fail to run.

## Changing models

1. Edit the `model` entries in your merged config (see `opencode.example.json`).
2. A subagent with no configured model **inherits the invoking session's model** (V2 docs).
3. The orchestrator deliberately has no override in this setup: the user chooses the primary session model manually, and the orchestrator prompt forbids it from claiming it switched its own model.
4. Re-run `opencode agent list` and start a new session to pick up changes; existing sessions keep their stored model/agent selections.
