# Phase 6 — findings from the reference build setup

Gathered by Claude while Codex was rate-limited. These are the concrete
gotchas in wiring the frozen backend to the new client.

## 1. Client output path must be `dist/client`

`src/server.ts:266` resolves the client as:

```ts
const clientDist = path.join(__dirname, 'client');
```

`tsup` bundles the server to `dist/server.js`, so `__dirname` is `dist/` and the
server serves `dist/client/`. The reference `vite.config.ts` matches that with
`root: 'client'` + `build.outDir: '../dist/client'`.

Our `client/vite.config.ts` currently sets no `outDir`, so it defaults to
`client/dist` — the server would 404 on every request. **Set
`build.outDir` to resolve to `<repo>/dist/client`** (and keep the existing
`manualChunks` config).

## 2. Build order — `tsup` has `clean: true`

Reference `tsup.config.ts`:

```ts
entry: ['src/server.ts','src/mcp-server.ts','src/hive-mcp-server.ts','src/daemon/daemon.ts'],
format: ['esm'], target: 'node20', outDir: 'dist',
clean: true, sourcemap: true, external: ['node-pty'],
```

`clean: true` wipes `dist/` — including `dist/client/`. So the full build must
run **`tsup` first, then `vite build`** (the reference's `"build": "tsup && vite build"`
does exactly this). Getting this backwards produces a server with no client.

## 3. `node-pty` is external and optional

It's an `optionalDependency` and marked `external` in tsup. If the native build
fails, the **web server should still start** — only real PTY terminals break.
Note the failure honestly rather than working around it.

## 4. Two processes

`termhive.service` (web, serves client + `/api` + `/ws`) and the daemon
(owns PTYs, `127.0.0.1:3210`). `start:all` runs both via `concurrently`.
The web server proxies to the daemon; the client only ever talks to the web server.

## 5. Runtime checks worth doing

- `GET /api/projects` returns JSON (not the SPA fallback).
- The SPA catch-all (`app.get('*')`) doesn't swallow `/api` routes.
- WebSocket upgrade on `/ws` succeeds — the reference has a known trap where a
  plain `curl` returns a misleading `400`; look for HTTP `101`.
- Creating a project then an agent through the real API, and the client listing them.
