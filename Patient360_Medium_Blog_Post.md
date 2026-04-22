# Building Patient360: A Unified AI-Powered Patient View on Snowflake Cortex and Salesforce

*How we eliminated fragmented patient data by combining Snowflake Cortex AI, FastAPI, React, and Salesforce Lightning into a single, intelligent dashboard*

---

## The Problem: Data Silos Kill Clinical Efficiency

In healthcare, a care coordinator, billing specialist, or patient support rep often needs to answer a deceptively simple question: **"Who is this patient, and what do they need right now?"**

The answer is almost never in one place.

- Demographics live in the CRM.
- Insurance authorizations are buried in a legacy system.
- Physician orders sit in the EHR.
- Oxygen flow rates and therapy equipment details are in yet another portal.
- Account status — whether a patient is serviceable, on hold, or pending — requires a phone call or a ticket to another team.

The result is wasted minutes per interaction multiplied across thousands of daily touchpoints. Worse, context-switching errors happen: wrong insurance billed, wrong physician contacted, care delayed.

We set out to fix this with **Patient360** — a unified patient intelligence application that brings every critical data point into a single screen, augmented by a natural-language AI agent that lets anyone on the care team ask questions in plain English.

---

## What We Built

Patient360 is a full-stack application that gives care teams:

1. **A single-pane-of-glass dashboard** showing demographics, contact info, insurance, medical details, account status, and physician information — all surfaced in real time from Snowflake.

2. **A conversational AI agent** powered by Snowflake Cortex that translates natural-language questions ("What is this patient's primary diagnosis and LPM setting?") into SQL against a semantic data model, then returns a clean, structured answer — no SQL knowledge required.

3. **A native Salesforce integration** via a Lightning Web Component (LWC) that embeds the entire Patient360 dashboard directly into the Salesforce Account record page, so reps never leave the CRM.

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│            Salesforce CRM               │
│  ┌──────────────────────────────────┐   │
│  │  Lightning Web Component (LWC)   │   │
│  │  patient360Embed                 │   │
│  │  ↓ iFrame ↓                      │   │
│  └──────────────────────────────────┘   │
└────────────────┬────────────────────────┘
                 │ HTTPS
┌────────────────▼────────────────────────┐
│         React + Vite Frontend           │
│         (TypeScript, Tailwind CSS)      │
│         Port 5173 (dev) / SPCS (prod)   │
└────────────────┬────────────────────────┘
                 │ /api proxy (X-API-Key)
┌────────────────▼────────────────────────┐
│         FastAPI Backend (Python)        │
│         Port 8002                       │
│         JWT key-pair authentication     │
└────────────────┬────────────────────────┘
                 │ REST + SSE streaming
┌────────────────▼────────────────────────┐
│         Snowflake Cortex                │
│  ┌──────────────────────────────────┐   │
│  │  Cortex Agent                   │   │
│  │  PATIENT_360_AGENT              │   │
│  │  ↓ tool: cortex_analyst_text_to_sql  │
│  └──────────────────────────────────┘   │
│  ┌──────────────────────────────────┐   │
│  │  Semantic View                   │   │
│  │  CORTEX_ANALYST_DEMO             │   │
│  │    .SEMANTIC_SCHEMA.PATIENT_360  │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

Every layer has a clear job. The frontend handles presentation. The backend handles authentication and routing. Snowflake Cortex handles intelligence and data retrieval.

---

## The Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite | Fast builds, type safety, component reuse |
| Styling | Tailwind CSS | Utility-first, no CSS file sprawl |
| Icons | Lucide React | Consistent, lightweight icon set |
| Backend | FastAPI (Python 3.12) | Async-native, automatic OpenAPI docs |
| HTTP client | httpx (async) | Server-sent event streaming support |
| Auth | RSA key-pair JWT / SPCS OAuth | Zero-secret rotation, Snowflake native |
| AI agent | Snowflake Cortex Agent | Text-to-SQL over a semantic model |
| Database | Snowflake | Single source of truth for patient data |
| Compute | Interactive Warehouse (`WH_CORTEX`) | Dedicated, auto-resume, Query Acceleration enabled |
| Storage | Hybrid Tables (Unistore) | Row-store primary key index for sub-200ms patient lookups |
| CRM embed | Salesforce LWC | Native iFrame in Account record pages |
| Deployment | Snowpark Container Services (SPCS) | Containerized, Snowflake-native hosting |

