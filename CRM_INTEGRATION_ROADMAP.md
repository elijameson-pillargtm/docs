# PILLAR CRM Integration Roadmap

## Salesforce, HubSpot & Microsoft Dynamics 365

**Version:** 1.0
**Date:** March 2025
**Status:** Architecture Defined — Implementation Pending

---

## 1. Executive Summary

PILLAR's dev environment uses a canonical data model with seed data. Production deployments must ingest live data from customer CRMs. This roadmap defines the three-phase approach to building bidirectional CRM connectors for the three dominant CRM platforms:

1. **Salesforce** (dominant in enterprise/mid-market)
2. **HubSpot** (strong in SMB/growth-stage companies)
3. **Microsoft Dynamics 365** (prevalent in Microsoft-ecosystem organizations)

### Architecture Principle: Canonical Model + Mapping Engine

```
┌─────────────────────┐     ┌───────────────────────┐     ┌─────────────────────┐
│   SOURCE CRM        │     │   INTEGRATION LAYER   │     │   PILLAR CANONICAL  │
│                     │     │                       │     │                     │
│  Salesforce         │────▶│  OAuth Manager        │────▶│  accounts           │
│  HubSpot            │────▶│  Field Mapper         │────▶│  contacts           │
│  Dynamics 365       │────▶│  Sync Engine          │────▶│  opportunities      │
│                     │◀────│  Writeback Engine      │◀────│  activities         │
│  Custom fields      │     │  Conflict Resolution  │     │  signals            │
│  Picklists          │     │  Rate Limiter         │     │  leads              │
│  Workflows          │     │  Error Recovery       │     │  renewals           │
└─────────────────────┘     └───────────────────────┘     └─────────────────────┘
```

---

## 2. What Already Exists

### Database Infrastructure (Ready)
- `accounts.crm_external_id` + `accounts.crm_source` — tracks which CRM an account originated from
- `contacts.crm_external_id` — external contact ID mapping
- `opportunities.crm_external_id` — external opportunity ID mapping
- `activities.crm_external_id` — external activity ID mapping
- `connector_sync_log` — tracks sync runs (connector_name, sync_type, status, records_synced, errors)

### Environment Configuration (Ready)
- Salesforce OAuth env vars defined (client ID, secret, redirect URI, login URL)
- HubSpot OAuth env vars defined (client ID, secret, redirect URI)
- Google Calendar OAuth env vars defined (for activity sync)

### Connector Pattern (Established)
- `src/lib/connectors/starbridge.ts` — reference implementation showing API client, webhook processing, data mapping, and sync patterns

### Canonical Data Model (Ready)
- All PILLAR tables use UUIDs with `org_id` multi-tenancy
- Jsonb columns available for CRM-specific overflow fields
- Signal framework supports `GROWTH`, `RISK`, `PIPELINE`, `USAGE`, `COMPETITIVE` families

---

## 3. Phase 1: Connector Foundation (Weeks 1-3)

### 3.1 Connection Manager

Manages OAuth 2.0 flows, token storage, and refresh cycles for all three CRMs.

**New table: `crm_connections`**
```
id                  uuid PK
org_id              uuid FK → organizations
provider            varchar(20)     — "salesforce" | "hubspot" | "dynamics"
status              varchar(20)     — "connected" | "disconnected" | "error" | "syncing"
access_token        text (encrypted)
refresh_token       text (encrypted)
token_expires_at    timestamp
instance_url        text            — Salesforce instance URL or Dynamics org URL
scopes              jsonb           — granted OAuth scopes
metadata            jsonb           — provider-specific config (HubSpot portal ID, etc.)
last_sync_at        timestamp
next_sync_at        timestamp
created_at          timestamp
updated_at          timestamp
```

**OAuth Flows:**
| CRM | Auth Type | Token Refresh | Notes |
|-----|-----------|---------------|-------|
| Salesforce | OAuth 2.0 Web Server Flow | Refresh token (no expiry) | Instance URL varies per org |
| HubSpot | OAuth 2.0 | Refresh token (6-month expiry) | Portal-scoped |
| Dynamics | OAuth 2.0 via Azure AD | Refresh token (90-day expiry) | Requires Azure AD app registration |

