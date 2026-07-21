---
name: 6-Deploy-Mobile
when_to_use: "User says 'put my app on my phone', 'deploy the mobile app', 'make it work away from home', 'app icon on my phone', 'ship the backend + mobile', or types /6-Deploy-Mobile."
disable-model-invocation: true
description: MUST BE USED when a project has a mobile app (Expo/React Native) plus a backend that currently runs locally, and the user wants the app working on their phone anywhere — backend to Railway + Neon, secured with an API token, and the app either in Expo Go (dev) or as a standalone installed APK via EAS (Android). Heavy on click-by-click guidance for non-developers. Trigger on `/6-Deploy-Mobile`, "put this on my phone", "deploy mobile".
---

# /6-Deploy-Mobile

You walk the user — assume a non-developer — through making a phone app work
anywhere: backend to the cloud, security first, then the app on the phone.
Every browser/dashboard step gets numbered click-by-click instructions.
**Never use placeholder URLs/tokens in commands you give the user — fill in
their real values first; users copy-paste examples literally.**

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and any `DEPLOYMENT.md` if present
- Run `secret-scanner` (working tree + full git history) — must be clean before anything goes to a cloud host or GitHub
- Detect the backend stack yourself (do NOT assume npm — e.g. FastAPI/uvicorn has no package.json; build/start commands differ)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, deployed URL, key decisions, suggested next step
- Update the project's `DEPLOYMENT.md` with anything learned during the run (real port answers, real dashboard labels)

## Critical

- **Security before exposure.** The backend must refuse unauthenticated requests BEFORE it gets a public URL: API token on every endpoint (constant-time compare, lockout on repeated failures), fail-closed local-only mode when no token is set, interactive docs disabled in deployment, secrets only in host env vars. Verify from outside after deploy: `/api/health` → 200, data route without key → 401, with key → 200.
- **Git pushes redeploy the server and kill in-flight background jobs** (data backfills, imports). Hold pushes until such jobs finish, or make jobs idempotent (skip-existing) and re-trigger after deploy.
- Never commit `.env` (mobile or backend). Verify `.gitignore` covers both.

## When to Use

- App works in local dev (Expo + local backend) and the user wants it on their phone away from home
- First cloud deploy of a mobile-backed project
- "How do I get an app icon on my phone?"

## When NOT to Use

- Web-only projects → `/6-Deploy`
- App/Play Store public release — this skill ends at a personal installed build; store submission is a separate effort

## Procedure

### Phase 1: Secure the backend (Claude-side)

Add/verify before anything is public:
1. Token middleware on all routes except `/api/health` (host healthchecks need it open; it must return static status only)
2. Fail-closed default: no token configured → refuse non-local/proxied traffic (check `X-Forwarded-*`: behind any cloud proxy the socket peer looks private — never trust it as "local")
3. Generate command for the user: `python -c "import secrets; print(secrets.token_urlsafe(32))"` (or Node equivalent)
4. Mobile client: read server URL + key from env (`EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_API_KEY`), fall back to dev-host detection so local dev keeps working

### Phase 2: Database (user-side, guided)

Neon (free tier): create project → copy connection string. **SQLAlchemy needs
the driver spelled out: change `postgresql://` to `postgresql+psycopg://` and
add `psycopg[binary]` to requirements.** Cloud DB starts EMPTY — plan the
data re-sync/import step and say so up front (blank screens ≠ broken).

### Phase 3: Backend to Railway (user-side, guided)

