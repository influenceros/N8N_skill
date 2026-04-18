---
name: n8n-integration-builder
description: Build, deploy, and manage N8N integration workflows for customer platforms via the N8N REST API. Handles credential creation, workflow templating, activation, testing, and monitoring. Use when a customer needs a CRM, calendar, or third-party API connected to OneVa.ai.
---

# N8N Integration Builder Skill

## Purpose

Programmatically create and manage N8N workflows that connect OneVa.ai to customer platforms (CRMs, calendars, lead systems). All operations happen via the N8N REST API — no UI access required.

## N8N REST API Reference

Base URL: `{N8N_BASE_URL}` (set in environment)
Auth: `X-N8N-API-KEY: {N8N_API_KEY}` header on all requests

### Credentials API

**Create credential:**
```
POST /api/v1/credentials
Content-Type: application/json

{
  "name": "org_{org_id}_{platform}_api",
  "type": "httpHeaderAuth",
  "data": {
    "name": "Authorization",
    "value": "Bearer {customer_api_key}"
  },
  "nodesAccess": [
    { "nodeType": "n8n-nodes-base.httpRequest" }
  ]
}

Response: { "id": "credential_id", ... }
```

**Delete credential:**
```
DELETE /api/v1/credentials/{id}
```

**Update credential:**
```
PUT /api/v1/credentials/{id}
Content-Type: application/json

{
  "name": "org_{org_id}_{platform}_api",
  "type": "httpHeaderAuth",
  "data": {
    "name": "Authorization",
    "value": "Bearer {new_token}"
  }
}
```

### Workflows API

**Create workflow:**
```
POST /api/v1/workflows
Content-Type: application/json

{
  "name": "org_{org_id}_{platform}_sync",
  "nodes": [...],
  "connections": {...},
  "settings": { "executionOrder": "v1" },
  "active": false
}

Response: { "id": "workflow_id", ... }
```

**Activate workflow:**
```
PATCH /api/v1/workflows/{id}
Content-Type: application/json

{ "active": true }
```

**Deactivate workflow:**
```
PATCH /api/v1/workflows/{id}
Content-Type: application/json

{ "active": false }
```

**Delete workflow:**
```
DELETE /api/v1/workflows/{id}
```

**Execute workflow (manual test):**
```
POST /api/v1/workflows/{id}/run
Content-Type: application/json

{ "data": { "test": true } }
```

### Executions API

**Get recent executions:**
```
GET /api/v1/executions?workflowId={id}&limit=5&status=error
```

## Workflow Template Pattern

Every integration workflow follows this structure:

### Inbound Sync (External Platform → OneVa.ai)

```
1. Webhook Trigger (POST /webhook/org_{id}_{platform}_inbound)
   ↓
2. Validate payload (check required fields)
   ↓
3. Transform data (map external fields → OneVa.ai schema)
   ↓
4. HTTP Request to OneVa.ai API
   POST {APP_URL}/api/integrations/sync
   Headers: Authorization: Bearer {oneva_service_key}
   Body: { organization_id, platform, entity_type, data }
   ↓
5. Log result to Supabase
   POST {SUPABASE_URL}/rest/v1/integration_logs
```

### Outbound Sync (OneVa.ai → External Platform)

```
1. Webhook Trigger (POST /webhook/org_{id}_{platform}_outbound)
   or Schedule Trigger (every 5 min)
   ↓
2. Fetch pending sync items from Supabase
   GET {SUPABASE_URL}/rest/v1/integration_queue
   ?organization_id=eq.{id}&platform=eq.{platform}&status=eq.pending
   ↓
3. For each item:
   a. Transform data (OneVa.ai schema → external fields)
   b. HTTP Request to External API (using org-scoped credential)
   c. Update sync status in Supabase
   ↓
4. Log results
```

## Credential Naming Convention

All credentials follow this pattern:
```
org_{organization_id}_{platform}_{type}
```

Examples:
- `org_abc123_followupboss_api`
- `org_abc123_hubspot_api`
- `org_abc123_google_oauth`

