---
name: onboard
description: Onboard a repo onto Telescope so its web app can be previewed on a phone, at real desktop size, while Claude Code Cloud edits it. Use when the user wants to set up, onboard or "telescope up" a repo, preview their app on their phone, pair a phone with a preview, push env vars to a preview, debug a broken preview, or manage or remove Telescope previews.
---

Telescope gives a repo a private **preview box** in the cloud. The box runs the app's `next dev`, plus any backend the app needs, and serves it to the user's phone as a home-screen app inside a 1440×900 "laptop frame". Opening the app syncs the newest `claude/telescope/*` branch that Claude Code Cloud pushed.

You drive Telescope through its CLI, `telescope` (npm package `usetelescope`). This skill covers onboarding a repo that already exists on this machine, fixing what stops it from working in the preview, and managing previews afterwards.

## How to talk to the user

The user is usually setting this up on the go and wants to act, not decode. Every time they have to do something themselves (in GitHub, Supabase, Google Cloud, Clerk, a terminal, their phone):

- **Lead with the action.** Put "here's what you need to do" first, as a numbered list, before explanations. Never bury a required step in a paragraph.
- **Name the exact place:** the service, then the exact menu path, then the field. For example, "**Supabase** (not GitHub) → your project → Authentication → URL Configuration → Redirect URLs → **Add URL**".
- **Give the exact value, fully filled in**, in its own code block so it can be copied. Never hand over a template like `https://<preview-host>/...` once you know the real host.
- **Say what kind of thing it is** ("this is a URL, not a key") and **what it's for** in one short clause ("so Google can send you back to the preview after sign-in").
- **Be explicit about scope:** "add, don't replace", "delete both of these", "only this repo".
- **Never paste secrets, keys or pairing links into chat.** Point to where they are instead.
- After the user reports back, say plainly whether it worked and what's next.

## 0. CLI and sign-in

- Check for the CLI with `command -v telescope`. If it's missing, use `npx usetelescope <command>` for everything below. Only install it globally (`npm install -g usetelescope`, Node 20+) if the user says yes.
- Run `telescope whoami`. If it says "Not signed in", run `telescope login` and tell the user: "A browser tab opened. Sign in with GitHub, then click **Authorize** on the Telescope CLI page."

## 1. Find the repo

You need the **local checkout**, because env files live there, and you run `telescope up` from inside it. Its GitHub remote (`git remote get-url origin`) must be on github.com. If the user names a project without a path, look for a `package.json` that depends on `next` in their usual project folders.

## 2. Survey the repo: what can and can't be previewed

Finding facts is your job; don't ask the user for anything you can read. Look at the whole repo, not only the web app.

**What Telescope can run:**
- **The web app:** a Next.js app, in the directory whose `package.json` depends on `next` (the repo root or a subfolder such as `frontend/`, `web/` or `apps/web`). Installs are automatic from its lockfile (npm, pnpm, yarn or bun).
- **Local backends the web app needs** (FastAPI, Express, Django, …): find each one's folder, one-time install command, start command and port (from the README, `requirements.txt`/`pyproject.toml`, `package.json` scripts, `Procfile` or `docker-compose.yml`). These run as extra services on the box, bound to `127.0.0.1`, with file watching (`uvicorn --reload`, `nodemon`, `tsx watch`) so synced changes apply without a restart.

**What it can't run.** Find these and tell the user up front; set up the parts that can run anyway:
- browser extensions (`manifest.json` with `manifest_version`), mobile apps (React Native, Expo, Swift, Kotlin), desktop apps, CLIs, cron jobs and background workers that aren't started by the web app;
- Docker-only services (Postgres or Redis via `docker-compose`). Suggest a hosted dev instance (Neon, Supabase, Upstash) in the env file instead;
- popup-based sign-in (`signInWithPopup`, popup modes). Suggest the provider's redirect mode;
- web frameworks other than Next.js aren't supported yet. Say so, and offer to send feedback (see below).

**Env files:** `.env`, `.env.local` and `.env.development*` in the app folder and at the repo root. Never include `.env.production*`. **Print variable names only, never values:** `sed -E 's/=.*//' <file> | grep -v '^#'`. When you need to know where a URL points, print only its host and port.

