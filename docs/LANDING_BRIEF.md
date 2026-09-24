# Landing Page Brief — TermHive

Design, theme and copy are final (Claude). Codex implements in React, strictly
following `docs/PLAN.md` + `docs/CODEX_BRIEF.md` conventions. Copy below is
**verbatim** — put it in `constants.ts`, do not rewrite it.

---

## 1. Positioning (context for the implementer — not page copy)

TermHive is a **vendor-neutral control room for AI coding agents**. It does not
compete with Claude Code / Codex / Gemini CLI / OpenCode — it runs them, side by
side, as a team:

- Real CLI sessions (full PTY, xterm in the browser), kept alive by a daemon when
  the browser closes; scrollback replays on reconnect.
- Mix vendors in one project. Agents message each other over MCP
  (`message_agent`, `list_teammates`).
- **The Keeper** — an orchestrator agent you chat with; it can list/create
  projects and agents, start/stop them, `ask_agent` and `broadcast`.
- Shared files + a per-project wiki (persistent memory every agent reads).
- Runs on the subscriptions users already pay for (interactive CLI sessions, not
  metered API calls). In-browser sign-in for each CLI.
- Layouts: single / 2-up / 3-up / grid / free canvas, drag-and-drop panes.
  Structured Codex view (commands, diffs, tool calls as cards). Voice. Per-CLI
  usage meters. Activity feed. Installable PWA — works on a phone.

Market map we position against:

| Category | Examples | Their limit |
|---|---|---|
| Single-agent CLIs | Claude Code, Codex CLI, Gemini CLI, OpenCode | One agent, one vendor, one terminal |
| AI IDEs | Cursor, Windsurf, Copilot in VS Code | Desktop-bound; agents live inside one editor |
| Cloud autonomous agents | Devin, Copilot coding agent, Codex cloud | Black-box runs, metered, one vendor's model |
| Parallel-agent runners | Conductor, Claude Squad, Vibe Kanban | Parallel, but agents don't talk and there's no orchestrator |

**What makes us stand out:** cross-vendor *teams* that talk to each other,
directed by an orchestrator, on your existing subscriptions, from any device.

---

## 2. Visual direction

**"Night shift in the hive."** Dark, terminal-native, editorial. Calm and
precise — not neon, no purple-blue AI gradients, no stock 3D blobs. One warm
accent (honey) against cool graphite; the existing app blue is secondary.
Everything should look like it could be a screenshot of the real product.

### Tokens — scope all of these under `.landing` in `features/landing/styles.css`

```css
.landing {
  --l-bg: #0d0d0c;            /* page */
  --l-bg-raised: #151514;     /* cards */
  --l-bg-sunken: #0a0a09;     /* terminal bodies */
  --l-line: rgba(255,255,255,.07);
  --l-line-strong: rgba(255,255,255,.13);
  --l-text: #f2efe8;          /* warm white */
  --l-text-2: rgba(242,239,232,.66);
  --l-text-3: rgba(242,239,232,.42);
  --l-honey: #f2b544;         /* primary accent */
  --l-honey-soft: rgba(242,181,68,.12);
  --l-honey-glow: rgba(242,181,68,.28);
  --l-blue: #529cca;          /* = app --accent, secondary */
  /* per-CLI identity colors (pane dots, chips, message lines) */
  --l-claude: #d97757;
  --l-codex: #e8e6e3;
  --l-gemini: #7aa7ff;
  --l-opencode: #8fd18f;
  --l-radius: 14px;
  --l-radius-sm: 8px;
  --l-max: 1200px;
  --l-ff-display: 'Inter Tight', 'Inter', system-ui, sans-serif;
  --l-ff-sans: var(--ff-sans);
  --l-ff-mono: var(--ff-mono);
}
[data-theme='light'] .landing {
  --l-bg: #f7f5f0; --l-bg-raised: #ffffff; --l-bg-sunken: #16161a;
  --l-line: rgba(20,18,12,.08); --l-line-strong: rgba(20,18,12,.14);
  --l-text: #16140f; --l-text-2: rgba(22,20,15,.68); --l-text-3: rgba(22,20,15,.45);
  --l-honey: #c98a12; --l-honey-soft: rgba(201,138,18,.10); --l-honey-glow: rgba(201,138,18,.22);
}
```
Terminal mockups stay dark in light mode (`--l-bg-sunken` is dark in both; text
inside mockups uses fixed light colors).