## Platform-Specific API Reference

### Follow Up Boss
- Base URL: `https://api.followupboss.com/v1`
- Auth: Basic Auth (API key as username, blank password)
- Header format: `Authorization: Basic {base64(apikey:)}`
- Key endpoints:
  - `GET /people` — list contacts
  - `POST /people` — create contact
  - `POST /notes` — add note to contact
  - `GET /calls` — list calls
  - `POST /events` — create event/call log

### GoHighLevel
- Base URL: `https://services.leadconnectorhq.com`
- Auth: Bearer token
- Header format: `Authorization: Bearer {api_key}`
- Key endpoints:
  - `GET /contacts/` — list contacts
  - `POST /contacts/` — create contact
  - `PUT /contacts/{id}` — update contact
  - `GET /calendars/events` — list appointments
  - `POST /calendars/events` — create appointment

### HubSpot
- Base URL: `https://api.hubapi.com`
- Auth: Bearer token (private app token)
- Header format: `Authorization: Bearer {token}`
- Key endpoints:
  - `POST /crm/v3/objects/contacts` — create contact
  - `PATCH /crm/v3/objects/contacts/{id}` — update contact
  - `POST /crm/v3/objects/deals` — create deal
  - `POST /crm/v3/objects/calls` — log call

### Salesforce
- Base URL: `https://{instance}.salesforce.com/services/data/v59.0`
- Auth: Bearer token (OAuth access token)
- Header format: `Authorization: Bearer {access_token}`
- Key endpoints:
  - `POST /sobjects/Lead` — create lead
  - `PATCH /sobjects/Contact/{id}` — update contact
  - `POST /sobjects/Task` — create task (call log)

### Google Calendar
- Base URL: `https://www.googleapis.com/calendar/v3`
- Auth: Bearer token (OAuth access token)
- Header format: `Authorization: Bearer {access_token}`
- Key endpoints:
  - `POST /calendars/{calendarId}/events` — create event
  - `GET /calendars/{calendarId}/events` — list events

## Integration Setup Procedure

When you receive a `create_integration` task:

### Step 1: Validate
- Confirm the platform is supported
- Confirm required credential fields are present
- Confirm the organization exists in Supabase

### Step 2: Create Credential
- Decrypt customer API key from Supabase reference
- Create N8N Header Auth credential via API
- Store the returned credential ID in Supabase `organization_integrations` table

### Step 3: Build Workflow
- Load the platform template
- Customize with:
  - Organization ID
  - Credential ID
  - Webhook URLs
  - Field mappings (if custom)
  - Sync direction and entity types
- Create workflow via N8N API (inactive)

### Step 4: Test
- Execute the workflow manually with a test payload
- Verify:
  - Credential works (no 401/403)
  - Data transforms correctly
  - Round-trip completes
- If test fails: deactivate, report error, do NOT activate

### Step 5: Activate
- Activate the workflow via API
- Update Supabase integration status to "active"
- Record webhook URLs for the customer's reference
- Notify the Onboarding Concierge

### Step 6: Monitor (Ongoing)
- Check execution history for errors periodically
- If error rate > 20% in last hour: deactivate and alert
- If credential expires (401): deactivate and request re-authentication

## Error Recovery

| Error | Action |
|-------|--------|
| 401 Unauthorized | Deactivate workflow, mark integration "credentials_expired", notify customer |
| 403 Forbidden | Check API permissions, report specific missing scope |
| 429 Rate Limited | Add delay nodes, reduce sync frequency |
| 500 External Error | Retry 3x with exponential backoff, then deactivate |
| N8N API Error | Log full error, retry once, escalate to CTO if persistent |
| Webhook not firing | Verify webhook URL is registered in external platform |

## Security Rules

1. **Never log raw API keys** — use reference IDs only
2. **Always scope by organization_id** — every query, every workflow
3. **Encrypt credentials at rest** — use Supabase column encryption
4. **Rotate credentials on demand** — update N8N credential via PUT, no workflow changes needed
5. **Audit trail** — log every credential creation, workflow activation, and test result
