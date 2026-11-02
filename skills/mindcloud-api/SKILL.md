---
name: mindcloud-api
description: Manage a MindCloud account over REST. Use when the user asks you to build, run, or manage MindCloud workflows, inspect workflow runs and debug failures, manage app connections, members, roles, environments, or API keys, or check task usage. Requires a MindCloud API key.
homepage: https://mindcloud.co/docs/api/rest/introduction
metadata: {"source": "https://github.com/mindcloud-inc/skills/tree/main/skills/mindcloud-api", "requires": {"env": ["MINDCLOUD_API_KEY"]}}
---

# MindCloud API

REST API for a MindCloud company account: building and running workflows, runs, connections, members, roles, environments, usage, and API keys. To call third-party apps (Slack, NetSuite, Shopify) directly, use the Universal API skill instead: https://mindcloud.co/skills/mindcloud-universal-api/SKILL.md

- Base URL: `https://connect.mindcloud.co`
- Auth: send `Authorization: Bearer $MINDCLOUD_API_KEY` on every request. A key belongs to one company and acts on it. To act on a child organization of that company, add `X-Company-Id: <childCompanyId>`: https://mindcloud.co/docs/api/rest/introduction/child-organizations
- Every response is JSON: `{"success": true, "data": ...}` or `{"success": false, "error": {"code", "message", "details"}}`. `error.code` is a small fixed set; `error.details.reason` names the specific cause. Lists add `meta.pagination` with `totalCount`, `hasNextPage`, `hasPreviousPage`.
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

1. Reads are safe. Before a write (POST, PUT, DELETE), confirm with the user. Testing a workflow step is a write: it runs the step for real.
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

Bracket operators: `ne`, `gt`, `gte`, `lt`, `lte`, `contains`, `starts`, `ends`, `in`, `nin` (comma lists for the last two). In `contains`, `starts`, and `ends`, `*` is the only wildcard. A `where=` parameter accepts RSQL for advanced filters; `;` (AND) binds tighter than `,` (OR). Details: https://mindcloud.co/docs/api/rest/introduction/requests

Use only names the endpoint lists. An unknown parameter, filter, sort or `fields` name, body field, or a value the field cannot hold returns `400 VALIDATION_FAILED`, and `error.details.accepted` lists what works: read it and retry. (The long-standing workflow, run, usage, and `/v2/me` reads still ignore unknown names for now.)

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

**Build a workflow.** Every change goes to the workflow's draft; nothing runs until you publish. Draft writes take the draft's revision in `If-Match` and answer the next one in `meta.revision`; a stale revision answers `412` with the current one in `error.details.revision`.

```bash
# 1. Create it (starts disabled; "type": "function" makes one other workflows call)
curl -s -X POST "https://connect.mindcloud.co/v2/workflows" \
  -H "Authorization: Bearer $MINDCLOUD_API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "Sync new orders"}'

# 2. Read the draft's revision (data.revision)
curl -s "https://connect.mindcloud.co/v2/workflows/{workflowId}/definition" ...

# 3. Add a trigger (app: schedule, webhook, runWorkflow, or an app slug), then a step (app and action slugs from the Universal API's app and action search)
curl -s -X POST "https://connect.mindcloud.co/v2/workflows/{workflowId}/triggers" ... \
  -H 'If-Match: "{revision}"' -d '{"app": "schedule"}'
curl -s -X POST "https://connect.mindcloud.co/v2/workflows/{workflowId}/steps" ... \
  -H 'If-Match: "{revision}"' -d '{"app": "slack", "action": "send-message"}'

# 4. Read what a step takes, then set it; values can reference a trigger's or an earlier step's output
curl -s "https://connect.mindcloud.co/v2/workflows/{workflowId}/steps/{stepId}/schema" ...
curl -s -X PUT "https://connect.mindcloud.co/v2/workflows/{workflowId}/steps/{stepId}/configuration" ... \
  -H 'If-Match: "{revision}"' -d '{"arguments": {"text": "New order {{ steps.<stepId>.output[0].id }}"}, "connection": {"mode": "connection", "connectionId": "{connectionId}"}}'

# 5. Test a step (real side effects), check the draft, publish, then enable
curl -s -X POST "https://connect.mindcloud.co/v2/workflows/{workflowId}/steps/{stepId}/test" ...
curl -s -X POST "https://connect.mindcloud.co/v2/workflows/{workflowId}/validate" ...
curl -s -X POST "https://connect.mindcloud.co/v2/workflows/{workflowId}/publish" ... -H 'If-Match: "{revision}"' -d '{}'
curl -s -X PUT "https://connect.mindcloud.co/v2/workflows/{workflowId}" ... -d '{"isActive": true}'
```

