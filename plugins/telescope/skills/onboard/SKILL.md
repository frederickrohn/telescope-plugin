---
name: onboard
description: Onboard a repo onto Telescope so its Next.js app can be previewed on a phone, at real desktop size, while Claude Code Cloud edits it. Use when the user wants to set up, onboard or "telescope up" a repo, preview their app on their phone, pair a phone with a preview, push env vars to a preview, or manage or remove Telescope previews.
---

Telescope gives a repo a private preview box in the cloud. It runs the app's `next dev` (plus any backend it needs) and serves it to the user's phone as a home-screen app inside a 1440×900 "laptop frame". Opening the app syncs the newest `claude/telescope/*` branch that Claude Code Cloud pushed.

You drive Telescope through its CLI, `telescope` (npm package `usetelescope`). This skill covers onboarding a repo that already exists on this machine, plus the commands for managing previews afterwards.

## 0. CLI and sign-in

- Check the CLI with `command -v telescope`. If it's missing, ask the user before installing anything globally: `npm install -g usetelescope`. Node 20+ is required. Alternatively, run every command below as `npx usetelescope <command>`.
- Run `telescope whoami`. If it says "Not signed in", run `telescope login`. It opens a browser for "Sign in with GitHub" and an "Authorize Telescope CLI" confirmation; tell the user to complete both.

## 1. Find the repo

You need the **local checkout**, because env files live there. Run every `telescope up` from inside it. Its GitHub remote (`git remote get-url origin`) must be on github.com. If the user names a project without a path, look for a `package.json` that depends on `next` in their usual project folders.

## 2. Inspect what it needs to run

Finding facts is your job; don't ask the user for anything you can read.

- **Next app folder:** the directory whose `package.json` depends on `next`. That may be the repo root or a subfolder such as `frontend/`, `web/` or `apps/web`. A lockfile should sit next to it; installs are automatic.
- **Env files:** `.env`, `.env.local` and `.env.development*` in the app folder and at the repo root. Never include `.env.production*`. **Print variable names only, never values:** `sed -E 's/=.*//' <file> | grep -v '^#'`.
- **Backends the frontend calls:** look for URL-like env vars (`*_URL`, `*_API*`, `*_HOST`) that point at `localhost:<port>` or `127.0.0.1`. Print only the host and port, nothing else. Each one means a local backend must also run on the preview box.
- **For each local backend:** find its folder, language, one-time install command and start command (from the README, `requirements.txt`/`pyproject.toml`, `package.json` scripts, `Procfile` or `docker-compose.yml`), the port it listens on, and the env file it loads. Bind it to `127.0.0.1` and use file watching (`uvicorn --reload`, `nodemon`, `tsx watch`) so synced changes apply without a restart.
- **Docker-only services** (Postgres, Redis via compose) aren't supported. Suggest a hosted dev instance (Neon, Supabase, Upstash) in the env file instead.
- **Auth:** note the provider (`@supabase/ssr` or `@supabase/supabase-js`, `next-auth`, `@clerk/nextjs`, others) and how sign-in starts. Popup-based sign-in (`signInWithPopup`) isn't supported in the preview; suggest the provider's redirect mode.

## 3. Write the project config

Write the config to a temporary file **outside the repo** (for example `$(mktemp -d)/telescope.json`). Nothing gets committed; Telescope stores the config with the project.

```json
{
  "name": "<lowercase-dashes, 2-40 chars; becomes the project name>",
  "title": "<short home-screen name>",
  "app": "<Next app folder relative to the repo root, or .>",
  "env": [{ "from": "<path to the local env file, relative to the repo root>", "to": "<same path relative to the repo root>" }],
  "services": [
    { "name": "backend", "dir": "backend", "setup": "<one-time install>", "cmd": "<binary; relative to dir if it contains a slash>", "args": ["..."] }
  ],
  "syncHooks": [
    { "match": "^backend/requirements\\.txt$", "cwd": "backend", "run": ["<reinstall argv>"], "restart": ["backend"] }
  ]
}
```

- `services` is only for extra processes; `next dev` and Telescope's own proxy are added automatically. Omit `services` and `syncHooks` for a plain Next app. Service names `next` and `telescope` are reserved.
- Add a `syncHooks` entry for each non-JS dependency file, so a synced change reinstalls and restarts that service. JS lockfiles (including ones in subfolders) are handled automatically.

**Show the config to the user before running anything**, and call out:
- every env file that will be copied to the preview box, with its variable names. Env files go from this machine through Telescope's API straight to the user's own box; they're never stored by Telescope;
- any **paid API keys** among them (OpenAI, Anthropic, Stripe, and so on): clicks in the preview spend real money. Recommend dev or test keys with a spend cap set at the provider, never production secrets.

Wait for their go-ahead.

## 4. Run `telescope up`

Run it from the repo root **in a terminal the user can see** (not a background command), because it may stop for input:

```bash
telescope up --config <path to the config file>
```

- **First time for a repo:** it opens the **Telescope GitHub App** install page. Tell the user to choose their account, pick **Only select repositories**, select this repo and install, then press Enter in the terminal. The app only gets read access to the repos they choose.
- It creates the preview box, copies env files, installs dependencies, starts the services and prints the preview URL. Expect one to two minutes on the first run.
- If no phone is paired yet, it opens a **QR code** in the browser at the end.
- It's safe to re-run: `telescope up` with no `--config` reuses the stored config and repairs the box.

Errors you may see:
- **"You can have up to 3 projects"**: suggest `telescope down <project>` for an unused one.
- **"suspended" or "paused"**: the account is suspended or Telescope is temporarily paused. Nothing to fix locally.
- **"doesn't have access to <repo>"**: the signed-in GitHub account can't read the repo.

