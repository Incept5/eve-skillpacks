---
name: eve-staging-operations
description: Deploy and operate Eve Horizon on staging infrastructure. Use when creating staging overlays, deploying to staging, verifying health, troubleshooting staging issues, or managing staging environments.
---

# Eve Staging Operations

Use this skill to deploy, verify, and troubleshoot Eve Horizon on the staging environment.

## When to Use

- Creating or updating the Kustomize staging overlay
- Deploying Eve Horizon to the staging VPS
- Verifying staging health after deployment
- Troubleshooting staging failures (DNS, DB, registry, ingress, jobs)
- Updating or rolling back staging deployments

## Prerequisites

**Before deploying to staging:**

1. VPS provisioned with k3s (see [eve-aws-provisioning](../eve-aws-provisioning/SKILL.md))
2. External PostgreSQL database accessible from the VPS
3. DNS records configured:
   - `api.eve-staging.incept5.dev` A record pointing to VPS IP
   - `*.eve-staging.incept5.dev` wildcard A record pointing to VPS IP
4. Container registry credentials (GHCR or private)
5. SSH access to the staging VPS

**Environment Values:**
| Variable | Value |
|----------|-------|
| Domain | `eve-staging.incept5.dev` |
| API URL | `api.eve-staging.incept5.dev` |
| EVE_DEFAULT_DOMAIN | `eve-staging.incept5.dev` |
| Route 53 Zone | incept5.dev (Z03843481L61UH2BEHZM8) |
| Namespace | `eve` |

## Create Staging Overlay

### Directory Structure

Create or verify the staging overlay exists at `k8s/overlays/staging/`:

```
k8s/overlays/staging/
├── kustomization.yaml
├── api-deployment-patch.yaml
├── orchestrator-deployment-patch.yaml
├── worker-deployment-patch.yaml
├── api-ingress-patch.yaml
├── db-migrate-job-patch.yaml
└── remove-postgres.yaml
```

### kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: eve
patches:
  - path: api-ingress-patch.yaml
  - path: api-deployment-patch.yaml
  - path: orchestrator-deployment-patch.yaml
  - path: worker-deployment-patch.yaml
  - path: db-migrate-job-patch.yaml
  - path: remove-postgres.yaml
```

### api-ingress-patch.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: eve-api
  namespace: eve
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - $patch: replace
      host: api.eve-staging.incept5.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: eve-api
                port:
                  number: 4701
```

### api-deployment-patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eve-api
  namespace: eve
spec:
  template:
    spec:
      imagePullSecrets:
        - name: eve-registry
      containers:
        - name: api
          image: ghcr.io/incept5/eve-horizon-api:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: eve-app
                  key: DATABASE_URL
```

### worker-deployment-patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eve-worker
  namespace: eve
spec:
  template:
    spec:
      imagePullSecrets:
        - name: eve-registry
      containers:
        - name: worker
          image: ghcr.io/incept5/eve-horizon-worker:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: eve-app
                  key: DATABASE_URL
            - name: EVE_RUNNER_IMAGE
              value: ghcr.io/incept5/eve-worker-full:latest
            - name: EVE_DEFAULT_DOMAIN
              value: eve-staging.incept5.dev
```

### orchestrator-deployment-patch.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eve-orchestrator
  namespace: eve
spec:
  template:
    spec:
      imagePullSecrets:
        - name: eve-registry
      containers:
        - name: orchestrator
          image: ghcr.io/incept5/eve-horizon-orchestrator:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: eve-app
                  key: DATABASE_URL
```

### remove-postgres.yaml

```yaml
$patch: delete
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: eve
---
$patch: delete
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: eve
---
$patch: delete
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: eve
```

## Deploy to Staging

### 1. Create the Namespace

```bash
kubectl create namespace eve --dry-run=client -o yaml | kubectl apply -f -
```

### 2. Create Registry Secret

```bash
kubectl -n eve create secret docker-registry eve-registry \
  --docker-server=ghcr.io \
  --docker-username=$GHCR_USERNAME \
  --docker-password=$GHCR_TOKEN \
  --docker-email=$GHCR_EMAIL
