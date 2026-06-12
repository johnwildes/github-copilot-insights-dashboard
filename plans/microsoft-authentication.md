# Plan: Microsoft (Entra ID) Authentication for Copilot Insights

## Goal

Require every visitor to sign in with their Microsoft account before they can
reach the dashboard. Today the app is publicly reachable (the only protection is
an optional shared `DASHBOARD_PASSWORD` and an `ADMIN_PASSWORD` gate on Settings).

Two distinct concerns, and what this plan delivers:

- **Authentication (AuthN) — who you are.** Implement now. Users must log in with
  a Microsoft (Microsoft Entra ID / Azure AD) account.
- **Authorization (AuthZ) — what you can do/see.** Explicitly out of scope for
  now: any authenticated user sees the entire dashboard. The design must *not*
  block adding role/section-based authorization later.

> Note on terminology: the request labels the login feature "AuthZ" and the
> future per-section restriction "AuthN". In standard usage it is the reverse —
> logging in is authentication (AuthN), and restricting access to sections is
> authorization (AuthZ). This plan uses the standard meanings. The intent is
> unchanged: log everyone in now, keep the door open for fine-grained access later.

## Current state (what exists today)

- **Framework:** Next.js 16 (App Router), React 19, deployed to Azure Container
  Apps via `azd` (Bicep in `infra/`), behind external ingress.
- **Existing "auth" model** is a password gate, not identity:
  - `app/src/lib/auth.ts` — HMAC-signed, stateless session cookies
    (`dashboard_session`, `admin_session`), rate limiting, lockout, timing-safe
    compare. Keyed off `DASHBOARD_PASSWORD` / `ADMIN_PASSWORD` env vars.
  - `app/src/proxy.ts` — Edge middleware that protects `/api/*`: public
    (`/api/auth/*`), admin (`/api/admin`, `/api/settings`, `/api/ingest`,
    `/api/audit-log`), and dashboard (everything else). Falls open when no
    password is configured.
  - `app/src/app/api/auth/verify-dashboard/route.ts` and `verify-admin` —
    POST endpoints that validate a password and set the session cookie.
  - `app/src/components/auth/auth-gate.tsx` and `admin-gate.tsx` — client
    components that prompt for the password and gate rendering.
  - `app/src/app/layout.tsx` — wraps the whole app in `<AuthGate>`.
  - `app/src/lib/audit.ts` — structured audit logging (`logAudit`, `getClientIp`)
    used for `dashboard_login_*` / admin events.
- **Infra:** `infra/resources.bicep` provisions Key Vault, User-Assigned Managed
  Identity, Container App + Container Apps Environment. Secrets (DB URL, admin /
  dashboard passwords, GitHub token) are stored in Key Vault and surfaced as
  Container App secrets + env vars.

The new Microsoft sign-in should *coexist with or replace* this password gate
(see "Relationship to the existing password gate" below).

## Recommended approach

Two viable options. Recommendation: **Option A (Auth.js / NextAuth) with the
Microsoft Entra ID provider**, because it keeps identity inside the app, makes
future role-based authorization straightforward (claims/roles are available in
session and middleware), and is portable across hosting environments.

Option B (Container Apps built-in "Easy Auth") is listed as a lighter-weight
alternative for teams that want zero application code and are content to stay on
Azure Container Apps.

### Option A — Auth.js (NextAuth) with Microsoft Entra ID  ← recommended

High-level steps:

1. **Register an app in Microsoft Entra ID.**
   - Create an App Registration (single-tenant unless multi-tenant access is
     desired). Record Tenant ID, Client ID.
   - Create a client secret (or configure a federated credential against the
     existing User-Assigned Managed Identity to avoid a stored secret — preferred
     for production).
   - Add redirect URI(s): the deployed Container App FQDN
     `https://<app>/api/auth/callback/microsoft-entra-id` and
     `http://localhost:3000/api/auth/callback/microsoft-entra-id` for local dev.
   - Request minimal delegated scopes: `openid`, `profile`, `email`
     (and `User.Read` if you want display name/photo from Graph).

