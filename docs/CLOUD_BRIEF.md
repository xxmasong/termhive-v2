# TermHive Cloud — build brief (M2–M6)

Supersedes the open questions in `docs/MULTI_TENANT_PLAN.md`. Decisions are
final (owner, 2026-09-25):

| Topic | Decision |
|---|---|
| Hosting | Everything on SG, inside **CT102** (`sg1-termhive2`, 10.10.1.12, Debian 13, node 22.14, systemd 257, nft available, cgroup v2 with memory/cpu/pids). |
| Public domain | **None.** Stays tailnet-only at `https://sg1-termhive2.tailfa2e0b.ts.net` (tailscale serve → `127.0.0.1:3200`, sidecar socket `/var/run/tailscale-termhive2.sock`). |
| Sign-in | **Firebase Authentication** for everything: email+password, Google, GitHub. |
| Email | **Firebase sends it** (verification + password reset from `noreply@<project>.firebaseapp.com`). No SMTP, no domain. |
| Resources | Use everything the CT has — no per-plan RAM/CPU caps. Plans limit only project/agent counts. Grow CT102 disk (`pct resize 102 rootfs +30G` on the SG host) before provisioning users. |
| Prices | Dummy: Pro `$12/mo`, Pro Plus `$29/mo` (mark as placeholder in code). No billing: the plan chosen at signup is granted; admin can change it. |
| Signups | `SIGNUP_MODE=open` by default (tailnet is the gate); `invite` mode supported. |
| Isolation | **One Linux user per account** (not containers — simplest; no Docker in CT102). |

## 1. Runtime topology (CT102)

```
tailscale serve ─► :3200  termhive-cloud.service  (root)   NEW control plane
                           • serves dist/client (landing, /login…, /app shell)
                           • /auth/* (session exchange, me, logout, config)
                           • proxies /api, /ws, /claude-chat… (HTTP + WS upgrade)
                             to the caller's workspace on 127.0.0.1:<port_base>
                           • SQLite at /var/lib/termhive-cloud/cloud.db (node:sqlite)
   ├─► 4000-4002  termhive2.service (root)  = admin's existing workspace (unchanged data)
   ├─► 4010-4012  termhive-ws@th-xxxx.service (User=th-xxxx, HOME=/home/th-xxxx)
   └─► 4020-4022  …   (port_base = 4000 + 10·n → web, daemon, claude bridge)
```
Workspace processes are today's `dist/server.js` + `dist/daemon/daemon.js`
unchanged apart from §4, configured by env: `PORT`, `TERMHIVE_DAEMON_PORT`,
`CLAUDE_BRIDGE_PORT`, `TERMHIVE_MAX_PROJECTS`, `TERMHIVE_MAX_AGENTS`,
`TERMHIVE_CONFINE_HOME=1`, `HOME`. Env files `/etc/termhive/ws/<user>.env`, root 0600.
Agent CLIs (`claude codex gemini opencode`) are already in `/usr/local/bin`; each
user logs them in inside their own HOME via the existing Connect flow.

## 2. Control plane (`src/cloud/`, bundled by tsup to `dist/cloud/*.js`)

- **DB (SQLite, migrations in code):** `users(id, firebase_uid unique, email, name, avatar_url, plan, role, status, created_at)`, `workspaces(user_id unique, unix_user, port_base unique, state provisioning|running|stopped|error, created_at, last_active_at)`, `sessions(id_hash pk, user_id, created_at, expires_at, ip, user_agent)`, `invites(code_hash pk, max_uses, uses, expires_at, created_at)`, `audit_log(id, user_id, action, detail json, at)`.
- **Firebase ID-token verification without firebase-admin:** RS256 against `https://www.googleapis.com/robot/v1/metadata/x509/securetoken@system.gserviceaccount.com` (cache by `Cache-Control` max-age), `aud = FIREBASE_PROJECT_ID`, `iss = https://securetoken.google.com/<id>`, `exp/iat/auth_time` sane, `sub` non-empty. Password accounts require `email_verified=true`.
- **Endpoints:**
  - `GET /auth/config` → Firebase web config from env (`FIREBASE_API_KEY, FIREBASE_AUTH_DOMAIN, FIREBASE_PROJECT_ID, FIREBASE_APP_ID`); `{configured:false}` if unset.
  - `POST /auth/session {idToken, plan?, inviteCode?}` → verify; first sign-in creates the user (checks `SIGNUP_MODE`/invite → `403 SIGNUPS_CLOSED`/`INVALID_INVITE`; unverified → `403 EMAIL_UNVERIFIED`), provisions the workspace, sets cookie `th_session` (random 256-bit, stored hashed; HttpOnly, Secure, SameSite=Lax, 30 days) → `200 {ok:true}`. Rate-limit per IP.
  - `GET /auth/me` → `{user:{email,name,avatarUrl,role}, plan:{id,maxProjects,maxAgents}, workspace:{state}}` or 401.
  - `POST /auth/logout` → delete session.
  - CSRF: reject state-changing `/auth/*` requests whose `Origin` isn't the site origin.