Fonts: add to `client/index.html` one Google Fonts stylesheet link for
`Inter:wght@400;500;600`, `Inter+Tight:wght@500;600;700`,
`JetBrains+Mono:wght@400;500` with `display=swap`, plus preconnect to
`fonts.googleapis.com` and `fonts.gstatic.com`.

### Type scale
- Display H1: `--l-ff-display`, 600, `clamp(2.6rem, 6vw, 4.75rem)`, line-height 1.02, letter-spacing -0.035em, `text-wrap: balance`.
- H2: display, 600, `clamp(2rem, 4vw, 3rem)`, lh 1.08, ls -0.03em.
- H3: sans 600, 1.125rem.
- Body: 1.0625rem / 1.6, `--l-text-2`, max 62ch.
- Eyebrow: mono 500, 0.75rem, uppercase, ls 0.12em, `--l-honey`, preceded by a 6px honey square.
- Mono text in mockups: 12.5px / 1.55.

### Surfaces & motifs
- Background: `--l-bg` + a very faint hex grid (inline SVG pattern, stroke `--l-line`, ~28px hexes) masked by a radial gradient so it only shows around the hero; plus one soft honey radial glow (`--l-honey-glow`, heavy blur, ~600px) behind the hero visual. No other gradients.
- Cards: `--l-bg-raised`, 1px `--l-line`, radius `--l-radius`; hover: border `--l-line-strong`, translateY(-2px), 180ms ease.
- Section rhythm: `padding-block: clamp(80px, 12vw, 144px)`; container max `--l-max`, side gutter 24px (16px under 480px).
- Buttons: primary = honey bg, `#16140f` text, radius 999px, 44px tall, weight 600, subtle inset top highlight. Secondary = transparent, 1px `--l-line-strong`, `--l-text`. Focus ring 2px `--l-honey`, offset 2px.
- Motion: animate only `transform`/`opacity`. Sections fade-up 12px on first intersection. **All motion disabled under `prefers-reduced-motion: reduce`** (HiveSim shows its final frame, chat mock shows all messages).
- Mobile: works at 360px with no horizontal page scroll. Everything stacks at ≤760px (reuse `MOBILE_BREAKPOINT`).

---

## 3. Page structure & verbatim copy

