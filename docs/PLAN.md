# TermHive v2 — Frontend Rebuild Plan

**Repo:** https://github.com/xxmasong/termhive-v2 (owner: xxmasong)
**Target host:** SG CT102 `sg1-termhive2` @ 10.10.1.12
**Reference impl:** SG CT226 `sg1-termhive` @ /opt/termhive

## Scope

Rebuild the **frontend only**. Backend (`src/`) is copied verbatim from the
reference and MUST NOT be modified. The React client is rebuilt from scratch
under atomic design + feature-based architecture.

The reference client is a 720-line `App.tsx` god component holding ~25 pieces of
state, plus 14 flat components. That is what we are replacing.

## Non-negotiable constraints

1. **Backend is frozen.** `src/**` copied as-is. Every REST route, WS message
   type, and payload shape must be honored exactly. Do not "improve" the API.
2. **Wire contract is law.** See CONTRACT.md — generated from the reference
   `src/types.ts` and `src/daemon/protocol.ts`. Client types must match.
3. **No backend behavior change.** If the client needs data the API doesn't
   expose, derive it client-side. Do not add endpoints.

## Stack

- React 18 + TypeScript (strict), Vite
- **Recoil** for client/UI state (atoms + selectors)
- **TanStack Query** for all server state (REST). No server data in Recoil.
- **TanStack Virtual** for long lists (activity feed, messages)
- **TanStack Table** where tabular (usage/org views)
- xterm.js (`@xterm/xterm` + `@xterm/addon-fit`) for terminals

### State ownership rule (SOLID/SRP — enforce strictly)
| Kind | Owner |
|---|---|
| REST resources (projects, agents, content, wiki) | TanStack Query |
| WS-pushed live data (terminal output, codex items, brain events) | Recoil atoms, fed by a single WS hook |
| UI/ephemeral (selection, tab, layout, modals, theme) | Recoil atoms (persisted subset to localStorage) |

WS events that invalidate server state call `queryClient.invalidateQueries` —
they do not duplicate it into Recoil.

## Folder structure

```
client/src/
  app/                      # composition root only
    App.tsx                 # <50 lines: providers + shell
    providers/              # RecoilRoot, QueryClientProvider, ThemeProvider
    routes/
  components/               # ATOMIC DESIGN — generic, feature-agnostic
    atoms/                  # Button, Icon, Badge, Spinner, Input, Kbd
    molecules/              # FormField, Toolbar, ListItem, Modal, Tooltip
    organisms/              # Sidebar shell, TabBar, CommandPaletteShell
    templates/              # AppShell, SplitPane, GridLayout
  features/                 # FEATURE-BASED — vertical slices
    projects/
      api/                  # query fns + queryKeys
      hooks/                # useProjects, useCreateProject (business logic)
      state/                # recoil atoms/selectors
      components/           # ProjectList, ProjectCard (feature-specific)
      constants.ts
      types.ts
      index.ts              # public surface — other features import ONLY this
    agents/
    terminal/
    codex/
    brain/                  # orchestrator "Keeper"
    messages/
    content/
    wiki/
    activity/
    voice/
    settings/
  lib/
    api/                    # fetch client, error types, base config
    ws/                     # useWebSocket — single connection, typed
    hooks/                  # cross-cutting: useLocalStorage, useDebounce
    utils/
  constants/                # app-wide: routes, ws message types, storage keys
  types/                    # shared/wire types mirroring backend
  styles/
```

### Rules
- **Atomic components know nothing about the domain.** No `Agent`/`Project`
  types under `components/`. If it needs domain types, it belongs in `features/`.
- **Features never deep-import each other.** `features/x/index.ts` is the only
  public surface. No `../agents/components/Foo`.
- **Business logic lives in hooks**, never in components. A component renders
  and wires handlers; it does not fetch, transform, or decide.
- `React.FC<Props>` for every component, explicit props interfaces.
- **No magic values.** Strings/numbers/keys go in `constants.ts`.
- `useMemo` for derived data, `useCallback` for every handler passed as a prop,
  `useEffect` only for real side effects (subscriptions, imperative DOM).
  Do not use `useEffect` to derive state — compute or use a selector.

## Naming
- Components/types `PascalCase`; hooks `useCamelCase`; constants
  `SCREAMING_SNAKE`; files match default export; `queryKeys` factory per feature.

## Phases

**P0 — Scaffold.** Vite+TS strict, deps, folders, path aliases (`@/`), ESLint
+ Prettier, `lib/api` client, `constants/`, `types/` from CONTRACT.md. Backend
copied verbatim. Builds clean, `tsc --noEmit` passes.

**P1 — Design system.** atoms → molecules → organisms → templates. Port the
reference CSS variables/themes (dark/light/amber/mono) into `styles/`. No
feature logic.

**P2 — Projects + Agents.** Query hooks, Recoil selection atoms, sidebar,
create/delete modals, agent lifecycle (start/stop/restart). First vertical slice
proving the pattern.

**P3 — WS + Terminal.** Single typed WS hook w/ reconnect + backoff. xterm
attach/detach/input/resize. Terminal buffer in Recoil keyed by agentId.
Layout modes (single/2up/3up/grid/canvas).

**P4 — Codex + Brain.** Structured `CodexItem` card stream; brain conversation
panel w/ streaming `BrainEvent`, conversation switcher, abort.

**P5 — Remaining features.** messages, shared content, wiki, activity feed
(virtualized), notifications, command palette, settings, voice.

**P6 — Polish.** Error boundaries per feature, loading/empty states,
accessibility, responsive.

## Definition of done (each phase)
- `npx tsc --noEmit` clean, `npm run build` clean, no `any` (except narrow
  documented escapes), no ESLint errors.
- No feature deep-imports; no domain types under `components/`.
- Commit on a branch, push to origin.

## Division of labor
- **Codex writes 100% of the code** (via MCP delegate, persistent thread).
- **Claude**: plans, reviews diffs, runs builds/tests, reports bugs back to
  Codex, verifies fixes. Claude does not write feature code.
