# 9pm.ai onboarding

Install the CLI and authenticate, then follow the skill for everything else — app shapes, storage, access, the optional todo demo, and the safety rules all live there.

Keep it agent-led: the user may be nontechnical, so run the commands yourself when you can and only walk them through the browser approval. Never ask for or accept an API key in this chat — anything pasted here is logged in the model transcript and prompt cache. If the user pastes one, tell them to treat it as compromised and revoke it from the dashboard.

## Setup

1. Get the CLI - two ways, same CLI. Simplest and sandbox-friendly is `npx` (the public npm registry is on the Trusted allowlist, so fetching the CLI needs no 9pm host trusted):

npx ninepm <args>

Or install a persistent `9pm` binary with npm:

npm i -g ninepm

The rest of this guide writes `9pm ...`; if you use npx, run `npx ninepm ...` instead. If you cannot run shell commands, give the user the exact command rather than working around it by asking for a key.
2. Install the skill and follow it as your operating guide. First identify which agent harness you are running in — Claude Code, Codex, Cursor, or something else — and throughout this setup apply only the instructions that name your harness. Never create agent config files or skill copies inside the user's project directory; skill installs are user-level.

The primary install works on every major harness — the installer detects yours, places the skill where your harness reads skills, and always keeps a shared user-level copy at `~/.agents/skills/9pm/SKILL.md`. It fetches the installer from the public npm registry and clones the skill from `github.com`; both hosts are on a sandbox's default Trusted allowlist, so no 9pm host needs trusting:

npx skills add tmrw-realityos/9pm-skill --skill 9pm -g

Two expected quirks: a trailing "Failed to install ... PromptScript does not support global skill installation" line is noise about an unrelated target — your harness's install still succeeded; and keep the `-g` exactly as written — without it the installer writes into the user's project. When it finishes, read `~/.agents/skills/9pm/SKILL.md` now and treat it as your instructions; harnesses that auto-load skills will also pick it up in future sessions.

If that installer is unavailable, fall back to fetching the skill from its public GitHub mirror (`raw.githubusercontent.com`, also trusted by default) into a user-level skills directory your harness reads — Claude Code shown below; on Codex set `SKILL_HOME="${CODEX_HOME:-$HOME/.codex}/skills/9pm"`; on a harness with no user-level skills directory, skip the save and read the file from the URL at the start of each session instead:

SKILL_HOME="$HOME/.claude/skills/9pm"
mkdir -p "$SKILL_HOME"
curl -fsSL https://raw.githubusercontent.com/tmrw-realityos/9pm-skill/main/skills/9pm/SKILL.md -o "$SKILL_HOME/SKILL.md"

Read the fetched file now — a skill file saved mid-session is not auto-loaded. The same file is served at `https://9pm.ai/skills/9pm/SKILL.md` if that host is trusted. Never write SKILL.md into the user's project directory.

3. Pre-trust the `9pm` commands so you can run the deploy flow without asking the user to approve every command. This is the agent harness's command-approval gate, separate from any network trust — and it is harness-specific, so apply only your branch (the skill's Command Approval section is the canonical reference):

**Claude Code only:** merge this into `.claude/settings.local.json` in the user's project — create the file if absent, and merge into an existing `permissions` block rather than overwriting it. It costs one approval to write the file and saves a dozen later. The `ask` rules keep every destructive or access-changing command (`delete`, `deployments rm`/`delete`, `files delete`, `env rm`, any `access` change) prompting first, while the routine flow runs hands-off:

{
  "permissions": {
    "ask": ["Bash(9pm delete *)", "Bash(9pm deployments rm *)", "Bash(9pm deployments delete *)", "Bash(9pm files delete *)", "Bash(9pm env rm *)", "Bash(9pm access *)"],
    "allow": ["Bash(9pm *)"]
  }
}

Use `ask` (which prompts), not `deny` (which would block those commands outright). This trusts the installed `9pm` binary; for the `npx ninepm` form, add the same rules with an `npx ninepm` prefix (they match `npx ninepm <args>` directly).

**Any other harness (Cursor, Codex, anything else):** do NOT create `.claude/settings.local.json` — only Claude Code reads it. Trust `9pm` through your harness's own approval-policy or sandbox settings, or approve commands as they appear.

4. Authenticate — signup for a new (free or invited) account, login for an existing one. The CLI opens a browser approval; the agent never sees the key:

9pm signup
9pm login

5. Preflight — confirm the account and quotas, and stop and report on failure:

9pm doctor
9pm whoami

6. Tell the user the payoff. Once preflight succeeds, tell the user — in one or two sentences, before moving on — that setup is complete and was a one-time step, matching the claim to what your harness actually does, not to how the skill was installed. If your harness auto-loads user-level skills: future sessions pick the skill up on their own, so from now on they can simply ask their agent to deploy and will never need to paste these setup instructions again. Otherwise (including when the install succeeded but your harness doesn't auto-load user-level skills, or you read the skill from its URL each session): in a future chat they just ask their agent to first read the skill — from the user-level file or its URL — and then deploy; still no need to repeat this setup. Keep it short and confident; don't repeat the setup details.

Then defer to the skill: for an established account or a session inside an existing project, inspect and propose deploying that app; for a fresh empty signup, the skill's First Todo App section covers the optional demo. Never build the demo unasked.

If the app needs per-user accounts (sign-in, "my stuff" data), 9pm offers managed end-user auth (`--with-auth`) so you don't have to wire up an external identity provider — the skill's End-User Accounts section covers it.