- **Proxy:** everything that is not a static client asset, a client route (`/`, `/login`, `/signup`, `/verify-email`, `/forgot-password`, `/reset-password`, `/account/*`, `/app*`) or `/auth/*` requires a session and is piped to the user's workspace (plain `http`/`net`, no new dep; handle WS upgrade; strip the cookie header before forwarding). No session → 401 JSON for XHR, redirect `/login` for navigations. Workspace `stopped` → start it first.
- **Provisioner** (root): `useradd -m -d /home/th-<8 hex> -s /bin/bash -K UID_MIN=20000 -K UID_MAX=29999`, chmod 700 home, write env file, `systemctl enable --now termhive-ws@<user>`, wait for `GET /api/daemon/status`, add the per-user nft rules (§3). Idempotent; errors → workspace `error` + audit log.
- **Admin:** `ADMIN_EMAILS=xxmasong@gmail.com` → role admin; the first admin sign-in is linked to the **existing root workspace** (`unix_user=root`, `port_base=4000`) instead of provisioning a new one. CLI `/usr/local/bin/termhive-admin` (`users`, `set-plan <email> <free|pro|pro-plus>` → rewrites env + restarts that workspace, `invite create [--uses N]`, `suspend <email>`).
- Units: `termhive-cloud.service` (root, :3200), template `termhive-ws@.service` (`User=%i`, `EnvironmentFile=/etc/termhive/ws/%i.env`, `WorkingDirectory=/opt/termhive-v2`, `ExecStart=/usr/local/bin/npm run start:all`, `Restart=always`, `NoNewPrivileges=yes`, `ProtectSystem=full`). Keep `/opt/termhive-v2` root-owned, world-readable.

## 3. Isolation hardening (CT102)
- `nft` table `inet termhive`, output hook: sockets with `meta skuid 20000-29999` → drop to `10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 100.64.0.0/10 169.254.0.0/16`; on `lo`, drop tcp to `3210, 3300, 4000-4999` except the user's own three ports (per-user rule added by the provisioner). Persist in `/etc/nftables.d/termhive.nft` + load at boot.
- `/root`, `/etc/termhive`, `/var/lib/termhive-cloud` = 0700 root. Try `/proc` `hidepid=invisible` (skip if the CT forbids it; note it).
- Verify as a test user: cannot read `/home/<other>`, cannot `curl` another workspace port, cannot reach `10.10.0.1`, can reach `https://api.anthropic.com`.

## 4. Workspace backend changes (`src/`, contract-compatible)
- `storage.createProject` / `storage.createAgent`: enforce `TERMHIVE_MAX_PROJECTS` / `TERMHIVE_MAX_AGENTS` (unset = unlimited; agents counted across all projects) → `PlanLimitError`. `routes.ts` → `403 {error, code:'PLAN_LIMIT', kind:'project'|'agent', limit, used}`; `hive.ts` Keeper dispatch returns the same message as a tool error.
- `TERMHIVE_CONFINE_HOME=1`: project/agent `cwd` must resolve inside `$HOME` → `400`.