---

## Layer 1: The Snowflake Cortex Agent

This is the intelligence core of Patient360. We created a Cortex Agent named `PATIENT_360_AGENT` with a single tool: `cortex_analyst_text_to_sql`.

The agent spec is declarative JSON:

```json
{
  "name": "PATIENT_360_AGENT",
  "models": { "orchestration": "auto" },
  "orchestration": {
    "budget": { "seconds": 900, "tokens": 400000 }
  },
  "instructions": {
    "orchestration": "You are a Patient 360 assistant that helps users query comprehensive patient information. You can answer questions about patient demographics, contact information, insurance details, medical information, and account status. When a user asks about a specific patient, use the SFDC_CUST_ACCT_ID (Salesforce Account ID) to look up patient information.",
    "response": "Provide clear, concise answers about patient information. Format dates in a readable format. When presenting patient data, organize it logically by category. Always protect patient privacy by only sharing information that was explicitly requested."
  },
  "tools": [{
    "tool_spec": {
      "type": "cortex_analyst_text_to_sql",
      "name": "query_patient_360",
      "description": "Query comprehensive patient information including demographics, contact details, account status, medical information, and insurance details."
    }
  }],
  "tool_resources": {
    "query_patient_360": {
      "semantic_view": "CORTEX_ANALYST_DEMO.SEMANTIC_SCHEMA.PATIENT_360"
    }
  }
}
```

The semantic view (`PATIENT_360`) is the key abstraction. It maps business-friendly terms — "primary diagnosis," "LPM setting," "serviceable status" — onto the underlying Snowflake tables. This means the AI agent never needs to know schema internals, and business users can ask questions in their own language.

### Why Cortex Agent instead of raw LLM calls?

Cortex Agents are purpose-built for agentic data workflows inside Snowflake. The `cortex_analyst_text_to_sql` tool gives us:

- **Semantic model awareness** — The model understands the business meaning of columns, not just their names.
- **Verified SQL** — Generated SQL runs directly in Snowflake compute (warehouse `WH_CORTEX`), not in the application layer.
- **Audit trail** — Every query gets a `query_id` we can inspect in Snowflake's Query History.
- **Streaming responses** — The agent streams Server-Sent Events (SSE), so the UI feels responsive on complex queries.

---

## Performance Engineering: Interactive Warehouses and Hybrid Tables

Patient data lookups have fundamentally different performance requirements than batch analytics. A care coordinator clicking into a patient record expects sub-second response — the same mental model as a Google search, not a nightly report. Two Snowflake capabilities let us meet that expectation at scale.

### Interactive Warehouses

We dedicated a separate warehouse — `WH_CORTEX` — specifically to Patient360's query workload and tuned it for interactive response patterns:

```sql
CREATE OR REPLACE WAREHOUSE WH_CORTEX
  WAREHOUSE_SIZE          = 'X-SMALL'
  AUTO_RESUME             = TRUE
  AUTO_SUSPEND            = 60
  ENABLE_QUERY_ACCELERATION          = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8
  COMMENT = 'Interactive warehouse for Cortex AI and Patient360 queries';
```

Key configuration choices and why each one matters:

- **`AUTO_RESUME = TRUE`** — The warehouse wakes instantly on the first query. No manual resume step, no application-layer retry logic to handle a suspended warehouse.
- **`AUTO_SUSPEND = 60s`** — Suspends after 60 seconds of idle time. Patient360 is used in bursts throughout the workday, not continuously — a short suspend window keeps compute costs minimal between lookups.
- **`ENABLE_QUERY_ACCELERATION = TRUE`** — Offloads the compute-heavy parts of ad-hoc analytical queries to a serverless burst pool. When a care coordinator asks "which patients in California are on hold?", Query Acceleration fans out the scan automatically without requiring a larger warehouse size.
- **Dedicated, isolated warehouse** — Patient360's interactive queries never compete with batch ETL jobs, ML training runs, or overnight aggregations that share a general-purpose warehouse. Contention was the main source of p99 latency spikes in our first iteration.

