---
name: 9pm
description: Deploy and manage apps on 9pm.ai with the public 9pm CLI. Use when a user asks to deploy an app, inspect app compatibility for 9pm.ai, create a hosted app, configure app access, use managed storage, set environment variables or secrets, regenerate deployment commands, or delete a 9pm.ai app.
---

# 9pm.ai

Product guidance for working with 9pm.ai after the CLI is installed and the user has authorized access. Not the one-time setup prompt.

## Operating Rules

- Inspect before changing.
- Ask before creating files for a new app or modifying an existing app.
- Deploy what the user built. Prefer the smallest compatibility change that preserves their stack and intent.
- Deploy the working copy the user is editing — 9pm uploads the directory you point at, not a git branch. When the user is on a branch or in a git worktree, confirm the deploy path resolves to that checkout before running (see Previews and Branches).
- Never refactor an app — especially its storage layer — to fit 9pm.ai without explicit user approval. If an app already has a working database, attach storage to it (see Persistence) rather than rewriting it.
- Run the app's existing build/test checks before deploying when available.
- Run `9pm deploy <dir> --check` before the real deploy when supported.
- Report the deployed URL and any follow-up risks clearly.

## Authentication

This chat agent has no secure secret input. Anything pasted into the conversation is logged in the model provider transcript, tool-call history, and prompt cache. Do not request the user's API key. Do not accept one if offered. If the user pastes a key anyway, stop, tell them to treat it as compromised, and instruct them to revoke and reissue from the dashboard.

Authentication paths, in order of preference:

1. `9pm signup` for a new account, when public signup or an invite is available. The browser page says invite-only if signup is closed.
2. `9pm login` for an existing account. Device-code flow; the token is stored in native protected storage. Plaintext file storage is opt-in only when native storage is unavailable. The agent never sees the key.
3. User-set environment in their own terminal: `export NINEPM_API_KEY=9pm_...`. Warn that the value can land in shell history.
4. `NINEPM_API_KEY` overrides stored login for CI.

The endpoint defaults to `https://api.9pm.ai` unless the user specifies another. When a command needs it set explicitly:

```sh
export NINEPM_API_URL=https://api.9pm.ai
```

Preflight:

```sh
9pm doctor
9pm whoami
```

Fallback only when `whoami` is unavailable and the user has set `NINEPM_API_KEY` themselves:

```sh
curl -fsS "$NINEPM_API_URL/v1/sites" -H "authorization: Bearer $NINEPM_API_KEY" >/dev/null
```

## Inventory

Before deploying or modifying anything, list what the user already has so you don't collide with an existing app slug or operate on the wrong project:

```sh
9pm apps
9pm apps --json
```

The slug column is the handle for every subsequent command (`deploy`, `access`, `delete`, `env`, `files --project`). `active` means the app has a finalized deployment; `no-deploy` means it was created but never finalized.

## Traffic analytics

Every deployed app has a built-in **Analytics** view in the dashboard (per app, selectable 24h / 7d / 30d window): views, unique visitors, top referrers, countries, human-vs-crawler split, and the paths that returned 404. It's captured automatically — nothing to add to the app and no third-party script. Visitor counts are privacy-preserving (a daily rotating hash, no raw IP, no cookies). Point the user to their app's page in the dashboard to see it.

## Versioning

`9pm doctor` reports a `CLI:` line comparing installed vs latest. If the user hits an "unknown command" error or behavior that doesn't match this guidance, suspect an out-of-date CLI and run:

```sh
9pm latest
```

Upgrade with:

```sh
curl -fsSL https://9pm.ai/install.sh | bash
```

The CLI is also published as the npm package `ninepm` on 9pm's own npm registry, so `npx` works once you point npm at it (the public-npmjs rollout is in progress; until it completes, pass the registry explicitly):

```sh
npx --registry https://9pm.ai/npm ninepm <args>
```

Once the public-npm rollout completes, plain `npx ninepm <args>` works with no registry flag. `install.sh` remains the primary path; the npm channel is additive. Reads are anonymous; no token needed to install.

## Sandboxed Environments

If you run inside a sandbox with restricted network egress (the Claude cloud sandbox, or the Claude Code CLI bash sandbox - both default to a "Trusted" allowlist), install and deploy fail until the relevant hosts are trusted. The first symptom is a refused `curl https://9pm.ai/install.sh`.

Hosts to trust:

- `9pm.ai` - hosted `install.sh` and the CLI tarball/manifest under `/downloads/9pm/*`. (Required today only for the hosted installer; once `ninepm` ships on npm - see #69 - only `api.9pm.ai` is needed.)
- `api.9pm.ai` - the deploy API the CLI calls (`9pm login`, `deploy`, `apps`, ...).
- Container deploys (`--runtime container`) build the image **locally**, so they pull base layers from Docker Hub (`registry-1.docker.io` / `auth.docker.io` are on the Trusted default; the layer-blob CDN may need adding). The built image is then **uploaded to `api.9pm.ai`** (the `/v1/container-uploads/<id>/image` PUT - same host as above). The push to the managed image registry happens **server-side**, so no registry host needs allowlisting in the sandbox.

How you trust them depends on the environment:

- **Claude cloud (claude.ai/code)**: set this in the environment's network settings (the cloud icon), not in any settings file. Choose **Custom** network access and add the hosts in the **Allowed domains** field. The policy is applied when the environment is (re)built, so set it **before** the session - an agent already inside the sandbox cannot change it mid-session, and `~/.claude/settings.json` is not loaded in cloud sessions. See https://code.claude.com/docs/en/claude-code-on-the-web.
- **Claude Code CLI (local bash sandbox)**: add the hosts to `sandbox.network.allowedDomains` in `.claude/settings.json` (wildcards like `*.9pm.ai` work). See https://code.claude.com/docs/en/sandboxing.

This guidance and the installer are themselves served from `9pm.ai`, so in a sandbox that has not trusted it yet you cannot fetch them from inside - apply the allowlist from outside the sandbox (the cloud environment settings, or your CLI settings) before the first run.

## Command Approval

Separate from the network egress above: a coding agent harness may ask the user to approve **every** `9pm` command before it runs, so a single deploy turns into a dozen approval prompts. This is the agent's own command-approval gate, not a 9pm setting - nothing about `9pm` is inherently untrusted, the agent just has no standing rule that it is safe to run. Pre-trust the `9pm` command surface in the agent's permission config so the routine deploy flow runs hands-off, while the one destructive command still asks first.

On Claude Code, add this to `.claude/settings.local.json` in the user's project, **merging into any existing `permissions` block rather than replacing it**:

```json
{
  "permissions": {
    "ask": [
      "Bash(9pm delete *)",
      "Bash(9pm deployments rm *)",
      "Bash(9pm deployments delete *)",
      "Bash(9pm files delete *)",
      "Bash(9pm env rm *)",
      "Bash(9pm access *)"
    ],
    "allow": ["Bash(9pm *)"]
  }
}
```

`ask` rules take precedence over `allow`, so the routine surface (`deploy`, `apps`, `doctor`, `logs`, `whoami`, `env ls`, `files` reads, ...) runs hands-off while every destructive or access-changing command still prompts first: `9pm delete` (destroys an app), `9pm deployments rm`/`deployments delete` (removes a deployment), `9pm files delete`, `9pm env rm`, and any `9pm access` change. Use `ask`, not `deny` - `deny` would block those commands outright instead of prompting for them.

This trusts the installed `9pm` binary. If you run the `npx ninepm` form instead, add the same rules with an `npx ninepm` prefix (including `Bash(npx ninepm delete *)` under `ask`). Note that until the public-npm rollout completes the invocation carries an explicit registry flag (`npx --registry https://9pm.ai/npm ninepm ...`), which a plain `Bash(npx ninepm delete *)` glob will not match - gate that interim form with `Bash(npx --registry * ninepm delete *)` (and the same prefix for the other destructive subcommands). Once plain `npx ninepm ...` works, the `npx ninepm`-prefixed rules match directly.

On Codex and other agents there is no clean per-binary allowlist - trust `9pm` through that agent's own approval-policy or sandbox settings, or approve commands as they appear.

## Supported App Shapes

- Static web builds with an output directory containing `index.html` and assets. Common examples: Vite, React, Vue, Svelte, Solid, Astro static output, vanilla HTML/CSS/JS, and static exports. Each individual file is limited to 25 MiB. A single asset larger than that — most often a large WebAssembly engine binary from a Godot/Unity/Flutter web export — cannot be hosted on the static or server runtime; the deploy is rejected up front at `9pm deploy --check` and at create time with an `asset_file_too_large` error naming the file and the limit. This is a hard, deterministic limit, not a transient failure — retrying will not help. Split or externally host the oversized file, or package the app as a container instead.
- Single-page apps with client-side routing, deployed from their build output directory.
- Runtime apps with a bundled JavaScript server entry that exports a `fetch`-compatible handler. Common examples: Hono-style apps, small API backends, and framework runtime adapters that produce one bundled entry.
- Framework apps deploy with zero flags: running `9pm deploy` in a Next.js or Vite project detects the framework and runs the full pipeline itself — installing dependencies if needed, running the production build, then uploading. The build may add an adapter config file to the project; commit it if the CLI says so. Build errors stream through verbatim — they are the framework's own; fix the app code and re-run. `--no-build` skips detection; `--bundle <dir>` deploys a pre-assembled bundle (entry `worker.js` plus an `assets/` directory). Astro/SvelteKit are not auto-built yet — `9pm deploy` says so and tells you what to do: run the framework's Worker-runtime adapter build yourself (`9pm deploy` names the adapter to run and the exact deploy command), then deploy the output with `9pm deploy . --bundle <adapter output dir>` (Worker bundle) or `9pm deploy <static output dir> --no-build` (static output). Apps needing more than ~25 MB gzipped of server code should deploy as containers instead. Known Next.js limitation: routes that use `next/og` (`ImageResponse`) pull in `yoga.wasm`, which the runtime build cannot resolve, so the build fails on it. Pre-render those Open Graph images to static files (or drop the dynamic OG routes) before deploying.
- Container apps with a `Dockerfile` when the app cannot fit the static or server runtime. The CLI builds locally for `linux/amd64`, requires `--port`, accepts `--health-path`, and supports provider-neutral `--region` (`auto`, `us-east`, `us-west`, `europe`, `asia`, `south-america`, `middle-east`, `oceania`, `africa`). The app must listen on `0.0.0.0:$PORT` (or the selected port) and expose an HTTP server. The filesystem is ephemeral by default. A persistent disk (`--volume <gb>`, mounted at `/data`), an always-running instance (`--always-on`), and custom sizing (`--cpus <n>`, `--memory <size>` like `4g`) are available only on placements that support them and may need platform enablement — if they are not enabled, the deploy fails fast with a `container_capability_unavailable` error rather than silently dropping the option.
- Apps that need managed storage when deployed with `--with-db`. Server/runtime code receives a managed storage handle from the runtime; do not expose secrets in browser code.
- Apps that need per-user accounts when deployed with `--with-auth` — 9pm-managed end-user sign-in (email one-time codes), with the current user injected into the app. Implies `--with-db`. See End-User Accounts.
- Private apps using 9pm.ai access commands when the user requests restricted access.

## Persistence

Most apps that store data already use a local database or files. Deploy what they have — do not rewrite the storage layer to fit the platform unless the user explicitly asks. Two persistence options, in order of least disruption:

- **Persistent disk for a container app (`--volume <gb>`), where enabled.** This is the smallest change for a container that already has a local database (SQLite file, embedded store, on-disk data): the disk mounts at `/data`, so point the app's database/data directory at `/data` and keep its current code. A container's filesystem is otherwise ephemeral, so on-disk data is lost between deploys/restarts without a volume. Note this needs a placement that supports volumes and may require platform enablement — if `--volume` returns `container_capability_unavailable`, the disk option is off for this account; fall back to managed SQL below rather than guessing.
- **Managed SQL (`--with-db`).** Always available, maintenance-free, and the right choice for server-runtime apps with no disk of their own or when a volume is not enabled. For a container app that already has its own database, switching to managed SQL means rewriting its data access against the SQL bridge below — propose that as a deliberate change and get the user's approval first; never silently refactor an app's storage just to deploy it.

### Managed SQL

- Server-runtime apps deployed with `--with-db` receive a `DB` binding. Use prepared statements: `env.DB.prepare("SELECT ...").bind(...).all()`.
- Container apps deployed with `--with-db` receive `NINEPM_SQL_URL` and `NINEPM_DB_BINDINGS`. Send `POST` requests to `NINEPM_SQL_URL` with JSON shaped like `{ "binding": "DB", "sql": "SELECT 1", "params": [] }`.
- Keep SQL compatible with SQLite. For local dev, use a local SQLite/Postgres adapter behind a tiny repository layer and switch to the 9pm.ai binding or SQL bridge only in the deployed runtime.
- Initialize schema idempotently with `CREATE TABLE IF NOT EXISTS` unless the app already has a migration system.

## End-User Accounts

When an app needs **per-user accounts** — sign-in, "my stuff" views, data scoped to whoever is logged in — 9pm provides this as a managed feature. Reach for it before wiring up an external identity provider: it removes the same login/session/identity plumbing 9pm already removes for hosting and storage. Offer it whenever the user describes per-user data or a login.

Enable it at deploy time:

```sh
9pm deploy ./app --name "App Name" --slug app-name --with-auth
```

`--with-auth` implies `--with-db` (per-user data needs a managed database) and is one-way — enabling it is enable-only. v1 sign-in is **email one-time codes only**; there is no managed third-party identity provider yet (for that, see the external-provider fallback in the Deploy Workflow).

What you get:

- **9pm hosts the entire sign-in flow** at `/__9pm/auth` on the app's own domain — the app does not build login UI or handle session secrets. A hosted login page is served at `/__9pm/auth`, and the flow is also available as endpoints the app's own UI can call: `POST /__9pm/auth/request-code` (`{ email }`, emails a code), `POST /__9pm/auth/verify` (`{ email, code }`, starts a session), `GET /__9pm/auth/me` (returns the current user or null), and `POST /__9pm/auth/logout`.
- **The app reads the current user from one place.** Worker-runtime apps read `ctx.user` on the request context — `{ id, email }` when signed in, `null` when signed out. Container apps read the `X-9pm-User` request header (JSON `{ id, email }`, absent when signed out); 9pm verifies and sets this header, so trust it and ignore any client-supplied value of the same name.
- **The user `id` is stable** for the same person on that app, so use it as the per-user foreign key in the app's own managed tables (see Managed SQL) — e.g. `WHERE user_id = ?`. 9pm owns the identity; the app owns its data.

Composing with access modes (see Access): managed accounts work on a `public` app and on `private_otp` / `private_idp` apps. On a private app the visitor is already identified by the access gate, so `ctx.user` is populated with **no second login** and the email-code endpoints aren't used. It cannot be enabled on a `private_service` app — there is no human behind a service token.

## Share-by-link Apps

When one URL equals one private workspace (a shared board, doc, or the todo demo below), the URL itself is the access secret:

- Mint a `workspace_id` per workspace with `crypto.randomUUID()`. Reject malformed `:workspace` path values.
- Scope every database read/write by `workspace_id` so workspaces stay isolated.
- Set `Referrer-Policy: no-referrer` and `Cache-Control: private, no-store` on every workspace response.
- Do not load third-party assets from the SPA — they can leak the workspace path through the `Referer` header.
- Tell the user explicitly that the URL is the access secret before deploying.

## Deploy Workflow

1. Confirm auth (`9pm whoami`) and inventory (`9pm apps`) so you don't collide with an existing slug.
2. Inspect the app: stack, package manager, build command, output directory, runtime shape, and persistence needs.
3. If changes are required, explain the smallest compatible change and wait for explicit approval.
4. Build and test with the app's existing commands when available.
5. Run a local preflight:

```sh
9pm deploy ./dist --check
```

6. Deploy static build output:

```sh
9pm deploy ./dist --name "App Name" --slug app-name
```

   Pass an optional `--description "<short note>"` to label what this deploy changed (e.g. `--description "Fix login redirect"`). It appears next to the version in the dashboard and helps the user tell deploys apart later. Keep it short (a sentence; 200 characters max) and prefer including one on every deploy. It is optional — omit it and the deploy just shows its version number. Descriptions can be edited from the dashboard afterward. Works with any runtime:

   ```sh
   9pm deploy ./dist --name "App Name" --slug app-name --description "Initial deploy"
   ```

7. For app-owned server code, deploy a runtime entry:

```sh
9pm deploy ./app --name "App Name" --slug app-name --runtime worker --entry src/index.js
```

8. For Docker/container apps, confirm Docker is installed and the app listens on the selected port, then:

```sh
9pm deploy ./app --check --runtime container --port 8080
9pm deploy ./app --name "App Name" --slug app-name --runtime container --port 8080 --health-path /health --region auto
```

   For persistence, see the Persistence section: prefer `--volume <gb>` (mounts at `/data`) to keep an app's existing local database where it is enabled, and fall back to `--with-db` (managed SQL — the app reads `NINEPM_SQL_URL` and `NINEPM_DB_BINDINGS` for the bridge) when a volume is unavailable or the app has no disk of its own. If deployment is blocked by image size, report whether it hit the per-app image limit, account retained-image quota, or platform capacity. Report `queued`, `publishing`, `active`, or `failed` exactly as printed by the CLI.

   Container readiness checklist: Dockerfile present, HTTP server binds to `0.0.0.0`, selected `--port` matches the app's listen port, optional health endpoint returns 2xx, persistent state uses an attached `--volume` (writing to `/data`) or managed SQL, and secrets are not baked into the image — set them with `9pm env` instead.

   If a container app fails to come up or you need to investigate it after the fact, retrieve its most recent recorded logs:

```sh
9pm logs <app>                # recent container logs + the failure reason, if any
9pm logs <app> --lines 50     # last N lines
9pm logs <app> --json         # machine-readable
```

   `9pm logs` reports the latest container deployment's status, health, the platform-neutral failure reason, and the captured build/startup output when a deploy failed to come up. It is read-only and does not affect deploys. A deployment reported `active` that only errors on live requests may have no captured logs yet; in that case the command says so rather than implying the app is healthy.

9. Use managed files for generated artifacts or handoff files. Never request storage credentials:

```sh
9pm files put ./report.pdf --project app-name
9pm files share <file-id> --expires 24h
9pm files share <file-id> --project app-name --login-required
```

10. Report both the live URL and the dashboard URL printed by the CLI.

For per-user accounts, prefer 9pm's managed end-user auth (`--with-auth`) — see End-User Accounts — so the user doesn't have to run their own identity stack. Only when the user specifically wants their own external provider does the following apply.

If the app uses an external auth/identity provider (Supabase, Auth0, Clerk, Firebase, ...), its login, password-reset, and magic-link redirects are configured in that provider's dashboard, not in the app's env. After deploying to the 9pm URL, tell the user to update the provider's Site URL and allowed redirect URLs to `https://<slug>.9pm.ai` — otherwise auth emails redirect to the old domain or `localhost`. The agent cannot change the provider's dashboard, so flag this as a required manual step alongside setting any `*_SITE_URL` env the app reads.

## Previews and Branches

9pm deploys the files at the path you give it under the slug you pass. It has no notion of git branches or worktrees — deploying does not "pick up the current branch," it uploads whatever is in the deploy directory. Two consequences to handle so a deploy ships the code the user means:

- **Deploy the checkout the user is editing.** When the user is iterating on a branch or in a git worktree and asks to deploy "this", resolve the deploy path to *that* working copy — not a path carried over from an earlier deploy, and not the repo's default checkout. Re-running a previous `9pm deploy` command verbatim is the common way an agent redeploys the old code while the user expected the worktree changes; confirm the path (and your working directory) before running.
- **Same slug replaces the live app; a new slug is a separate preview.** Deploying with an existing slug overwrites that app's live deployment. To preview work-in-progress without disturbing the live app, deploy the branch/worktree under a distinct slug so it gets its own URL:

```sh
9pm deploy ./dist --name "App Name (preview)" --slug app-name-preview
```

When the intent is to preview unfinished work, default to a separate preview slug and confirm with the user whether they want that or to replace the live app at its existing slug. Delete previews you no longer need with `9pm delete <slug> --confirm <slug>` (the `--confirm` flag is required and must match the slug).

## First Todo App

The todo demo is optional. Skip it entirely for established accounts (existing credentials, `9pm login`, or prior apps in `9pm apps`) and whenever you are working inside an existing repository or project — inspect and deploy that app instead. For a fresh signup in an empty workspace, ask the user whether they want the demo and build it only on an explicit yes. When building it, apply the share-by-link pattern above, ask for the target directory, and confirm the file list before writing.

Build spec for the demo:

- Runtime shape: `--runtime worker --with-db`; the entry receives `env.DB`. Initialize schema lazily on the first request.
- Routing: on `GET /` with no workspace, mint a `workspace_id` and redirect to `/w/<id>`; scope every query by `workspace_id`; validate `:workspace` and reject malformed values (per the share-by-link pattern above).
- Schema:

```sql
CREATE TABLE IF NOT EXISTS todos (
  id TEXT PRIMARY KEY,
  workspace_id TEXT NOT NULL,
  text TEXT NOT NULL,
  done INTEGER NOT NULL DEFAULT 0,
  created_at INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS todos_workspace_created ON todos(workspace_id, created_at DESC);
```

- API: `GET`/`POST` `/w/:workspace/api/todos`, `PATCH`/`DELETE` `/w/:workspace/api/todos/:id { text?, done? }`.
- Stack: Vite + React + TypeScript with vite-plugin-singlefile so the SPA collapses to one HTML file; bundle the worker entry as ESM exporting a default `fetch`-compatible handler (`export { app as default }` is supported).
- Deploy:

```sh
9pm deploy ./deploy --check --runtime worker --entry src/index.js
9pm deploy ./deploy --name "Todo" --slug todo --runtime worker --entry src/index.js --with-db
```

- Polish bar: intentional typography, sensible spacing, dark mode via `prefers-color-scheme`, keyboard support, aria-labels on icon buttons, inline SVG icons only, no third-party assets. Report the live URL and dashboard URL the CLI prints.

## Access

Access controls *who may reach the app at all* (the perimeter) — distinct from end-user accounts, which identify *who the visitor is* once inside (see End-User Accounts). The two compose: an app can be private and also have per-user accounts.

Apps are public by default. Manage access only when requested:

```sh
9pm access show app-name
9pm access otp app-name
9pm access idp app-name --provider google
9pm access allow email app-name user@example.com
9pm access allow domain app-name example.com
9pm access revoke email app-name user@example.com
9pm access service create app-name --name backend
9pm access public app-name
```

## Environment Variables and Secrets

Per-app configuration is encrypted at rest and applies on the next deploy. Values are secret (write-only) by default; pass `--var` for a readable config value:

```sh
9pm env set <app> KEY=value [KEY2=value2 ...]      # secret by default (write-only)
9pm env set <app> KEY=value --var                  # readable config value
9pm env set <app> --from-file ./.env               # KEY=value lines; # comments and quotes ok
9pm env set <app> SECRET_NAME --stdin              # value read verbatim from stdin (never the chat)
9pm env request <app> SECRET_NAME                  # prints a one-time browser link to enter the value
9pm env ls <app>
9pm env rm <app> <NAME>
```

Names are UPPER_SNAKE_CASE; `PORT` and `NINEPM_*` are platform-reserved. After changing env, redeploy the app to apply.

When an app needs a real secret you don't already hold — a third-party API key, a token — **do not ask the user to paste it into this chat** (see Authentication: anything pasted here is logged). Instead run `9pm env request <app> SECRET_NAME` and give the user the printed link. They open it, sign in as the app owner, and enter the value in the browser; it is sealed and stored, and never enters this conversation. The link works once and expires quickly. Use `9pm env ls <app>` to confirm the secret now exists (names only — secret values are never readable back), then redeploy. Pass `--var` only for a non-secret readable config value.

## Files

Store generated artifacts or share files without handling storage credentials:

```sh
9pm files list [--project app-name]
9pm files put <local-path> [--path remote/path] [--project app-name]
9pm files get <file-id> [--output ./local] [--project app-name]
9pm files share <file-id> [--expires 24h] [--login-required] [--project app-name]
9pm files delete <file-id> [--project app-name]
```

## Delete

`9pm delete` destroys the app, managed database, managed container app, retained image, and project files. Rows are not recoverable. Before running, state this to the user verbatim and get explicit confirmation, even if they already asked for deletion.

```sh
9pm delete app-name --confirm app-name
```

## When something fails

Most failures are self-fixable — read the error the CLI prints, then work down this list before escalating:

- **Auth / "not signed in" / 401:** run `9pm doctor` (checks the stored credential and reachability), then `9pm whoami` to confirm who you are. If the credential is missing or expired, `9pm login` again.
- **"Unknown command" or behavior that doesn't match this guide:** suspect an out-of-date CLI. Run `9pm latest`, and upgrade if it's behind (see *Versioning*).
- **Install or deploy refused before any upload, or `curl https://9pm.ai/install.sh` is refused:** you're in a sandbox with restricted egress. Trust the required hosts (see *Sandboxed Environments*) — this is applied from *outside* the sandbox, before the session.
- **Deploy/build failed:** the CLI prints the reason. Common fixes: ensure the build output directory exists and is non-empty; re-run with `9pm deploy <dir> --check` to validate the directory before deploying; for `--runtime container`, confirm the image builds locally and the app listens on `0.0.0.0:$PORT`; if a requested capability (always-on, volume, custom sizing) is rejected, it may not be enabled for this account — drop it or escalate.
- **`asset_file_too_large`:** a single static file exceeds the 25 MiB per-file limit (see *Supported App Shapes*). This is deterministic — do not retry. Split or externally host the oversized asset (usually a large WASM binary), or deploy the app as a container.

If you're still stuck after that, escalate with context. From the terminal:

```sh
9pm support --site app-name -m "<what you tried and what happened>"
9pm support -m "<what happened>"          # when it isn't about a specific app
```

This files a tracked request with your account and (with `--site`) the app's last-deploy status and error attached automatically — you don't need to copy any of that in. A human replies to the operator's account email, not to this conversation.
