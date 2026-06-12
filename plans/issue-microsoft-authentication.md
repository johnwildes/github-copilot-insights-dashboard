# Issue: Add Microsoft (Entra ID) sign-in authentication

> Draft issue body. Implementation reference: [`plans/microsoft-authentication.md`](./microsoft-authentication.md).

## Summary

The dashboard is currently publicly reachable; the only protection is an optional
shared `DASHBOARD_PASSWORD` gate and an `ADMIN_PASSWORD` gate on Settings. We want
every visitor to **sign in with their Microsoft account (Microsoft Entra ID)**
before reaching any part of the dashboard.

Scope for this issue is **authentication only (AuthN)** — once signed in, a user
sees the entire dashboard. **Authorization (AuthZ)** — restricting access to
specific sections/reports by role — is explicitly *out of scope*, but the
implementation must not block adding it later.

> Terminology note: logging in = authentication (AuthN); restricting sections =
> authorization (AuthZ). The original request used these labels in reverse; the
> intent is "log everyone in now, keep the door open for fine-grained access later."

## Background / current state

- Next.js 16 (App Router), React 19, deployed to Azure Container Apps via `azd`
  (Bicep in `infra/`), external ingress with `corsPolicy.allowedOrigins: ['*']`.
- Password-gate "auth" today:
  - `app/src/lib/auth.ts` — HMAC-signed stateless session cookies, rate limiting,
    lockout, timing-safe compare.
  - `app/src/proxy.ts` — Edge middleware protecting `/api/*` (public / dashboard /
    admin tiers). **Fails open** when no password is configured.
  - `app/src/app/api/auth/verify-dashboard` + `verify-admin` routes.
  - `app/src/components/auth/{auth-gate,admin-gate}.tsx` + `app/src/app/layout.tsx`.
  - `app/src/lib/audit.ts` — structured audit logging.

## Proposed approach

Use **Auth.js (NextAuth v5) with the Microsoft Entra ID provider** (recommended in
the plan), JWT session strategy, identity claims + a placeholder `roles` array on
the session as the seam for future authorization. (Alternative: Container Apps
"Easy Auth" — see plan for trade-offs.)

See `plans/microsoft-authentication.md` for the full step-by-step approach,
affected files, and infra/secret wiring.

## Success criteria

- [ ] An unauthenticated user visiting **any** page is redirected to a sign-in
      screen and cannot view dashboard content or chrome.
- [ ] Unauthenticated requests to protected `/api/*` routes return **401**.
- [ ] `/api/health` and Container App probes remain reachable **without** auth.
- [ ] A user can click **"Sign in with Microsoft"**, complete the Entra ID flow,
      and land back on the dashboard authenticated.
- [ ] The session persists across page reloads and navigation until expiry.
- [ ] The signed-in user's name/email is shown and a **"Sign out"** action ends
      the session and returns the user to the sign-in screen.
- [ ] After sign-in, **all existing reports/pages load and function** unchanged
      (no regression to data, charts, filters, i18n, or theming).
- [ ] Auth strings are present in **all four locales** (en/ar/es/fr) and the
      sign-in UI respects light/dark/RTL.
- [ ] Sign-in success, sign-in failure, and sign-out emit **audit log** events
      (with client IP) via the existing `logAudit` helper.
- [ ] Secrets (`AUTH_SECRET`, Entra client id/secret, tenant id) are sourced from
      **Key Vault** in `infra/resources.bicep`; none are hardcoded or in plain env.
      Redirect URI is wired to the Container App FQDN.
- [ ] Ingress `corsPolicy.allowedOrigins` is tightened from `['*']` to the app's
      own origin.
- [ ] The app **fails closed** (denies access) on auth misconfiguration in
      production, rather than failing open.
- [ ] `app/.env.example` documents all new environment variables.
- [ ] A documented local-development path exists so the app runs without Azure.
- [ ] Authorization seam is in place: `session.user.roles` is populated (default
      all-access) and Entra **App Roles** are defined for future use.
- [ ] `pnpm run build` (types + lint) and `pnpm run test` pass.

## Tests

### Unit tests (Vitest — `app/src/**/*.test.ts`)

- [ ] **Session/JWT callback**: given Entra claims (oid/sub, email, name), the
      session callback returns a user object with those fields and a `roles`
      array (default value when no roles claim present).
- [ ] **Token/identity helpers**: any new helper (e.g. principal parsing, role
      extraction) returns expected output for valid input and safe defaults for
      missing/malformed input.
- [ ] **Middleware route classification**: public prefixes (`/api/auth/*`,
      `/api/health`) are allowed without a session; protected routes require one;
      admin routes still require the admin tier. Cover both page and `/api/*`
      paths.
- [ ] **Fail-closed behavior**: with auth enabled but session missing/invalid, the
      guard denies (redirect for pages, 401 for APIs).

### Integration / behavioral tests

- [ ] Request to a protected API route **without** a session cookie → `401`.
- [ ] Request to `/api/health` without a session → `200`.
- [ ] Request to a page without a session → redirect (3xx) to the sign-in route.
- [ ] With a valid (mocked) session, a protected page/API returns content/`200`.

### End-to-end (Playwright — config already present)

- [ ] Visiting `/` while unauthenticated lands on the sign-in screen.
- [ ] Completing sign-in (mocked Entra/provider) reaches the dashboard, shows the
      user identity, and renders a report page.
- [ ] Clicking **Sign out** returns to the sign-in screen and a subsequent
      protected navigation is blocked.
- [ ] Sign-in UI renders correctly in dark mode and in an RTL locale (ar).

### Manual verification checklist

- [ ] Single-tenant restriction works (only directory users can sign in); decide
      and document multi-tenant/personal-account handling.
- [ ] Deployed Container App redirect URI + Key Vault secrets resolve correctly.
- [ ] Audit events appear for login success/failure and logout.

## Out of scope

- Per-section / per-report role-based authorization (tracked separately; this
  issue only lays the groundwork via `session.user.roles` + Entra App Roles).
- Migrating existing `dim_user` data to auth identities.
