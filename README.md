# Quicknode Earn skill

An [Agent Skill](https://agentskills.io) that teaches coding agents the [Quicknode Earn](https://earn.quicknode.com) public API: rank Morpho USDC vaults, create and fund strategies, read performance and rebalance history, and build SIWE-signed writes and deposit/withdraw calldata. The API never holds keys or broadcasts transactions; the agent prepares them and your wallet signs.

One skill, packaged for every major harness from this repo.

## Install

### Claude Code

```bash
claude plugin install earn --marketplace quicknode/earn-skill
```

Inside a session: `/plugin install earn --marketplace quicknode/earn-skill`. On Claude Code older than v2.1.292 (shell) or v2.1.275 (session), use two steps:

```bash
claude plugin marketplace add quicknode/earn-skill
claude plugin install earn@quicknode
```

Update with `claude plugin update earn@quicknode`. The skill loads automatically when you work with Earn, or call it with `/earn:earn-api`.

### Codex

As a plugin:

```bash
codex plugin marketplace add quicknode/earn-skill
```

Then open the Plugins Directory in the ChatGPT desktop app, choose the **Quicknode** marketplace, and install **Quicknode Earn**.

As a plain skill (CLI only):

```bash
npx skills add quicknode/earn-skill -a codex -g
```

### Cursor

In Cursor, open **Customize**, choose **From GitHub Repository**, and enter `quicknode/earn-skill`. Teams can add it once for everyone under **Dashboard → Plugins & MCPs → Team Marketplaces → Add Marketplace → Import from Repo**.

As a plain skill:

```bash
npx skills add quicknode/earn-skill -a cursor -g
```

### Gemini CLI

```bash
gemini extensions install https://github.com/quicknode/earn-skill
```

Or just the skill: `gemini skills install https://github.com/quicknode/earn-skill.git --path skills/earn-api`

### GitHub Copilot (CLI, VS Code, cloud agent)

```bash
gh skill install quicknode/earn-skill earn-api --scope user
```

Drop `--scope user` to install into the current repo's `.agents/skills/` so the whole team gets it. Needs a GitHub CLI release with `gh skill` (April 2026 or later).

### Everything else

Windsurf, Cline, OpenCode, Amp, Goose, Kiro, Roo Code, Continue and 40+ other agents:

```bash
npx skills add quicknode/earn-skill            # detects your installed agents
npx skills add quicknode/earn-skill -a windsurf -g
```

`gh skill install quicknode/earn-skill earn-api --agent <agent>` works too; see `gh skill install --help` for agent ids.

### Manual

Copy `skills/earn-api/` into your agent's skills directory. For most agents that is `.agents/skills/` in a repo or `~/.agents/skills/` for your user; Claude Code uses `~/.claude/skills/`.

```bash
mkdir -p ~/.agents/skills/earn-api && curl -fsSL https://raw.githubusercontent.com/quicknode/earn-skill/main/skills/earn-api/SKILL.md -o ~/.agents/skills/earn-api/SKILL.md
```

## Repo layout

| Path | Read by |
| --- | --- |
| `skills/earn-api/SKILL.md` | The skill itself. Every harness below loads this one file. |
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` | Claude Code plugin and marketplace |
| `.codex-plugin/plugin.json`, `.agents/plugins/marketplace.json` | Codex plugin and marketplace |
| `.cursor-plugin/plugin.json`, `.cursor-plugin/marketplace.json` | Cursor plugin and marketplace |
| `gemini-extension.json` | Gemini CLI extension |

`npx skills` and `gh skill` need no manifest; they find `skills/*/SKILL.md` directly.

## Releasing

1. Edit `skills/earn-api/SKILL.md`. Keep `name: earn-api` matching the folder name (Cursor and the Agent Skills spec require it) and the `description` under 1024 characters.
2. Bump `version` in `.codex-plugin/plugin.json`, `.cursor-plugin/plugin.json` and `gemini-extension.json`. The Claude Code manifest has no version on purpose, so every commit to `main` is a release there.
3. Run `claude plugin validate .` and push to `main`.
4. Keep `https://earn.quicknode.com/skill.md` serving the same file as `skills/earn-api/SKILL.md`.
