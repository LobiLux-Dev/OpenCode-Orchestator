# opencode-orchestrator

A delegation-based engineering workflow for **OpenCode V2**: one orchestrator agent owns scope, routing, verification, and delivery, while specialists with varying tool access handle exploration, implementation, review, testing, architecture, and UI/UX.

This repository contains:

```text
agents/                  8 agent prompts (Markdown, copied verbatim from a working setup)
docs/architecture.md     How the two-layer design fits together
docs/models.md           Exact provider/model/variant mapping and conceptual tiers
docs/adding-an-agent.md  How to add a new agent safely
opencode.example.json    Sanitized reference configuration (models + permissions)
CONTRIBUTING.md
LICENSE                  MIT, © 2026 José Luis Sánchez Vázquez
```

The configuration schema targets **OpenCode V2** (`agents` map, list-form `permissions`, `default_agent`). It was verified against the official V2 documentation (opencode.ai/v2/docs) on **2026-10-28**, and the reference setup ran `opencode v2.2.0`. V1's singular `agent`/`permission` keys are deprecated and not used here.

## Agent roster

The roster is exactly the reference setup: 4 built-in OpenCode agents (configured) plus 8 custom agents (prompts in `agents/`, models/permissions in config). **No other agents exist in this repository.**

| Agent | Kind | Responsibility | Model in reference config | Configured restrictions |
| --- | --- | --- | --- | --- |
| `build` | built-in, primary | Default coding agent (stays the default) | `opencode-go/deepseek-v4.1-flash#low` | None (built-in defaults) |
| `plan` | built-in, primary | Planning and analysis | `opencode-go/deepseek-v4.1-flash#high` | Built-in restrictions (no normal file edits) |
| `orchestrator` | custom, primary | Scope, delegation, integration, verification, delivery | No override — runs on the session model the user picks | None; delegation limited by its subagent allowlist |
| `general` | built-in, subagent | Scoped non-UI implementation | `opencode-go/mimo-v2.6-flash` | None (may edit) |
| `explore` | built-in, subagent | Code search and call tracing | `opencode-go/mimo-v2.6-flash` | Built-in: no edits |
| `architect` | custom, subagent | Hard system-level decisions and diagnoses | `openai/gpt-6.1-sol#high` | Edit + shell denied |
| `tester` | custom, subagent | Reproduce issues, run checks, report results | `opencode-go/mimo-v2.6-flash` | Edit tools denied; **shell unrestricted** |
| `reviewer` | custom, subagent | Independent routine code review | `opencode-go/deepseek-v4.1-flash#high` | Edit + shell denied |
| `reviewer-critical` | custom, subagent | High-stakes audit (security, integrity, migrations) | `openai/gpt-6.1-terra#high` | Edit + shell denied |
| `ui-designer` | custom, subagent | Design brief and visual direction | `openai/gpt-6.1-terra#medium` | Edit + shell denied |
| `ui-builder` | custom, subagent | Implement approved UI in the existing stack | `opencode-go/mimo-v2.6-flash` | None in config — its "UI files only" scope is prompt-level |
| `visual-qa` | custom, subagent | Critique of the **rendered** interface | `openai/gpt-6-luna#high` | Edit + shell denied |

"Denied" here means the matching **tools** are denied — `edit` covers the write/edit/patch tools and `shell` covers bash. Agents that deny both still keep read/search/web tools; this is least-privilege scoping, not a read-only sandbox. Model availability depends on your providers; see [docs/models.md](docs/models.md).

## Workflow

Dotted arrows are **conditional**: the orchestrator delegates only where value is clear and handles small, localized work itself.

```mermaid
flowchart TD
    U([User request]) --> O[orchestrator<br/>scope, route, integrate, deliver]

    O -->|small and well-localized| D[Handled directly<br/>no subagents launched]

    O -.->|unknown code location| E[explore<br/>no edits (built-in)]
    O -.->|scoped non-UI work| G[general<br/>implementation]
    O -.->|architecture uncertainty| A[architect<br/>decision, no edit/shell]
    O -.->|reproduce / run checks| T[tester<br/>optional; edit tools denied]
    O -.->|material behavior change| R[reviewer<br/>optional, no edit/shell]
    O -.->|serious security or data risk| RC[reviewer-critical<br/>optional, no edit/shell]

    O -.->|substantial UI work| UD[ui-designer<br/>no edit/shell, only if direction unclear]
    UD --> UB[ui-builder<br/>implements the brief]
    O -.->|UI direction already clear| UB
    UB --> EV{Rendered evidence<br/>available?}
    EV -->|screenshots or running app| VQ[visual-qa<br/>no edit/shell, inspects rendered UI]
    EV -->|no evidence, e.g. browser MCP disabled| NV[Visual review reported<br/>as NOT VERIFIED]

    D --> I[Evaluate findings, fix, re-verify]
    E --> I
    G --> I
    A --> I
    T --> I
    R --> I
    RC --> I
    VQ --> I
    NV --> I
    I --> F([Final report: changes, checks,<br/>actual evidence, open risks])
```