**API Routes:**
- `GET /api/connectors/[provider]/authorize` — initiate OAuth
- `GET /api/connectors/[provider]/callback` — handle OAuth callback, store tokens
- `POST /api/connectors/[provider]/disconnect` — revoke tokens, mark disconnected
- `GET /api/connectors/status` — return connection status for all providers

### 3.2 Field Mapping Configuration

Allows per-org customization of how CRM fields map to PILLAR canonical fields.

**New table: `crm_field_mappings`**
```
id                  uuid PK
org_id              uuid FK → organizations
provider            varchar(20)
pillar_table        varchar(50)     — "accounts", "contacts", "opportunities", etc.
pillar_field        varchar(100)    — canonical field name
crm_object          varchar(100)    — "Account", "Contact", "Opportunity" (SF) or "companies", "contacts", "deals" (HS)
crm_field           varchar(200)    — CRM field API name (e.g., "AnnualRevenue", "annual_revenue")
transform           varchar(50)     — "direct", "lookup", "formula", "picklist_map"
transform_config    jsonb           — transformation rules (picklist mappings, formulas)
direction           varchar(10)     — "inbound", "outbound", "bidirectional"
is_required         boolean
is_active           boolean
created_at          timestamp
```

**Default Mapping Templates:**

Each CRM ships with a pre-built mapping template that customers can customize:

| PILLAR Field | Salesforce | HubSpot | Dynamics 365 |
|-------------|------------|---------|--------------|
| `accounts.name` | `Account.Name` | `companies.name` | `account.name` |
| `accounts.arr` | `Account.AnnualRevenue` | `companies.annualrevenue` | `account.revenue` |
| `accounts.segment` | `Account.Type` | `companies.hs_company_size` | `account.accountcategorycode` |
| `accounts.industry` | `Account.Industry` | `companies.industry` | `account.industrycode` |
| `accounts.health_score` | Custom: `Account.PILLAR_Health__c` | Custom property | Custom field |
| `contacts.name` | `Contact.Name` | `contacts.firstname + lastname` | `contact.fullname` |
| `contacts.email` | `Contact.Email` | `contacts.email` | `contact.emailaddress1` |
| `contacts.title` | `Contact.Title` | `contacts.jobtitle` | `contact.jobtitle` |
| `opportunities.name` | `Opportunity.Name` | `deals.dealname` | `opportunity.name` |
| `opportunities.amount` | `Opportunity.Amount` | `deals.amount` | `opportunity.estimatedvalue` |
| `opportunities.stage` | `Opportunity.StageName` | `deals.dealstage` | `opportunity.stepname` |
| `opportunities.close_date` | `Opportunity.CloseDate` | `deals.closedate` | `opportunity.estimatedclosedate` |
| `opportunities.probability` | `Opportunity.Probability` | `deals.hs_deal_stage_probability` | `opportunity.closeprobability` |
| `leads.name` | `Lead.Name` | `contacts.firstname + lastname` | `lead.fullname` |
| `leads.email` | `Lead.Email` | `contacts.email` | `lead.emailaddress1` |
| `leads.status` | `Lead.Status` | `contacts.hs_lead_status` | `lead.statuscode` |
| `leads.lead_score` | `Lead.Lead_Score__c` | `contacts.hubspotscore` | `lead.leadqualitycode` |

### 3.3 Picklist / Enum Mapping

CRM picklist values rarely match PILLAR enums. The mapping engine translates between them.

**New table: `crm_picklist_mappings`**
```
id                  uuid PK
org_id              uuid FK → organizations
provider            varchar(20)
pillar_table        varchar(50)
pillar_field        varchar(100)
pillar_value        varchar(100)     — PILLAR enum value
crm_value           varchar(200)     — CRM picklist value
is_default          boolean          — use when no match found
created_at          timestamp
```

