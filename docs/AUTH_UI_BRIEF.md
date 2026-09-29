# Auth UI Brief — login / signup (M1)

Design + copy by Claude; Codex implements following `docs/PLAN.md` rules. Same
token system as the landing (`.landing` tokens, fonts, buttons). **Frontend
only** — the endpoints below are implemented by the control plane in M2; pages
must render and validate without a backend (network errors → form error banner).

## Routes (replace the temporary redirects in `AppRouter`)

| Path                          | Page                                 |
| ----------------------------- | ------------------------------------ |
| `/login`                      | LoginPage                            |
| `/signup` (`?plan=free        | pro                                  | pro-plus`) | SignupPage |
| `/verify-email` (`?email=`)   | VerifyEmailPage ("check your inbox") |
| `/forgot-password`            | ForgotPasswordPage                   |
| `/reset-password` (`?token=`) | ResetPasswordPage                    |

All lazy-loaded as one `auth` chunk; no xterm/recoil/tanstack in it.

## Layout — `AuthLayout`

- ≥1024px: split screen. **Left** 520px column, `--l-bg`: logo (→ `/`) top-left
  at 32px, form vertically centered, max-width 380px, footer line bottom
  (`© 2026 TermHive`). **Right** fills the rest: `--l-bg-raised`, hex pattern +
  honey glow, the landing `HiveSim` (export it from `@/features/landing` index)
  centered at ~88% width, and under it the line
  `Four agents. One team. Zero copy-paste.` (display font 600 1.5rem) +
  `Claude Code · Codex · Gemini CLI · OpenCode` (mono 12px `--l-text-3`).
- <1024px: right panel hidden; form centered with 24px gutters.
- Page title via `useDocumentTitle`: `Sign in — TermHive`, `Create account — TermHive`, etc.

## Shared components (feature `auth`)

`AuthLayout`, `AuthCard` (heading + sub + children), `OAuthButtons`
(Google, GitHub — full-width secondary buttons with brand-neutral monochrome
inline SVG marks), `OrDivider` (`or`), `TextField` (label, input, hint, error,
`aria-describedby`), `PasswordField` (show/hide toggle button with aria-label),
`PasswordStrength` (rule checklist), `PlanPicker`, `FormError` (banner, role=alert),
`SubmitButton` (loading spinner, disabled while pending).

Inputs: 44px tall, `--l-bg-sunken`-ish in dark (`rgba(255,255,255,.03)`),
1px `--l-line-strong`, radius 10px; focus: border honey + 3px `--l-honey-soft`
ring. Error: border `oklch(64% 0.19 25)` + message 13px below. Labels 13px 500.

## Pages & copy (verbatim → `constants.ts`)

### Login

- H1 `Welcome back` · sub `Sign in to your hive.`
- OAuth: `Continue with Google` (→ `/auth/google`), `Continue with GitHub` (→ `/auth/github`)
- Divider `or`
- Email (`Email`, autocomplete `email`), Password (`Password`, autocomplete `current-password`), right-aligned link `Forgot password?` → `/forgot-password`
- Submit `Sign in`
- Footer `New to TermHive?` + link `Create an account` → `/signup`
- Errors by `code`: `INVALID_CREDENTIALS` → `That email and password don't match.` ·
  `EMAIL_UNVERIFIED` → `Please verify your email first.` + link `Resend email` (→ `/verify-email?email=`) ·
  `RATE_LIMITED` → `Too many attempts. Try again in a few minutes.` ·
  other/network → `Something went wrong. Please try again.`
- Success → `window.location.assign('/app')`.

### Signup

- H1 `Create your hive` · sub `Free forever on one project. Upgrade anytime.`
- **PlanPicker**: 3-option radio group styled as segmented cards (Free / Pro /
  Pro Plus), each showing name + limits line (`1 project · 3 agents`,
  `3 projects · 10 agents`, `Unlimited projects · 30 agents`). Preselect from
  `?plan=`, default Free. Reuse plan data — move the plan definitions into a
  shared place both features read (`client/src/constants/plans.ts`, exported
  from `@/constants`; landing pricing must use it too).
  Paid choice shows hint under picker: `Paid plans are in early access — we'll email you before any billing starts.`
