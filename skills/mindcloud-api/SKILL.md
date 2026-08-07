---
name: mindcloud-api
description: Manage a MindCloud account over REST. Use when the user asks you to run or manage MindCloud workflows, inspect workflow runs and debug failures, manage app connections, members, roles, environments, or API keys, or check task usage. Requires a MindCloud API key.
homepage: https://mindcloud.co/docs/api/rest/introduction
metadata: {"source": "https://github.com/mindcloud-inc/skills/tree/main/skills/mindcloud-api", "requires": {"env": ["MINDCLOUD_API_KEY"]}}
---

# MindCloud API

REST API for a MindCloud company account: workflows, runs, connections, members, roles, environments, usage, and API keys. To call third-party apps (Slack, NetSuite, Shopify), use the Universal API skill instead: https://github.com/mindcloud-inc/skills/tree/main/skills/mindcloud-universal-api

- Base URL: `https://connect.mindcloud.co`
- Auth: send `Authorization: Bearer $MINDCLOUD_API_KEY` on every request. A key belongs to one company; every request is scoped to it.
- Every response is JSON: `{"success": true, "data": ...}` or `{"success": false, "error": {"code", "message"}}`. Lists add `meta.pagination` with `totalCount`, `hasNextPage`, `hasPreviousPage`.
- Rate limits per key: 240 reads/min, 60 writes/min. On 429, wait `Retry-After` seconds.

## Setup

Get an API key at https://app.mindcloud.co/user/api-keys and export it as `MINDCLOUD_API_KEY`. Key access levels:

- `read_only`: all GET routes.
- `run_workflows`: reads, plus run workflow, cancel run, and run universal actions.
- `full_access`: everything, including writes and key management.

Verify the key and see its access level:

```bash
curl -s https://connect.mindcloud.co/v2/me \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

## Hard rules

1. Reads are safe. Before a write (POST, PUT, PATCH, DELETE), confirm with the user.
2. Workflow runs are asynchronous. `POST .../run` returns 202 immediately; poll the run until it reaches a terminal status.
3. An API key's plaintext is shown once, at creation. Store it immediately; you cannot fetch it again.
4. Member and role writes act as the key's creator. They can fail with 403 even on a `full_access` key.

## List query grammar

Every list endpoint accepts the same parameters:

```
?limit=100&offset=0            # limit 1-1000, default 100
&sort=-createdOn,name          # "-" prefix means descending
&fields=id,name,status         # select fields, dot paths allowed
&status=failed                 # plain equality filter
&startedOn[gte]=2026-08-01     # bracket operators
```

Bracket operators: `ne`, `gt`, `gte`, `lt`, `lte`, `contains`, `starts`, `ends`, `in`, `nin` (comma lists for the last two). A `where=` parameter accepts RSQL for advanced filters. Details: https://mindcloud.co/docs/api/rest/introduction/requests

## Common flows

**Run a workflow and wait for the result:**

```bash
# 1. Find the workflow
curl -s "https://connect.mindcloud.co/v2/workflows?name[contains]=sync&isActive=true" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"

# 2. Start it (202; body: version "published" (default) or "draft", optional environmentId)
curl -s -X POST "https://connect.mindcloud.co/v2/workflows/{workflowId}/run" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY" \
  -H "Content-Type: application/json" -d '{}'