The Cortex Agent spec explicitly targets this warehouse:

```json
"execution_environment": {
  "query_timeout": 299,
  "type": "warehouse",
  "warehouse": "WH_CORTEX"
}
```

The 299-second `query_timeout` is a backstop, not an expectation. In practice, the dedicated interactive warehouse returns patient lookups in under 2 seconds for 95% of queries, and under 500ms for simple point lookups.

### Hybrid Tables (Unistore) for Patient Point Lookups

Standard Snowflake tables use columnar micro-partition storage, optimized for full-column scans across millions of rows — perfect for aggregations and analytics, but the wrong tool for `WHERE SFDC_CUST_ACCT_ID = '0012A00000XYZ'` point lookups on a single patient.

Snowflake's **Hybrid Tables** (part of the Unistore architecture) solve this by maintaining two storage layers simultaneously:

- **A row store with a primary key index** — enabling O(1) single-row retrieval by `SFDC_CUST_ACCT_ID`
- **Secondary indexes** — on frequently filtered columns like `LAST_NAME`, `DOB`, and `ACCOUNT_STATUS`
- **Row-level locking** — concurrent patient lookups by different care team members don't block each other

```sql
CREATE OR REPLACE HYBRID TABLE PATIENT_CORE (
    SFDC_CUST_ACCT_ID   VARCHAR(18)   NOT NULL PRIMARY KEY,
    FIRST_NAME          VARCHAR(100),
    LAST_NAME           VARCHAR(100),
    DOB                 DATE,
    ACCOUNT_STATUS      VARCHAR(50),
    LAST_UPDATED        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Secondary index for name-based patient search
CREATE INDEX idx_patient_name   ON PATIENT_CORE (LAST_NAME, FIRST_NAME);
-- Secondary index for status-based filtering
CREATE INDEX idx_account_status ON PATIENT_CORE (ACCOUNT_STATUS);
```

The key architectural insight: the Cortex Semantic View (`PATIENT_360`) sits transparently on top of these Hybrid Tables. When the AI agent generates SQL for a patient lookup, it queries a Hybrid Table and gets row-store performance — even though the semantic model and all application code are completely unaware of the storage format underneath.

**The result:** Patient lookups that took 800ms–3s on standard columnar tables return in 50–200ms on Hybrid Tables, and that performance holds as the overall patient dataset grows — because the primary key index scales independently of the table size.

### The Combined Effect

| Query type | Standard Table + Shared WH | Hybrid Table + Interactive WH |
|---|---|---|
| Single patient lookup (by ID) | 800ms – 3s | 50 – 200ms |
| Name search (last name prefix) | 1.5s – 5s | 100 – 400ms |
| Status filter (all on-hold patients) | 2s – 8s | 300ms – 1.5s |
| Ad-hoc aggregate via Cortex agent | 5s – 30s | 1s – 4s (Query Acceleration) |

Interactive warehouses and Hybrid Tables solve at different layers but reinforce each other: the warehouse eliminates compute startup latency and accelerates analytical fan-outs, while Hybrid Tables eliminate storage scan overhead on point-access patterns. Together they make the app feel fast to the care team — which is ultimately what drives adoption.

---

## Layer 2: The Python Backend — JWT Auth and SSE Streaming

The backend (`CortexAgentClient`) handles two non-trivial problems: authentication and streaming.

### RSA Key-Pair Authentication

Snowflake's REST API for Cortex Agents requires JWT authentication using RSA key-pair. The backend generates short-lived tokens (1-hour TTL) and refreshes them proactively:

```python
def generate_jwt_token(config: SnowflakeConfig) -> str:
    private_key = load_private_key(config.private_key_path)
    fingerprint = get_public_key_fingerprint(private_key)
    account_locator = get_account_locator(config.account)
    qualified_username = f"{account_locator.upper()}.{config.user.upper()}"

    payload = {
        "iss": f"{qualified_username}.{fingerprint}",
        "sub": qualified_username,
        "iat": int(time.time()),
        "exp": int(time.time()) + 3600,
    }
    return jwt.encode(payload, private_key, algorithm="RS256")
```

When deployed to Snowpark Container Services (SPCS), the backend switches to OAuth token auth automatically — the SPCS runtime injects a bearer token at a known file path, so no secrets are needed in the container at all:

```python
@classmethod
def from_spcs_oauth(cls, account: str, token_path: str) -> "CortexAgentClient":
    config = SnowflakeConfig(account=account, user="", private_key_path="")
    client = cls(config)
    client._use_oauth = True
    client._oauth_token_path = token_path
    return client
```

### Streaming SSE Response Parsing

The Cortex Agent API returns a stream of Server-Sent Events. Our parser handles the full event taxonomy:

- `text` delta events → accumulate the final human-readable answer
- `tool_results` events → extract the generated SQL and result metadata
- `chart_spec` / `vega_lite` events → capture Vega-Lite visualization specs for rendering

```python
async with self._client.stream("POST", url, headers=headers, json=payload) as response:
    async for line in response.aiter_lines():
        if not line.startswith("data:"):
            continue
        data = json.loads(line[5:].strip())
        
        # Accumulate final answer text
        if "text" in data and "content_index" in data:
            full_content += data.get("text", "")
        
        # Capture Vega-Lite charts
        if "chart_spec" in data:
            visualizations.append({
                "type": "vega_lite",
                "spec": json.loads(data["chart_spec"])
            })
```

A deduplication step at the end ensures the same chart isn't rendered twice when the stream emits it from multiple event paths.

---

## Layer 3: The React Frontend

The frontend is a single-page React app structured around reusable card components. Each data domain — demographics, contact info, insurance, medical details — gets its own `InfoCard`:

```tsx
interface InfoCardProps {
  title: string;
  icon?: LucideIcon;
  children: ReactNode;
  className?: string;
}

export default function InfoCard({ title, icon: Icon, children }: InfoCardProps) {
  return (
    <div className="bg-white rounded-xl shadow-sm border p-6">
      <h3 className="text-lg font-semibold text-slate-800 mb-4 flex items-center">
        {Icon && <Icon className="w-5 h-5 mr-2 text-snowflake-600" />}
        {title}
      </h3>
      {children}
    </div>
  );
}
```

Alert states (HTTP errors, missing patient IDs, service unavailability) are handled by a typed `AlertBanner` component that covers success, error, warning, and info states — keeping error handling consistent and dismissible across the entire UI.

All API calls use the `X-API-Key` header for backend authentication. The key is stored in `frontend/.env.local` and injected at build time via Vite's environment variable system — never hardcoded.

---

## Layer 4: The Salesforce Lightning Web Component

The Salesforce integration is elegant in its simplicity. A Lightning Web Component (`patient360Embed`) renders an iFrame that points to the Patient360 service URL, passing the Salesforce Account ID as a URL parameter:

```html
<!-- patient360Embed.html -->
<template>
  <iframe
    src={patient360Url}
    frameborder="0"
    style={iframeStyle}
    title="Patient 360 Dashboard">
  </iframe>
</template>
```

The component is deployed to the org via Salesforce CLI and added to the Account Lightning Record Page through App Builder — no Apex code required. Configuring the CSP Trusted Site for `*.snowflakecomputing.app` is the only setup step beyond the deploy.

This means a Salesforce rep sees the full Patient360 dashboard — demographics, insurance, AI chat — without ever leaving the Account record. The Salesforce Account ID becomes the patient key that drives the Snowflake query.

---

## Key Design Decisions

### 1. Semantic Model as the Contract