## 5. Allow the preview URL in the app's auth provider

Only if the app has sign-in. Providers only redirect back to allowlisted URLs, and the preview has its own address. Get it from `telescope ls`, then tell the user exactly what to add, fully filled in:

- **Supabase Auth:** find the app's `redirectTo` (for example `${origin}/auth/callback?next=...`), then add `https://<preview-host>/auth/callback**` under Authentication → URL Configuration → Redirect URLs. The GitHub or Google side doesn't change.
- **Auth.js:** the provider's OAuth app needs `https://<preview-host>/api/auth/callback/<provider>`. GitHub OAuth Apps allow only one callback URL, so create a separate "preview" OAuth app, put its ID and secret in the env file, and run `telescope env push`. Telescope sets `AUTH_URL` and `NEXTAUTH_URL` automatically.
- **Clerk:** a development instance (`pk_test_…`) works without changes; a production instance (`pk_live_…`) needs the preview domain added in the Clerk dashboard.

Never suggest a wildcard across `*.sprites.app`: every Telescope preview shares that domain. The URL is stable for the life of the project, so this is a one-time step.

## 6. Pair the phone

If `telescope up` didn't open a QR code (a phone is already paired), run `telescope link`. Tell the user: scan the QR code with the phone's camera within 10 minutes (it works once), then in Safari tap Share → Add to Home Screen, with **Open as Web App** turned on. **Never paste pairing links into chat.** The QR page keeps the code out of the conversation.

## 7. Wire Claude Code Cloud to the preview

Offer to add this to the repo's **root** `CLAUDE.md` (create it if missing), on a **review branch, never main**. Use a temporary git worktree so the user's checkout isn't disturbed:

```markdown
## Telescope preview

This repo is previewed on a phone through Telescope, which syncs the newest `claude/telescope/*` branch into a live preview.

- In cloud sessions, work on a branch named `claude/telescope/<short-task-name>`, cut from the branch you were asked to work on (usually `main`).
- End every turn that changed code with `git commit` and `git push` to that branch, so the preview can sync it.
```

Push the branch only with the user's OK, and give them the compare link to open a PR. Also remind them that **Claude Code Cloud needs the Claude GitHub App installed on this repo to push**. A public repo clones fine without it but fails on the first push with a 403. Check at github.com/settings/installations → Claude → Repository access ("Only select repositories").

## Done when

The preview is live, the user's phone has the home-screen app showing their app in the laptop frame, and they know the loop: work in a Claude Code Cloud session → open the home-screen app → it syncs.

## When the preview misbehaves: find out whose problem it is

When a page errors, sign-in fails, data doesn't load, or the app looks different from local, work out whether it's the **project** or **Telescope** before changing anything:

1. **Read the box's logs:** `telescope logs [project] --service next` (also `--service backend` for each extra service, and `--service telescope` for the preview proxy; `--lines 300` for more). Look for the actual error, such as a failed database connection, a missing env var, or a crash.
2. **Reproduce the same thing locally** with the same env files: run the app or call the same endpoint on this machine. Also check that external services the app depends on are reachable from here (database host, a deployed API it calls, third-party APIs).
3. **Decide:**
   - **Fails locally too:** it's the project or its services, for example a paused or deleted database, an expired key, or a deployed backend that's down. Tell the user plainly what's broken and where to fix it; Telescope is fine.
   - **Works locally, fails only in the preview:** it's preview-specific. Common causes: a URL in the env pointing at `localhost` that the browser uses (the phone can't reach it; route those calls through the Next app instead), an auth provider or CORS list that doesn't include the preview URL, or env that `telescope env push` hasn't sent yet. Fix what you can on a `claude/telescope/*` branch; if it looks like a Telescope bug, send feedback (below).

Apps that swallow errors (for example an empty list when a fetch fails) are common, so don't trust an empty page; check the logs.

## When something doesn't work: send feedback

Telescope learns which stacks and setups to support next from feedback. **Offer to send it** whenever onboarding fails, part of the repo can't be previewed, the stack isn't supported (a framework other than Next.js, Docker-only services, a sandbox or agent-orchestrator setup, an unusual auth flow), or the user asks:

```bash
telescope feedback "<what was being set up, what failed or isn't supported, and what the user wanted>" --project <name>
```

Write the message as a short, specific report, e.g. `Onboarding email-tracker: Next 16 dashboard + FastAPI backend. Dashboard calls the backend from the browser at NEXT_PUBLIC_BACKEND_URL=localhost:8000, which can't work on the phone; worked around with a Next rewrite to /backend. User also has a Chrome extension (not previewable).` The CLI attaches its version, the OS, the repo's name and the last error automatically. **Never include code, env values, secrets or personal data**, and show the user the message before sending it.

## Managing previews afterwards

| User wants to | Run |
| --- | --- |
| See their projects | `telescope ls` |
| Update env vars after editing a local env file | `telescope env push [project]` (from anywhere; restarts the services) |
| Pair another phone or browser | `telescope link [project]` |
| See or remove paired devices | `telescope devices [project]`, `telescope devices revoke <id> [project]` |
| Repair a preview | `telescope up` inside the repo |
| Delete a preview (destroys the box and its env files) | `telescope down <project>`. Confirm with the user first, then pass `--yes`. |
| See what the preview box is logging | `telescope logs [project] [--service <name>] [--lines <n>]` |
| Report a problem or a missing stack | `telescope feedback "<message>" [--project <name>]` |

Limits: 3 projects per account, one preview awake at a time (opening another puts the previous one to sleep), and 4 awake hours per day (resets at midnight UTC). Previews sleep when unused and wake when opened. Keep-warm (up to 30 minutes) and **Sleep now** are in the toolbar inside the preview.
