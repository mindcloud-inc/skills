---
name: mindcloud-universal-api
description: Read and write data in 3,400+ business apps (Slack, NetSuite, Shopify, QuickBooks, HubSpot, and more) through one REST API. Use when the user asks you to fetch or change data in any external app. Covers app discovery, action schemas, lookup resolution, and running actions with curl. Requires a MindCloud API key.
homepage: https://mindcloud.co/docs/universal/rest/home/latest
metadata: {"source": "https://github.com/mindcloud-inc/skills/tree/main/skills/mindcloud-universal-api", "requires": {"env": ["MINDCLOUD_API_KEY"]}}
---

# MindCloud Universal API

One REST API for every app MindCloud supports. You discover an app, read an action's schema, then run the action. MindCloud handles provider auth, tokens, and API quirks behind one contract.

- Base URL: `https://connect.mindcloud.co`
- Auth: send `Authorization: Bearer $MINDCLOUD_API_KEY` on every request.
- Organizations: calls act in the key's own organization. To act in another one the key can reach, find it with `GET /v2/companies?name[contains]=<name>&fields=id,name,access` and send its `id` as `X-Company-Id` on every call; if it is not listed, stop and say so, never fall back to the key's own: https://mindcloud.co/docs/api/rest/introduction/organizations
- Every response is JSON: `{"success": true, "data": ...}` or `{"success": false, "error": {"code", "message", "details"}}`. `error.code` is a small fixed set; `error.details.reason` names the specific cause.
- Rate limits per key: 240 reads/min, 60 writes/min. Watch `X-RateLimit-Remaining`; on 429, wait `Retry-After` seconds.

## Setup

Get an API key at https://app.mindcloud.co/user/api-keys and export it:

```bash
export MINDCLOUD_API_KEY=your_key_here
```

Key access levels: `read_only` can discover apps and schemas. `run_workflows` or `full_access` can also resolve lookups and run actions. Verify the key works:

```bash
curl -s https://connect.mindcloud.co/v2/me \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

## When the user is missing something

Stop and send the user to the right page instead of retrying:

- No MindCloud account: sign up at https://app.mindcloud.co
- Account but no API key, or every request returns 401 (wrong or revoked key): create one at https://app.mindcloud.co/user/api-keys
- API key but the app is not connected (`NO_CONNECTION` error, or the app never appears in `/v2/connections`): connect it at https://app.mindcloud.co/credentials, then retry the request.

## Hard rules

1. Never guess an `appSlug` or `actionSlug`. Discover them first (steps 1 and 2 below).
2. Read the action schema before you run an action. It tells you the required arguments.
3. Never guess provider IDs (channel IDs, customer IDs). Resolve them with the lookup endpoint.
4. Send only documented parameters and body fields. Unknown ones return `400 VALIDATION_FAILED` naming the bad one, with the accepted names in `error.details.accepted`.
5. The run endpoint takes a JSON body only. It accepts no query parameters.
6. Read actions are safe. Before you run an action that creates, updates, or deletes data, confirm with the user.

## Core workflow

Five steps. Skip steps you already have answers for.

**1. Find the app:**

```bash
curl -s "https://connect.mindcloud.co/v2/universal/apps?q=slack" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

Returns `slug`, `id`, `publishedVersion`, `actionsTotal`, and a preview of action slugs per app. You can also pass `actionQ=send a message` to describe your intent; when one action clearly matches, the response includes `meta.suggestedAction`.

**2. List the app's actions:**

```bash
curl -s "https://connect.mindcloud.co/v2/universal/apps/slack/actions?q=message" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

Compact by default, with required arguments included. Add `verbosity=full` or `includeArguments=all` for more detail.

**3. Get the action schema:**

```bash
curl -s "https://connect.mindcloud.co/v2/universal/apps/slack/actions/send-channel-message" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

Returns `requiredArguments`, `optionalArguments`, `responseFields`, and `supports` (which response controls the action honors).

**4. Resolve lookup arguments.** When an argument needs a provider ID, resolve it by name instead of guessing:

```bash
curl -s "https://connect.mindcloud.co/v2/universal/apps/slack/actions/send-channel-message/lookups/channel?q=general" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

Check `data.status`. When it is `resolved`, use `data.selected.value`. When it is `ambiguous`, pick from `data.candidates` or ask the user. `notFound` means nothing matched `q` (`search` is accepted as the same input).

**5. Run the action:**

```bash
curl -s -X POST "https://connect.mindcloud.co/v2/universal/apps/slack/actions/send-channel-message/run" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"arguments": {"channel": "C0123456789", "text": "Hello from an agent"}}'
```

Action arguments go inside `arguments`. Response controls (`fields`, `limit`, `offset`, `sort`, `where`) go at the top level of the body, next to `arguments`, and only when the schema's `supports` block lists them.

## Endpoint reference

<!-- BEGIN GENERATED ENDPOINT REFERENCE: edit the endpoint docs in projects/publicApi/v2, then run npm run docs:website:generate there -->

Each endpoint links to its full docs: parameters, body, response fields, and errors.

### Universal

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Universal Apps](https://mindcloud.co/docs/api/rest/actions/list-universal-apps) | GET `/v2/universal/apps` | Any key | Find published apps with Universal API actions. |
| [List Universal Actions](https://mindcloud.co/docs/api/rest/actions/list-universal-actions) | GET `/v2/universal/apps/{appSlug}/actions` | Any key | List the published actions for one app. |
| [Get Universal Action](https://mindcloud.co/docs/api/rest/actions/get-universal-action) | GET `/v2/universal/apps/{appSlug}/actions/{actionSlug}` | Any key | Get one published action schema, including its arguments, response fields, and supported controls. |
| [Resolve Universal Lookup](https://mindcloud.co/docs/api/rest/actions/resolve-universal-lookup) | GET `/v2/universal/apps/{appSlug}/actions/{actionSlug}/lookups/{argumentKey}` | `universal:read` | Resolve choices for one action argument, such as finding Slack channels before sending a message. |
| [Run Universal Action](https://mindcloud.co/docs/api/rest/actions/run-universal-action) | POST `/v2/universal/apps/{appSlug}/actions/{actionSlug}/run` | `universal:read`, `universal:create`, `universal:update`, `universal:delete`: the one matching the action's operation | Run one published Universal API action through a connection. |

<!-- END GENERATED ENDPOINT REFERENCE -->

`version` defaults to `latest` everywhere; omit it unless you need an older app version.

## Connections

A connection is the company's stored credential for an app. You can omit `connectionId` from run and lookup requests when the app has a default connection or exactly one valid connection; MindCloud selects it. Otherwise pass one:

```bash
# The app id comes from step 1 (/v2/universal/apps)
curl -s "https://connect.mindcloud.co/v2/connections?appId=123&status=valid" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

Use the connection's `id` as `connectionId` in the run body: `{"connectionId": "eQfDE3Jk1GT7", "arguments": {...}}`. A connection's `status` tells you if it is usable; `valid` means yes.

## Errors

`error.code` says what kind of failure it is; `error.details.reason` says which.

| Code                                          | Reason                                                                 | What to do                                                                                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `VALIDATION_FAILED`                           | `UNKNOWN_PARAM`, `UNKNOWN_BODY_KEY`, ...                               | `details.accepted` lists the accepted names; fix and retry                                                                            |
| `NOT_FOUND`                                   | `UNIVERSAL_APP_NOT_FOUND`                                              | Re-run app discovery with `q`                                                                                                         |
| `NOT_FOUND`                                   | `UNIVERSAL_ACTION_NOT_FOUND`                                           | Re-run action discovery with `q`                                                                                                      |
| `CONFLICT`                                    | `UNIVERSAL_ACTION_SLUG_CONFLICT`                                       | Slug matches several actions; pick one and retry                                                                                      |
| `VALIDATION_FAILED`                           | `NO_CONNECTION`                                                        | Send the user to https://app.mindcloud.co/credentials to connect it                                                                   |
| `VALIDATION_FAILED`                           | `CONNECTION_REQUIRED`                                                  | List connections (above) and pass `connectionId`                                                                                      |
| `FORBIDDEN`                                   | `API_KEY_ACCESS_DENIED`                                                | Use a `run_workflows` or `full_access` key                                                                                            |
| `FORBIDDEN`                                   | `CONNECTION_ACCESS_DENIED`                                             | Another member's personal connection, or limited to roles the key's creator lacks; use a shared or own connection                     |
| `RATE_LIMITED`                                | none                                                                   | Wait `Retry-After` seconds, then retry                                                                                                |
| `PROVIDER_ERROR`                              | the provider's own code                                                | The app itself refused or failed (502); the message carries its error and `details.provider.status` its HTTP status; adjust arguments |
| `NOT_FOUND`, `FORBIDDEN`, `VALIDATION_FAILED` | `CONNECTION_NOT_FOUND`, `CONNECTION_APP_MISMATCH`, `INVALID_OPERATION` | MindCloud refused the run before calling the app; fix the connection or the request                                                   |

## Learn more

- Quickstart: https://mindcloud.co/docs/universal/rest/home/latest/introduction/quickstart
- Per-app action docs with runnable examples: https://mindcloud.co/docs/universal/rest/home/latest/introduction/browse-apps
- Machine-readable spec (all v2 routes, no auth needed): https://connect.mindcloud.co/v2/openapi.json
- Building and managing workflows, runs, members, and keys: https://mindcloud.co/skills/mindcloud-api/SKILL.md
