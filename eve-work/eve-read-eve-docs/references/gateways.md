# Gateway Plugins (Current)

## Overview

The gateway plugin architecture provides a unified interface for external messaging platforms to interact with Eve agents. Each provider implements a standard contract and registers itself with the provider registry.

## Transport Models

| Model | Mechanism | Example |
|-------|-----------|---------|
| **Webhook** | Platform sends HTTP POST to Eve endpoint | Slack |
| **Subscription** | Eve connects to external relay and listens for events | Nostr |

Webhook providers receive inbound messages via HTTP. Subscription providers maintain a persistent connection to an external service and process messages as they arrive.

## Provider Interface Contract

Every gateway provider implements:

| Method | Purpose |
|--------|---------|
| `name` | Unique provider identifier (e.g., `slack`, `nostr`) |
| `transport` | `webhook` or `subscription` |
| `capabilities` | Feature flags (threads, reactions, file uploads, etc.) |
| `validate(request)` | Verify inbound request authenticity (signatures, tokens) |
| `parse(request)` | Extract normalized message from provider-specific payload |
| `send(channel, message)` | Deliver a response back to the platform |

## Providers

### Slack (Webhook)

- **Webhook endpoint**: `POST /gateway/providers/slack/webhook`
- **Legacy endpoint**: `POST /integrations/slack/events` (preserved for existing installations)
- Validates requests using Slack signing secret
- Parses Events API payloads (message, app_mention, etc.)
- Sends responses via Slack Web API (`chat.postMessage`)

### Nostr (Subscription)

- **Transport**: Relay subscription (no inbound webhook)
- **Message types**: Kind 4 (encrypted DMs), Kind 1 (public mentions)
- **Encryption**: NIP-04 for DM content
- Connects to configured relays and subscribes to events targeting the platform pubkey
- Sends responses as signed Nostr events published back to relays

## Gateway Discovery Policy

Agents must opt in to be visible and routable from chat gateways. See `agents-teams.md` for full details.

| Policy | Directory visible | Direct chat | Internal dispatch |
|--------|-------------------|-------------|-------------------|
| `none` | No | Rejected | Always works |
| `discoverable` | Yes | Rejected (hint to use route) | Always works |
| `routable` | Yes | Works | Always works |

The directory endpoint (`GET /internal/orgs/{org_id}/agents`) filters out `none` agents and supports `?client=slack` to further filter by provider.

Slug-based routing (`POST /internal/orgs/{org_id}/chat/route`) enforces:
1. Agent must be `routable` (not `none` or `discoverable`)
2. If `gateway_clients` is set, request provider must be in the list

## Agent Slug Extraction

Each provider defines how to extract the target agent slug from an inbound message:

| Provider | Pattern | Example |
|----------|---------|---------|
| Slack | `@eve <agent-slug> <text>` or channel-bound agent | `@eve mission-control review PR` |
| Nostr | `/agent-slug <text>` or `agent-slug: <text>` | `/mission-control review PR` |

If no slug is matched, the org's default agent is used as fallback.

## Thread Key Format

Thread continuity across providers uses a canonical key:

```
provider:account_id:channel[:thread_id]
```

Examples:
- Slack: `slack:T123ABC:C456DEF:1234567890.123456`
- Nostr: `nostr:<platform-pubkey>:<sender-pubkey>`

The thread key uniquely identifies a conversation and maps to Eve's internal thread system for context persistence.

## Provider Registry and Factory

Providers register themselves at startup via the provider registry. The factory resolves the correct provider by name when:

1. An inbound webhook arrives at the generic controller
2. A subscription event is received
3. An outbound message needs delivery

### Generic Webhook Controller

```
POST /gateway/providers/:provider/webhook
```

The controller looks up the provider by the `:provider` path parameter, delegates to `validate()` and `parse()`, then routes the normalized message to the agent resolution pipeline.