Deploy from a **private** GitHub repo. In service Settings:
- Root Directory = backend folder (monorepo: first build FAILS before this is set — warn the user it's expected)
- Custom Start Command with `$PORT` (e.g. `uvicorn app.main:app --host 0.0.0.0 --port $PORT`)
- Healthcheck Path = `/api/health`
- Attach a Volume (e.g. `/data`) if the app writes files — explain it as "a hard drive that survives restarts"
- Variables tab: explain the name/value boxes explicitly ("left box: DATA_DIR, right box: /data — literal text"). All secrets go here.
- Generate Domain: **it asks for the listening port — have the user enter it AND add a matching `PORT` variable** so app and domain agree.

Verify security from outside (Critical section) before continuing.

### Phase 4: Phone app pointed at the cloud (user-side, guided)

`mobile/.env` with `EXPO_PUBLIC_API_URL` + `EXPO_PUBLIC_API_KEY`; restart with
`npx expo start -c`. Explain the two-halves model plainly: data now comes from
the cloud, but Expo Go still loads the app's screens from the PC — full
independence arrives in Phase 6.

Common stumbles (preempt them):
- Expo commands run from the MOBILE folder, not the backend folder ("The expected package.json path ... does not exist" = wrong folder)
- Port 8081 busy → answer Y to use the next port; leftover terminal from an earlier try
- "Restart the app" means: Ctrl+C the terminal running expo, run `npx expo start -c`, re-scan the QR with Expo Go

### Phase 5: Third-party session tokens (if the backend logs into an external service)

Cloud/data-center IPs often get **429 / rate-limited on password logins**
(Garmin, and similar consumer services). Fix pattern:
1. Add a token-seeding endpoint (behind the API key) that accepts the session-token files; write them owner-only (0600)
2. Refresh the tokens LOCALLY (home IP is trusted) **immediately before** uploading — stale tokens fail and fall back to the blocked password path
3. Surface the token-resume failure reason in error messages — a silent `except: pass` around resume cost a debugging round-trip
4. Long backfills: run in background, poll a status endpoint, chunked commits + skip-existing so interruptions are cheap

### Phase 6: Standalone app with its own icon (Android via EAS)

Expo Go from the stores lags behind new SDKs (store review delays; Apple has
gone months). If "project is incompatible with this version of Expo Go":
- Android: sideload the matching official Expo Go — if the expo.dev/go download stalls, use the same file from `github.com/expo/expo-go-releases/releases` (official; version index at `exp.host/--/api/v2/versions`)
- iOS: no free path; requires Apple Developer ($99/yr) — which a real iPhone install needs anyway

Standalone build (the icon):
1. BEFORE first build set `app.json`: display `name` (it defaults to the folder name — "mobile" on a home screen looks broken), `slug`, `android.package`
2. `eas.json` with a `preview` profile, `"buildType": "apk"`, `"distribution": "internal"`
3. **`.easignore` that mirrors ignores but KEEPS `mobile/.env`** — EAS respects .gitignore by default, which would strip the env vars and the built app would silently point at localhost
4. User creates a free expo.dev account; `npx eas-cli login` is interactive — the USER runs it (browser flow), Claude cannot
5. `npx eas-cli build --platform android --profile preview` → yes/Enter to first-run prompts → 10–20 min free queue → install link/QR on the phone
6. Set expectations explicitly — non-developers assume an installed app auto-updates. The real model:
   - **Server-side changes** (insights, sync, security, anything backend): git push → host redeploys → installed app benefits instantly. No rebuild ever.
   - **Screen/JS changes**: with a plain APK, each one needs a full rebuild + reinstall (~15+ min).
   - **Native changes** (Expo SDK upgrade, new native module): always a real rebuild, no way around it. Rare — a few times a year.

### Phase 6.5: Wire EAS Update (over-the-air screen updates)

Do this before the SECOND screen change, so the user's next rebuild is their
last routine one. EAS Update ships JS changes to installed apps in seconds
(downloaded silently on next app launch); free tier is generous.

1. `npx expo install expo-updates` then `npx eas-cli update:configure`
2. **One more rebuild is required** after wiring it in — the update-receiver must be baked into the APK. Warn the user this final rebuild is expected, then it's OTA from there.
3. Publishing a screen change afterwards:
   `npx eas-cli update --channel preview --environment preview --non-interactive --message "what changed"`
   — seconds, no reinstall, no phone touching. **`--environment` is mandatory
   everywhere `eas update` runs (manual AND CI): without it the bundle ships
   with NO baked `EXPO_PUBLIC_*` vars, the app can't authenticate, and the
   user sees demo mode that reads as "my app reverted to an old version".**
4. Phones fetch the LATEST update group on the branch — a bad publish after a
   good one wins. Fix by republishing correctly; waiting never helps.
5. Native changes still require a rebuild; EAS Update only carries JS/assets.
   **Rollback trap: with `runtimeVersion: {"policy": "appVersion"}`, adding a
   native module does NOT change the runtime version — old installed binaries
   ACCEPT the new OTA, crash at launch on the missing native module, and
   expo-updates silently rolls back to the previous bundle. The phone "looks
   old" with no error surfaced anywhere. A new build + sideload must land
   BEFORE any OTA that references new native modules.**

### Phase 6.6: Auto-publish OTA on git push (GitHub Actions)

Wire pushes touching `mobile/**` to publish the OTA automatically:
1. Workflow (`.github/workflows/eas-update.yml`): checkout → setup-node →
   `expo/expo-github-action` with `token: ${{ secrets.EXPO_TOKEN }}` →
   `npm ci` → `eas update --branch <channel> --environment <env> --auto --non-interactive`
2. The USER creates the token (expo.dev → Account → Access tokens) and adds it
   as the `EXPO_TOKEN` repo secret (GitHub → Settings → Secrets and variables
   → Actions). **Without the secret the workflow fails on every push while the
   push itself succeeds — screens silently never reach the phone.**
3. VERIFY after the first push: `gh run list --workflow eas-update` shows
   success. Backend deploy (host) and OTA (EAS) are independent pipelines —
   never report "it'll reach your phone" without checking both: curl a NEW
   endpoint on the deployed backend, and `eas update:list --branch <ch>` for
   what phones actually fetch.

### Phase 7: Verify end-to-end

- Phone on cellular (wifi OFF), PC off if standalone: app loads, data appears
- Security spot-check from any browser: data endpoint without key → auth error (that error IS the success)
- Background sync/import completed or progressing (status endpoint)
