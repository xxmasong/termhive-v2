# Multi-tenant TermHive — Plan (DRAFT, not approved for build)

Goal: anyone can sign up at the landing page and get **their own complete,
isolated TermHive environment**: their own projects, agents, wiki, Keeper
history, and their own Claude/Codex/Gemini/OpenCode logins.

## 1. Where we are today (single-tenant)

| Concern | Today |
|---|---|
| Data | JSON files under `os.homedir()/.termhive/` (projects, shared_content, wiki, brain, mcp-configs, hook-configs, codex-history, voice). No database. |
| CLI credentials | `~/.claude`, `~/.codex`, `~/.gemini`, `~/.local/share/opencode` of the service user (root). |
| Processes | one web server (:3200) + one daemon (:3210) owning every PTY, running as root. |
| Auth | Google OAuth + email allowlist, stateless HMAC cookie; `/api/auth/me`. |
| Hardcoded infra | `DEFAULT_PUBLIC_BASE_URL` = tailnet host in `src/auth.ts`; ports 3200/3210/3300. |

Every path is derived from `os.homedir()`. That is the key fact for the design.

## 2. Why we can't just add a `userId` column

Agents are real shells with full permissions. If two users share one process /
one Unix user, user A's agent can `cat` user B's `~/.claude/.credentials.json`,
read B's repos, or kill B's PTYs. Row-level scoping in code is not isolation.
Tenancy must be enforced by the OS.

## 3. Recommended architecture: control plane + one container per user

```
            internet / CF tunnel
                    │
        ┌───────────▼────────────┐
        │  termhive-cloud        │  NEW service (control plane)
        │  • landing, /login,    │  • Postgres: users, identities,
        │    /signup (static)    │    sessions, workspaces, audit
        │  • auth + sessions     │  • provisions / wakes containers
        │  • reverse proxy /app, │  • proxies HTTP + WebSocket to the
        │    /api, /ws  ─────────┼──► caller's own workspace
        └───────────┬────────────┘
                    │ internal network only
   ┌────────────────┼────────────────┐
   ▼                ▼                ▼
 ws-<userA>       ws-<userB>       ws-<userC>     one container per user
 server+daemon    server+daemon    server+daemon  = today's code, nearly unchanged
 HOME=/home/th    HOME=/home/th    HOME=/home/th  non-root uid, cpu/mem/disk limits
   │                │                │
 /srv/termhive/workspaces/<userId>/home   ← per-user volume
```

Why this shape:
- **The existing backend barely changes.** Because everything keys off `HOME`,
  giving each user their own container + volume isolates projects, wiki,
  Keeper state and CLI logins for free. The frozen client↔server contract
  (`docs/CONTRACT.md`) stays exactly as is; the proxy just routes it.
- Real isolation (namespaces, cgroups, separate uid, separate filesystem).
- Agents keep running with the browser closed — a headline feature — because
  the container stays up independently of the browser.

Rejected: shared process with per-user folders (no isolation); per-user Linux
accounts on one host (workable stopgap, weaker limits, harder cleanup).

Constraint: both Proxmox hosts are nested VPSes without `/dev/kvm`, so no
Firecracker/Kata. Docker/Podman inside an LXC with nesting enabled works (Coolify
already does this on SG).

## 4. Database design (Postgres 17, control plane only)

```sql
users            (id uuid pk, email citext unique, name, avatar_url,
                  password_hash text null,          -- argon2id
                  email_verified_at timestamptz null,
                  role text default 'user',          -- 'user' | 'admin'
                  status text default 'active',      -- active | suspended | waitlist
                  created_at, updated_at)
auth_identities  (id, user_id fk, provider text,     -- google | github
                  provider_uid text, unique(provider, provider_uid))
sessions         (id text pk /* random 256-bit, stored hashed */, user_id fk,
                  created_at, last_seen_at, expires_at, ip inet, user_agent)
email_tokens     (token_hash pk, user_id fk, purpose text, -- verify | reset
                  expires_at, used_at)
workspaces       (id uuid pk, user_id fk unique, state text,
                  -- provisioning | running | sleeping | error | deleted
                  container_id, internal_host, internal_port, volume_path,
                  image_tag, cpu_limit, mem_limit_mb, disk_limit_gb,
                  last_active_at, created_at)
invites          (code_hash pk, created_by, max_uses, uses, expires_at)
audit_log        (id bigserial, user_id, action, detail jsonb, ip, at)
```
Server-side sessions (not the current stateless cookie) so we can revoke,
list devices, and log out everywhere.