**Then check for the problems that break previews** (each of these came up in real onboardings):

1. **Browser calls to a local backend.** If a `NEXT_PUBLIC_*` variable (or code running in the browser) points at `localhost` or `127.0.0.1`, it can't work: on the phone, `localhost` is the phone itself. Plan to route those calls through the Next app instead (step 5).
2. **CORS and other domain allowlists.** Check the backend's allowed origins (for example FastAPI `CORSMiddleware`, Express `cors()`), and anything else that only works on listed domains: OAuth providers, reCAPTCHA site keys, Google Maps keys, Stripe, Supabase redirect URLs. The preview has its own address, so each of these needs it added (step 6).
3. **Production data.** If an env var points at a real production database, storage bucket or API, warn the user that actions in the preview change real data, and suggest a dev instance.
4. **Paid API keys** (OpenAI, Anthropic, Stripe, and so on): clicks in the preview spend real money. Recommend dev or test keys with a spend cap set at the provider.
5. **External services are actually up.** For each external host in the env files (database, a deployed API, third-party services), check from this machine that it resolves and answers (`nslookup <host>`, `curl -sS -o /dev/null -w '%{http_code}' https://<host>`). Free-tier databases get paused and deployed backends get removed. Catch that now, so a dead dependency isn't mistaken for a Telescope problem later.

## 3. Write the project config

Write it to a temporary file **outside the repo** (for example `$(mktemp -d)/telescope.json`). Nothing gets committed; Telescope stores the config with the project.

```json
{
  "name": "<lowercase-dashes, 2-40 chars; becomes the project name>",
  "title": "<short home-screen name>",
  "app": "<Next app folder relative to the repo root, or .>",
  "env": [{ "from": "<local env file, relative to the repo root>", "to": "<where the app reads it, relative to the repo root>" }],
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
- **Preview-only env values:** when the preview needs a different value than local development (for example `NEXT_PUBLIC_BACKEND_URL=/backend`, or a dev database), copy the env file to a preview version such as `.env.telescope.local`, change only those values, make sure it's gitignored, and point `from` at it (keeping `to` as the file the app reads). The user's normal local setup stays untouched.

**Before running anything, show the user one summary, in this order:**
1. **What will be previewed** (the web app, and each backend as a service) and **what won't be**, with the reason (for example "your Chrome extension: Telescope can't run browser extensions").
2. **What will be copied to the preview box:** each env file, with its variable names. Say that env files go from this machine through Telescope straight to their own box and are never stored by Telescope.
3. **Risks found:** production data, paid keys, dead external services, browser calls to localhost.
4. **What they'll need to do after setup** (allowlists, the GitHub App), so nothing surprises them.

Wait for their go-ahead.

## 4. Run `telescope up`

Run it from the repo root **in a terminal the user can see** (not a background command), because it may stop for input:

```bash
telescope up --config <path to the config file>
```

- **The Telescope GitHub App:** on a repo's first run, the CLI opens GitHub's page for the `telescope-preview` app. Tell the user exactly what they'll see:
  - **If the app isn't installed on their account yet** (they see an **Install** button): choose their account → **Only select repositories** → select this repo → **Install**.
  - **If it's already installed for other repos** (they see **Configure**): under **Repository access**, keep **Only select repositories** → **Select repositories** → add this repo → **Save**.
  - Then press **Enter** in the terminal. The app only gets read access to the repos they choose.
- It creates the preview box, copies env files, installs dependencies, starts the services and prints the preview URL. Expect one to two minutes on the first run.
- If no phone is paired yet, it opens a QR code in the browser at the end (step 7).
- It's safe to re-run: `telescope up` with no `--config` reuses the stored config and repairs the box.

Errors you may see:
- **"You can have up to 3 projects"**: suggest `telescope down <project>` for one they no longer use.
- **"suspended" or "paused"**: the account is suspended or Telescope is paused. Nothing to fix locally.
- **"doesn't have access to <repo>"**: the signed-in GitHub account can't read the repo.

## 5. Fix what would break in the preview (on a branch)

For problems from step 2 that need code changes, make them on a `claude/telescope/<short-name>` branch (the preview follows these). Never change the user's main branch or production config without asking. Keep every change **inert in production**.

- **Browser calls to a local backend:** add a rewrite in `next.config.*` so the browser calls the Next app and Next forwards to the backend on the box, then set the preview-only env value to the relative path:

  ```ts
  async rewrites() {
    return [{ source: "/backend/:path*", destination: "http://127.0.0.1:8000/:path*" }];
  }
  ```

  Preview env (`.env.telescope.local`): `NEXT_PUBLIC_BACKEND_URL=/backend`. Production keeps its absolute URL, so the rewrite is never used there. These requests now come from the same origin, so CORS no longer applies to them.
- Push the branch with the user's OK, then run `telescope env push` if the env changed. The preview picks up the branch the next time it's opened.

## 6. External settings the user has to change

The preview's address comes from `telescope ls` (for example `https://email-tracker-aa801e-b4u72.sprites.app`). It stays the same for the life of the project, so each of these is a one-time step. Give each item in the format from "How to talk to the user", fully filled in. Common ones:

