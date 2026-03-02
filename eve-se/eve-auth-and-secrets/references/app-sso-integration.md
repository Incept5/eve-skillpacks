# App SSO Integration Reference

Add Eve SSO login to Eve-deployed apps. The platform auto-injects environment variables; the shared packages handle token verification and session management.

**Related**: [SKILL.md](../SKILL.md) for quick start, Eve platform docs for SSO broker internals.

## Contents

- [Auto-Injected Environment Variables](#auto-injected-environment-variables)
- [Backend API (@eve-horizon/auth)](#backend-api-eveauth)
- [Frontend API (@eve-horizon/auth-react)](#frontend-api-eveauth-react)
- [Token Flow](#token-flow)
- [Advanced Patterns](#advanced-patterns)

## Auto-Injected Environment Variables

Eve's deployer injects these into every deployed service:

| Variable | Description |
|----------|-------------|
| `EVE_API_URL` | Internal API URL for server-to-server calls |
| `EVE_PUBLIC_API_URL` | Public-facing API URL (optional) |
| `EVE_SSO_URL` | SSO broker URL |
| `EVE_ORG_ID` | Organization ID |
| `EVE_PROJECT_ID` | Project ID |
| `EVE_ENV_NAME` | Environment name |

For local development, set `EVE_SSO_URL`, `EVE_ORG_ID`, and `EVE_API_URL` manually. Use manifest interpolation for custom env vars:

```yaml
services:
  web:
    environment:
      MY_SSO_URL: "${SSO_URL}"
```

## Backend API (@eve-horizon/auth)

Install: `npm install @eve-horizon/auth`

### eveUserAuth(options?)

Express middleware. Verifies Eve RS256 tokens and checks org membership.

- **Non-blocking**: unauthenticated requests pass through. Use `eveAuthGuard()` to enforce.
- Attaches `req.eveUser: { id, email, orgId, role }` on success.
- Extracts tokens from `Authorization: Bearer` header or `?token=` query param.
- Options:
  - `orgId?: string` -- override `EVE_ORG_ID`
  - `eveApiUrl?: string` -- override `EVE_API_URL`
  - `strategy?: 'local' | 'remote'` -- JWKS verification (default) or HTTP verification

### eveAuthGuard()

Express middleware. Returns 401 if `req.eveUser` is not set. Place after `eveUserAuth()`.

### eveAuthConfig()

Express handler. Returns `{ sso_url, eve_api_url, eve_public_api_url, eve_org_id }` from environment variables. The frontend provider fetches this to discover SSO configuration.

### eveAuthMiddleware(options?)

Lower-level middleware for agent/job token verification. Attaches `req.agent` with full `EveTokenClaims`. Returns 401 on failure (blocking, unlike `eveUserAuth`).

- Options:
  - `eveApiUrl?: string` -- override `EVE_API_URL`
  - `strategy?: 'remote' | 'local'` -- HTTP (default) or JWKS verification

### verifyEveToken(token, eveApiUrl?)

Verify a token locally using JWKS (fetched and cached from Eve API). Faster for high-throughput scenarios. Returns `EveTokenClaims`.

### verifyEveTokenRemote(token, eveApiUrl?)

Verify a token by calling Eve API `/auth/token/verify` endpoint. Simplest approach -- no key management. Returns `EveTokenClaims`.

## Frontend API (@eve-horizon/auth-react)

Install: `npm install @eve-horizon/auth-react`

### EveAuthProvider

Context provider. Handles session bootstrap, token caching, SSO probing. Fetches `/auth/config` from the backend to discover SSO URL.

```tsx
<EveAuthProvider apiUrl="/api">
  {children}
</EveAuthProvider>
```

### useEveAuth()

Hook returning `{ user, loading, error, config, loginWithSso, loginWithToken, logout }`.

- `user: { id, email, orgId, role } | null`
- `loginWithSso()` -- redirect to SSO broker login page
- `loginWithToken(token: string)` -- validate and store a pasted token
- `logout()` -- clear stored token and reset state

### EveLoginGate

Renders children when authenticated, login form otherwise.

Props:
- `fallback?: ReactNode` -- custom login component (defaults to `EveLoginForm`)
- `loadingFallback?: ReactNode` -- custom loading component (defaults to null)

### EveLoginForm

Built-in login UI with SSO and token paste modes. SSO button is disabled when `sso_url` is not configured.

### createEveClient(baseUrl?)

Fetch wrapper with automatic Bearer token injection. Returns `{ fetch, getToken }`.

```typescript
const client = createEveClient('/api');
const res = await client.fetch('/users');
```

### Token Storage

- `getStoredToken()` -- read cached token from `sessionStorage`
- `storeToken(token)` -- cache a token
- `clearToken()` -- clear cached token and reset state

## Token Flow

1. User visits app -- `EveAuthProvider` checks `sessionStorage` for cached token.
2. No cached token -- probes SSO broker `/session` endpoint (uses root-domain cookie).
3. SSO session exists -- gets fresh Eve RS256 token, caches in `sessionStorage`.
4. No SSO session -- shows login form (SSO redirect or token paste).
5. All API requests include `Authorization: Bearer <token>` header.

## Advanced Patterns

### SSE Authentication

The middleware supports `?token=` query parameter for Server-Sent Events:

```
GET /api/events?token=eyJ...
```

### Token Paste Mode

For development or headless environments, get a token from the CLI and paste it into the login form:

```bash
eve auth token  # prints Bearer token
```

### Token Staleness

The `orgs` claim in JWT tokens reflects membership at token mint time. With default 1-day TTL, membership changes take up to 24h to reflect. For immediate membership checks, use `strategy: 'remote'` in `eveUserAuth()`.