- OAuth buttons (`Sign up with Google`, `Sign up with GitHub`) — pass `?plan=` and `?invite=` through.
- Sign-ups are invite-only at launch. Show an `Invite code` field (autocomplete off, mono input) above Name when `?invite=` exists, prefilling it from that parameter. If the signup API returns `SIGNUPS_CLOSED`, reveal and focus the field with `Sign-ups are invite-only right now. Enter your invite code.` as its field error.
- Name (`Name`, autocomplete `name`), Email, Password (autocomplete `new-password`) + PasswordStrength
  rules (live ✓/○): `At least 10 characters` · `A letter and a number` · `Not your email`
- Submit `Create account`
- Legal line (12px `--l-text-3`): `By creating an account you agree to the Terms and Privacy Policy.` (plain text for now — pages don't exist yet)
- Footer `Already have an account?` + `Sign in` → `/login`
- Errors: `EMAIL_TAKEN` → field error on email `An account with this email already exists.` · `WEAK_PASSWORD` → field error · `SIGNUPS_CLOSED` → `Sign-ups are invite-only right now.` · `RATE_LIMITED` / generic as login.
- Success (`{status:'verify_email'}`) → navigate to `/verify-email?email=<email>`.

### Verify email

- Icon: 48px honey-ring envelope. H1 `Check your inbox` · sub `We sent a verification link to` + bold email.
- Button `Resend email` (secondary) → POST resend; 60s cooldown showing `Resend in 42s`; success toast-line `Sent. Check your spam folder too.`
- Link `Back to sign in` → `/login`.

### Forgot password

- H1 `Reset your password` · sub `Enter your email and we'll send you a reset link.`
- Email + submit `Send reset link` → always show success state: `If an account exists for <email>, a reset link is on its way.` + `Back to sign in`.

### Reset password

- H1 `Choose a new password`; PasswordField + PasswordStrength + `Confirm password` (must match: `Passwords don't match.`); submit `Update password`.
- Missing token → error state `This reset link is invalid or has expired.` + link `Request a new one` → `/forgot-password`.
- Success → `Password updated.` + primary `Sign in` → `/login`.

## API — Firebase + control plane

Identity lives in **Firebase Authentication**; the control plane only exchanges a
Firebase ID token for its own session cookie. Firebase sends the verification
and password-reset emails.

Client (Firebase JS SDK, loaded only in the auth chunk; config from `GET /auth/config`):

| Page | Firebase calls |
|---|---|
| Login | `signInWithEmailAndPassword` → unverified → `/verify-email`; else session exchange → `/app` |
| Signup | `createUserWithEmailAndPassword` → `updateProfile(name)` → `sendEmailVerification` → `/verify-email` (plan + invite kept in `sessionStorage` until the first exchange) |
| Google / GitHub | `signInWithPopup` → session exchange |
| Verify email | `sendEmailVerification(currentUser)` (60 s cooldown); `I've verified my email` → `reload()` → session exchange |
| Forgot password | `sendPasswordResetEmail` |
| `/account/action` | Firebase email links: `mode=verifyEmail` → `applyActionCode`; `mode=resetPassword` → `confirmPasswordReset` (`/reset-password` redirects here) |

Control plane (same-origin JSON, errors `{ error: string, code?: string }`):

```
GET  /auth/config    → {configured:true, apiKey, authDomain, projectId, appId} | {configured:false}
POST /auth/session   {idToken, plan?, inviteCode?} → 200 {ok:true} + th_session cookie
                     403 EMAIL_UNVERIFIED | SIGNUPS_CLOSED | INVALID_INVITE, 429 RATE_LIMITED
GET  /auth/me        → {user, plan, workspace:{state}} | 401
POST /auth/logout    → 204
```

Firebase error codes map to the copy above (`auth/invalid-credential`,
`auth/email-already-in-use`, `auth/weak-password`, `auth/too-many-requests`;
`auth/popup-closed-by-user` is silent). Business logic stays in hooks.

## Accessibility

One `h1` per page; labels bound to inputs; errors linked via `aria-describedby`
and announced (`role="alert"` banner); focus moves to the first invalid field on
submit; all flows keyboard-only usable; 44px touch targets; reduced-motion honored.

## Definition of done

tsc, lint, build, prettier clean · screenshots at 1440 and 390 for every page
look right · fine-grained commits (layout+primitives, login, signup+plans,
verify/forgot/reset, routing).
