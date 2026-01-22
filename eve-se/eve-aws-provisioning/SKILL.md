---
name: eve-aws-provisioning
description: Provision AWS VPS (k3s) for Eve Horizon using the P0 baseline overlay, including DNS, registry auth, external DB, and smoke checks.
---

# Eve AWS Provisioning (P0)

Use this skill to provision a single-node AWS VPS (k3s) for the Eve Horizon P0 baseline.

## When to Use

- Deploying Eve Horizon to AWS in the P0 scope (k3s on a single VPS).
- Setting up external Postgres, registry auth, and ingress domains.

## Preconditions

- AWS VPS (Ubuntu 22.04+), SSH access
- Public domain with DNS control
- External Postgres (RDS or managed)
- Container registry credentials (GHCR or private)

## Instructions

1. **Confirm inputs**: VPS IP, domain, registry, DB URL, and desired image tags.
2. **Install k3s**:
   - `curl -sfL https://get.k3s.io | sh -`
   - Export `KUBECONFIG=/etc/rancher/k3s/k3s.yaml`
3. **DNS setup**:
   - `api.<domain>` A record -> VPS IP
   - `*.apps.<domain>` wildcard A record -> VPS IP
4. **Build + push images**:
   - Build `eve-horizon-api`, `eve-horizon-orchestrator`, `eve-horizon-worker`
   - Push to registry and record tags
5. **Update overlay placeholders**:
   - `k8s/overlays/aws/*-patch.yaml`
   - Set `DATABASE_URL`, Ingress host, `EVE_DEFAULT_DOMAIN`, and image names
6. **Create registry secret**:
   - `kubectl -n eve create secret docker-registry eve-registry ...`
7. **Create app secret**:
   - Include `EVE_INTERNAL_API_KEY`, `EVE_SECRETS_MASTER_KEY`, Claude token, registry creds
8. **Apply manifests**:
   - `kubectl apply -k k8s/overlays/aws`
9. **Smoke checks**:
   - `curl -f https://api.<domain>/health`
   - Check logs if any step fails

## Required Confirmations (ask explicitly)

- DNS changes applied and propagated
- Registry credentials verified
- External database reachable from the VPS

## Output

Provide:
- Applied overlay path and any edits made
- Deployed image tags
- Health check results and any follow-up actions

## Recursive skill distillation

- Capture new AWS quirks (security groups, storage class defaults, ingress changes).
- If EKS steps become stable, split to a dedicated EKS skill.
