# Eve Auth SDK Reference

Two shared packages that eliminate auth boilerplate in Eve-compatible apps.

| Package | Scope | Purpose |
|---------|-------|---------|
| `@eve-horizon/auth` | Backend (Express/NestJS) | Token verification, org check, route protection |
| `@eve-horizon/auth-react` | Frontend (React) | SSO session bootstrap, login gate, token cache |

## Backend: `@eve-horizon/auth`

### Setup (Express)

```bash
npm install @eve-horizon/auth
```

```typescript
import { eveUserAuth, eveAuthGuard, eveAuthConfig } from '@eve-horizon/auth';

app.use(eveUserAuth());                                     // Parse tokens (non-blocking)
app.get('/auth/config', eveAuthConfig());                   // Serve SSO discovery
app.get('/auth/me', eveAuthGuard(), (req, res) => {         // Protected endpoint
  res.json(req.eveUser);
});
app.use('/api', eveAuthGuard());                            // Protect all API routes
```

### Exports

| Export | Type | Purpose |
|--------|------|---------|
| `eveUserAuth(options?)` | Middleware | Verify user token, check org membership, attach `req.eveUser` |
| `eveAuthGuard()` | Middleware | Return 401 if `req.eveUser` not set |
| `eveAuthConfig()` | Handler | Serve `{ sso_url, eve_api_url, ... }` from env vars |
| `eveAuthMiddleware(options?)` | Middleware | Agent/job token verification (blocking), attach `req.agent` |
| `verifyEveToken(token, url?)` | Function | JWKS-based local verification (15-min cache) |
| `verifyEveTokenRemote(token, url?)` | Function | HTTP verification via `/auth/token/verify` |

### Middleware Behavior

**`eveUserAuth()`** is non-blocking — unauthenticated requests pass through. Pair with `eveAuthGuard()` to enforce authentication on specific routes.

**`eveAuthMiddleware()`** is blocking — returns 401 immediately on any verification failure. Use for agent-facing APIs.

### Verification Strategies

| Strategy | Default for | Latency | Freshness |
|----------|-------------|---------|-----------|
| `'local'` | `eveUserAuth` | Fast (JWKS cached 15 min) | Stale up to 15 min |
| `'remote'` | `eveAuthMiddleware` | ~50ms per request | Always current |

### Token Types

```typescript
interface EveTokenClaims {
  valid: true;
  type: 'user' | 'job' | 'service_principal';
  user_id: string;
  email?: string;
  org_id?: string | null;     // Job tokens: single org
  orgs?: Array<{              // User tokens: all memberships
    id: string;
    role: string;
  }>;
  project_id?: string;
  job_id?: string;
  permissions?: string[];
  is_admin?: boolean;
  role?: string;
}

interface EveUser {
  id: string;
  email: string;
  orgId: string;
  role: 'owner' | 'admin' | 'member';
}
```

## Frontend: `@eve-horizon/auth-react`

### Setup (React)

```bash
npm install @eve-horizon/auth-react
```

```tsx
import { EveAuthProvider, EveLoginGate } from '@eve-horizon/auth-react';

function App() {
  return (
    <EveAuthProvider apiUrl="/api">
      <EveLoginGate>
        <ProtectedApp />
      </EveLoginGate>
    </EveAuthProvider>
  );
}
```

### Exports

| Export | Type | Purpose |
|--------|------|---------|
| `EveAuthProvider` | Component | Context provider, session bootstrap |
| `useEveAuth()` | Hook | `{ user, loading, error, config, loginWithSso, loginWithToken, logout }` |
| `EveLoginGate` | Component | Render children when authed, login form when not |
| `EveLoginForm` | Component | SSO + token paste login UI |
| `createEveClient(baseUrl?)` | Function | Fetch wrapper with Bearer injection |
| `getStoredToken()` / `storeToken()` / `clearToken()` | Functions | Direct sessionStorage access |

### Session Bootstrap Sequence

1. Check `sessionStorage` for cached token → validate via `GET /auth/me`
2. Fetch `GET /auth/config` to get `sso_url`
3. Probe `GET {sso_url}/session` (root-domain cookie) → get fresh token
4. If no session → show login form

### Token Lifecycle

| Token | Storage | TTL |
|-------|---------|-----|
| Eve RS256 access token | `sessionStorage` | 1 day |
| SSO refresh cookie | httpOnly cookie (root domain) | 30 days |

## Auto-Injected Environment Variables

The platform deployer injects these into every deployed app:

| Variable | Purpose |
|----------|---------|
| `EVE_API_URL` | Internal API URL (JWKS fetch, remote verify) |
| `EVE_PUBLIC_API_URL` | Public-facing API URL |
| `EVE_SSO_URL` | SSO broker URL (`eveAuthConfig()` response) |
| `EVE_ORG_ID` | Org membership check |

Use `${SSO_URL}` in manifest env blocks for frontend-accessible SSO URL.

## Migration from Custom Auth

Replace ~750 lines of hand-rolled auth with ~50 lines:

| Delete | Replacement |
|--------|-------------|
| JWKS setup, org check, role mapping | `eveUserAuth()` |
| Bearer extraction middleware | Built into `eveUserAuth()` |
| Route protection guard | `eveAuthGuard()` |
| SSO URL discovery (api. → sso. hack) | `eveAuthConfig()` reads `EVE_SSO_URL` |
| Frontend `useAuth` hook | `useEveAuth()` |
| Token storage and Bearer injection | `createEveClient()` |
| Login form | `EveLoginGate` / `EveLoginForm` |

## NestJS Integration

Wrap the Express middleware in a thin NestJS guard:

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class EveGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    return !!ctx.switchToHttp().getRequest().eveUser;
  }
}
```

## SSE Authentication

The middleware supports `?token=` query parameter for Server-Sent Events:
```
GET /api/events?token=eyJ...
```
