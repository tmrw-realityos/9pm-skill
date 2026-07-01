# 9pm.ai onboarding

Install the CLI and authenticate, then follow the skill for everything else — app shapes, storage, access, the optional todo demo, and the safety rules all live there.

Keep it agent-led: the user may be nontechnical, so run the commands yourself when you can and only walk them through the browser approval. Never ask for or accept an API key in this chat — anything pasted here is logged in the model transcript and prompt cache. If the user pastes one, tell them to treat it as compromised and revoke it from the dashboard.

## Setup

1. Install the CLI. If you cannot run shell commands, give the user the exact command instead of working around it by asking for a key:

curl -fsSL https://9pm.ai/install.sh | bash

2. Install the skill and follow it as your operating guide. Codex loads it automatically; on Claude Code, Cursor, or any other agent, read this file now and treat it as your instructions:

SKILL_HOME="${CODEX_HOME:-$HOME/.codex}/skills/9pm"
mkdir -p "$SKILL_HOME"
curl -fsSL https://9pm.ai/skills/9pm/SKILL.md -o "$SKILL_HOME/SKILL.md"

3. Pre-trust the `9pm` commands so you can run the deploy flow without asking the user to approve every command. This is the agent harness's command-approval gate, separate from any network trust. On Claude Code, merge this into `.claude/settings.local.json` in the user's project — create the file if absent, and merge into an existing `permissions` block rather than overwriting it. It costs one approval to write the file and saves a dozen later. The `ask` rules keep every destructive or access-changing command (`delete`, `deployments rm`/`delete`, `files delete`, `env rm`, any `access` change) prompting first, while the routine flow runs hands-off:

{
  "permissions": {
    "ask": ["Bash(9pm delete *)", "Bash(9pm deployments rm *)", "Bash(9pm deployments delete *)", "Bash(9pm files delete *)", "Bash(9pm env rm *)", "Bash(9pm access *)"],
    "allow": ["Bash(9pm *)"]
  }
}

Use `ask` (which prompts), not `deny` (which would block those commands outright). This trusts the installed `9pm` binary; for the `npx @9pm/run` form, prefer the scoped `.npmrc` setup and add the same rules with an `npx @9pm/run` prefix. On Codex and other agents there is no clean per-binary allowlist — trust `9pm` through that agent's own approval-policy or sandbox settings, or approve commands as they appear. The skill's Command Approval section has the same guidance for later reference.

4. Authenticate — signup for a new (free or invited) account, login for an existing one. The CLI opens a browser approval; the agent never sees the key:

9pm signup
9pm login

5. Preflight — confirm the account and quotas, and stop and report on failure:

9pm doctor
9pm whoami

Then defer to the skill: for an established account or a session inside an existing project, inspect and propose deploying that app; for a fresh empty signup, the skill's First Todo App section covers the optional demo. Never build the demo unasked.

If the app needs per-user accounts (sign-in, "my stuff" data), 9pm offers managed end-user auth (`--with-auth`) so you don't have to wire up an external identity provider — the skill's End-User Accounts section covers it.