2. **Add the Auth.js dependency.**
   - Add `next-auth` (v5 / Auth.js) to `app/package.json` and install with
     `pnpm install`. Run the security advisory check before adding.

3. **Create the Auth.js configuration.**
   - Add `app/src/lib/auth/` (e.g. `auth.config.ts` + `index.ts`) wiring the
     Microsoft Entra ID provider with Tenant ID / Client ID / secret from env.
   - Configure JWT session strategy (stateless, no DB needed) so it fits the
     existing serverless/Container App model. Set a strong `AUTH_SECRET`.
   - Add a `session`/`jwt` callback that copies stable identity claims (oid/sub,
     email, name) and a placeholder `roles` array onto the token/session. This is
     the seam for future authorization — populate roles now (empty/default) so
     consumers already read `session.user.roles`.

4. **Expose the Auth.js route handler.**
   - Add `app/src/app/api/auth/[...nextauth]/route.ts` exporting the generated
     GET/POST handlers. Note this lives under `/api/auth/*`, which `proxy.ts`
     already treats as public — confirm the catch-all is allowed through.

5. **Protect the application via middleware.**
   - Replace/augment `app/src/proxy.ts` logic so that, instead of (or in addition
     to) the password cookie check, it requires a valid Auth.js session for all
     non-public routes — both pages and `/api/*`.
   - Keep `/api/auth/*` and static/health (`/api/health`) public.
   - Preserve the "admin" distinction: admin API prefixes can continue to require
     the admin gate (or later, an admin role) on top of authentication.
   - Unauthenticated page requests redirect to the sign-in page; unauthenticated
     API requests return 401.
   - Confirm `middleware.ts` vs `proxy.ts` wiring: ensure the file Next.js loads
     as middleware invokes this logic and the `matcher` covers app routes, not
     just `/api/*`.

6. **Add a sign-in experience.**
   - Add a sign-in page (e.g. `app/src/app/login/page.tsx`) with a single
     "Sign in with Microsoft" button that triggers the Entra ID flow.
   - Replace the password-prompt `AuthGate` in `layout.tsx` with a thin
     server-side session check (or a session provider) so the chrome only renders
     for authenticated users. Reuse the existing loading/locked visual style.

7. **Surface the signed-in user + sign-out.**
   - Show the user's name/email/avatar and a "Sign out" action in
     `components/layout/sidebar.tsx`.
   - Add i18n strings (`signIn`, `signOut`, `signInWithMicrosoft`, etc.) to all
     four locale files under `app/src/lib/i18n/translations/{en,ar,es,fr}.ts`,
     following the existing `t()` key conventions.

8. **Audit logging.**
   - Emit `logAudit` events for sign-in success/failure and sign-out, mirroring
     the existing `dashboard_login_*` events, including client IP via
     `getClientIp`.

9. **Configuration / secrets.**
   - New env vars: `AUTH_SECRET`, `AUTH_MICROSOFT_ENTRA_ID_ID` (client id),
     `AUTH_MICROSOFT_ENTRA_ID_SECRET`, `AUTH_MICROSOFT_ENTRA_ID_TENANT_ID` (or
     issuer URL), and `NEXTAUTH_URL`/`AUTH_URL` for the deployed origin.
   - Document them in `app/.env.example`.
   - In `infra/resources.bicep`: add Key Vault secrets for the client secret and
     `AUTH_SECRET`, expose them as Container App secrets + env vars (follow the
     existing `kvSecret*` + `baseSecrets`/`baseEnvVars` pattern). Wire the
     redirect URI to the Container App FQDN output. Prefer a federated
     credential on the existing Managed Identity over a stored client secret.
   - Tighten the ingress `corsPolicy` (currently `allowedOrigins: ['*']`) to the
     app's own origin.

10. **Tests & validation.**
    - Add/adjust Vitest unit tests for the session/role callback and any token
      helpers (`app/src/**/*.test.ts`).
    - Run `pnpm run build` (type + lint) and `pnpm run test`.
    - Manually verify: unauthenticated redirect, successful Microsoft login,
      session persistence, sign-out, and that all reports load post-login.