**Example: Opportunity Stage Mapping**
| PILLAR Stage | Salesforce | HubSpot | Dynamics |
|-------------|------------|---------|----------|
| Discovery | Qualification | appointmentscheduled | 0 (Qualify) |
| Evaluation | Needs Analysis | qualifiedtobuy | 1 (Develop) |
| Proposal | Proposal/Price Quote | presentationscheduled | 2 (Propose) |
| Negotiation | Negotiation/Review | contractsent | 3 (Close) |
| Closed Won | Closed Won | closedwon | 4 (Won) |
| Closed Lost | Closed Lost | closedlost | 5 (Lost) |

---

## 4. Phase 2: Sync Engine (Weeks 4-6)

### 4.1 Sync Orchestration

**Sync Types:**
1. **Initial Sync (Full)** — First connection: bulk import all records
2. **Incremental Sync** — Scheduled: fetch only changed records since last sync
3. **Real-Time Webhook** — Instant: CRM triggers push changes to PILLAR
4. **Writeback** — PILLAR pushes calculated fields back to CRM

**Sync Schedule:**
| Sync Type | Frequency | Method |
|-----------|-----------|--------|
| Accounts | Every 15 min | Incremental (modified date filter) |
| Contacts | Every 15 min | Incremental |
| Opportunities | Every 5 min | Incremental (pipeline changes are time-sensitive) |
| Activities | Every 30 min | Incremental |
| Leads | Every 10 min | Incremental |
| Writeback (scores) | Every 1 hour | Batch upsert |

### 4.2 CRM-Specific API Patterns

**Salesforce:**
- Auth: OAuth 2.0 Web Server Flow
- Bulk reads: SOQL queries via REST API (`/services/data/v59.0/query`)
- Incremental: `WHERE LastModifiedDate > :lastSyncTimestamp`
- Bulk writes: Composite API (`/services/data/v59.0/composite/sobjects`)
- Real-time: Outbound Messages or Platform Events → PILLAR webhook
- Rate limits: 100,000 API calls/24hr (Enterprise), concurrent limit of 25

**HubSpot:**
- Auth: OAuth 2.0
- Bulk reads: Search API (`/crm/v3/objects/[object]/search`) with filters
- Incremental: `lastmodifieddate` filter
- Bulk writes: Batch API (`/crm/v3/objects/[object]/batch/update`)
- Real-time: Webhooks via workflow triggers
- Rate limits: 100 requests/10 sec (OAuth), 500K/day (Enterprise)

**Dynamics 365:**
- Auth: OAuth 2.0 via Azure AD (MSAL)
- Bulk reads: OData queries (`/api/data/v9.2/[entity]`)
- Incremental: `$filter=modifiedon gt [timestamp]`
- Bulk writes: Batch requests (`$batch` endpoint)
- Real-time: Webhooks registered via Plugin Registration Tool
- Rate limits: 6,000 requests/5 min per user, concurrent limit of 52

### 4.3 Conflict Resolution

When a record is modified in both PILLAR and the CRM between syncs:

| Scenario | Resolution | Rationale |
|----------|-----------|-----------|
| CRM field changed, PILLAR field unchanged | CRM wins | CRM is system of record for raw data |
| PILLAR score changed, CRM score unchanged | PILLAR wins | PILLAR owns computed/scored fields |
| Both changed, same field | CRM wins + flag signal | Alert user to review |
| Record deleted in CRM | Soft-delete in PILLAR | Preserve history, flag for review |
| Record created in CRM | Create in PILLAR | Auto-map using field mappings |

### 4.4 Error Handling & Recovery

```
Retry Strategy:
  Attempt 1: Immediate
  Attempt 2: 30 seconds
  Attempt 3: 2 minutes
  Attempt 4: 15 minutes
  Attempt 5: 1 hour
  After 5 failures: Mark sync as "error", generate RISK signal

Token Refresh:
  On 401 response → attempt token refresh
  On refresh failure → mark connection as "error"
  Generate signal: "CRM connection lost — {provider}"
```

---

## 5. Phase 3: Writeback & Advanced Features (Weeks 7-10)

### 5.1 PILLAR → CRM Writeback

Push PILLAR's computed intelligence back into the CRM so reps see it natively:

| PILLAR Field | CRM Custom Field | Update Frequency |
|-------------|-----------------|-----------------|
| Account Health Score | `PILLAR_Health_Score__c` | Every sync |
| Renewal Risk Score | `PILLAR_Renewal_Risk__c` | Every sync |
| Pipeline Hygiene Score | `PILLAR_Hygiene_Score__c` | Every sync |
| Expansion Propensity | `PILLAR_Expansion_Score__c` | Every sync |
| Forecast Category | `PILLAR_Forecast_Category__c` | On override |
| Next Best Action | `PILLAR_Next_Action__c` | On signal generation |

### 5.2 Custom Object Support

Beyond standard objects, customers may have custom CRM objects that need mapping:

- **Salesforce**: Custom objects (`__c` suffix), custom fields on standard objects
- **HubSpot**: Custom objects (Enterprise only), custom properties on standard objects
- **Dynamics**: Custom entities, custom fields via solution import

**Approach:**
1. During onboarding, run a schema discovery API call to enumerate all objects + fields
2. Present a mapping UI showing PILLAR canonical fields → discovered CRM fields
3. Customer confirms/adjusts mappings
4. Store in `crm_field_mappings` table
5. Sync engine uses mappings at runtime

### 5.3 Managed Package / App (Future)

For deeper integration, create installable packages:

- **Salesforce**: Managed Package (AppExchange) — installs custom fields, layouts, Lightning component
- **HubSpot**: Public App (App Marketplace) — installs timeline events, custom cards
- **Dynamics**: Managed Solution (AppSource) — installs entities, forms, dashboards

---

## 6. File Architecture

```
src/lib/connectors/
├── types.ts                     # Shared connector types & interfaces
├── connection-manager.ts        # OAuth flows, token management, refresh
├── field-mapper.ts              # Field mapping resolution engine
├── sync-engine.ts               # Orchestration, scheduling, conflict resolution
├── salesforce/
│   ├── client.ts                # Salesforce REST API client
│   ├── auth.ts                  # SF OAuth 2.0 Web Server Flow
│   ├── mapper.ts                # SF-specific default field mappings
│   ├── sync.ts                  # SF sync logic (SOQL queries, bulk API)
│   └── writeback.ts             # SF writeback (Composite API)
├── hubspot/
│   ├── client.ts                # HubSpot API client
│   ├── auth.ts                  # HS OAuth 2.0
│   ├── mapper.ts                # HS-specific default field mappings
│   ├── sync.ts                  # HS sync logic (Search API, batch updates)
│   └── writeback.ts             # HS writeback (batch API)
├── dynamics/
│   ├── client.ts                # Dynamics 365 OData client
│   ├── auth.ts                  # D365 Azure AD / MSAL auth
│   ├── mapper.ts                # D365-specific default field mappings
│   ├── sync.ts                  # D365 sync logic (OData queries, $batch)
│   └── writeback.ts             # D365 writeback ($batch)
└── starbridge.ts                # Existing Starbridge connector (reference)

src/app/api/connectors/
├── status/route.ts              # GET — all connector statuses
├── [provider]/
│   ├── authorize/route.ts       # GET — initiate OAuth
│   ├── callback/route.ts        # GET — handle OAuth callback
│   ├── disconnect/route.ts      # POST — disconnect
│   ├── sync/route.ts            # POST — trigger manual sync
│   ├── mappings/route.ts        # GET/PUT — field mappings
│   └── webhook/route.ts         # POST — real-time webhook receiver
└── schema-discovery/route.ts    # POST — discover CRM schema for mapping UI
```

---

## 7. Database Migrations Required

### New Tables
1. `crm_connections` — OAuth tokens, connection status, sync timestamps
2. `crm_field_mappings` — per-org field mapping configuration
3. `crm_picklist_mappings` — enum/picklist value translation
4. `crm_sync_queue` — pending sync operations (outbound writeback queue)

### Schema Modifications
- Add `crm_last_synced_at` timestamp to accounts, contacts, opportunities, activities, leads
- Add `crm_sync_hash` varchar(64) to detect changes without full comparison
- Extend `connector_sync_log` with `provider`, `object_type`, `direction` fields

---

## 8. Onboarding Workflow