# 3. Poll the returned run id until status is terminal
curl -s "https://connect.mindcloud.co/v2/runs/{runId}" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY"
```

Run statuses: `waiting`, `running`, `cancelling` are in flight; `finished`, `failed`, `cancelled`, `interrupted`, `limitExceeded` are terminal. Poll every few seconds.

**Debug a failed run:** drill down from the run to the step that broke.

```bash
curl -s "https://connect.mindcloud.co/v2/workflows/{workflowId}/runs?status=failed&limit=5" ...
curl -s "https://connect.mindcloud.co/v2/runs/{runId}/steps" ...            # flat index, no payloads
curl -s "https://connect.mindcloud.co/v2/runs/{runId}/steps/{stepInstanceId}" ...  # request + response payloads
curl -s "https://connect.mindcloud.co/v2/runs/{runId}/operations?actionStatus=error" ...  # per-record results
```

**Check connection health:**

```bash
curl -s "https://connect.mindcloud.co/v2/connections?status[ne]=valid" ...   # find broken connections
curl -s -X POST "https://connect.mindcloud.co/v2/connections/{connectionId}/test" ...  # live test (full_access)
```

Deleting a connection that workflows still use returns `409 CONNECTION_IN_USE` with the blocking workflows in `error.workflows`.

## Endpoint reference

Tier column: the minimum key access level. Full parameter docs per endpoint: https://mindcloud.co/docs/api/rest/introduction

| Method and path | Tier | Purpose |
|---|---|---|
| GET `/v2/workflows`, GET `/v2/workflows/{id}` | read_only | List and read workflows |
| GET `/v2/workflows/{id}/versions` | read_only | Draft plus published snapshots |
| POST `/v2/workflows` | full_access | Create (body: `name`, optional `tags`) |
| PUT `/v2/workflows/{id}` | full_access | Update `name`, `isActive`, `tags` |
| POST `/v2/workflows/{id}/duplicate` | full_access | Copy; the copy starts disabled |
| DELETE `/v2/workflows/{id}` | full_access | Soft delete and disable |
| POST `/v2/workflows/{id}/run` | run_workflows | Start a run, returns 202 |
| GET `/v2/workflows/{id}/runs` | read_only | Run history, default sort `-startedOn` |
| GET `/v2/runs/{id}` | read_only | Run detail and status |
| POST `/v2/runs/{id}/cancel` | run_workflows | Cancel, returns 202; 409 if already terminal |
| GET `/v2/runs/{id}/steps`, `/steps/{stepInstanceId}` | read_only | Step index; step request and response |
| GET `/v2/runs/{id}/operations` | read_only | Per-record operation log |
| GET `/v2/connections`, `/v2/connections/{id}` | read_only | List and read connections (never secrets) |
| POST `/v2/connections/{id}/test` | full_access | Live credential test |
| DELETE `/v2/connections/{id}` | full_access | Delete; revokes provider-side when supported |
| GET `/v2/members` | read_only | Company members |
| POST `/v2/members` | full_access | Invite (body: `invites` array of `{email, roleId?}`) |
| PUT `/v2/members/{userId}` | full_access | Change role (body: `roleId`) |
| DELETE `/v2/members/{userId}` | full_access | Remove member, cancels pending invites |
| GET `/v2/roles`, `/v2/roles/{id}` | read_only | Roles; detail includes permissions |
| POST, PUT, DELETE `/v2/roles...` | full_access | Manage custom roles and their permissions |
| GET `/v2/companies` | read_only | Companies visible to the key |
| PATCH `/v2/companies/{id}` | full_access | Update own company `name`, `description`, `timezone` |
| GET `/v2/environments` | read_only | Environments for workflow runs |
| GET `/v2/usage`, `/v2/usage/daily`, `/v2/usage/workflows` | read_only | Task usage: 30-day summary, daily, per workflow |
| GET `/v2/api-keys` | read_only | Key audit (no secrets) |
| POST `/v2/api-keys` | full_access | Mint a key (body: `name`, `accessLevel`); plaintext returned once |
| DELETE `/v2/api-keys/{id}` | full_access | Revoke, idempotent |
| GET `/v2/me` | read_only | Current key identity and access level |
| GET `/v2/openapi.json` | none | Full OpenAPI 3.1 spec, no auth |

## Errors

| Code | Meaning | What to do |
|---|---|---|
| `INVALID_QUERY` | Bad parameter or body field | The message names the problem; fix and retry |
| `NOT_FOUND` | No such resource in this company | Check the id; list the collection to find it |
| `API_KEY_ACCESS_DENIED` | Key tier too low for this route | Use a higher-access key |
| `NO_FIELDS_TO_UPDATE` | Empty update body | Send at least one field |
| `WORKFLOW_DISABLED`, `WORKFLOW_NOT_PUBLISHED` | Run blocked by workflow state | Enable it, or run with `{"version": "draft"}` |
| `RUN_NOT_CANCELLABLE` | Run already terminal | Nothing to do |
| `CONNECTION_IN_USE` | Delete blocked by workflows | `error.workflows` lists them; detach first |
| `RATE_LIMITED` | Over the per-key budget | Wait `Retry-After` seconds, then retry |

## Learn more

- Getting started: https://mindcloud.co/docs/api/rest/introduction
- Authentication and access levels: https://mindcloud.co/docs/api/rest/introduction/authentication
- Query grammar in depth: https://mindcloud.co/docs/api/rest/introduction/requests
- Machine-readable spec: https://connect.mindcloud.co/v2/openapi.json
- Calling third-party apps: https://github.com/mindcloud-inc/skills/tree/main/skills/mindcloud-universal-api