- **Supabase Auth:** **Supabase** → the app's project → **Authentication → URL Configuration → Redirect URLs → Add URL**, then paste the app's callback path on the preview host with a trailing `**` (find the path in the app's `redirectTo`), for example `https://<host>/auth/callback**` → **Save**. Nothing changes on the GitHub or Google side.
- **Google OAuth (Auth.js/NextAuth, or direct):** **Google Cloud Console** → APIs & Services → **Credentials** → the app's OAuth 2.0 Client ID → **Authorized redirect URIs → Add URI**: `https://<host>/api/auth/callback/google` → **Save**. Google allows several redirect URIs, so this is added alongside the existing ones.
- **GitHub OAuth (Auth.js):** GitHub OAuth Apps allow only **one** callback URL, so the user creates a separate "preview" OAuth App (github.com → Settings → Developer settings → **OAuth Apps → New OAuth App**) with callback `https://<host>/api/auth/callback/github`, puts its client ID and secret in the preview env file, and you run `telescope env push`.
- **Clerk:** a development instance (`pk_test_…`) needs nothing; a production instance (`pk_live_…`) needs the preview domain added in the Clerk dashboard.
- **Backend CORS:** if the browser still calls a backend directly, add `https://<host>` to its allowed origins (a code change, on the branch).
- **reCAPTCHA, Google Maps and other key-restricted services:** add the preview's domain (`<host>`, no `https://`) to the key's allowed domains in that service's console.

Telescope sets `AUTH_URL` and `NEXTAUTH_URL` to the preview address automatically. Never suggest a wildcard across `*.sprites.app`: every Telescope preview shares that domain.

## 7. Pair the phone

If `telescope up` didn't open a QR code (a phone is already paired), run `telescope link`. Tell the user:

1. Scan the QR code on your laptop screen with your phone's camera. It works once and expires in 10 minutes.
2. When the preview opens in Safari: **Share → Add to Home Screen**, with **Open as Web App** turned **on**.
3. The QR page on your laptop switches to "Paired! You can close this tab".

Never paste the pairing link into chat; the QR page keeps it out of the conversation.

## 8. Verify it actually works

You can't open the preview yourself (only paired devices can), so verify in two ways:

- **Ask the user to check specific things,** for example "Does the dashboard load? Tap **Sign in with Google**: do you end up signed in? Do your emails show up?"
- **Read the box's logs** while they try: `telescope logs --service next`, plus `--service backend` for each backend. Look for real errors even if the page "looks fine". Apps often swallow errors and just show an empty list.

If something fails, follow "When the preview misbehaves" below before changing anything.

## 9. Wire Claude Code Cloud to the preview

1. **The Claude GitHub App must be installed on this repo**, or Cloud sessions can't push. A public repo clones fine without it, then fails on the first push with a 403 that looks like an account problem. Tell the user: **github.com → Settings → Applications → Installed GitHub Apps → Claude → Configure → Repository access** → add this repo → **Save** (or install it from claude.ai/connect-github if Claude isn't listed).
2. Offer to add this to the repo's **root** `CLAUDE.md` (create it if missing), on a **review branch, never main**, using a temporary git worktree so the user's checkout isn't disturbed:

```markdown
## Telescope preview

This repo is previewed on a phone through Telescope, which syncs the newest `claude/telescope/*` branch into a live preview.

- In cloud sessions, work on a branch named `claude/telescope/<short-task-name>`, cut from the branch you were asked to work on (usually `main`).
- End every turn that changed code with `git commit` and `git push` to that branch, so the preview can sync it.
```

Push the branch only with the user's OK, and give them the compare link to open a PR.

## Done when

The preview is live, the user has checked on their phone that the app works (sign-in and data, not just the first page), they know what wasn't previewable and why, and they know the loop: work in a Claude Code Cloud session → open the home-screen app → it syncs.

## When the preview misbehaves: find out whose problem it is

When a page errors, sign-in fails, data doesn't load, or the app looks different from local, work out whether it's the **project** or **Telescope** before changing anything:

1. **Read the box's logs:** `telescope logs [project] --service next` (also `--service backend` for each extra service, and `--service telescope` for the preview proxy; `--lines 300` for more). Look for the actual error, such as a failed database connection, a missing env var, or a crash.
2. **Reproduce the same thing locally** with the same env files: run the app or call the same endpoint on this machine, and re-check that external services are reachable (step 2, item 5).
3. **Decide:**
   - **Fails locally too:** it's the project or its services, for example a paused or deleted database, an expired key, or a deployed backend that's down. Tell the user plainly what's broken and exactly where to fix it; Telescope is fine.
   - **Works locally, fails only in the preview:** it's preview-specific. Usually a browser call to `localhost` (step 5), a missing allowlist entry (step 6), or env that hasn't been pushed (`telescope env push`). Fix it on a `claude/telescope/*` branch; if it looks like a Telescope bug, send feedback.

## When something doesn't work: send feedback

Telescope learns which stacks and setups to support next from feedback. **Offer to send it** whenever onboarding fails, part of the repo can't be previewed, the stack isn't supported (a framework other than Next.js, Docker-only services, a sandbox or agent-orchestrator setup, an unusual auth flow), or the user asks:

```bash
telescope feedback "<what was being set up, what failed or isn't supported, and what the user wanted>" --project <name>
```

Write a short, specific report, for example `Onboarding email-tracker: Next 16 dashboard + FastAPI backend. Dashboard called the backend from the browser at NEXT_PUBLIC_BACKEND_URL=localhost:8000, which can't work on the phone; worked around with a Next rewrite to /backend. Chrome extension not previewable.` The CLI attaches its version, the OS, the repo's name and the last error automatically. **Never include code, env values, secrets or personal data**, and show the user the message before sending it.

## Managing previews afterwards

| User wants to | Run |
| --- | --- |
| See their projects | `telescope ls` |
| Update env vars after editing a local env file | `telescope env push [project]` (works from anywhere; restarts the services) |
| Pair another phone or browser | `telescope link [project]` |
| See or remove paired devices | `telescope devices [project]`, `telescope devices revoke <id> [project]` |
| Repair a preview | `telescope up` inside the repo |
| See what the preview box is logging | `telescope logs [project] [--service <name>] [--lines <n>]` |
| Report a problem or a missing stack | `telescope feedback "<message>" [--project <name>]` |
| Delete a preview (destroys the box and its env files) | `telescope down <project>`. Confirm with the user first, then pass `--yes`. |

Limits: 3 projects per account; one preview awake at a time (opening another puts the previous one to sleep); 4 awake hours per day (resets at midnight UTC). Previews sleep when unused and wake when opened. Inside the preview, the toolbar has sync, a branch picker ("Latest Claude branch (auto)" or a pinned branch), screen sizes, keep warm (up to 30 minutes) and **Sleep now**. Each project gets its own home-screen icon; iOS caches icons, so a changed icon needs the app removed and re-added.