## 5. Client
- `PLANS`: add dummy prices (`$12/mo`, `$29/mo`, comment `// placeholder prices`); pricing cards show them plus a small chip `No charge during early access`.
- Auth pages switch to **Firebase JS SDK** (`firebase` npm dep, modular, imported only in the auth chunk; config from `GET /auth/config`; show a clear notice if `configured:false`):
  - Login: `signInWithEmailAndPassword` → unverified → `/verify-email`; else `getIdToken()` → `POST /auth/session` → `/app`.
  - Signup: `createUserWithEmailAndPassword` → `updateProfile(name)` → `sendEmailVerification` → `/verify-email`. Keep `plan` + `invite` in `sessionStorage` until the first session exchange.
  - Google / GitHub: `signInWithPopup` → `POST /auth/session {idToken, plan, inviteCode}` → `/app`.
  - Verify page: resend via `sendEmailVerification(currentUser)` (60 s cooldown) + button `I've verified my email` (`reload` → session exchange).
  - Forgot: `sendPasswordResetEmail`. New client route `/account/action` handles Firebase action links (`mode=verifyEmail` → `applyActionCode`; `mode=resetPassword` → the reset form with `confirmPasswordReset`). `/reset-password` redirects there.
  - Map Firebase error codes to the brief's messages (`auth/invalid-credential`, `auth/email-already-in-use`, `auth/weak-password`, `auth/too-many-requests`, `auth/popup-closed-by-user` silently).
  - Update `docs/AUTH_UI_BRIEF.md` API section to this flow.
- `/app`: `WorkspaceRoute` first `GET /auth/me` → 401 → `/login`; `provisioning` → full-screen "Preparing your workspace…" (poll 2 s); then the shell. Landing nav uses `/auth/me`.
- Workspace header: user menu (avatar/name, plan badge, `Sign out` → `POST /auth/logout` → `/`).
- `PLAN_LIMIT` 403 on create project/agent → upgrade dialog: `You've reached your plan's limit` / `<Plan> includes <n> <projects|agents>. Upgrade to add more.` + `See plans` (→ `/#pricing`) + `Close`.

## 6. Deploy (CT102) — in this order
1. Grow disk (`ssh sg1-proxmox pct resize 102 rootfs +30G`). ⚠️ A second root operator works on SG: check `who`, never `pct start` a stopped CT.
2. Ship code: CT101 `product` branch → git bundle → CT102 `/opt/termhive-v2` (fast-forward from `feat/in-browser-cli-signin`; CT102's untracked `CLAUDE.md` stays), `npm ci`, `npm run build` there.
3. Check no agents are mid-task (`/api/daemon/status`, agent statuses); the move restarts the root workspace.
4. `termhive2.service`: `PORT=4000 TERMHIVE_DAEMON_PORT=4001 CLAUDE_BRIDGE_PORT=4002`. Start `termhive-cloud.service` on :3200. tailscale serve unchanged.
5. `/etc/termhive/cloud.env` with `ADMIN_EMAILS`, `SIGNUP_MODE=open`, and the Firebase keys once the owner supplies them (§7). Until then `/auth/config` says not configured.
6. Smoke test: landing loads; `/app` without a session → `/login`; with a test session (seed one via `termhive-admin` test helper or a fake-token dev flag that is **off** in production) the proxy reaches the workspace incl. WebSocket terminals; plan limit returns 403; isolation checks from §3.

## 7. Owner steps (can't be automated — list them in the final report)
1. Firebase console → new project → Authentication → enable **Email/Password**, **Google**, **GitHub**.
2. GitHub → Settings → Developer settings → OAuth App, callback `https://<project-id>.firebaseapp.com/__/auth/handler`; paste client id/secret into Firebase's GitHub provider.
3. Authentication → Settings → Authorized domains → add `sg1-termhive2.tailfa2e0b.ts.net`.
4. Templates → set the action URL to `https://sg1-termhive2.tailfa2e0b.ts.net/account/action`.
5. Project settings → Web app → send the config (apiKey, authDomain, projectId, appId) → goes into `/etc/termhive/cloud.env`.

## Rules
PLAN.md conventions for all client code. Fine-grained commits, one purpose
each, on branch `product` in CT101 `/opt/termhive-v2`. tsc, lint, build,
prettier clean at every commit. Don't push.
