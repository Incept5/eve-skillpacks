# Integrations Reference

External provider integrations (Slack, GitHub) and identity resolution for Eve orgs.

## Overview

Integrations connect external providers to Eve events and chat routing. Each integration is **org-scoped** — one row per provider account (e.g., one Slack workspace per org).

Key tables:

| Table | Purpose |
|-------|---------|
| `integrations` | Maps provider accounts to orgs |
| `external_identities` | Maps provider user IDs to Eve users |
| `membership_requests` | Tracks pending/approved/denied access requests |

## Slack Integration

### Connect

```bash
# Connect Slack workspace to org
eve integrations slack connect \
  --org <org_id> \
  --team-id <T-ID> \
  --token xoxb-...

# Full bootstrap with all tokens
eve integrations slack connect \
  --org <org_id> \
  --team-id <T-ID> \
  --tokens-json '{"access_token":"xoxb-...","bot_user_id":"U...","team_id":"T...","app_id":"A..."}'

# Verify
eve integrations list --org <org_id>
eve integrations test <integration_id> --org <org_id>
```

### Routing

- `@eve <agent-slug> <command>` — resolves slug to project/agent (unique per org)
- If first word after `@eve` is not a known slug → routes to org `default_agent_slug`
- Channel messages without mention → dispatched to channel/thread listeners

### Auth

- **Signing secret**: Verifies inbound webhook signatures (`EVE_SLACK_SIGNING_SECRET`)
- **Bot token**: `xoxb-...` for outbound messages, stored in integration `tokens_json`

### Required Bot Events

| Event | Purpose |
|-------|---------|
| `app_mention` | `@eve` commands |
| `message.channels` | Public channel listeners |
| `message.groups` | Private channel listeners |
| `message.im` | Direct messages |

### Integration Settings

```bash
# Set admin notification channel for membership requests
eve integrations update <integration_id> --org <org_id> \
  --setting admin_channel_id=C-ADMIN-CHANNEL
```

## GitHub Integration

```bash
# Set up GitHub integration for a project
eve github setup
```

- Webhook endpoint: `/integrations/github/events/:projectId`
- Auth: `EVE_GITHUB_WEBHOOK_SECRET` + project-scoped secret override
- Events: Push, pull request, and configured GitHub webhook events trigger Eve pipelines and workflows

## Identity Resolution

When an external user (e.g., Slack) messages `@eve`, the platform resolves their identity through three tiers. The first match short-circuits the rest.

### Tier 1: Email Auto-Match

The gateway fetches the provider user's email and checks if an Eve user with that email exists and is an org member. If so, the identity is automatically bound. No user action required.

### Tier 2: Self-Service CLI Link

An existing Eve user can link their external identity:

```bash
eve identity link slack --org <org_id>
```

Generates a one-time token. User sends `@eve link <token>` in Slack. The gateway validates and binds.

### Tier 3: Admin Approval

When neither Tier 1 nor 2 resolves, a **membership request** is created.

Admins handle requests via:
- **CLI**: `eve org membership-requests list --org <org_id>`
- **Slack**: Block Kit Approve/Deny buttons (if `admin_channel_id` configured)

On approval: Eve user created (if needed), org membership added, identity bound, user notified.

### Resolution Decision Table

| Has Eve email? | Is org member? | Result |
|----------------|---------------|--------|
| Yes | Yes | Tier 1: auto-bind |
| Yes | No | Tier 3: membership request |
| No / unknown | -- | Tier 2 (self-link) or Tier 3 |

### Membership Request CLI

```bash
eve org membership-requests list --org <org_id>
eve org membership-requests approve <request_id> --org <org_id>
eve org membership-requests deny <request_id> --org <org_id>
```

## External Identities

External identities map provider user IDs to Eve users. Once bound, subsequent messages skip resolution entirely.

Lifecycle:
1. **Created** when provider user first seen (`eve_user_id` = null)
2. **Bound** when resolution succeeds (`eve_user_id` set)
3. **Unbound** if Eve user deleted (returns to unresolved)

## CLI Quick Reference

| Command | Purpose |
|---------|---------|
| `eve integrations list --org <org>` | List integrations |
| `eve integrations test <id> --org <org>` | Test integration health |
| `eve integrations slack connect --org <org> --team-id <id> --token <token>` | Connect Slack |
| `eve integrations update <id> --org <org> --setting key=value` | Update settings |
| `eve identity link slack --org <org>` | Self-service identity link |
| `eve org membership-requests list --org <org>` | List pending requests |
| `eve org membership-requests approve <id> --org <org>` | Approve request |
| `eve org membership-requests deny <id> --org <org>` | Deny request |
| `eve github setup` | GitHub webhook setup |

## Related Skills

- **Chat gateway details**: `references/gateways.md`
- **Auth and access control**: `references/secrets-auth.md`
- **App SSO integration**: `references/auth-sdk.md`
