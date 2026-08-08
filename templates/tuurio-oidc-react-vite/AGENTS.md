# AGENTS.md

React SPA using Tuurio ID through standards-based OpenID Connect. Keep the
browser app a public client and keep every credential outside the agent context.

## Stack

- React 18+, TypeScript strict, and Vite 5+ on Node 20+.
- `manage-tuurio-id` 1.1.6 for client provisioning.
- `oidc-client-ts` 3.x for Authorization Code with PKCE S256.
- Package manager and test runner: use the versions already locked by the repo.

## Commands

Inspect the runnable app before substituting its directory for `<app-root>`:

```bash
npx manage-tuurio-id@1.1.6 init --framework react \
  --project-dir <app-root> --auth browser --yes --output json \
  --campaign agents_md_directory --no-open --no-wait
```

- Show the exact resolved command and wait for explicit human approval.
- Run it once. On `authorization_required`, show `verificationUriComplete` and
  stop without leaving a process open.
- After the human completes the browser step, rerun the exact same command.
  Never start overlapping runs, delete saved state, or add `--fresh-handoff`
  without new approval.
- Never accept contracts or authenticate for the user.

Use the repository's commands after implementation, for example:

```bash
pnpm lint && pnpm typecheck && pnpm test && pnpm build
```

## Before Provisioning

- Locate the actual Vite app root; do not assume the workspace root.
- Identify every environment file the CLI may update, including `.env`,
  `.env.local`, and mode-specific files.
- Stop if a target is tracked or staged. If an untracked target is not ignored,
  add only its exact path to `.gitignore` and verify it is ignored.
- Never ignore example or template environment files.
- Repeat these checks after provisioning.

## Public Client Configuration

- A browser app is a public client: never add a client secret.
- Import the generated public configuration from the requested project path.
  It may contain issuer, client ID, exact redirects, and deployment origins.
- Select exactly one target matching `window.location.origin`; fail closed on
  zero or multiple matches.
- Require HTTPS for production issuer endpoints and redirects. Permit HTTP only
  for explicit loopback development on `localhost`, `127.0.0.1`, or `[::1]`.

Example target selection:

```ts
const matches = config.targets.filter(
  (target) => new URL(target.deploymentBaseUrl).origin === window.location.origin,
);
if (matches.length !== 1) throw new Error("OIDC target mismatch");
```

## OIDC Requirements

- Use Authorization Code with PKCE S256 through `oidc-client-ts`; never disable
  PKCE or hand-write token parsing or cryptographic validation.
- Let the library validate stored authorization state and nonce. Process each
  callback only once and reject replay, missing state, or origin mismatch.
- `oidc-client-ts` deliberately does not verify ID-token signatures locally in
  this flow. Do not claim that it does. Accept its decoded browser claims only
  when the code was redeemed directly at the configured HTTPS token endpoint
  and PKCE, state, and nonce checks succeeded.
- Before exposing a returned user to application state, fail closed unless the
  issuer equals `config.issuer`, the audience contains `config.clientId`, and
  the token has not expired. Clear stored user state on any mismatch.
- Treat browser ID-token claims as session and display metadata, never as proof
  of backend authorization. APIs must validate access-token signature, issuer,
  audience, and expiry server-side.
- If policy requires local cryptographic ID-token verification, stop and use a
  reviewed library or BFF that supports it; never implement JWT crypto by hand.
- Add explicit callback, protected-route, sign-out, and post-logout handling.
- Enable renewal only when deliberately configured. Apply the same checks to
  renewed users, avoid retry loops, and clear state when renewal fails.

Example public-client manager options:

```ts
const manager = new UserManager({
  authority: config.issuer,
  client_id: config.clientId,
  redirect_uri: target.redirectUri,
  post_logout_redirect_uri: target.postLogoutRedirectUri,
  response_type: "code",
  scope: config.scope,
  automaticSilentRenew: false,
});
```

## Verification

- Run lint, typecheck, tests, and the production build.
- Test state and nonce mismatch, callback replay, wrong redirect URI, open
  redirects, expired tokens, renewal failure, and logout cleanup.
- Inspect the bundle and generated public config for client secrets.
- Ask the human to verify one real sign-in and sign-out on every registered
  deployment origin. Never request codes, tokens, cookies, or `.env` contents.

## Security / Boundaries

- Do inspect and preserve existing authentication unless migration is requested.
- Do let the OIDC library process codes and tokens as protocol values.
- Do keep exact CLI-generated variable names and redirect URIs.
- Ask first before provisioning, changing identity providers, or replacing an
  established OIDC library.
- Never disclose, log, expose, or commit passwords, bootstrap tokens, client
  secrets, authorization codes, access tokens, refresh tokens, or cookies.
- Never place secrets in `VITE_*` variables or browser bundles; use public client
  metadata only.
- Never commit secrets, including `.env` files; do keep them ignored and verify
  both the Git index and working tree before finishing.
