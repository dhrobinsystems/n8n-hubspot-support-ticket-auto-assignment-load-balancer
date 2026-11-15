# n8n-hubspot-support-ticket-auto-assignment-load-balancer

**A support triage workflow that assigns each incoming HubSpot ticket to the agent with the lowest current load, so work spreads evenly and no one is buried.**

I built this as a reference architecture for my solo practice, **dhRobin Systems**. It replaces round-robin, which ignores how much work each agent already has, with capacity-aware load balancing: every new ticket goes to the agent whose open-ticket count is lowest relative to their personal capacity.

> **Safe by default.** The workflow ships with `DRY_RUN=true` and `SANDBOX_MODE=true`. It makes **no** HubSpot writes, and no HubSpot reads at all, until you deliberately turn both off. The file holds no credentials, tokens, portal IDs, or real owner IDs. Every value is a placeholder.

---

## Table of Contents

1. [Why I Built This](#why-i-built-this)
2. [Architecture](#architecture)
3. [The Load-Balancing Algorithm](#the-load-balancing-algorithm)
4. [Repository Structure](#repository-structure)
5. [Prerequisites](#prerequisites)
6. [Setup](#setup)
7. [Environment Variables](#environment-variables)
8. [Node Configuration](#node-configuration)
9. [Operational Workflow](#operational-workflow)
10. [Testing](#testing)
11. [Go-Live Checklist](#go-live-checklist)
12. [Security Model](#security-model)
13. [Troubleshooting](#troubleshooting)
14. [Known Limitations](#known-limitations)
15. [Disclaimer & License](#️-disclaimer--liability-limitation)

---

## Why I Built This

Round-robin assignment looks fair, but it isn't. If one agent is working through a backlog of complex escalations and another has cleared their queue, round-robin still sends them the same number of new tickets. Over a few weeks the gap widens, response times slip, and the most dependable people burn out first.

This workflow measures each agent's **live open-ticket count** in HubSpot at the moment a ticket arrives. It then routes the ticket to the agent with the most headroom, so assignment follows actual workload and not a turn order.

---

## Architecture

```mermaid
flowchart LR
    A[HubSpot<br/>ticket.creation] -->|POST| B[HubSpot Ticket Webhook]
    B -->|200 OK at once| A
    B --> C[Open Ticket Count Check<br/>& Load Balancer<br/><i>Code node</i>]
    C -->|GET ticket state<br/>POST search per agent| H[(HubSpot CRM API)]
    C --> D{Should Assign?<br/>action == assign}
    D -->|true| E[HubSpot Owner Update<br/>PATCH hubspot_owner_id]
    E -->|PATCH| H
    E -->|success| F[Audit Log]
    E -->|error| G[Sanitized Error Handler]
    G --> I[Raise Sanitized Error<br/>→ your Error Workflow]
    D -->|false| J[Dry Run / Skip Report]
```

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **HubSpot Ticket Webhook** | Webhook (v2) | Receives ticket events and acknowledges at once (`200`), so HubSpot never times out or retries because of slow processing. |
| 2 | **Open Ticket Count Check & Load Balancer** | Code (v2) | Authenticates the request, normalises the payload, deduplicates events, checks the ticket's current owner, counts open tickets per agent, and picks the least-loaded eligible agent. |
| 3 | **Should Assign?** | IF (v2) | Lets only `action = "assign"` through to the write step. |
| 4 | **HubSpot Owner Update** | HTTP Request (v4.2) | `PATCH /crm/v3/objects/tickets/{id}` with `hubspot_owner_id`. Retries 3× and sends errors to a separate output. |
| 5 | **Audit Log** | Code | Builds a compact assignment record with no secrets in it. |
| 6 | **Sanitized Error Handler** | Code | Strips tokens and request internals, then adds a fix hint based on the HTTP status. |
| 7 | **Raise Sanitized Error** | Stop and Error | Fails the execution with the clean message, so your n8n Error Workflow can alert on it. |
| 8 | **Dry Run / Skip Report** | No Op | Collects dry-run decisions and skipped events so you can inspect them. |

**Why the owner update is an HTTP Request node.** I chose the HTTP Request node over the built-in HubSpot node so the exact v3 endpoint and the exact property (`hubspot_owner_id`) are visible and version-stable. It also gives explicit retry and error-output control. If you prefer the native node, swap it in: use *Ticket → Update* and map *Ticket Owner* to `{{ $json.selectedOwnerId }}`.

---

## The Load-Balancing Algorithm

For each ticket, the Code node:

1. **Builds a load snapshot** once per execution. For every active agent in `AGENT_POOL_JSON`, it runs a CRM Search for tickets where `hubspot_owner_id = agent` **and** `hs_pipeline_stage NOT IN CLOSED_STAGE_IDS` (or `IN OPEN_STAGE_IDS` if you set it). It reads `total` from the response.
2. **Filters candidates:**
   - The agent is `active`.
   - The count was read successfully. If an agent's count fails, that agent is **excluded**, not guessed at.
   - `openTickets < capacity`.
   - The agent's `priorities` include the ticket's `hs_ticket_priority`. If no one with capacity matches the priority, it falls back to anyone with capacity and records why.
3. **Ranks candidates** by:
   1. lowest **utilisation** (`openTickets / capacity`)
   2. then the lowest raw open count
   3. then the agent assigned least recently (from workflow static data)
   4. then a deterministic rotation hashed from the ticket ID
4. **Adds 1 to the chosen agent's in-memory load**, so a batch of tickets in one webhook call spreads across the team instead of all going to the same person.
5. **Handles overflow:** if every agent is at capacity, the ticket goes to `FALLBACK_OWNER_ID` if you set one. Otherwise it is left unassigned with `action = "no_capacity"`.

**Worked example** with the placeholder pool in sandbox mode:

| Agent | Open | Capacity | Utilisation | Eligible |
|---|---|---|---|---|
| Agent A | 7 | 15 | 0.467 | ✅ |
| Agent B | 3 | 15 | **0.200** | ✅ ← selected |
| Agent C | 4 | 10 | 0.400 | ✅ (LOW/MEDIUM only) |
| Agent D | — | 15 | — | ❌ inactive |

Ticket 1001 goes to Agent B (B is now at 4/15 = 0.267). Ticket 1002 also goes to Agent B, because 0.267 is still the lowest. A retried copy of 1001 in the same batch is skipped as a duplicate.

---

## Repository Structure

```
n8n-hubspot-support-ticket-auto-assignment-load-balancer/
├── README.md          ← this guide
├── workflow.json      ← importable n8n workflow (with pinned sample data)
├── DISCLAIMER.md      ← liability limitation + license notice
└── LICENSE            ← MIT License
```

---

## Prerequisites

- **n8n** 1.x, self-hosted or Cloud, with access to environment variables from inside nodes (see below).
- A **HubSpot** account with Service Hub tickets. **Start with a developer test account or sandbox.**
- A HubSpot **private app** (or public app) access token with these scopes:
  - `tickets`: read and write tickets (required)
  - `crm.objects.owners.read`: only needed to look up owner IDs (optional)
- One of these to send events:
  - **Option A:** a HubSpot ticket-based **workflow** with a *Send a webhook* action. Check that your subscription tier includes this action.
  - **Option B:** an app **webhook subscription** to `ticket.creation`.

### n8n instance settings

| Setting | Value | Why |
|---|---|---|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Lets the Code node and HTTP node read `$env.*`. |
| `NODE_FUNCTION_ALLOW_BUILTIN` | `crypto` | Only needed if `WEBHOOK_AUTH_MODE=hubspot_v3`. |

> If your security policy doesn't allow env access in nodes, move `HUBSPOT_ACCESS_TOKEN` into an n8n **HubSpot App Token** credential. On the *HubSpot Owner Update* node, choose *Authentication → Predefined Credential Type → HubSpot App Token*. In the Code node, swap `this.helpers.httpRequest` for `this.helpers.httpRequestWithAuthentication.call(this, 'hubspotAppToken', options)`.

---

## Setup

### 1. Import the workflow

In n8n, go to **Workflows → Import from File** and select `workflow.json`. The workflow imports **inactive**.

### 2. Find your HubSpot owner IDs

In your **test** portal, list owners (replace the placeholder token):

```bash
curl -s https://api.hubapi.com/crm/v3/owners \
  -H "Authorization: Bearer YOUR_HUBSPOT_PRIVATE_APP_TOKEN"
```

Copy the `id` of each support agent. You can also find owner IDs under **Settings → Users & Teams** in HubSpot.

### 3. Find your closed stage IDs

```bash
curl -s https://api.hubapi.com/crm/v3/pipelines/tickets \
  -H "Authorization: Bearer YOUR_HUBSPOT_PRIVATE_APP_TOKEN"
```

In HubSpot's default *Support Pipeline*, the closed stage ID is usually `4`. Custom pipelines have their own IDs, so list **every** closed stage from **every** pipeline in `CLOSED_STAGE_IDS`.

### 4. Set environment variables

Copy the block from [Environment Variables](#environment-variables) into your n8n environment (`.env`, Docker Compose, Kubernetes secret, or n8n Cloud variables). Keep `DRY_RUN=true` and `SANDBOX_MODE=true` for now. Restart n8n.

### 5. Connect HubSpot

**Option A: HubSpot workflow "Send a webhook"**
1. Create a **ticket-based** workflow. Enrollment trigger: *Ticket is created* (and optionally *Ticket owner is unknown*).
2. Add **Send a webhook**: method `POST`, URL = your n8n **production** webhook URL (`https://YOUR_N8N_HOST/webhook/hubspot-ticket-auto-assign`).
3. Authentication: **API key**, sent in the header `x-webhook-token`, value = your `WEBHOOK_SHARED_SECRET`.
4. Include at least `hs_object_id` (the ticket ID). `hs_ticket_priority` and `hs_pipeline` are optional, because live mode reads them again from the API.

**Option B: app webhook subscription**
1. In your app's **Webhooks** settings, set the target URL to your n8n production webhook URL.
2. Subscribe to **Ticket → Created** (`ticket.creation`).
3. Set `WEBHOOK_AUTH_MODE=hubspot_v3`, `HUBSPOT_CLIENT_SECRET`, and `PUBLIC_WEBHOOK_URL` (the exact public URL HubSpot calls).

### 6. Activate in dry-run mode

Turn the workflow **Active**. Create a few test tickets in your **test** portal, then review the executions (see [Operational Workflow](#operational-workflow)).

---

## Environment Variables

```dotenv
# ---------- Safety switches (fail-safe: ON unless exactly "false") ----------
DRY_RUN=true                       # true = decide but never write to HubSpot
SANDBOX_MODE=true                  # true = mock loads, no HubSpot API calls at all

# ---------- HubSpot ----------
HUBSPOT_ACCESS_TOKEN=YOUR_HUBSPOT_PRIVATE_APP_TOKEN

# ---------- Agent pool (JSON array, single line) ----------
AGENT_POOL_JSON=[{"ownerId":"YOUR_OWNER_ID_1","name":"Agent One","capacity":15,"priorities":["LOW","MEDIUM","HIGH","URGENT"]},{"ownerId":"YOUR_OWNER_ID_2","name":"Agent Two","capacity":10,"priorities":["LOW","MEDIUM"]},{"ownerId":"YOUR_OWNER_ID_3","name":"Agent Three","capacity":15,"active":false}]
DEFAULT_AGENT_CAPACITY=20

# ---------- Load definition ----------
CLOSED_STAGE_IDS=4                 # comma-separated; every closed stage, every pipeline
OPEN_STAGE_IDS=                    # optional; if set, used INSTEAD of CLOSED_STAGE_IDS
ALLOWED_PIPELINE_IDS=              # optional; only route tickets in these pipelines

# ---------- Behaviour ----------
ACCEPTED_EVENT_TYPES=ticket.creation
REASSIGN_IF_OWNED=false            # opt-in: OFF unless exactly "true"
FALLBACK_OWNER_ID=                 # optional; overflow owner when everyone is at capacity
SEARCH_DELAY_MS=250                # pause between search calls (rate-limit courtesy)

# ---------- Webhook authentication ----------
WEBHOOK_AUTH_MODE=shared_secret    # shared_secret | hubspot_v3 | none (none = sandbox only)
WEBHOOK_SHARED_SECRET=YOUR_WEBHOOK_SHARED_SECRET
WEBHOOK_TOKEN_HEADER=x-webhook-token
HUBSPOT_CLIENT_SECRET=YOUR_HUBSPOT_APP_CLIENT_SECRET   # hubspot_v3 only
PUBLIC_WEBHOOK_URL=https://YOUR_N8N_HOST/webhook/hubspot-ticket-auto-assign  # hubspot_v3 only
```

| Variable | Default | Description |
|---|---|---|
| `DRY_RUN` | `true` | If anything other than `false`, the workflow decides but never writes. |
| `SANDBOX_MODE` | `true` | If anything other than `false`, it uses `mockOpenTickets`, makes no API calls, and forces dry run. |
| `HUBSPOT_ACCESS_TOKEN` | — | Private app token. Required when `SANDBOX_MODE=false`. |
| `AGENT_POOL_JSON` | built-in placeholders | Array of `{ownerId, name, capacity, active?, priorities?, mockOpenTickets?}`. Live mode requires numeric owner IDs. |
| `DEFAULT_AGENT_CAPACITY` | `20` | Used when an agent has no `capacity`. |
| `CLOSED_STAGE_IDS` | `4` | Stages that do **not** count as load. |
| `OPEN_STAGE_IDS` | empty | Stages that **do** count as load. Overrides `CLOSED_STAGE_IDS`. |
| `ALLOWED_PIPELINE_IDS` | empty (all) | Restricts routing to specific pipelines. |
| `ACCEPTED_EVENT_TYPES` | `ticket.creation` | App-webhook subscription types to process. |
| `REASSIGN_IF_OWNED` | `false` | Overwrite an existing owner. Keep this `false` to stay idempotent. |
| `FALLBACK_OWNER_ID` | empty | Receives tickets when every agent is at capacity. |
| `SEARCH_DELAY_MS` | `250` | Delay between per-agent search calls. |
| `WEBHOOK_AUTH_MODE` | `shared_secret` | How inbound requests are verified. |
| `WEBHOOK_SHARED_SECRET` | — | Long random string. Generate with `openssl rand -hex 32`. |
| `WEBHOOK_TOKEN_HEADER` | `x-webhook-token` | Header that carries the shared secret. |
| `HUBSPOT_CLIENT_SECRET` | — | App client secret for v3 signature checks. |
| `PUBLIC_WEBHOOK_URL` | n8n's `webhookUrl` | The exact URL HubSpot signs. Set it if n8n runs behind a proxy. |

**Mode matrix**

| `SANDBOX_MODE` | `DRY_RUN` | HubSpot reads | HubSpot writes | Use for |
|---|---|---|---|---|
| `true` | any | ❌ | ❌ | Import test, logic review, pinned data |
| `false` | `true` | ✅ | ❌ | Shadow mode against a real portal |
| `false` | `false` | ✅ | ✅ | **Live** assignment |

---

## Node Configuration

### 1. HubSpot Ticket Webhook
- **Method:** `POST` · **Path:** `hubspot-ticket-auto-assign`
- **Respond:** *Immediately* (`onReceived`, HTTP 200). HubSpot app webhooks expect a fast 2xx response. Answering at once also means an unauthenticated caller learns nothing from the response.
- Comes with **pinned sample data**: two `ticket.creation` events and one retried duplicate, all placeholders.

### 2. Open Ticket Count Check & Load Balancer
- **Mode:** *Run Once for All Items*
- The code is organised into commented sections: Configuration → Agent pool → Live-mode guard rails → Helpers → Authentication → Payload normalisation → Load snapshot → Balancer → Main.
- **Output fields per ticket:** `ticketId`, `action`, `reason`, `selectedOwnerId`, `selectedAgentName`, `selectedAgentLoadBefore`, `selectedAgentCapacity`, `priority`, `pipeline`, `loadSnapshot[]`, `mode`, `authCheck`, `correlationId`, `evaluatedAt`.
- **Guard rails:** with `SANDBOX_MODE=false`, the node refuses to run if the token is missing or still a placeholder, if the agent pool is still the placeholder pool, or if any owner ID isn't numeric.

| `action` | Meaning |
|---|---|
| `assign` | Live assignment approved. Goes to *HubSpot Owner Update*. |
| `dry_run_would_assign` | What *would* have happened. |
| `skip_already_owned` | The ticket already has an owner (idempotent against HubSpot retries). |
| `skip_duplicate_event` | The same ticket appeared twice in one payload. |
| `skip_event_type` | The subscription type isn't in `ACCEPTED_EVENT_TYPES`. |
| `skip_pipeline` | The pipeline isn't in `ALLOWED_PIPELINE_IDS`. |
| `no_capacity` | Every agent is full and there's no fallback owner. |
| `rejected` | Authentication failed (live mode). |
| `error` | Invalid ticket ID, empty payload, or failed ticket lookup (sanitised). |

### 3. Should Assign?
- Condition: `{{ $json.action }}` **equals** `assign` (string, case-sensitive).

### 4. HubSpot Owner Update
- `PATCH https://api.hubapi.com/crm/v3/objects/tickets/{{ $json.ticketId }}`
- Header: `Authorization: Bearer {{ $env.HUBSPOT_ACCESS_TOKEN }}`
- Body: `{ "properties": { "hubspot_owner_id": "<selectedOwnerId>" } }`
- **Retry on fail:** 3 tries, 2 s apart · **Timeout:** 10 s · **On error:** *Continue (using error output)*

### 5–8. Audit Log · Sanitized Error Handler · Raise Sanitized Error · Dry Run / Skip Report
- **Audit Log** records the owner HubSpot confirmed back (`hubspotConfirmedOwnerId`). Connect it to Google Sheets, a database, or Slack if you want a lasting record.
- **Sanitized Error Handler** removes `Bearer …` and `pat-…` strings and maps 400/401/403/404/429 to fix hints.
- **Raise Sanitized Error** fails the run with only the clean message. Go to *Workflow Settings → Error Workflow* and point it at your alerting workflow.
- **Dry Run / Skip Report** is a No Op you can replace with a Slack or Teams notification for `no_capacity` and `error` items.

---

## Operational Workflow

1. A customer submits a form, an email arrives, or an agent creates a ticket → HubSpot creates the ticket.
2. HubSpot POSTs the event → n8n answers `200` at once.
3. The balancer verifies the request, drops duplicates, and re-reads the ticket. If it already has an owner, it stops there.
4. It counts each agent's open tickets (one search per agent, once per execution).
5. It picks the agent with the lowest utilisation who is eligible for the ticket's priority.
6. If live: it PATCHes `hubspot_owner_id`, and HubSpot notifies the agent. The audit record is written.
7. If anything fails: a sanitised error fires your Error Workflow. The ticket stays unassigned, which keeps it visible in the unassigned-tickets view rather than wrongly assigned.

**Day-to-day operations**
- **An agent goes on leave:** set `"active": false` in `AGENT_POOL_JSON` and restart n8n (or reload the variables).
- **A new hire is ramping up:** give them a lower `capacity`, such as 5, and raise it over time.
- **A specialist queue:** limit `priorities` so senior agents take `HIGH`/`URGENT` and juniors take `LOW`/`MEDIUM`.
- **Weekly review:** filter executions by `no_capacity`. Frequent overflow means the team needs more capacity, not a better algorithm.

---

## Testing

### Stage 1: Sandbox (no HubSpot at all)
1. Open the workflow and click **Execute Workflow**. The pinned payload runs.
2. Open **Dry Run / Skip Report**. You should see:
   - `dry_run_would_assign` · `1001` → `OWNER_ID_AGENT_B`
   - `dry_run_would_assign` · `1002` → `OWNER_ID_AGENT_B`
   - `skip_duplicate_event` · `1001`
3. Edit `mockOpenTickets` and capacities in `AGENT_POOL_JSON` to try your own scenarios.

### Stage 2: Webhook smoke test (still sandbox)
```bash
curl -X POST "https://YOUR_N8N_HOST/webhook-test/hubspot-ticket-auto-assign" \
  -H "Content-Type: application/json" \
  -H "x-webhook-token: YOUR_WEBHOOK_SHARED_SECRET" \
  -d '[{"subscriptionType":"ticket.creation","objectId":1001,"eventId":1}]'
```

### Stage 3: Shadow mode (`SANDBOX_MODE=false`, `DRY_RUN=true`) in a HubSpot test account
- Check that each `loadSnapshot[].openTickets` matches HubSpot's ticket board filtered by owner and open stages.
- Check that no `countSource: "error"` entries appear.
- Let it run alongside your current process for a few days and compare its decisions with what actually happened.

### Stage 4: Live in the test account (`DRY_RUN=false`)
- Create 10 or more tickets quickly and confirm that owners are spread according to load.
- Revoke the token's scope on purpose and confirm the sanitised 403 alert reaches your Error Workflow.

---

## Go-Live Checklist

- [ ] Tested end to end in a **HubSpot developer test account or sandbox**
- [ ] `AGENT_POOL_JSON` contains real, numeric owner IDs and realistic capacities
- [ ] `CLOSED_STAGE_IDS` covers every closed stage in every routed pipeline
- [ ] `WEBHOOK_SHARED_SECRET` is a fresh 32+ byte random value, or `hubspot_v3` is configured
- [ ] Error Workflow is configured and tested
- [ ] Any existing HubSpot assignment rules or round-robin on the same tickets are **disabled** (to avoid two assigners competing)
- [ ] Execution data retention matches your privacy policy
- [ ] Ran shadow mode before setting `DRY_RUN=false`
- [ ] Team informed about how assignment now works

---

## Security Model

- **No secrets in the repo.** All credentials are read from environment variables. `workflow.json` contains only placeholders.
- **The token goes to one host only.** The API base URL is hard-coded to `https://api.hubapi.com` and can't be changed by an env var or payload.
- **Inbound authentication.** A shared-secret header compared in constant time, or HubSpot v3 HMAC signature verification with a 5-minute replay window. `none` is refused outside sandbox.
- **Input validation.** Ticket IDs must be numeric in live mode, which blocks path injection into the PATCH URL.
- **Sanitised errors.** Bearer tokens and `pat-` strings are redacted before anything is logged or raised. Rejected requests record no payload.
- **Least privilege.** Grant only the `tickets` scope, and use a dedicated private app for this workflow.
- **Fail-safe flags.** A typo in `DRY_RUN` or `SANDBOX_MODE` (anything other than exactly `false`) keeps the safe mode on.

> **Note on `hubspot_v3`:** n8n hands the Code node a parsed body, which the node re-serialises before computing the signature. HubSpot sends compact JSON, so this normally matches. If verification fails behind a proxy that alters the body, use `shared_secret` with a HubSpot workflow webhook instead.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `access to env vars denied` | Env access is blocked in nodes | Set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, or switch to an n8n credential (see Prerequisites). |
| All items are `rejected` | Wrong or missing secret header | Check that the header name matches `WEBHOOK_TOKEN_HEADER` and the value matches exactly. |
| `countSource: "error"`, status 401/403 | Token invalid or missing the `tickets` scope | Rotate the token and add the scope. |
| Loads look too high | Closed stages from custom pipelines aren't listed | Add them to `CLOSED_STAGE_IDS`, or use `OPEN_STAGE_IDS`. |
| PATCH returns 400 | Owner ID not valid in this portal | Re-check the IDs from `/crm/v3/owners`. |
| Everything is `skip_already_owned` | Another rule assigns first | Disable competing HubSpot assignment rules. |
| 429 errors | Search rate limit | Raise `SEARCH_DELAY_MS`, or reduce how many webhook executions run at once. |
| `crypto_module_not_allowed…` | Builtin module is blocked | Set `NODE_FUNCTION_ALLOW_BUILTIN=crypto`. |

---

## Known Limitations

- **Search indexing lag.** The HubSpot CRM Search index can trail recent writes by a few seconds. The balancer adjusts its own counts within a batch, but two **separate** executions arriving at the same moment may both see the same load. For very high volume, set n8n to process webhooks one at a time (queue mode with concurrency 1) or add a short `Wait` node.
- **Counts, not complexity.** Every open ticket counts as one unit of load. A weighting by priority (for example URGENT = 3) is an easy extension inside `buildLoadSnapshot`.
- **No working-hours awareness.** Agent availability comes from `active` in `AGENT_POOL_JSON` only. You could connect it to a calendar or HubSpot user availability.
- **API cost per execution.** Each run makes one ticket GET per ticket plus one search per active agent. That is fine for teams of up to about 30 agents. Beyond that, cache the snapshot for a short time.

---

## ⚠️ Disclaimer & Liability Limitation

**Use at your own risk.** The code, workflows, and scripts in this repository (`n8n-hubspot-support-ticket-auto-assignment-load-balancer`) are provided for educational and architectural demonstration purposes.

* **No Liability:** The author and dhRobin Systems accept no responsibility or liability for any direct, indirect, incidental, or consequential damages that result from using, misusing, or deploying these scripts.
* **No Warranty:** All software, n8n JSON blueprints, and JavaScript modules are provided "AS IS", without warranty of any kind, express or implied. This includes, but is not limited to, warranties of merchantability, fitness for a particular purpose, and non-infringement.
* **Production Testing Required:** Always test thoroughly in an isolated staging environment, such as a HubSpot developer test account or sandbox, with `SANDBOX_MODE=true` and `DRY_RUN=true` enabled, before you connect live HubSpot credentials. Treat every automated owner assignment as a live change: it notifies agents, starts SLA clocks, and appears in customer-facing records.
* **Third-Party APIs & Compliance:** You alone are responsible for complying with the terms of service, rate limits, data privacy regulations (GDPR/CCPA), and security policies of HubSpot and any connected services. That includes confirming a lawful basis for processing ticket and customer data in n8n, applying appropriate retention to execution logs, respecting the workload policies that govern your support team, and reviewing automated assignment decisions on a regular basis.

The full disclaimer is also available in [DISCLAIMER.md](https://github.com/dhrobinsystems/n8n-hubspot-support-ticket-auto-assignment-load-balancer/blob/main/DISCLAIMER.md).

## License

Released under the MIT License. Copyright (c) 2026 DH Robin (dhRobin Systems). See [LICENSE](https://github.com/dhrobinsystems/n8n-hubspot-support-ticket-auto-assignment-load-balancer/blob/main/LICENSE).

---

Full text: DISCLAIMER.md · Licensed under the MIT License. License: MIT © 2026 dhRobin Systems · Provided as-is, see the disclaimer.
