# Codex Working Brief — TermHive v2

You are the sole implementer. Claude plans, reviews, builds, tests, and reports
bugs; Claude does not write feature code. Everything in `client/src/**` is yours.

## Where things are (on CT102, you are root)
```
/opt/termhive-v2/            # git repo, remote: github.com/xxmasong/termhive-v2
  src/**                     # BACKEND — FROZEN, copied from reference. DO NOT EDIT.
  client/**                  # YOUR WORK
  docs/PLAN.md               # architecture + phases
  docs/CONTRACT.md           # frozen API/WS contract — authoritative
  reference/                 # read-only copy of the OLD client, for behavior only
```

`reference/` is there so you can see *what the UI does*. Do not copy its
structure — its `App.tsx` is a 720-line god component; that is the thing we are
replacing. Copy **behavior and visual design**, not architecture.

## Hard rules
1. **Never edit `src/**` (backend) or `reference/**`.**
2. **CONTRACT.md is law.** Exact routes, payloads, WS message types. If the API
   doesn't expose something, derive it client-side — never add an endpoint.
3. **TanStack Query owns all REST state. Recoil owns UI + WS-pushed state.**
   Never store fetched server resources in Recoil.
4. **Business logic lives in hooks.** Components render and wire handlers only —
   no fetching, no transforms, no decisions in component bodies.
5. **Atomic components are domain-blind.** No `Agent`/`Project`/`BrainState` type
   may appear under `components/`. Those belong in `features/`.
6. **Features expose `index.ts` only.** No deep cross-feature imports.
7. **No magic values.** Literals go in a `constants.ts`.
8. `React.FC<Props>` with an explicit props interface for every component.
9. `useMemo` for derived values; `useCallback` for handlers passed as props;
   `useEffect` ONLY for real side effects. Never `useEffect` to derive state.
10. TypeScript strict. No `any` unless narrowly scoped with a comment.
11. **Terminal output must never go through React state per-chunk** — write ANSI
    chunks straight into the xterm instance.

## Definition of done for every phase
- `npx tsc --noEmit` clean
- `npm run build` clean
- no ESLint errors
- no deep cross-feature imports, no domain types under `components/`
- commit with a clear message and push to `origin`

## Workflow
Work **one phase at a time**. At the end of a phase: run the checks above, commit,
push, then STOP and report:
- what you built (files//dirs)
- the verification output
- anything in PLAN.md that proved wrong or ambiguous

Do not start the next phase until Claude confirms. Claude will send you build
failures and bug reports — fix them and re-verify.

Keep commits scoped. Do not reformat files you did not otherwise change.

## Phase order
P0 scaffold -> P1 design system -> P2 projects+agents -> P3 WS+terminal ->
P4 codex+brain -> P5 remaining features -> P6 polish.
(Full detail in docs/PLAN.md.)