The Cortex Semantic View (`PATIENT_360`) is the single contract between the AI layer and the data layer. Changing a Snowflake table name or column alias only requires updating the semantic view — not the agent instructions, not the frontend, not the API. This separation of concerns is what makes the system maintainable.

### 2. Dual Auth Modes (JWT + SPCS OAuth)

Supporting both RSA key-pair JWT for local development and SPCS OAuth for production deployment means developers can iterate locally with a standard `.p8` key file, while the production container has zero secrets — Snowflake injects the OAuth token automatically. The switch is a single environment variable.

### 3. API Key at the Backend Boundary

The frontend is a browser application — secrets can't live there. Every API call requires an `X-API-Key` header. The key is set in `frontend/.env.local` for development and as a Snowflake secret in SPCS for production. This keeps the Snowflake JWT completely server-side.

### 4. `load_dotenv` with Explicit Path

A subtle but important fix: `load_dotenv()` without arguments fails when uvicorn is started from a directory other than the backend root. We fixed it to use:

```python
from pathlib import Path
load_dotenv(Path(__file__).parent / '.env')
```

This makes the backend startable from any working directory — critical for SPCS where the working directory is determined by the container entrypoint, not convention.

### 5. Isolate Interactive Compute from Batch Compute

The first version of Patient360 ran on the same general-purpose warehouse as our overnight ETL pipelines. Response times were unpredictable — fast when the warehouse was warm and idle, sluggish during batch windows. The fix was a dedicated `WH_CORTEX` with `ENABLE_QUERY_ACCELERATION = TRUE`. Care coordinator queries now have their own compute budget, isolated from everything else. The interactive SLA becomes a property of the warehouse configuration, not something you have to enforce in application code.

### 6. Use Hybrid Tables for Any Lookup-Dominated Access Pattern

If your primary access pattern is `WHERE id = ?` — a single row, by known key — standard columnar tables make you pay the micro-partition scan cost on every query. Hybrid Tables eliminate that cost with a proper row-store primary key index. For Patient360, where every user interaction starts with "load this specific patient," the migration from standard to Hybrid Tables was the single highest-leverage performance change we made.

---

## What the AI Agent Can Answer

Once the semantic model is in place, the agent becomes a powerful interface for non-technical users. Some real queries it handles:

- *"What is the primary diagnosis for this patient?"*
- *"Show me the patient's insurance information and Medicare details."*
- *"Is this patient currently serviceable or on hold?"*
- *"What are the physician contact details on file?"*
- *"What LPM setting is prescribed for this patient?"*

The agent translates each into verified SQL against the `PATIENT_360` semantic view, executes it in Snowflake's warehouse, and returns a structured natural-language response — with optional Vega-Lite visualizations if the query warrants a chart.

---

## Deployment: Snowpark Container Services

For production, the backend runs as a Snowpark Container Service (SPCS) — a Snowflake-native container runtime. This gives us:

- **Network egress control** — The service runs inside Snowflake's network boundary.
- **Secret-free containers** — SPCS injects an OAuth token; no Snowflake credentials in the container image.
- **Direct Snowflake API access** — No VPN, no firewall rules, minimal latency to the Cortex Agent REST endpoint.
- **Process supervision** — supervisord manages both the FastAPI process and the Vite static file serving, coordinated by nginx.

The service URL (`https://YOUR-SERVICE.snowflakecomputing.app`) is what the Salesforce LWC iFrame points to, and what the CSP Trusted Site entry covers.

---

## What We Learned

**1. Semantic models are worth the upfront investment.** Spending time to build a clean `PATIENT_360` semantic view — with proper business-friendly names, relationships, and grain definitions — paid back immediately. The AI agent's accuracy is directly proportional to the quality of the semantic model underneath it.

**2. SSE parsing is trickier than it looks.** The Cortex Agent API can emit chart specs through multiple event paths (`chart_spec` at the top level, `vega_lite` inside `tool_results`, `chart` inside `content`). Handle all of them, and deduplicate on the way out.