```

### 3. Create Application Secrets

```bash
kubectl -n eve create secret generic eve-app \
  --from-literal=DATABASE_URL="postgres://eve:PASSWORD@db-host:5432/eve" \
  --from-literal=EVE_INTERNAL_API_KEY="$(openssl rand -hex 32)" \
  --from-literal=EVE_SECRETS_MASTER_KEY="$(openssl rand -hex 32)" \
  --from-literal=CLAUDE_ACCESS_TOKEN="$CLAUDE_ACCESS_TOKEN" \
  --from-literal=GHCR_USERNAME="$GHCR_USERNAME" \
  --from-literal=GHCR_TOKEN="$GHCR_TOKEN"
```

### 4. Apply Manifests

```bash
kubectl apply -k k8s/overlays/staging
```

### 5. Wait for Rollout

```bash
kubectl -n eve rollout status deployment/eve-api --timeout=180s
kubectl -n eve rollout status deployment/eve-orchestrator --timeout=180s
kubectl -n eve rollout status deployment/eve-worker --timeout=180s
```

## Verify Staging Health

### Health Endpoint Check

```bash
curl -f https://api.eve-staging.incept5.dev/health
```

Expected response:
```json
{"status":"ok","version":"...","db":"connected"}
```

### CLI Verification

```bash
# Configure CLI for staging
eve config set api_url https://api.eve-staging.incept5.dev

# Verify connectivity
eve org list
```

### Deployment Checklist

- [ ] All pods running: `kubectl -n eve get pods`
- [ ] No restart loops: `kubectl -n eve get pods` (RESTARTS column)
- [ ] Health endpoint returns 200
- [ ] Database migrations completed: `kubectl -n eve logs job/db-migrate`
- [ ] Ingress routing works: `curl -v https://api.eve-staging.incept5.dev/health`
- [ ] Worker can pull runner image: check worker logs
- [ ] Test job executes successfully

## Update Staging

### Rolling Update (Image Change)

```bash
# Update image tag in patch files, then:
kubectl apply -k k8s/overlays/staging

# Or use kustomize edit:
cd k8s/overlays/staging
kustomize edit set image ghcr.io/incept5/eve-horizon-api:v1.2.3
kubectl apply -k .
```

### Secrets Rotation

```bash
# Delete and recreate the secret
kubectl -n eve delete secret eve-app
kubectl -n eve create secret generic eve-app \
  --from-literal=DATABASE_URL="..." \
  # ... other secrets

# Restart deployments to pick up new secrets
kubectl -n eve rollout restart deployment/eve-api
kubectl -n eve rollout restart deployment/eve-orchestrator
kubectl -n eve rollout restart deployment/eve-worker
```

### Rollback

```bash
# Rollback to previous revision
kubectl -n eve rollout undo deployment/eve-api
kubectl -n eve rollout undo deployment/eve-orchestrator
kubectl -n eve rollout undo deployment/eve-worker

# Rollback to specific revision
kubectl -n eve rollout undo deployment/eve-api --to-revision=2
```

## Troubleshooting

### DNS Issues

**Symptom:** `curl: (6) Could not resolve host: api.eve-staging.incept5.dev`

**Check:**
```bash
dig api.eve-staging.incept5.dev
nslookup api.eve-staging.incept5.dev
```

**Fix:**
- Verify A record exists in Route 53 (zone Z03843481L61UH2BEHZM8)
- Wait for DNS propagation (up to 5 minutes)
- Check for CNAME vs A record conflicts

### Database Connection

**Symptom:** Pod crash loop with `ECONNREFUSED` or `timeout` to database

**Check:**
```bash
kubectl -n eve logs deployment/eve-api | grep -i database
kubectl -n eve describe secret eve-app
```

