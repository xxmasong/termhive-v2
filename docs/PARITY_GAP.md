# v1 → v2 Parity Gap (P7)

Derived by diffing `reference/client/src` (v1, frozen) against `client/src` (v2),
plus side-by-side screenshots of both running apps at the same viewport.

The backend is frozen and already serves everything below. **No backend work.**

## The headline problem

v2 renders a *bare xterm rectangle*. v1 renders an **agent pane**: avatar,
name, role, cli · cwd, status chip, and per-pane start/stop/restart/delete.
That single missing component is most of the visual delta.

`reference/.../AgentGrid.tsx` is 947 lines; `features/terminal/.../AgentTerminal.tsx`
is 265. The difference is pane chrome + layout engine, not terminal I/O
(terminal I/O is correct and must not be touched).

---

## G1 — Agent pane chrome (highest impact)

Port `AgentPane` from `reference/client/src/components/AgentGrid.tsx:110-200`.

Each pane needs a header with:

| Element | Source |
| --- | --- |
| Avatar (monogram + deterministic hue) | `utils/agentIdentity.ts` — **not ported at all** |
| Name + role pill | `agent.name`, `agent.role` |
| `cli · cwd` subtitle | `agent.cli`, `agent.cwd` |
| Status chip + pulsing dot | `statusLabel()` at `AgentGrid.tsx:26` |
| Stop / Start (conditional), Restart, Delete | existing `useAgentLifecycle` |

Port `utils/agentIdentity.ts` verbatim to `features/agents/utils/agentIdentity.ts`.
It is pure, dependency-free, and already matches our `--h-*` tokens.

**Alive rule** (v1 `AgentGrid.tsx:148`) — an agent is alive for *any* non-stopped
status, i.e. `status !== 'stopped'`. Do not test `=== 'running'`; `awaiting_input`
and `idle` are alive and must still show Stop.

The pane header wraps **both** `AgentTerminal` and `CodexAgentView` — in v1 the
Codex pane has the same chrome. Right now only the raw Codex view renders.

## G2 — Tab bar actions

v1 `App.tsx:562-587`, missing entirely in v2:

- `Start all` — disabled when `stoppedCount === 0`
- `Stop all` — disabled when `aliveCount === 0`
- `New agent` (primary) — always visible
- Both bulk buttons render **only** on the Terminals tab.

v2 has a stray `Command` button in that slot; v1 has none there (the palette
lives in the header). Remove it.

## G3 — Terminals tab count badge

v1 `App.tsx:536`: `<span className="count">{projectAgents.length}</span>`.
v2 shows a badge on Messages but not Terminals. v1 shows it on Terminals only.

## G4 — "Shared" vs "Content"

v1's tab is labelled **Shared** with a folder icon and renders a file **tree**
(`SharedContent.tsx`, 307 lines — collapsible dirs, per-extension colouring).
v2 labels it "Content" with a file icon. Match v1: label, icon, and the tree.

## G5 — Sidebar agent rows

v1 `Sidebar.tsx` rows carry a status dot, the `CLAUDE`/`CODEX` badge, the role
subtitle, and the `Ctrl1..5` hint. v2 has the badge and hint but the status dot
does not reflect live status, and the row is not click-to-focus-pane.

## G6 — Layout engine

v1 has five modes — `single | 2up | 3up | grid | canvas` — with drag-to-reorder,
drag-to-swap, resize dividers, and per-project persisted canvas positions.
v2's `GridLayout.tsx` (758 lines) covers the modes but not drag/resize.

**Lowest priority.** Do G1–G5 first; they are what the screenshot shows.

---

## Rules

- `src/**` and `reference/**` stay frozen and read-only.
- Terminal WebSocket I/O in `AgentTerminal.tsx` is correct — wrap it, don't rewrite it.
- Keep the architecture: atomic design, feature slices, `React.FC`, extracted
  constants, business logic in hooks, Recoil for UI/WS state, TanStack Query for REST.
- Reuse existing atoms/molecules; do not introduce a second button or badge.
- `tsc --noEmit` and `eslint` clean before handing back.
