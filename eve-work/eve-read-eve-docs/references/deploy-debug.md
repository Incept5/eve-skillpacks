# Deployment + Debugging (Current)

## Default Environment (Staging)

Default to **staging** for user guidance. Use local/docker only when explicitly asked to do local development.

## Runtime Modes

- **Kubernetes** (`EVE_RUNTIME=k8s`): integration testing, staging, production
- **Docker Compose** (`EVE_RUNTIME=docker`): local dev (opt-in)

## Deploying Environments

```bash
eve env deploy test --ref main

eve env deploy staging --ref abc123
```

If `environments.<env>.pipeline` is set, `eve env deploy` triggers that pipeline. Use `--direct` to bypass.

## Ingress URLs

Default pattern:
```
{service}.{orgSlug}-{projectSlug}-{env}.{domain}
```

The `orgSlug` and `projectSlug` come from `eve org ensure --slug` and `eve project ensure --slug`.
K8s namespace follows the same pattern: `eve-{orgSlug}-{projectSlug}-{env}`.

Example: org slug `myorg`, project slug `myproj`, env `staging` →
- URL: `api.myorg-myproj-staging.eh1.incept5.dev`
- Namespace: `eve-myorg-myproj-staging`

Domain resolution order:
1) manifest `x-eve.ingress.domain`
2) `EVE_DEFAULT_DOMAIN`
3) no ingress if neither set

## Platform Env Vars (Injected)

- `EVE_API_URL`
- `EVE_PROJECT_ID`
- `EVE_ORG_ID`
- `EVE_ENV_NAME`

## CLI-First Debugging Ladder

1) **CLI**
   - `eve system health`
   - `eve job show <id> --verbose`
   - `eve job diagnose <id>`
   - `eve job follow <id>`
2) **Environment status**
   - Use `./bin/eh status` when available to confirm URLs and running services.
3) **kubectl** only if CLI lacks data.

## Common Debug Commands

```bash
eve job attempts <id>
eve job logs <id> --attempt 2
eve job result <id>
eve job watch <id>
```

## Notes

- CLI only needs `EVE_API_URL`; everything routes through the API.
- For k8s, ingress URL is typically `http://api.eve.lvh.me`.
