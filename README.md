# 9pm.ai agent skill

Deploy and manage apps on **[9pm.ai](https://9pm.ai)** straight from your coding agent.

This repository is a generated, read-only mirror of the 9pm skill. It drops product guidance
for 9pm.ai into your agent's context so it knows how to deploy an app, configure access, attach
managed storage, set environment variables, and more - using the public `9pm` CLI.

> **Requesting access:** 9pm.ai is currently invite-only. [Request access](https://9pm.ai/request-invite) to get
> on the list - once you're in, the skill below lets your agent drive deploys end to end.

## Install

Pick whichever your agent supports:

```sh
# skills registry
npx skills add tmrw-realityos/9pm-skill

# Codex / Cursor: install from this repo's plugin manifests
#   .codex-plugin/plugin.json
#   .cursor-plugin/plugin.json
```

The skill itself lives at [`skills/9pm/SKILL.md`](skills/9pm/SKILL.md) (and a top-level
[`9pm/SKILL.md`](9pm/SKILL.md) copy for the skills registry). [`AGENTS.md`](AGENTS.md) is the
one-time setup prompt that installs the CLI and authorizes your agent.

## What your agent can do once installed

- Deploy a static site or a server-runtime app to `*.9pm.ai`
- Inspect an app for compatibility before deploying
- Configure public or private (login-gated) access
- Use managed per-app storage and environment variables/secrets
- Regenerate deploy commands and tear apps down

## How this repo is maintained

The content here is **generated** from 9pm's internal source of truth and refreshed
automatically - do not edit it by hand. Open an issue in this repository instead.

## License

MIT - see [LICENSE](LICENSE).