```
Step 1: Customer selects CRM provider
  └─▶ Show "Connect [Salesforce/HubSpot/Dynamics]" button

Step 2: OAuth authorization
  └─▶ Redirect to CRM login → approve PILLAR scopes → callback

Step 3: Schema discovery
  └─▶ API call to enumerate CRM objects + fields → show mapping UI

Step 4: Field mapping review
  └─▶ Pre-filled with defaults → customer adjusts custom fields → save

Step 5: Picklist mapping
  └─▶ Auto-detect CRM picklist values → map to PILLAR enums → save

Step 6: Initial sync (full)
  └─▶ Bulk import all records → progress indicator → completion signal

Step 7: Enable incremental sync
  └─▶ Schedule recurring syncs → enable webhooks for real-time

Step 8: Verify & activate
  └─▶ Show sync summary → data quality check → activate for org
```

---

## 9. Implementation Priority

| Priority | Item | Est. Effort | Dependencies |
|----------|------|-------------|--------------|
| P0 | Connection Manager (OAuth) | 1 week | None |
| P0 | Salesforce connector (read) | 1.5 weeks | Connection Manager |
| P0 | Default field mappings (SF) | 3 days | SF connector |
| P1 | HubSpot connector (read) | 1 week | Connection Manager |
| P1 | Dynamics connector (read) | 1 week | Connection Manager |
| P1 | Field Mapping UI | 1 week | All connectors |
| P1 | Picklist mapping | 3 days | Field Mapping |
| P2 | Incremental sync engine | 1 week | All connectors |
| P2 | Conflict resolution | 3 days | Sync engine |
| P2 | Writeback (SF) | 1 week | SF connector |
| P2 | Writeback (HS + D365) | 1 week | HS/D365 connectors |
| P3 | Real-time webhooks | 1 week | All connectors |
| P3 | Schema discovery UI | 1 week | All connectors |
| P3 | Managed packages | 2 weeks | Writeback |

**Total estimated effort: 10-12 weeks**

---

## 10. Security Requirements

- All OAuth tokens encrypted at rest (AES-256 via Supabase Vault or application-level encryption)
- Refresh tokens stored separately from access tokens
- Token refresh happens server-side only (never exposed to client)
- All CRM API calls made server-side via API routes
- Rate limiting enforced per-provider per-org
- Sync logs retained for 90 days for audit
- Customer can revoke access at any time via Settings → Connectors → Disconnect
- PILLAR never stores CRM credentials (username/password) — OAuth only

---

## 11. Monitoring & Observability

**Signals generated by connector health:**
- `CRM_SYNC_FAILED` — Sync attempt failed after retries → RISK signal
- `CRM_CONNECTION_LOST` — OAuth token refresh failed → RISK signal
- `CRM_DATA_DRIFT` — Significant data discrepancy detected → INFO signal
- `CRM_RATE_LIMITED` — Approaching API rate limits → WARNING signal

**Dashboard metrics (future Connector Health page):**
- Sync success rate (last 24h, 7d, 30d)
- Records synced per cycle
- Average sync latency
- Error rate by object type
- Last successful sync timestamp per object

---

## 12. What the Dev Product Already Handles

The current PILLAR dev environment is well-positioned for CRM integration:

| Feature | Status | Notes |
|---------|--------|-------|
| `crm_external_id` on all core tables | Ready | Unique mapping to CRM records |
| `crm_source` on accounts | Ready | Identifies originating CRM |
| `connector_sync_log` table | Ready | Tracks sync runs |
| Multi-tenant `org_id` | Ready | Each customer gets isolated data |
| Jsonb columns | Ready | Overflow for CRM-specific fields |
| Signal framework | Ready | CRM health signals fit existing pattern |
| OAuth env vars | Ready | Salesforce + HubSpot placeholders defined |
| Connector code pattern | Ready | Starbridge connector is reference impl |

**The canonical data model is CRM-agnostic by design.** All dashboards, scoring algorithms, signal logic, and forecasting modules reference PILLAR's canonical tables — not CRM-specific schemas. The integration layer is purely a data ingestion/writeback concern that sits between the CRM and PILLAR's core.