### 3.0 Nav — sticky, 64px; once scrolled > 8px: `backdrop-filter: blur(12px)` over ~72%-opacity `--l-bg` + bottom 1px `--l-line`
- Left: existing `Icon name="logo"` + wordmark **TermHive**.
- Center anchor links: **Product** (#product) · **The Keeper** (#keeper) · **Compare** (#compare) · **FAQ** (#faq)
- Right: **Sign in** (text link → `ROUTES.LOGIN`) · **Get started** (small primary → `ROUTES.SIGNUP`).
- If `/api/auth/me` returns 200 (signed in, or auth disabled): replace both with one primary **Open workspace →** (→ `ROUTES.APP`).
- ≤760px: links collapse behind a menu button that opens a full-width sheet (Esc closes, focus returns to the button).

### 3.1 Hero (`#top`) — two columns ≥1024px (text 5/12, visual 7/12), stacked below
- Eyebrow: `AGENT ORCHESTRATION, VENDOR-NEUTRAL`
- H1: **Your coding agents,** (line break) **working as one team.**
- Sub: `Run Claude Code, Codex, Gemini and OpenCode side by side in real terminals. They message each other, share a project memory, and take direction from the Keeper — an orchestrator you just talk to.`
- CTAs: **Start your hive** (primary → SIGNUP) · **See how it works** (secondary → #how)
- Micro-line under CTAs (mono, `--l-text-3`): `Uses the subscriptions you already have · Runs in any browser · Keeps working when you close the tab`

**Hero visual — HiveSim** (the centerpiece). A faux TermHive window in pure HTML/CSS/SVG, no images:
- Window chrome: 36px bar, three muted dots, centered mono title `checkout-redesign — 4 agents`, right a small honey chip `● live`.
- Body: 2×2 grid of terminal panes (1px gaps showing `--l-line`). Pane header: CLI color dot + CLI name + role chip + status text (`running` green / `idle` grey / `awaiting input` honey). Body: mono lines.
  - `claude · frontend`, `codex · backend`, `gemini · qa`, `opencode · docs`.
- Scripted looping timeline (typed data in `constants.ts`, driven by `useHiveSimulation` with ONE `setInterval` step clock; reveal whole lines per step, no per-character state). ~14s loop:
  1. codex: `$ pnpm test api/orders` → `✓ 42 passed`
  2. codex → claude message: an SVG dashed path from codex pane to claude pane, stroke `--l-codex`, animated dash-offset; plus a toast inside claude's pane: `✉ from codex: orders API now returns {total, currency}`
  3. claude: `Updating CheckoutSummary.tsx…` → `+18 −6  CheckoutSummary.tsx`
  4. gemini: `Running e2e: checkout flow` → `✓ 9/9 scenarios`
  5. opencode: `Writing docs/checkout.md` → `✓ wiki updated`
  6. Keeper HUD (floating bubble bottom-right of the window, honey ring avatar): `All four agents done. PR ready: 3 files, tests green.`
  Then hold 2s and reset.
- Status dot pulses only while that pane is `running`.
- On mobile keep 2×2 with 11px mono — never taller than 420px.

### 3.2 Works-with strip
Label (mono, `--l-text-3`): `Bring the agents you already use`
Four text wordmarks in their CLI colors, thin dividers between:
**Claude Code** · **Codex** · **Gemini CLI** · **OpenCode**  (text only, no third-party logos)

### 3.3 Problem → solution (`#product`)
- Eyebrow: `THE PROBLEM`
- H2: **Four terminals. Four tabs. Zero coordination.**
- Body: `Every coding agent is brilliant alone and oblivious together. You copy output from one into another, lose track of who's doing what, and babysit sessions that die the moment your laptop sleeps.`
- Visual: four scattered, slightly rotated grey terminal cards with red `session ended` badges; on first intersection they transition into a neat 2×2 grid with green dots and the badges fade out (single CSS transition). Caption: `After: one hive.`

### 3.4 Feature bento (`#features`)
- Eyebrow: `WHAT YOU GET`
- H2: **A control room, not another chat box.**
- 6 cells. Desktop 3 columns with cells 1 and 4 spanning 2; tablet 2 columns; mobile 1. Each: small line icon, H3, body, tiny mono illustration.

| # | H3 | Body | Mini-illustration |
|---|---|---|---|
| 1 (wide) | **Real terminals, not wrappers** | `Every agent is the actual CLI in a full PTY — slash commands, plans, permissions, all of it. Sessions live in a daemon, so closing the browser never kills a run.` | mono lines: `$ claude` / `> /plan` / `● session restored · 2,418 lines` |
| 2 | **Agents that talk** | `Built-in messaging lets Claude ask Codex, Codex brief Gemini, anyone ping the team.` | two chips with an arrow: `codex → claude` |
| 3 | **Shared memory** | `A project wiki and shared files every agent reads and writes. Context survives restarts and handoffs.` | file list: `wiki/architecture.md`, `wiki/decisions.md`, `shared/api-contract.json` |
| 4 (wide) | **Any layout, any device** | `Single, split, grid or a free canvas — drag panes where you want them. Install it on your phone and check on the hive from the train.` | 5 tiny layout glyphs (single/2up/3up/grid/canvas) + a phone outline |
| 5 | **Pay nothing extra** | `TermHive drives your existing Claude, ChatGPT and Google plans through their own CLIs. No per-token markup.` | mini usage bars: `claude 34%`, `codex 12%`, `gemini 5%` |
| 6 | **See every move** | `Structured Codex view, a live activity feed and per-CLI usage meters — know what ran, what changed and what's left.` | 3 feed rows: `● codex ran pnpm test`, `● claude edited 2 files`, `● gemini → qa passed` |

### 3.5 How it works (`#how`)
- Eyebrow: `HOW IT WORKS`
- H2: **From zero to a working hive in three steps.**
- Three steps in a row joined by a thin honey line (vertical on mobile). Number in mono inside a 32px honey ring.
  1. **Connect your CLIs** — `Sign in to Claude, Codex, Gemini or OpenCode right in the browser. Your credentials stay in your own workspace.`
  2. **Assemble a team** — `Create a project, add agents, give each a role — frontend, backend, QA, docs. Mix vendors freely.`
  3. **Direct the hive** — `Type into any terminal, broadcast to all of them, or just tell the Keeper what you want done.`

### 3.6 The Keeper spotlight (`#keeper`) — text left, chat mock right
- Eyebrow: `MEET THE KEEPER`
- H2: **One conversation to run them all.**
- Body: `The Keeper is an orchestrator agent that can see your whole hive. Ask it to spin up a team, check on progress, or get an answer from a specific agent — it does the legwork and reports back.`
- Bullets (honey check marks): `Creates projects and agents on request` · `Asks any agent a question and waits for the answer` · `Broadcasts instructions to the whole team` · `Remembers every conversation`
- Chat mock (messages reveal one by one on first intersection):
  - **You:** `Set up a team to add Stripe checkout to the shop repo.`
  - tool row (mono, dim): `▸ create_project  shop-checkout`
  - tool row: `▸ create_agent  claude · frontend    ▸ create_agent  codex · backend`
  - **Keeper:** `Done — two agents are running. Codex is drafting the payments endpoint; Claude will build the form once the contract is in shared files.`
  - **You:** `How's backend doing?`
  - tool row: `▸ ask_agent  codex`
  - **Keeper:** `Codex says: "Endpoint and webhook handler done, 12 tests passing. Waiting on the Stripe test key."`

### 3.7 Comparison (`#compare`)
- Eyebrow: `HOW WE'RE DIFFERENT`
- H2: **Built for teams of agents, not a single one.**
- Sub: `Most tools give you one very good agent. TermHive gives you a coordinated team — and lets you keep the agents you already trust.`
- Columns: **TermHive** (highlighted: 2px honey top border, `--l-honey-soft` background) · **Single-agent CLIs** · **AI IDEs** · **Cloud agents** · **Parallel runners**. Header sub-labels (mono, small): `Claude Code, Codex CLI…` · `Cursor, Windsurf…` · `Devin, Copilot agent…` · `Conductor, Claude Squad…`
- Values: `yes` = ✓ (honey in TermHive column, `--l-text-2` elsewhere), `partial` = ◐ (`--l-text-3`), `no` = — (`--l-text-3`). Each symbol cell has an `aria-label` ("Yes"/"Partially"/"No").

| Row | TermHive | CLIs | IDEs | Cloud | Parallel |
|---|---|---|---|---|---|
| Mix agents from different vendors | yes | no | partial | no | partial |
| Agents message each other | yes | no | no | no | no |
| Orchestrator you can talk to | yes | no | partial | partial | no |
| Uses your existing subscriptions | yes | yes | partial | no | yes |
| Full native CLI, not a re-implementation | yes | yes | no | no | yes |
| Shared project memory across agents | yes | partial | partial | partial | no |
| Keeps running with the browser closed | yes | no | no | yes | partial |
| Works from your phone | yes | no | no | partial | no |

- Footnote (small, `--l-text-3`): `Categories summarize typical products as of 2026. Individual tools vary and change quickly.`
- Mobile: the table scrolls horizontally **inside its own container** with a sticky first column; the page never scrolls sideways.

### 3.8 Who it's for
- Eyebrow: `WHO IT'S FOR`
- H2: **Made for people who ship.**
- 3 cards:
  - **Solo builders** — `Run a whole team by yourself. Let agents parallelize while you review.`
  - **Small teams** — `Give every project its own hive, share the memory, and stop re-explaining context.`
  - **Agent tinkerers** — `Pit Claude against Codex on the same task, or chain them. Every vendor, one screen.`

### 3.9 FAQ (`#faq`) — accessible accordion (`<button aria-expanded aria-controls>`), one open at a time
1. **Do I need API keys?** — `No. TermHive runs each vendor's own CLI, so you sign in with your existing Claude, ChatGPT or Google account. API keys work too if you prefer them.`
2. **Is my code sent to TermHive?** — `Your agents work inside your own isolated workspace. Code goes only where your chosen CLI sends it — the same as running it on your laptop.`
3. **Which agents are supported?** — `Claude Code, OpenAI Codex, Gemini CLI and OpenCode today. More CLIs are on the way.`
4. **What happens when I close the browser?** — `Nothing stops. Agents run in a background daemon; reopen TermHive on any device and every terminal picks up where it left off.`
5. **Can agents break things?** — `Each agent runs with the permissions you choose. You decide per agent whether it asks before acting or runs autonomously.`
6. **What does it cost?** — `TermHive is free during early access. You only pay your AI vendors, through plans you likely already have.`

### 3.10 Final CTA band
Full-width card, `--l-bg-raised`, faint hex pattern + honey glow inside.
- H2: **Put your agents to work — together.**
- Sub: `Create your hive in under a minute.`
- CTAs: **Get started free** (primary → SIGNUP) · **Sign in** (secondary → LOGIN)

### 3.11 Footer
- Left: logo + `TermHive` + `© 2026 TermHive. Built for the multi-agent era.`
- Link columns — **Product**: Features (#features), The Keeper (#keeper), Compare (#compare) · **Resources**: FAQ (#faq), GitHub (`https://github.com/xxmasong/termhive-v2`) · **Account**: Sign in (LOGIN), Get started (SIGNUP).
- Small dark/light theme toggle reusing the existing theme mechanism (ThemeBootstrap / its storage key).

### SEO / head
- Title on `/`: `TermHive — your coding agents, working as one team`
- Meta description (in `index.html`, replacing the current one): `Run Claude Code, Codex, Gemini and OpenCode side by side. Agents that message each other, share memory, and take direction from an orchestrator.`
- `useDocumentTitle` sets the title while the landing is mounted and restores it on unmount.

---

## 4. Engineering spec (strict)

### Routing (no new dependencies)
- `client/src/constants/routes.ts`: `export const ROUTES = { LANDING: '/', LOGIN: '/login', SIGNUP: '/signup', APP: '/app' } as const;` (export from `constants/index.ts`).
- `client/src/lib/hooks/usePathname.ts`: subscribes to `popstate`, returns `{ pathname, navigate }` (`navigate` = `history.pushState` + dispatch a `popstate`). Typed, tiny.
- `client/src/app/routes/AppRouter.tsx`:
  - `/` → `LandingPage`
  - `/app` and `/app/*` → the existing `AuthGate` → `AppProviders` → `TermHiveShell` tree, moved verbatim out of `App.tsx` into `app/routes/WorkspaceRoute.tsx`, loaded with `React.lazy` + `Suspense` (fallback = existing `Spinner` centered) so the landing bundle does not pull in xterm / recoil / tanstack.
  - `/login`, `/signup` → temporary: `location.replace(ROUTES.APP)` (real auth pages are the next phase).
  - anything else → `LandingPage`.
- `App.tsx` becomes `ThemeBootstrap` + `AppRouter`, < 50 lines.
- `client/public/manifest.webmanifest`: `start_url` → `/app`.
- `src/auth.ts`: the post-login / logout `res.redirect('/')` calls → `res.redirect('/app')`. **This is the only backend edit allowed.**

### Feature slice `client/src/features/landing/`
```
landing/
  components/  LandingPage, LandingNav, MobileNavSheet, Hero, HiveSim,
               HiveSimPane, KeeperHud, WorksWith, ProblemSection, FeatureBento,
               FeatureCard, HowItWorks, KeeperSpotlight, ChatMock,
               ComparisonTable, AudienceCards, Faq, FaqItem, FinalCta,
               LandingFooter, Section (eyebrow + title wrapper), HexPattern,
               ThemeToggle
  hooks/       useHiveSimulation, useInView, useScrolled, useFaqAccordion,
               useSessionStatus (GET /api/auth/me once → 'signed-in' |
               'signed-out' | 'unknown'), useDocumentTitle, useReducedMotion
  constants.ts ALL copy above, nav links, sim timeline, comparison matrix,
               faq entries, timings (ms), thresholds
  types.ts     SimPane, SimStep, ComparisonRow, ComparisonValue, FaqEntry, …
  styles.css   all landing CSS, `landing-*` class names, tokens above
  index.ts     exports LandingPage only
```
- `LandingPage.tsx` imports the landing `styles.css` itself (NOT global.css) so it is code-split with the route.
- Data-driven: bento, steps, comparison, audience, FAQ, sim all `.map()` over typed constants. No copy strings inline in JSX.
- Reuse `Icon` / `Button` / `Spinner` from `@/components` where they fit; landing-only visuals stay in the feature.
- All PLAN.md rules: `React.FC<Props>` with explicit interface, `useCallback` for handlers passed as props, `useMemo` for derived data, `useEffect` only for real side effects (interval clock, IntersectionObserver, scroll/popstate listeners, fetch). Strict TS, no `any`, no magic values.
- Accessibility: `header/nav/main/section/footer` landmarks, exactly one `h1`, ordered headings, skip link to `#main`, decorative mockups `aria-hidden="true"` with a visually hidden text summary, body text ≥ 4.5:1 contrast, full keyboard reachability.

### Definition of done
- `npx tsc --noEmit`, `npm run lint`, `npm run build` all clean.
- `/` shows the landing; `/app` shows the existing workspace exactly as before.
- No horizontal page scroll at 360px; reduced motion honored.

### Commits
Fine-grained, one purpose per commit, e.g.:
1. routing: `ROUTES` + `usePathname` + `AppRouter`/`WorkspaceRoute`, App.tsx slimmed
2. `auth.ts` redirect + manifest `start_url` → `/app`
3. landing: tokens, fonts, `HexPattern`, `Section`, nav + footer
4. landing: hero + HiveSim
5. landing: works-with, problem, bento
6. landing: how-it-works + Keeper spotlight
7. landing: comparison table
8. landing: audience, FAQ, final CTA, SEO/title

Stage explicit paths only. Never `git add -A` / `git add .`. Every commit must typecheck.

---

## 5. Addendum — Pricing (added 2026-09-24)

Plans (authoritative, from the owner). "Agents" = agents that exist across **all**
projects, running or stopped. Deleting one frees a slot.

| Plan | Projects | Agents (total) |
|---|---|---|
| Free | 1 | 3 |
| Pro | up to 3 | up to 10 |
| Pro Plus | unlimited | up to 30 |

Prices for Pro / Pro Plus are **not decided yet**. Model price as
`price: string | null`; `null` renders the chip `Early access` in place of the
price. Free renders `$0` + `forever`.

### 5.1 Nav
Add **Pricing** (#pricing) between Compare and FAQ (desktop nav, mobile sheet,
footer "Product" column).

### 5.2 Pricing section (`#pricing`) — placed after Comparison, before Audience
- Eyebrow: `PRICING`
- H2: **Start free. Grow your hive.**
- Sub: `Every plan gets every CLI, the Keeper, agent messaging, shared memory and mobile access. Plans differ only in how big your hive can get.`
- Three cards in a row (stack on mobile, Pro first on mobile). Equal height.
  Card anatomy top→bottom: plan name (H3) · one-line pitch · price row ·
  two big limit stats · CTA · divider · "Everything included" list.
  - **Free** — pitch `Try a full hive on one project.` · `$0` + `forever` ·
    stats `1` `project` / `3` `agents` · CTA secondary **Start free** → `ROUTES.SIGNUP`
  - **Pro** (highlighted: 1px honey border, honey glow shadow, `Most popular` honey
    chip top-right, raised −8px on desktop) — pitch `For builders running several projects at once.` ·
    price (`null` → `Early access` chip) · stats `3` `projects` / `10` `agents` ·
    CTA primary **Choose Pro** → `ROUTES.SIGNUP + '?plan=pro'`
  - **Pro Plus** — pitch `For teams of agents at full scale.` · price (`null`) ·
    stats `∞` `projects` / `30` `agents` · CTA secondary **Choose Pro Plus** → `ROUTES.SIGNUP + '?plan=pro-plus'`
- Stats: number in `--l-ff-display` 600 2.25rem, label mono 12px `--l-text-3`
  under it; the two stats side by side separated by a 1px `--l-line` divider.
- "Everything included" (same list in every card, honey check icons, 0.925rem):
  `Claude Code, Codex, Gemini CLI & OpenCode` · `The Keeper orchestrator` ·
  `Agent-to-agent messaging` · `Shared wiki & files` · `Works on any device`
- Note under cards (small, `--l-text-3`, centered): `Agents count across all projects, running or stopped. AI usage is billed by your own Claude, ChatGPT or Google plan.`

### 5.3 FAQ changes
- Replace answer of **What does it cost?** with:
  `Free covers 1 project and 3 agents. Pro raises that to 3 projects and 10 agents; Pro Plus gives you unlimited projects and 30 agents. Your AI usage stays on your own Claude, ChatGPT or Google plan.`
- Add after it: **What counts as an agent?** — `Every agent you create, running or stopped, across all your projects. Delete an agent to free its slot.`
- Final CTA sub unchanged.