### Option B — Container Apps built-in authentication ("Easy Auth")

Lighter alternative requiring little/no app code:

1. Register an Entra ID app as above.
2. Enable the `authConfigs` resource on the Container App in
   `infra/resources.bicep` with the Microsoft provider, set
   `globalValidation.unauthenticatedClientAction` to `RedirectToLoginPage`, and
   store the client secret in Key Vault.
3. The platform injects identity headers (`X-MS-CLIENT-PRINCIPAL*`) and handles
   `/.auth/login/aad` and `/.auth/logout`. Add a small helper to read the
   principal header for displaying the user and (later) extracting roles.
4. Update the sidebar to link to `/.auth/logout` and show the user from the
   injected header.

Trade-offs: minimal code and no secret management in the app, but identity is
tied to the hosting platform (not portable), local development requires a
fallback path, and future fine-grained authorization is harder because role
logic lives partly outside the app.

## Relationship to the existing password gate

- The `DASHBOARD_PASSWORD` gate becomes redundant once Microsoft sign-in is
  required. Recommended: remove the dashboard password path (AuthGate,
  verify-dashboard route, related proxy branch, Bicep dashboard-password secret)
  and rely solely on Entra ID — but do this as a clearly separated cleanup step
  after the new flow is verified, to keep the change reviewable.
- The `ADMIN_PASSWORD` gate on Settings can be retained short-term, or replaced
  with an "admin" role check derived from Entra ID app roles/group claims — this
  is the natural first use of the authorization seam.

## Designing for future authorization (AuthZ) without implementing it now

- Standardize on reading `session.user.roles` everywhere, populated by the
  session callback (default empty / all-access today).
- Define Entra ID **App Roles** (e.g. `Admin`, `Viewer`) in the App Registration
  now, even if unused, so role claims flow through later with no schema change.
- Keep the middleware route classification (public / authenticated / admin)
  centralized so per-section rules can be added in one place.
- Optionally add an (unused-for-now) `dim_user`-adjacent mapping or a config list
  to associate Entra object IDs with roles, deferred until AuthZ is implemented.

## Files likely to change

- `app/package.json` — add `next-auth`.
- `app/src/lib/auth/` — new Auth.js config + role/session callbacks.
- `app/src/app/api/auth/[...nextauth]/route.ts` — new handler.
- `app/src/proxy.ts` (and the middleware entry it backs) — session-based
  protection for pages + APIs; broaden the matcher.
- `app/src/app/layout.tsx` — swap password `AuthGate` for session-based gating.
- `app/src/app/login/page.tsx` — new sign-in page.
- `app/src/components/layout/sidebar.tsx` — user info + sign-out.
- `app/src/lib/i18n/translations/{en,ar,es,fr}.ts` — auth strings.
- `app/src/lib/audit.ts` callers — sign-in/out audit events.
- `app/.env.example` — new env vars.
- `infra/resources.bicep` (+ `main.bicep` params) — Key Vault secrets, Container
  App secrets/env, tightened CORS, redirect URI wiring.
- Cleanup (separate step): `verify-dashboard` route, `auth-gate.tsx`, and the
  dashboard-password paths in `auth.ts` / `proxy.ts` / Bicep.

## Risks & considerations

- **Single vs multi-tenant:** decide who may sign in. Single-tenant restricts to
  your org's directory (recommended default); multi-tenant or personal Microsoft
  accounts need explicit handling and possibly an allow-list.
- **Secret handling:** prefer Managed Identity federated credentials over a
  stored client secret; if a secret is used, rotate via Key Vault.
- **Local development:** provide a documented dev path (separate dev app
  registration or an env flag) so the app remains runnable without Azure.
- **Edge runtime:** Auth.js v5 middleware must run in the configured runtime;
  verify compatibility with the existing Edge `proxy.ts` setup.
- **Health checks / probes:** keep `/api/health` and Container App probes
  unauthenticated.
- **Fail-closed:** unlike today's fail-open password gate, the new flow should
  fail closed (deny on misconfiguration) once auth is mandatory.
