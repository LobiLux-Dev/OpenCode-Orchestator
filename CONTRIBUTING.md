# Contributing

Thanks for helping improve this orchestrator setup. It is a small, opinionated configuration — changes that keep it focused and honest are the easiest to accept.

## Ground rules

- **Scope:** this repo ships exactly the agents documented in the README roster. New agents require an explicit, justified roster change (see [docs/adding-an-agent.md](docs/adding-an-agent.md)); speculative agents are declined.
- **No secrets, ever:** no API keys, tokens, credentials, machine-specific absolute paths (e.g. home-directory binaries), or personal project references — in prompts, configs, or docs.
- **English only** for prompts and documentation.
- **Honesty is a feature.** Every agent forbids fabricated evidence ("test passed", "screenshot captured", "review done"). Do not soften that in any edit.

## Workflow

1. Fork and branch from `main`.
2. Make your change; keep agent prompts coherent with the rest of the roster (shared severity rubric: `BLOCKING` / `IMPORTANT` / `OPTIONAL`; visual findings add `POLISH`).
3. If you change configuration, validate it:
   ```sh
   python3 -m json.tool opencode.example.json > /dev/null   # must parse
   ```
4. If you touch agents, test against a real OpenCode V2 install (`opencode agent list`, launch the agent, confirm permissions behave as described).
5. If models changed, update [docs/models.md](docs/models.md) with the exact `providerID` / `model` / `variant` values you actually verified, and the verification date.
6. If workflow, roster, or permissions changed, update the README (roster table, Mermaid diagram, security section) and [docs/architecture.md](docs/architecture.md) in the same commit.
7. Open a pull request describing *what* changed and *how you verified it*. Unverified claims will be treated as blockers.

## Editing agent prompts

- Preserve each agent's contract: role, boundaries, evidence requirements, report format.
- Subagent prompts must not instruct delegation; only the orchestrator routes.
- Restricted agents must keep explicit boundaries in their prompts ("do not edit", "do not run shell commands", "do not delegate" where applicable) — config permissions and prompt rules are separate layers, and config-denied tools are the only enforced boundary.
- Keep prompts model-agnostic — no model names or provider-specific instructions in Markdown.

## Changing permissions or models

- Permissions live only in JSON (config), never in the Markdown files. Prefer the least privilege that still lets the agent do its job.
- The orchestrator's `subagent` allowlist must remain deny-first, then explicit allows, matching the roster exactly.
- `default_agent` stays `build` unless the change explicitly concerns the default-selection story.

## Documentation drift

OpenCode's V2 schema evolves. This repo was verified against the official V2 docs on 2026-10-28 with runtime v2.2.0. If you verify a newer version, say so in your PR (with the doc pages you checked), and adjust any syntax that changed.

## License

By contributing you agree your contribution is released under the [MIT License](LICENSE) (© 2026 José Luis Sánchez Vázquez).