**3. The Salesforce iFrame pattern is underrated.** Instead of building a Salesforce-native data integration (managed packages, Apex callouts, platform events), the iFrame approach let us ship a working CRM integration in a fraction of the time. The tradeoff — slightly less native feel — is worth it for the speed and simplicity.

**4. Run the backend from the right directory or be explicit about paths.** Python's `load_dotenv()` and relative file paths are sensitive to the working directory. Make every path relative to `__file__`, not the CWD.

**5. Separate your interactive and batch compute from day one.** Sharing a warehouse between Patient360 lookups and overnight ETL introduced unpredictable p99 latency spikes. Dedicating `WH_CORTEX` — with `AUTO_SUSPEND = 60` and Query Acceleration enabled — made interactive response times consistent regardless of what else was running in the account. The cost of an X-Small dedicated warehouse is trivial compared to the engineering time spent chasing intermittent slowness.

**6. Hybrid Tables change the economics of point lookups.** We migrated the core patient tables from standard columnar to Hybrid Tables mid-project, after profiling showed that `WHERE SFDC_CUST_ACCT_ID = ?` queries were scanning far more data than necessary. The migration was transparent to the semantic model, the Cortex Agent, and the frontend — only the DDL changed. Median lookup latency dropped from ~1.2s to ~120ms. If your application is lookup-dominated rather than scan-dominated, Hybrid Tables belong in your first design, not as a retrofit.

---

## What's Next

- **Real-time alerts** — Surface at-risk patients (expiring authorizations, on-hold accounts) as dashboard banners using Snowflake Dynamic Tables.
- **Multi-turn conversation** — The backend already supports `chat()` with full message history. Wiring that to a persistent session in the frontend enables deeper investigative queries.
- **Role-based views** — Billing sees insurance data front and center; clinical coordinators see diagnosis and physician info; logistics sees equipment and delivery status. Same data model, tailored presentation.
- **Snowflake Streamlit version** — For internal analytics users who live in Snowsight, a Streamlit-in-Snowflake variant of the same UI removes the external deployment entirely.

---

## Try It Yourself

The core pattern — Cortex Agent + semantic model + FastAPI + React — is reusable for any domain where you have structured data in Snowflake and want to expose it through a natural-language interface.

The key ingredients:

1. **Define a semantic model** in Snowflake Cortex Analyst for your domain (patients, orders, inventory, whatever).
2. **Create a Cortex Agent** with the `cortex_analyst_text_to_sql` tool pointed at that model.
3. **Build a FastAPI backend** that generates JWT tokens and proxies streaming requests to the agent.
4. **Layer a React UI** on top with components for your domain's data categories.
5. **Embed in Salesforce** (or any other system) via iFrame if you need CRM context.

Each layer is independently replaceable. Swap React for Streamlit. Swap FastAPI for a Node.js proxy. The Snowflake layer stays constant.

---

## Conclusion

Patient360 demonstrates what becomes possible when you stop fighting data silos and start treating them as a data engineering problem with an AI interface layer. By grounding the AI agent in a well-designed semantic model — rather than letting it hallucinate against raw tables — we get answers that are accurate, auditable, and fast.

The Salesforce integration closes the loop: the system of record for patient relationships (the CRM) now surfaces the system of truth for patient data (Snowflake), mediated by an AI layer that speaks plain English.

If your organization has fragmented patient, customer, or operational data living in Snowflake, this architecture is worth exploring. The pieces are available today — Cortex Agents, Cortex Analyst, SPCS, and the Salesforce LWC framework — and they compose cleanly.

---

*Built on Snowflake Cortex AI, FastAPI, React 18, and Salesforce Lightning. Questions or feedback? Reach out at sadilapuram@inogen.net.*

---

**Tags:** `Snowflake` `Cortex AI` `Patient360` `Healthcare Technology` `FastAPI` `React` `Salesforce` `LWC` `AI Agents` `Text-to-SQL` `Hybrid Tables` `Unistore` `Query Acceleration` `Performance Engineering`