Testing a step runs it for real (a create step creates), so confirm with the user first. The endpoint reference below links each route's docs for exact bodies.

**Check connection health:**

```bash
curl -s "https://connect.mindcloud.co/v2/connections?status[ne]=valid" ...   # find broken connections
curl -s -X POST "https://connect.mindcloud.co/v2/connections/{connectionId}/test" ...  # live test (full_access)
```

Deleting a connection that workflows still use returns `409` with reason `CONNECTION_IN_USE` and the blocking workflows in `error.details.workflows`.

## Endpoint reference

<!-- BEGIN GENERATED ENDPOINT REFERENCE: edit the endpoint docs in projects/publicApi/v2, then run npm run docs:website:generate there -->

Each endpoint links to its full docs: parameters, body, response fields, and errors.

### Workflows

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Workflows](https://mindcloud.co/docs/api/rest/actions/list-workflows) | GET `/v2/workflows` | `workflow:view` | See every workflow in your company. |
| [Create Workflow](https://mindcloud.co/docs/api/rest/actions/create-workflow) | POST `/v2/workflows` | `workflow:create` | Create an empty workflow. |
| [Get Workflow](https://mindcloud.co/docs/api/rest/actions/get-workflow) | GET `/v2/workflows/{workflowId}` | `workflow:view` | Fetch one workflow, including the full definition your team built and its summary. |
| [Update Workflow](https://mindcloud.co/docs/api/rest/actions/update-workflow) | PUT `/v2/workflows/{workflowId}` | `workflow:edit` | Rename a workflow, enable or disable it with isActive, or replace its tags. |
| [Delete Workflow](https://mindcloud.co/docs/api/rest/actions/delete-workflow) | DELETE `/v2/workflows/{workflowId}` | `workflow:delete` | Delete a workflow. |
| [Duplicate Workflow](https://mindcloud.co/docs/api/rest/actions/duplicate-workflow) | POST `/v2/workflows/{workflowId}/duplicate` | `workflow:create` | Copy a workflow, including its steps and schedules. |
| [List Workflow Versions](https://mindcloud.co/docs/api/rest/actions/list-workflow-versions) | GET `/v2/workflows/{workflowId}/versions` | `workflow:view` | See a workflow's history: the current draft plus every published and unpublished version, newest first. |

### Workflow Runs

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [Run Workflow](https://mindcloud.co/docs/api/rest/actions/run-workflow) | POST `/v2/workflows/{workflowId}/run` | `workflow:run` | Start a run and get its id back right away. |
| [Cancel Run](https://mindcloud.co/docs/api/rest/actions/cancel-run) | POST `/v2/runs/{runId}/cancel` | `workflow:run` | Ask a run to stop. |
| [List Runs](https://mindcloud.co/docs/api/rest/actions/list-runs) | GET `/v2/workflows/{workflowId}/runs` | `log:view` | See every time a workflow ran. |
| [Get Run](https://mindcloud.co/docs/api/rest/actions/get-run) | GET `/v2/runs/{runId}` | `log:view` | Check one run: status, timing, task count, and, while it is still running, the step it is on. |
| [List Run Operations](https://mindcloud.co/docs/api/rest/actions/list-run-operations) | GET `/v2/runs/{runId}/operations` | `log:view` | See every record a run created or updated, and every error or warning it hit. |
| [List Run Steps](https://mindcloud.co/docs/api/rest/actions/list-run-steps) | GET `/v2/runs/{runId}/steps` | `log:view` | See which steps a run executed and whether each one succeeded, without downloading any payloads. |
| [Get a Run Step](https://mindcloud.co/docs/api/rest/actions/get-run-step) | GET `/v2/runs/{runId}/steps/{stepInstanceId}` | `log:view` | Inspect one step of a run: the exact data it received and the exact data it returned. |

### Workflow Building

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [Get Workflow Definition](https://mindcloud.co/docs/api/rest/actions/get-workflow-definition) | GET `/v2/workflows/{workflowId}/definition` | `workflow:view` | Read a workflow's whole structure in one call: its properties, its triggers, and its steps as a tree in execution order. |
| [List Workflow Steps](https://mindcloud.co/docs/api/rest/actions/list-workflow-steps) | GET `/v2/workflows/{workflowId}/steps` | `workflow:view` | List a workflow's steps flat, in the order they run, with the number the editor shows and where each one sits. |
| [Get Workflow Step](https://mindcloud.co/docs/api/rest/actions/get-workflow-step) | GET `/v2/workflows/{workflowId}/steps/{stepId}` | `workflow:view` | Read one step: what it runs, the number the editor shows, and where it sits in the workflow. |
| [List Workflow Triggers](https://mindcloud.co/docs/api/rest/actions/list-workflow-triggers) | GET `/v2/workflows/{workflowId}/triggers` | `workflow:view` | List what starts a workflow. |
| [Add Workflow Step](https://mindcloud.co/docs/api/rest/actions/add-workflow-step) | POST `/v2/workflows/{workflowId}/steps` | `workflow:edit` | Add a step to the workflow's draft. |
| [Update Workflow Step](https://mindcloud.co/docs/api/rest/actions/update-workflow-step) | PUT `/v2/workflows/{workflowId}/steps/{stepId}` | `workflow:edit` | Change a step in the workflow's draft: its name, whether it runs, its action, or its app version. |
| [Get Workflow Step Schema](https://mindcloud.co/docs/api/rest/actions/get-workflow-step-schema) | GET `/v2/workflows/{workflowId}/steps/{stepId}/schema` | `workflow:view` | Read what a step's configuration takes before you set it: its action's arguments as a tree with their types, choices and required ones; the options the action supports and their limits; the connection it needs; and the output fields later steps can reference. |
| [Get Workflow Step Configuration](https://mindcloud.co/docs/api/rest/actions/get-workflow-step-configuration) | GET `/v2/workflows/{workflowId}/steps/{stepId}/configuration` | `workflow:view` | Read how a step in the workflow's draft is set up: the arguments its action gets, nested as the action's schema nests them, with {{ steps.<stepId>.output... |
| [Update Workflow Step Configuration](https://mindcloud.co/docs/api/rest/actions/update-workflow-step-configuration) | PUT `/v2/workflows/{workflowId}/steps/{stepId}/configuration` | `workflow:edit` | Change how a step in the workflow's draft is set up. |
| [Validate Workflow Step Configuration](https://mindcloud.co/docs/api/rest/actions/validate-workflow-step-configuration) | POST `/v2/workflows/{workflowId}/steps/{stepId}/configuration/validate` | `workflow:view` | Send what you would send to Update Workflow Step Configuration and see what it would do: valid is false with every error in issues, or true with the configuration it would leave and the required values still missing. |
| [Run Workflow Step Lookup](https://mindcloud.co/docs/api/rest/actions/run-workflow-step-lookup) | POST `/v2/workflows/{workflowId}/steps/{stepId}/lookups` | `workflow:edit` | List the choices of one dropdown of a step in the workflow's draft, or find the one you mean with search, exactly as the editor loads them: the app's fixed options, a preset list, or a read from the app with the step's connection at its app version. |
| [Test Workflow Step](https://mindcloud.co/docs/api/rest/actions/test-workflow-step) | POST `/v2/workflows/{workflowId}/steps/{stepId}/test` | `workflow:run`, `workflow:edit` | Run one step of the workflow's draft on its own, exactly as Run in the editor does: with its saved configuration and connection, the saved sample outputs of the steps it references, and real side effects (a create step creates). |
| [Get Workflow Step Output](https://mindcloud.co/docs/api/rest/actions/get-workflow-step-output) | GET `/v2/workflows/{workflowId}/steps/{stepId}/output` | `workflow:view` | Read the sample output saved for a step in the workflow's draft, as the editor's Output tab shows it: its first records, how many there were, and the fields later steps can reference. |
| [Update Workflow Step Output](https://mindcloud.co/docs/api/rest/actions/update-workflow-step-output) | PUT `/v2/workflows/{workflowId}/steps/{stepId}/output` | `workflow:edit` | Save a sample output for a step in the workflow's draft without running it, as Edit Step Output in the editor does. |
| [Delete Workflow Step](https://mindcloud.co/docs/api/rest/actions/delete-workflow-step) | DELETE `/v2/workflows/{workflowId}/steps/{stepId}` | `workflow:edit` | Remove a step from the workflow's draft. |
| [Duplicate Workflow Step](https://mindcloud.co/docs/api/rest/actions/duplicate-workflow-step) | POST `/v2/workflows/{workflowId}/steps/{stepId}/duplicate` | `workflow:edit` | Copy a step and every step inside it, with their connections: each must be one the key's creator may use. |
| [Set Workflow Step Order](https://mindcloud.co/docs/api/rest/actions/set-workflow-step-order) | PUT `/v2/workflows/{workflowId}/step-order` | `workflow:edit` | Move any number of steps in one change: send the whole tree as you want it, the same shape Get Workflow Definition answers, with ids only. |
| [Add Workflow Trigger](https://mindcloud.co/docs/api/rest/actions/add-workflow-trigger) | POST `/v2/workflows/{workflowId}/triggers` | `workflow:edit` | Add a trigger to the workflow's draft. |
| [Update Workflow Trigger](https://mindcloud.co/docs/api/rest/actions/update-workflow-trigger) | PUT `/v2/workflows/{workflowId}/triggers/{triggerId}` | `workflow:edit` | Rename a trigger in the workflow's draft, or turn it off or on. |
| [Get Workflow Trigger Schema](https://mindcloud.co/docs/api/rest/actions/get-workflow-trigger-schema) | GET `/v2/workflows/{workflowId}/triggers/{triggerId}/schema` | `workflow:view` | Read what a trigger's configuration takes before you set it: its action's arguments as a tree with their types, choices and required ones; the options the action supports and their limits; the connection it needs; and the output fields later steps can reference. |
| [Get Workflow Trigger Configuration](https://mindcloud.co/docs/api/rest/actions/get-workflow-trigger-configuration) | GET `/v2/workflows/{workflowId}/triggers/{triggerId}/configuration` | `workflow:view` | Read how a trigger in the workflow's draft is set up: the arguments its action gets, nested as the action's schema nests them, with {{ steps.<stepId>.output... |
| [Update Workflow Trigger Configuration](https://mindcloud.co/docs/api/rest/actions/update-workflow-trigger-configuration) | PUT `/v2/workflows/{workflowId}/triggers/{triggerId}/configuration` | `workflow:edit` | Change how a trigger in the workflow's draft is set up. |
| [Validate Workflow Trigger Configuration](https://mindcloud.co/docs/api/rest/actions/validate-workflow-trigger-configuration) | POST `/v2/workflows/{workflowId}/triggers/{triggerId}/configuration/validate` | `workflow:view` | Send what you would send to Update Workflow Trigger Configuration and see what it would do: valid is false with every error in issues, or true with the configuration it would leave and the required values still missing. |
| [Run Workflow Trigger Lookup](https://mindcloud.co/docs/api/rest/actions/run-workflow-trigger-lookup) | POST `/v2/workflows/{workflowId}/triggers/{triggerId}/lookups` | `workflow:edit` | List the choices of one dropdown of a trigger in the workflow's draft, or find the one you mean with search, exactly as the editor loads them: the app's fixed options, a preset list, or a read from the app with the trigger's connection at its app version. |
| [Listen Workflow Trigger](https://mindcloud.co/docs/api/rest/actions/listen-workflow-trigger) | POST `/v2/workflows/{workflowId}/triggers/{triggerId}/listen` | `workflow:edit` | Wait for a call to a Webhook trigger in the workflow's draft: send a request to the trigger's URL while this waits. |
| [Delete Workflow Trigger](https://mindcloud.co/docs/api/rest/actions/delete-workflow-trigger) | DELETE `/v2/workflows/{workflowId}/triggers/{triggerId}` | `workflow:edit` | Remove a trigger from the workflow's draft. |
| [Get Workflow Inputs](https://mindcloud.co/docs/api/rest/actions/get-workflow-inputs) | GET `/v2/workflows/{workflowId}/inputs` | `workflow:view` | Read what a run of the workflow takes, where the editor keeps it. |
| [Update Workflow Inputs](https://mindcloud.co/docs/api/rest/actions/update-workflow-inputs) | PUT `/v2/workflows/{workflowId}/inputs` | `workflow:edit` | Replace a whole list, as the editor stores it; a list not sent stays as it is. |
| [Validate Workflow](https://mindcloud.co/docs/api/rest/actions/validate-workflow) | POST `/v2/workflows/{workflowId}/validate` | `workflow:view` | Check the workflow's draft the way publishing will: errors block publishing (an incomplete schedule, a broken graph), warnings do not (a step not set up yet). |
| [Publish Workflow](https://mindcloud.co/docs/api/rest/actions/publish-workflow) | POST `/v2/workflows/{workflowId}/publish` | `workflow:edit` | Make the draft the version that runs. |
| [Restore Workflow Version](https://mindcloud.co/docs/api/rest/actions/restore-workflow-version) | POST `/v2/workflows/{workflowId}/versions/{version}/restore` | `workflow:edit` | Copy a saved version into the draft, replacing it. |
| [List Workflow Tags](https://mindcloud.co/docs/api/rest/actions/list-workflow-tags) | GET `/v2/workflow-tags` | `workflow:view` | The company's workflow tags. |
| [Create Workflow Tag](https://mindcloud.co/docs/api/rest/actions/create-workflow-tag) | POST `/v2/workflow-tags` | `workflow:edit` | Add a tag to the company's catalog. |
| [Update Workflow Tag](https://mindcloud.co/docs/api/rest/actions/update-workflow-tag) | PUT `/v2/workflow-tags/{tagId}` | `workflow:edit` | Rename a tag or change its color. |
| [Delete Workflow Tag](https://mindcloud.co/docs/api/rest/actions/delete-workflow-tag) | DELETE `/v2/workflow-tags/{tagId}` | `workflow:edit` | Delete a tag. |

### Connections

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Connections](https://mindcloud.co/docs/api/rest/actions/list-connections) | GET `/v2/connections` | `credential:view` | See every connection in your company and whether it still works. |
| [Get Connection](https://mindcloud.co/docs/api/rest/actions/get-connection) | GET `/v2/connections/{connectionId}` | `credential:view` | Check one connection's health, expiry, and metadata. |
| [Test Connection](https://mindcloud.co/docs/api/rest/actions/test-connection) | POST `/v2/connections/{connectionId}/test` | `credential:edit` | Check a connection against the provider it signs into. |
| [Delete Connection](https://mindcloud.co/docs/api/rest/actions/delete-connection) | DELETE `/v2/connections/{connectionId}` | `credential:delete` | Delete a connection, revoking it with the provider first when the app supports that. |

### Embedded

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [Create Embedded Connect Link](https://mindcloud.co/docs/api/rest/actions/create-embedded-connect-link) | POST `/v2/embedded/end-users/{endUserId}/connect-links` | `embedded:create` | Create a single-use link for an end user to connect an app account. |
| [List End Users](https://mindcloud.co/docs/api/rest/actions/list-end-users) | GET `/v2/embedded/end-users` | `embedded:view` | See the people using MindCloud Embedded inside your product. |
| [Get End User](https://mindcloud.co/docs/api/rest/actions/get-end-user) | GET `/v2/embedded/end-users/{endUserId}` | `embedded:view` | See one Embedded end user. |
| [Create End User](https://mindcloud.co/docs/api/rest/actions/create-end-user) | POST `/v2/embedded/end-users` | `embedded:create` | Add someone from your product so they can connect their own accounts through Embedded. |
| [Update End User](https://mindcloud.co/docs/api/rest/actions/update-end-user) | PUT `/v2/embedded/end-users/{endUserId}` | `embedded:edit` | Update an end user's id, name, email, or metadata. |
| [Delete End User](https://mindcloud.co/docs/api/rest/actions/delete-end-user) | DELETE `/v2/embedded/end-users/{endUserId}` | `embedded:delete` | Remove an end user once their connections are gone. |
| [List Embedded End User Connections](https://mindcloud.co/docs/api/rest/actions/list-embedded-end-user-connections) | GET `/v2/embedded/end-users/{endUserId}/connections` | `embedded:view` | List the public app accounts owned by one end user. |

### Members

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Members](https://mindcloud.co/docs/api/rest/actions/list-members) | GET `/v2/members` | `member:view` | See who is in your company, the role they hold, and whether their invite is still pending. |
| [Get Member](https://mindcloud.co/docs/api/rest/actions/get-member) | GET `/v2/members/{userId}` | `member:view` | See one member: the role they hold, and whether they have accepted their invite. |
| [Get Member Permissions](https://mindcloud.co/docs/api/rest/actions/get-member-permissions) | GET `/v2/members/{userId}/permissions` | `member:view` | See exactly what one member can do, permission by permission, and why: from their role, a direct grant, or a company setting. |
| [Update Member](https://mindcloud.co/docs/api/rest/actions/update-member) | PUT `/v2/members/{userId}` | `member:edit` | Move a member to another role. |
| [Remove Member](https://mindcloud.co/docs/api/rest/actions/delete-member) | DELETE `/v2/members/{userId}` | `member:delete` | Remove someone from your company. |
| [List Invitations](https://mindcloud.co/docs/api/rest/actions/list-invitations) | GET `/v2/invitations` | `member:view` | See who you invited and has not signed up yet. |
| [Get Invitation](https://mindcloud.co/docs/api/rest/actions/get-invitation) | GET `/v2/invitations/{invitationId}` | `member:view` | See one invitation that has not been accepted yet. |
| [Create Invitation](https://mindcloud.co/docs/api/rest/actions/create-invitation) | POST `/v2/invitations` | `member:create` | Invite someone by email. |
| [Resend Invitation](https://mindcloud.co/docs/api/rest/actions/resend-invitation) | POST `/v2/invitations/{invitationId}/resend` | `member:create` | Send an invitation email again, for someone who lost or never got it. |
| [Revoke Invitation](https://mindcloud.co/docs/api/rest/actions/revoke-invitation) | DELETE `/v2/invitations/{invitationId}` | `member:delete` | Take back an invitation that has not been accepted. |
| [Get Member Spend Limit](https://mindcloud.co/docs/api/rest/actions/get-member-spend-limit) | GET `/v2/members/{userId}/spend-limit` | `subscription:view` | See one member's Cirra credit limit. |
| [Update Member Spend Limit](https://mindcloud.co/docs/api/rest/actions/update-member-spend-limit) | PUT `/v2/members/{userId}/spend-limit` | `subscription:manage` | Cap one member's Cirra credit use, or give them back the default. |
| [Check Authorization](https://mindcloud.co/docs/api/rest/actions/check-authorization) | POST `/v2/authorization/check` | `member:view` | Find out whether someone can do something before they try, and which role, grant, or rule is behind the answer. |

### Roles

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Roles](https://mindcloud.co/docs/api/rest/actions/list-roles) | GET `/v2/roles` | `member:view` | See the roles you can assign, including the global roles MindCloud provides to every company. |
| [Get Role](https://mindcloud.co/docs/api/rest/actions/get-role) | GET `/v2/roles/{roleId}` | `member:view` | See exactly what one role lets its members do, permission by permission. |
| [Create Role](https://mindcloud.co/docs/api/rest/actions/create-role) | POST `/v2/roles` | `role:manage` | Create a custom role with exactly the permissions you choose, then assign it to members. |
| [Update Role](https://mindcloud.co/docs/api/rest/actions/update-role) | PUT `/v2/roles/{roleId}` | `role:manage` | Rename a custom role or replace its permissions. |
| [Delete Role](https://mindcloud.co/docs/api/rest/actions/delete-role) | DELETE `/v2/roles/{roleId}` | `role:manage` | Delete a custom role that no member holds any more. |
| [List Permissions](https://mindcloud.co/docs/api/rest/actions/list-permissions) | GET `/v2/permissions` | Any key | See every permission you can build a role or an API key from, and which of them a custom role may carry. |

### Company

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Companies](https://mindcloud.co/docs/api/rest/actions/list-companies) | GET `/v2/companies` | `company:view` | See your company's profile, plan, and member count. |
| [Get Company](https://mindcloud.co/docs/api/rest/actions/get-company) | GET `/v2/companies/{companyId}` | `company:view` | See your company in full: its region, parent organization, how many members and roles it has, and whether domain discovery and Embedded are on. |
| [Update Company](https://mindcloud.co/docs/api/rest/actions/update-company) | PUT `/v2/companies/{companyId}` | `company:edit` | Change your company name, description, timezone, or logo, or turn domain discovery on or off. |
| [Leave Parent Organization](https://mindcloud.co/docs/api/rest/actions/leave-parent) | POST `/v2/companies/{companyId}/leave-parent` | `company:parent:manage` | Stop being a child organization. |
| [List Environments](https://mindcloud.co/docs/api/rest/actions/list-environments) | GET `/v2/environments` | `company:view` | See your environments, like Production and Sandbox, and how many scheduled workflows target each one. |
| [Create Environment](https://mindcloud.co/docs/api/rest/actions/create-environment) | POST `/v2/environments` | `environment:edit` | Add an environment, like Staging, to run workflows and map connections against. |
| [Update Environment](https://mindcloud.co/docs/api/rest/actions/update-environment) | PUT `/v2/environments/{environmentId}` | `environment:edit` | Rename one of your environments. |
| [Delete Environment](https://mindcloud.co/docs/api/rest/actions/delete-environment) | DELETE `/v2/environments/{environmentId}` | `environment:edit` | Remove an environment nothing uses any more. |

### Child Organizations

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [List Child Organizations](https://mindcloud.co/docs/api/rest/actions/list-child-companies) | GET `/v2/child-companies` | `child_company:access` | See your company's child organizations. |
| [Get Child Organization](https://mindcloud.co/docs/api/rest/actions/get-child-company) | GET `/v2/child-companies/{companyId}` | `child_company:access` | Get one child organization of your company. |
| [Create Child Organization](https://mindcloud.co/docs/api/rest/actions/create-child-company) | POST `/v2/child-companies` | `child_company:manage` | Create a new organization as a child of your company. |
| [Release Child Organization](https://mindcloud.co/docs/api/rest/actions/release-child-company) | DELETE `/v2/child-companies/{companyId}` | `child_company:manage` | End the link to a child organization. |
| [Close Child Organization](https://mindcloud.co/docs/api/rest/actions/close-child-company) | POST `/v2/child-companies/{companyId}/close` | `child_company:manage` | Delete a child organization your company created. |
| [List Link Requests](https://mindcloud.co/docs/api/rest/actions/list-link-requests) | GET `/v2/link-requests` | `child_company:access` | See the requests to link your company with another organization: the ones you sent and the ones waiting for your answer. |
| [Create Link Request](https://mindcloud.co/docs/api/rest/actions/create-link-request) | POST `/v2/link-requests` | `child_company:manage` | Ask an existing organization to become your child organization. |
| [Accept Link Request](https://mindcloud.co/docs/api/rest/actions/accept-link-request) | POST `/v2/link-requests/{requestId}/accept` | `company:parent:manage` | Accept a request to link your company with another organization. |
| [Decline Link Request](https://mindcloud.co/docs/api/rest/actions/decline-link-request) | POST `/v2/link-requests/{requestId}/decline` | `company:parent:manage` | Turn down a request to link your company with another organization. |
| [Cancel Link Request](https://mindcloud.co/docs/api/rest/actions/cancel-link-request) | POST `/v2/link-requests/{requestId}/cancel` | `child_company:manage` | Withdraw a link request your company sent before the other organization answers it. |

### Cirra Governance

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [Get Capability Catalog](https://mindcloud.co/docs/api/rest/actions/get-capability-catalog) | GET `/v2/capabilities` | Any key | See every Cirra capability key a role can allow or deny, its default, and how the keys fall back to each other. |
| [Get Member Capabilities](https://mindcloud.co/docs/api/rest/actions/get-member-capabilities) | GET `/v2/members/{userId}/capabilities` | `member:view` | See what Cirra lets one member do (web search, files, shell, and each app) and whether their role, their own choices, or the default decided it. |
| [Get Role Capabilities](https://mindcloud.co/docs/api/rest/actions/get-role-capabilities) | GET `/v2/roles/{roleId}/capabilities` | `member:view` | See what a role lets its members do in Cirra: web search, files, shell commands, and each app, model, or action. |
| [Update Role Capabilities](https://mindcloud.co/docs/api/rest/actions/update-role-capabilities) | PUT `/v2/roles/{roleId}/capabilities` | `role:manage` | Allow or deny what a role's members may do in Cirra, down to a single app action. |
| [Get Connection Roles](https://mindcloud.co/docs/api/rest/actions/get-connection-roles) | GET `/v2/connections/{connectionId}/roles` | `credential:view` | See which roles can use a shared connection when Cirra runs app actions. |
| [Update Connection Roles](https://mindcloud.co/docs/api/rest/actions/update-connection-roles) | PUT `/v2/connections/{connectionId}/roles` | `role:manage` | Limit a shared connection to certain roles, or open it to every member with an empty list. |
| [List Platforms](https://mindcloud.co/docs/api/rest/actions/list-platforms) | GET `/v2/platforms` | `company:view` | See how people reach Cirra from Teams and Slack, and which roles they get. |
| [Get Platform](https://mindcloud.co/docs/api/rest/actions/get-platform) | GET `/v2/platforms/{platformId}` | `company:view` | See the sign-in settings for Teams or Slack. |
| [Update Platform](https://mindcloud.co/docs/api/rest/actions/update-platform) | PUT `/v2/platforms/{platformId}` | `company:edit` | Set the email domains, guest access, automatic accounts, and group-to-role mappings for Teams or Slack sign-in. |

### API Keys

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [Who Am I?](https://mindcloud.co/docs/api/rest/actions/get-me) | GET `/v2/me` | Any key | Check which API key you are calling with, which company your requests will read from, and which MindCloud capabilities that company has enabled. |
| [List API Keys](https://mindcloud.co/docs/api/rest/actions/list-api-keys) | GET `/v2/api-keys` | `api_key:manage` | Audit your API keys: when each was last used and which have been revoked. |
| [Get API Key](https://mindcloud.co/docs/api/rest/actions/get-api-key) | GET `/v2/api-keys/{apiKeyId}` | `api_key:manage` | See one API key: what it may do, when it was last used, and whether it was revoked. |
| [Update API Key](https://mindcloud.co/docs/api/rest/actions/update-api-key) | PUT `/v2/api-keys/{apiKeyId}` | `api_key:manage` | Rename a key, or change what it may do. |
| [Create API Key](https://mindcloud.co/docs/api/rest/actions/create-api-key) | POST `/v2/api-keys` | `api_key:manage` | Create a new API key for your company. |
| [Revoke API Key](https://mindcloud.co/docs/api/rest/actions/revoke-api-key) | DELETE `/v2/api-keys/{apiKeyId}` | `api_key:manage` | Revoke an API key so it stops working immediately. |
| [List Audit Events](https://mindcloud.co/docs/api/rest/actions/list-audit-events) | GET `/v2/audit-events` | `api_key:manage` | See what your company's API keys changed, and what they were refused. |

### Usage

| Endpoint | Request | Permission | What it does |
| --- | --- | --- | --- |
| [Get Usage Summary](https://mindcloud.co/docs/api/rest/actions/get-usage) | GET `/v2/usage` | `subscription:view` | Check how many tasks you have used this period and how close you are to your limit. |
| [Get Daily Usage](https://mindcloud.co/docs/api/rest/actions/get-usage-daily) | GET `/v2/usage/daily` | `subscription:view` | See how your task usage trends day by day, up to 45 days at a time. |
| [Get Usage by Workflow](https://mindcloud.co/docs/api/rest/actions/get-usage-workflows) | GET `/v2/usage/workflows` | `subscription:view` | Find out which workflows use the most tasks, highest first, and how embedded usage splits across [installations](/docs/embedded/getting-started/mindcloud-embedded-overview#installation). |

<!-- END GENERATED ENDPOINT REFERENCE -->

## Errors

`error.code` says what kind of failure it is; `error.details.reason` says which.

| Code                | Common reasons                                                                         | What to do                                                        |
| ------------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| `VALIDATION_FAILED` | `UNKNOWN_PARAM`, `UNKNOWN_FILTER_FIELD`, `INVALID_FILTER_VALUE`, `NO_FIELDS_TO_UPDATE` | Fix the named field; `details.accepted` lists valid names         |
| `UNAUTHORIZED`      | none                                                                                   | Check the API key                                                 |
| `FORBIDDEN`         | `API_KEY_ACCESS_DENIED`, `MEMBER_ACTOR_NOT_RESOLVED`                                   | Use a higher-access key, or a key whose creator is still a member |
| `NOT_FOUND`         | none                                                                                   | Check the id; list the collection to find it                      |
| `CONFLICT`          | `WORKFLOW_DISABLED`, `WORKFLOW_NOT_PUBLISHED`                                          | Enable it, or run with `{"version": "draft"}`                     |
| `CONFLICT`          | `RUN_NOT_CANCELLABLE`                                                                  | Run already terminal; nothing to do                               |
| `CONFLICT`          | `CONNECTION_IN_USE`                                                                    | `details.workflows` lists them; detach first                      |
| `RATE_LIMITED`      | none                                                                                   | Wait `Retry-After` seconds, then retry                            |

The long-standing workflow, run, usage, and `/v2/me` reads answer the cause as `error.code` itself (`INVALID_QUERY`, `NOT_FOUND`), with no `details`.

## Learn more

- Getting started: https://mindcloud.co/docs/api/rest/introduction
- Authentication and access levels: https://mindcloud.co/docs/api/rest/introduction/authentication
- Query grammar in depth: https://mindcloud.co/docs/api/rest/introduction/requests
- Machine-readable spec: https://connect.mindcloud.co/v2/openapi.json
- Calling third-party apps: https://mindcloud.co/skills/mindcloud-universal-api/SKILL.md