Ground rules, taken from the agent prompts themselves:

- **Delegation is not reflexive.** Questions, simple edits, and one-off fixes are handled directly by the orchestrator.
- **`tester` and reviews are optional**, chosen by impact and remaining uncertainty. `reviewer` and `reviewer-critical` are never both launched automatically for routine edits.
- **The UI track is sequential, not automatically parallel.** `visual-qa` inspects the result of `ui-builder`, so it runs after implementation. Only independent *read-only* investigations may run concurrently; concurrent edits to overlapping files are forbidden.
- **Honest evidence is mandatory.** No agent may claim a test passed, a screenshot was captured, or a review happened without evidence. If rendered evidence is unavailable (the reference setup ships the browser MCP server *disabled*), the workflow ends visual assessment at **"VISUAL REVIEW NOT VERIFIED"** with a manual verification path — never with a fabricated approval.

## Setup (OpenCode V2)

This repository never modifies your live configuration; installation is copy-and-merge.

### 1. Install OpenCode V2

Verified install commands from the official V2 docs (2026-10-28):

```sh
# curl
curl -fsSL https://opencode.ai/v2/install | bash
# homebrew
brew install anomalyco/tap/opencode-v2
# npm (postinstall selects the native binary)
npm install -g @opencode/cli
```

`bun`, `pnpm`, `yarn`, `vite+`, AUR, and `mise` variants, plus standalone binaries, desktop apps, and Docker images, are listed in the same docs. Windows package managers are not supported.

### 2. Install the agent prompts

Copy the prompts into the global agent directory — **always back up first and never clobber existing files**:

```sh
mkdir -p ~/.config/opencode/agents            # global agents directory
cp -a ~/.config/opencode/agents ~/.config/opencode/agents.bak   # 1. back up (skip if none yet)
cp -n agents/*.md ~/.config/opencode/agents/                    # 2. no-clobber copy
diff -q agents/ ~/.config/opencode/agents/                      # 3. review the result
```

Or install them into a project's `.opencode/agents/` for per-project use (a nested path like `team/reviewer.md` becomes the agent ID `team/reviewer`). The file name is the agent ID; the frontmatter carries `description` and `mode`, and the Markdown body is the system prompt. Models and permissions are **not** in the Markdown files — they come from the JSON config below.

`cp -n` skips files that already exist instead of overwriting them — and it does so quietly, so always check `diff` afterward. Where a file already exists (for example, a customized `reviewer.md`), do **not** blindly replace it: open both versions, merge the changes you want into your copy, and keep the backup until you are satisfied.

### 3. Configure models and permissions

Copy `opencode.example.json` and merge its `agents` (and optionally `default_agent` and `mcp`) sections into your global `~/.config/opencode/opencode.json` or a project config. Back up the file you are merging into first:

```sh
cp ~/.config/opencode/opencode.json ~/.config/opencode/opencode.json.bak
```

Treat the example as a reference, not a drop-in replacement: merge entry-by-entry so you keep any providers, permissions, or settings you already have.