Workspace contents (projects, agents, wiki, brain) **stay as files inside the
user's volume** for now. Moving them into Postgres would force rewriting
`src/storage.ts` and the contract for no isolation benefit. Revisit only if we
need cross-user features (team sharing).

## 5. Folder structure

On the workspace host:
```
/srv/termhive/
  workspaces/<userId>/home/          → mounted at /home/th in the container
      .termhive/  (projects, shared_content, wiki, brain, mcp-configs, …)
      .claude/  .codex/  .gemini/  .local/share/opencode/   (their CLI logins)
      code/                          default root for project `cwd`s
  images/                            runtime image build context
```
In the repo:
```
src/                 existing workspace runtime (small changes, §6)
src/cloud/           NEW control plane: server.ts, auth/, db/ (migrations),
                     workspaces/ (provisioner, proxy, lifecycle), email/
client/src/features/auth/     NEW login / signup / verify / reset pages
client/src/features/landing/  done
docker/workspace.Dockerfile   runtime image: node + 4 CLIs + git + build tools
```

## 6. Changes to the existing backend (all internal, contract untouched)

1. `TERMHIVE_AUTH_MODE=proxy`: trust identity only from the control plane via
   a signed header (shared secret), reject everything else. Google OAuth stays
   for single-user self-host mode.
2. Remove the hardcoded tailnet `DEFAULT_PUBLIC_BASE_URL`; env only.
3. Confine project `cwd` to `$HOME` (today any path is accepted).
4. Run as non-root; drop the root-only assumptions.
5. Health endpoint for the provisioner.

## 7. Security must-haves before opening signups

- Egress firewall per workspace: internet yes, **private ranges no**
  (10.10.0.0/16, 100.64.0.0/10 tailnet, Proxmox/PBS, Postgres). Otherwise any
  user's agent can reach our internal services.
- cgroup limits (CPU, RAM, pids) + disk quota per workspace.
- Password hashing argon2id, login rate limiting, CSRF (SameSite=Lax + Origin
  check), email verification, reset tokens hashed + single-use.
- Abuse: agents run arbitrary code on our hardware → invite/waitlist first.

## 8. Capacity reality check

SG host: 12 GB RAM / 6 cores, already ~75 GB assigned. One workspace is roughly
300–500 MB idle plus ~200–400 MB per running agent. Realistically ≈ 5–10 active
users on SG. Idle policy: sleep a workspace only when **no agent is running**
and nobody connected for N hours; wake on login (cold start ~5–15 s).

## 9. Build phases (Codex implements, Claude designs + reviews)

| Phase | Deliverable |
|---|---|
| M0 | Decisions in §10 made; CT101/CT102 checkouts reconciled; landing branch merged |
| M1 | Login / signup / verify / reset UI (Claude designs, same token system as landing) against a mocked API |
| M2 | Control plane: Postgres schema + migrations, email+password + Google (+GitHub), server sessions, invites |
| M3 | Runtime image + workspace changes in §6 |
| M4 | Provisioner (Docker API), HTTP+WS reverse proxy, "preparing your workspace" screen, lifecycle/sleep |
| M5 | Hardening: egress firewall, quotas, rate limits, audit log, admin page |
| M6 | Migrate today's CT102 data into the first admin workspace; deploy; public DNS |

## 10. Decisions needed before M1

1. Isolation model — per-user containers (recommended) vs per-user Linux users.
2. Hosting — which host/CT runs the workspace fleet (SG capacity is tight).
3. Sign-in methods — email+password (needs an SMTP provider), Google, GitHub.
4. Signup policy — open, waitlist, or invite codes (recommended for launch).
5. Public domain — e.g. `termhive.xenitsystems.com` via the SG Cloudflare tunnel.
6. Per-user limits — RAM/CPU/disk per workspace, max concurrent agents.