**Fix:**
- Verify DATABASE_URL is correct in the secret
- Check RDS security group allows traffic from VPS IP
- Test connectivity from VPS: `psql $DATABASE_URL -c "SELECT 1"`

### Registry Pull Errors

**Symptom:** `ImagePullBackOff` or `ErrImagePull`

**Check:**
```bash
kubectl -n eve describe pod <pod-name> | grep -A5 "Events"
kubectl -n eve get secret eve-registry -o yaml
```

**Fix:**
- Verify eve-registry secret exists and has correct credentials
- Test registry auth: `docker login ghcr.io -u $GHCR_USERNAME -p $GHCR_TOKEN`
- Verify image exists: `docker pull ghcr.io/incept5/eve-horizon-api:latest`

### Ingress Routing

**Symptom:** 404 or connection refused on the API URL

**Check:**
```bash
kubectl -n eve get ingress
kubectl -n eve describe ingress eve-api
kubectl -n kube-system logs -l app.kubernetes.io/name=traefik
```

**Fix:**
- Verify ingress host matches DNS: `api.eve-staging.incept5.dev`
- Check traefik is running: `kubectl -n kube-system get pods -l app.kubernetes.io/name=traefik`
- Verify service exists: `kubectl -n eve get svc eve-api`

### Job Failures

**Symptom:** Jobs stuck or failing

**Check:**
```bash
eve job list
eve job diagnose <job-id>
kubectl -n eve-jobs get pods
kubectl -n eve-jobs logs <runner-pod>
```

**Fix:**
- Check worker logs: `kubectl -n eve logs deployment/eve-worker`
- Verify EVE_RUNNER_IMAGE is pullable
- Check eve-jobs namespace has registry secret
- See [eve-deploy-debugging](../eve-deploy-debugging/SKILL.md) for detailed job debugging

## Common Commands Reference

| Command | Purpose |
|---------|---------|
| `kubectl -n eve get pods` | List all Eve pods |
| `kubectl -n eve logs deployment/eve-api` | View API logs |
| `kubectl -n eve logs deployment/eve-worker` | View worker logs |
| `kubectl -n eve describe pod <name>` | Debug pod issues |
| `kubectl -n eve exec -it deployment/eve-api -- sh` | Shell into API pod |
| `kubectl -n eve rollout restart deployment/eve-api` | Restart API |
| `kubectl -n eve get events --sort-by='.lastTimestamp'` | Recent events |
| `kubectl apply -k k8s/overlays/staging` | Apply staging overlay |
| `kubectl -n eve scale deployment/eve-worker --replicas=2` | Scale workers |

## Environment Variables Reference

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgres://eve:pass@host:5432/eve` |
| `EVE_INTERNAL_API_KEY` | Internal service auth key | 64 char hex string |
| `EVE_SECRETS_MASTER_KEY` | Encryption key for secrets | 64 char hex string |
| `EVE_DEFAULT_DOMAIN` | Default domain for job ingress | `eve-staging.incept5.dev` |
| `EVE_RUNNER_IMAGE` | Docker image for job runners | `ghcr.io/incept5/eve-worker-full:latest` |
| `CLAUDE_ACCESS_TOKEN` | Claude API auth token | OAuth token |
| `GHCR_USERNAME` | GitHub Container Registry user | GitHub username |
| `GHCR_TOKEN` | GitHub Container Registry PAT | `ghp_...` |

## Related Skills

- [eve-aws-provisioning](../eve-aws-provisioning/SKILL.md) - VPS setup and k3s installation
- [eve-deploy-debugging](../eve-deploy-debugging/SKILL.md) - Deploy flows and job debugging
- [eve-auth-and-secrets](../eve-auth-and-secrets/SKILL.md) - OAuth tokens and secrets management

## Recursive Skill Distillation

- Add new staging-specific issues and fixes as they emerge
- Extract CI/CD workflow steps into a dedicated skill if needed
- Update this skill when staging infrastructure changes