> **The model IDs are real IDs from the reference setup, not placeholders — but you must configure the matching providers.** The `opencode-go/*` models require an [OpenCode Go](https://opencode.ai/v2/docs/console/go/) subscription; the `openai/*` models require OpenAI credentials. Connect providers in the TUI with `/connect` or through OpenCode Console. Run `opencode models` to see which models *your* account can actually use, and substitute equivalents if needed. Do not invent "tier" model names: `#low`, `#medium`, and `#high` are per-model variants and only work where the model defines them.

The example contains **no MCP servers, no machine-specific paths, and no credentials** — see step 5 for the optional MCP layer.

### 4. Keep `build` as the default; select the orchestrator per session

The example (like the reference setup) sets `"default_agent": "build"`: new sessions still start in the built-in `build` agent, with the orchestrator available on demand. To use the orchestrator in a session, pick it from the agent selector with the **`/agents`** command, or cycle agents with **Shift+Tab** (keybind `agent_cycle_reverse`; see the OpenCode keybinds docs for the current defaults). If you want the orchestrator as the default instead, change `default_agent` to `"orchestrator"` — per the V2 docs this affects only sessions that have not yet selected an agent and does **not** change existing sessions.

### 5. Optional: MCP servers

Everything works without MCP. The reference setup used a local codebase-memory graph server and a browser-screenshot server; their machine-specific binary paths and credentials are deliberately excluded from this repository. If you want either capability, add your own server. Example (a browser tooling server, kept disabled until you authorize it):

```json
{
  "mcp": {
    "servers": {
      "chrome-devtools": {
        "type": "local",
        "command": ["npx", "-y", "chrome-devtools-mcp@latest"],
        "disabled": true
      }
    }
  }
}
```

Agents degrade honestly without it: the prompts tell them to fall back to grep/glob when the code graph is unavailable, and to mark visual review **not verified** when no rendered evidence exists. Enabling an MCP server gives agents its tools — do it deliberately.

### 6. Verify

```sh
opencode agent list
```

You should see `orchestrator`, `architect`, `reviewer`, `reviewer-critical`, `tester`, `ui-designer`, `ui-builder`, and `visual-qa` alongside the built-ins.

## Security

- **Edit tools ≠ shell.** In OpenCode V2, `edit` gates the write/edit/patch *tools* and `shell` gates bash — they are independent. Denying `edit` does **not** stop an agent that retains shell from changing files (redirection, `sed -i`, package installs, …). `tester` has exactly this profile: no edit tools, **unrestricted shell**.
- **Capability split (config-enforced):** `architect`, `reviewer`, `reviewer-critical`, `ui-designer`, and `visual-qa` deny both `edit` and `shell`. `build`, `orchestrator`, `general`, and `ui-builder` have **no tool restrictions** in this config. `ui-builder`'s "UI files only" scope is **prompt-level, not permission-enforced** — it can technically edit anywhere.
- **Least privilege, not a read-only sandbox.** Agents that deny `edit` and `shell` still keep their other tools (read, glob, grep, webfetch, websearch, …). Never describe a configuration in this repo as "read-only" in the security sense.
- **Delegation allowlist (nine):** the orchestrator's permissions deny `subagent *` and then allowlist exactly nine subagents — the built-ins `general` and `explore`, plus the seven custom subagents `tester`, `reviewer`, `architect`, `reviewer-critical`, `ui-designer`, `ui-builder`, and `visual-qa`. It cannot spawn unlisted agents.
- **Prompt policy is not enforcement.** Four subagent prompts (`architect`, `reviewer`, `reviewer-critical`, `ui-designer`) explicitly tell the agent not to delegate; the others simply don't mention it. The orchestrator prompt forbids claiming tests, screenshots, or reviews without evidence. Per the V2 docs, `hidden` only controls visibility — **permissions are the enforcement mechanism**.
- **Honest limits:** permission enforcement follows OpenCode's documented behavior; verify it for your OpenCode version. The agents run with your user account — "deny" rules bind the agents, not a human operator, and the write-capable agents retain full shell access. Treat this as an agent workflow, not a sandbox.
- **No secrets in this repo:** the example config has no keys, no MCP paths, and no personal data. Keep your real configuration (which may reference local tools) out of commits.

## Limitations

- **Visual QA depends on rendered evidence.** With the browser MCP disabled or no running app, `visual-qa` cannot judge appearance and must report **VISUAL REVIEW NOT VERIFIED**. That is the designed, honest outcome — not a bug, and not something to paper over.
- **Model-dependent behavior.** Routing discipline, review quality, and UI taste vary with the selected models; the orchestrator itself runs on whatever session model the user picks.
- **Variants are per-model.** `#low`/`#medium`/`#high` availability depends on the model and your account; the reference combos are exact, not generic.
- **Optional checks are optional.** The orchestrator may skip `tester` or reviews when it judges risk low — that is a policy choice, not a quality guarantee.
- **Cost and latency.** Parallel subagents multiply token spend; the prompts cap visual correction loops at two before escalating instead of looping forever.
- **Documentation drift.** Syntax here was verified against the V2 docs on 2026-10-28 (runtime v2.2.0). OpenCode's schema evolves — recheck when upgrading.

## Future agent ideas (not implemented)

These are **recommendations only — no such agent exists in this repository**, and none is wired into the orchestrator's allowlist:

- `security-auditor` — read-only dependency/secret/headers audit
- `perf-analyst` — profiling and benchmark interpretation
- `docs-writer` — documentation updates restricted to `docs/**`
- `db-migration-guard` — migration-safety review before schema changes
- `i18n-auditor` — locale coverage and string-consistency checks
- `release-manager` — changelog, tagging, and release-draft assistance

Adding any of them means adding a real agent file and updating the orchestrator's `subagent` allowlist — see [docs/adding-an-agent.md](docs/adding-an-agent.md).

## Further reading

- [docs/architecture.md](docs/architecture.md) — two-layer design, delegation graph, permission semantics
- [docs/models.md](docs/models.md) — the exact model mapping and conceptual tiers
- [docs/adding-an-agent.md](docs/adding-an-agent.md) — extend the roster safely
- [CONTRIBUTING.md](CONTRIBUTING.md)

## License

[MIT](LICENSE) © 2026 José Luis Sánchez Vázquez.
