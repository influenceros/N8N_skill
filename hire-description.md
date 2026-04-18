# New Hire Request: Integration Engineer Agent

## Position Title
**Integration Engineer** — Automated API Connection Specialist

## Reports To
CTO

## Department
Engineering / Customer Operations

## Summary

We need an agent that automatically sets up API integrations for businesses that sign up for OneVa.ai. When a new customer says "I use Follow Up Boss" or "Connect me to GoHighLevel," this agent builds the entire integration pipeline in N8N — no human engineering time, no customer UI access to N8N required.

## Why We Need This

Today, connecting a customer's CRM, calendar, or lead system requires an engineer to manually build or configure N8N workflows. This doesn't scale. Every new customer means engineering hours for what is fundamentally a repeatable task: take customer's API credentials, build a workflow that syncs data between OneVa.ai and their platform.

**Target**: New customer integrations go from 2-4 hours of engineering time → fully automated in under 5 minutes.

## What This Agent Does

1. **Receives integration requests** — Customer signs up, selects their CRM/tools during onboarding, or requests a new integration via the dashboard
2. **Collects credentials securely** — Guides the onboarding flow to capture the customer's API key/token for their platform (stored encrypted in Supabase, scoped to their organization)
3. **Builds N8N workflows via API** — Creates the webhook listeners, HTTP Request nodes, data transforms, and scheduling needed to connect OneVa.ai ↔ Customer Platform
4. **Creates N8N credentials via API** — Programmatically creates the credential entries in N8N using `POST /api/v1/credentials` with the customer's encrypted keys
5. **Activates and tests** — Turns on the workflow, sends a test payload, verifies the round-trip works
6. **Reports status** — Updates the customer's integration status in Supabase, notifies the Onboarding Concierge that the integration is live

## Technical Architecture

### Why HTTP Request Nodes (Not Built-in Nodes)

N8N has built-in nodes for many platforms (HubSpot, Salesforce, etc.), but:
- Built-in nodes require credentials to be entered via the N8N UI
- Our customers never see N8N
- The N8N REST API **can** create credentials programmatically (`POST /api/v1/credentials`)
- But built-in node credentials have complex type structures that vary per integration

**Our approach**: Use HTTP Request nodes with Header Auth credentials for everything. This gives us:
- Uniform credential format (just an API key/token in a header)
- Full control via the N8N REST API
- No dependency on N8N's internal credential type schemas
- Easy rotation (update one credential, all workflows using it update automatically)

For OAuth-based integrations (Google Calendar, etc.), we handle the OAuth flow in our app, store the access/refresh tokens in Supabase, and use a token-refresh workflow in N8N that keeps the HTTP Request header credential current.

### Credential Storage Strategy

```
Customer enters API key in OneVa.ai dashboard
  → Encrypted and stored in Supabase (credentials table, scoped by org)
  → Agent creates N8N Header Auth credential via API:
      POST /api/v1/credentials
      { name: "org_{id}_fub_api", type: "httpHeaderAuth",
        data: { name: "Authorization", value: "Bearer {token}" } }
  → Credential ID stored in Supabase for reference
  → Workflows reference this credential by ID
```

### Dedicated N8N Instance

This agent operates on a **separate N8N instance** from internal operations:
- Customer-facing integrations only
- API key authentication (`X-N8N-API-KEY`)
- No customer access to the N8N UI
- All workflow creation/management via REST API

## Supported Integration Templates

The agent should ship with pre-built workflow templates for:

| Platform | Sync Type | Direction |
|----------|-----------|-----------|
| Follow Up Boss | Contacts, Notes, Calls | Bidirectional |
| GoHighLevel | Contacts, Appointments, Calls | Bidirectional |
| HubSpot | Contacts, Deals, Calls | Bidirectional |
| Salesforce | Leads, Contacts, Tasks | Bidirectional |
| Google Calendar | Appointments | OneVa → Google |
| Calendly | Appointments | Calendly → OneVa |
| Zapier (Webhook) | Generic | Bidirectional |

Each template is a JSON workflow that the agent customizes with the customer's:
- Credential ID (created via API)
- Organization ID (for data scoping)
- Webhook URLs (for receiving data back)
- Field mappings (if the customer has custom fields)

## Required Capabilities

- **N8N REST API**: Create workflows, credentials, activate/deactivate, execute
- **Supabase access**: Read customer org data, store credential references, update integration status
- **HTTP client**: Test integrations by making sample API calls
- **Template engine**: Customize pre-built workflow JSON with customer-specific values
- **Error handling**: Detect failed API calls, invalid credentials, rate limits — and report clearly

## Success Metrics

- Integration setup time: < 5 minutes from credential submission
- First-try success rate: > 90%
- Zero customer-facing N8N access required
- Zero manual engineering intervention for supported platforms

## Priority

High — this is a scaling blocker. Every new customer currently requires manual integration work.
