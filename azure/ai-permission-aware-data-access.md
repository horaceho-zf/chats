# Using Azure AI only on data a user is allowed to see

**Scenario:** we already run App Services, users sign in with the company Azure ID
(Microsoft Entra ID), and we use Azure databases (MySQL + PostgreSQL), Azure Databricks,
and SharePoint. We want to add Azure AI so it can answer users' enquiries — but we must
guarantee that the AI only ever processes data (in the databases) and documents (on
SharePoint) that the signed-in user's role actually permits.

---

## The golden rule (read this first)

**Run authorization *before* the AI ever sees any data. Filter first — never filter after.**

Azure OpenAI / Azure AI Foundry is a **stateless inference engine**, not a security boundary
and not a database. It has no idea about your row-level permissions or SharePoint ACLs, and it
will not reliably enforce them. If you hand the model a full data store and prompt it
"only answer with what this user is allowed to see," you *will* leak data: models don't obey
such instructions reliably, and document text can be crafted to trick the model
(prompt injection).

The safe design is:

1. **Identify the user** from their Azure ID (Entra ID) token — the same identity already
   used at login.
2. **Compute the allowed scope** from that identity: which database rows, which
   SharePoint items, which Databricks tables the user may read.
3. **Fetch only that scoped subset.** The AI is given the already-authorized result and
   nothing else.
4. **The model generates an answer from that subset.**

Because the model *never saw* the unauthorized data, it is impossible for it to reveal it.
The security decision is made by Entra ID + your data tier, **not** by the AI.

> Corollary — two different AI patterns, very different risk:
> - **Retrieval / RAG (search-and-answer):** safe, because retrieval searches a
>   permission-scoped index and only the authorized chunks go to the model.
> - **Text-to-SQL (let the AI write the query):** dangerous, because the AI *writes* the
>   query and may omit the permission predicate. Only do this if the database itself
>   enforces row-level security as a hard backstop that the AI cannot bypass. Never rely on
>   the model to remember the `WHERE user = ...` clause.

---

## Layman's tier

Think of the AI as a librarian who is very good at talking but has **no memory of which
books you may read**. If you give the librarian the whole library and say "only talk about
the books Alice is allowed to read," you cannot trust that. Alice might be told something
from a book she isn't allowed to open.

The fix is to make sure the librarian **never sees the whole library**. Instead, your
existing security system (the Azure login and the restrictions already on your databases
and SharePoint) does the checking. For each question, the system:

- figures out who is asking (the Azure login),
- works out exactly which rows and documents that person may see,
- grabs **only those**, and
- passes **just those** to the AI to answer.

So the AI answers only from the slice of data you already gave it. If Alice isn't allowed
to see a file, that file never reaches the AI, so it can't be leaked — no matter how clever
the question. The important work is still done by the security you already have, **not** by
the AI.

You already have three different kinds of data with three different security models:

- **Databases** lock down by *row* (who can see which row).
- **SharePoint** locks down by *document* (who can open which file).
- **Databricks** locks down by *table/column* (who can read which data).

Each needs a slightly different approach, and that's the "Pro" section below.

---

## Pro tier

### 1. Map your existing sources to their permission model

| Source | Native permission primitive | Who enforces it |
| --- | --- | --- |
| Azure SQL / Azure Database for PostgreSQL | Row-Level Security (RLS) policies | The database, per connection |
| Azure Database for MySQL | Row-level filtering via **views** (no native RLS) | Your app or a view + session variable |
| SharePoint (Microsoft 365) | SharePoint permissions / Entra groups, exposed via Microsoft Graph | SharePoint / Graph, per user |
| Azure Databricks | Unity Catalog: row filters + column masking | Databricks, per query principal |
| Azure AI Search (if you index content) | Document-level ACLs: security filters, or native ACL label/SharePoint ingestion | The search service, per query |
| Azure OpenAI / AI Foundry | **None** — stateless, no permission awareness | Your app tier + the above |

### 2. The three realistic architectures

**A — Enforce in the app tier; AI sees only the filtered result (recommended for most).**
Your App Service receives the user's Entra ID token, resolves their groups/roles, then uses
each data source's permission model to fetch an authorized slice. It passes only that slice
to the model. The AI never connects to a database or SharePoint directly.
(For SharePoint, the cleanest version is **delegated access**: the app calls Microsoft Graph
*as the user*, so SharePoint enforces the user's own permissions automatically — no app-side
ACL replication needed. For databases, use per-user connections or RLS.)

**B — Enforce in the data tier; the AI queries a permission-scoped view.**
Each user gets a connection whose RLS predicate is bound to their identity (see §3). Then
even if something tries to query beyond the user's scope, the database itself refuses.
Best combined with A — treat the DB RLS as the hard backstop and never let the AI
author SQL without it (see the text-to-SQL warning above).

**C — Azure AI Search over an indexed copy, with document-level access control.**
This is the standard RAG pattern for large corpora (e.g., all SharePoint). You index
documents (and their permissions) once; at query time you attach the user's Entra token and
the search service returns only documents that user may read. See §4. This is what you want
for "AI answers questions across thousands of SharePoint files the user can see."

### 3. Databases — row-level security

The idea: even if a query runs, it is *physically* limited to permitted rows by the
database, not by app code that a bug or a model could bypass.

- **Azure Database for PostgreSQL / Azure SQL:** native RLS.
  - PostgreSQL: `CREATE POLICY p ON t USING (owner = current_setting('app.user'))`,
    then `ALTER TABLE t ENABLE ROW LEVEL SECURITY;`. Set the per-user value
    (`SET app.user = ...`) on each connection.
  - Azure SQL: `CREATE SECURITY POLICY` with an inline predicate function that reads
    `SESSION_CONTEXT(N'user_id')`. Set `SESSION_CONTEXT` per connection.
  - Per-user connection (the DB does the check) versus a **middle-tier** approach where the
    app sets the context then queries. Both work; the former is the strongest because the
    predicate is guaranteed.
- **Azure Database for MySQL:** there is **no native row-level security**. The common
  pattern is a **view** with a `WHERE` clause that reads a session variable:
  `CREATE VIEW safe_t AS SELECT * FROM t WHERE owner = @user;`, setting `SET @user = ...`
  per connection. Alternatively, enforce filtering in the app layer. Say so in any design
  doc — it is a real limitation and a reason to prefer Postgres/Azure SQL here.

**If you go text-to-SQL (AI writes SQL), RLS is mandatory**, because model-produced SQL
cannot be trusted to include the permission predicate. The database must enforce it.

### 4. SharePoint — document-level permissions

Two routes:

- **Route 1 — live, delegated Graph access (no index).** The app calls Microsoft Graph for
  SharePoint files using the *user's* token (delegated permissions). SharePoint/Graph
  enforces the user's real permissions, so results are correct by construction. Good for a
  smaller, on-demand set of files; you don't duplicate permissions anywhere.
- **Route 2 — indexed RAG with document-level access control.** Index SharePoint content
  into Azure AI Search once, with permission metadata. Azure AI Search now has
  **native document-level access control** (in preview at the 2026-08-01-preview API) that
  can ingest SharePoint ACLs, ADLS Gen2 ACLs / RBAC scopes, or Microsoft Purview sensitivity
  labels. At query time you attach the user's Entra token via the
  `x-ms-query-source-authorization` header and the service trims results to only what that
  token's claims permit. For older/indexing models, use the general-availability
  **security filter** pattern: store a `Collection(Edm.String)` field of principal IDs per
  document and filter with
  `search.in(security_field, 'id1, id2, ...')` using the caller's group IDs.

Key caveat for indexed ACLs: **permission changes are reflected only after the index is
synchronized** (next indexer run / push-API update). SharePoint ACL changes on items with
unique permissions are picked up incrementally per indexer run; changes inherited from a
parent scope (site/library/list/folder) need an explicit refresh. So to honor a just-revoked
permission promptly you need a fast sync cadence (see §5).

### 5. Databricks — clear governance required

Databricks uses **Unity Catalog** for governance. Set **row filters** and **column masking**
on Delta tables so that any query — including one issued by an AI agent — only reads what
the principal is allowed to see. Use the fact that Unity Catalog filters are enforced
server-side (in the query engine), which is exactly the hard backstop you want. Make sure the
AI's service identity is scope-limited (e.g., a service principal bound to a specific
credential/role) so it can't run as a super-user.

### 6. The guardrails to bake in

- **Never send the raw superset to the model with instructions to limit it.** Filter first.
- **Treat document content as untrusted input** (it can carry prompt-injection text). Apply
  system prompts / content filters as defense-in-depth, but know that the *downstream* (only
  authorized data retrieved) is what actually prevents leakage.
- **Don't accumulate user-slice data into shared training.** Use stateless inference +
  retrieval; don't fine-tune a shared model on per-user data.
- **Idempotent permission re-check per request** — never cache an authorization decision
  across requests if permissions can change.
- **Audit.** Log which user asked, which documents/rows were retrieved, and that the
  retrieval was permission-scoped. This is what makes it defensible.
- **Don't let app-only (service) identity bypass user permissions for user-facing queries.**
  Where possible use the user's token (delegated) so the source of truth enforces access.
  If you must use an app identity, then you own replicating and trimming permissions — more
  code, more risk.
- **Least-privilege service identities**: managed identities with narrowly-scoped roles, not
  broad storage/database admin.

### 7. A worked flow (what a request looks like)

1. User signs in to the App Service via Entra ID; app gets an access token for the user.
2. App gets the user's groups/roles (from the token or Microsoft Graph).
3. **Databases:** open a connection bound to that identity (RLS context set), run the
   (possibly model-augmented) query → get only permitted rows.
   **SharePoint:** call Graph as the user, or query Azure AI Search with the user's
   `x-ms-query-source-authorization` token → get only permitted documents/chunks.
   **Databricks:** run the query under a Unity-Catalog-scoped principal → only permitted
   data.
4. Combine the authorized slices into a small grounding context.
5. Send that context (and the user's question) to Azure OpenAI / AI Foundry. The model
   answers using *only* that context.
6. User sees the answer — and it can contain nothing outside their granted scope, because
   that data was never presented to the model.

---

## Azure services touched (with caveats)

- **Microsoft Entra ID** — identity source (`learn.microsoft.com/azure/entra`).
- **Azure App Service** — the enforcement point / orchestrator; use managed identity for
  service-to-service, user token for user-scoped access.
- **Azure OpenAI / Azure AI Foundry** — the stateless model.
- **Azure AI Search** — indexed RAG with document-level access control:
  - Doc-level ACL overview (preview): `learn.microsoft.com/azure/search/search-document-level-access-overview`
  - Security-filter pattern (GA): `learn.microsoft.com/azure/search/search-security-trimming-for-azure-search`
  - SharePoint ACL indexer (preview): `learn.microsoft.com/azure/search/search-indexer-sharepoint-access-control-lists`
- **Azure Database for PostgreSQL** (`learn.microsoft.com/azure/postgresql`) / **Azure SQL**
  (`learn.microsoft.com/azure/azure-sql`) — RLS.
- **Azure Database for MySQL** (`learn.microsoft.com/azure/mysql`) — no native RLS; use views.
- **Azure Databricks / Unity Catalog** (`learn.microsoft.com/azure/databricks`) — row filters
  and column masking.
- **SharePoint / Microsoft Graph** (`learn.microsoft.com/graph`) — permission-aware access.

> **Note:** a number of the strongest Azure AI Search access-control features (built-in ACL
> ingestion for SharePoint/ADLS Gen2, Purview sensitivity labels, token-based query
> enforcement) are **preview** and tied to a specific preview API version (2026-08-01-preview
> at the time of writing). Behaviors, supported data sources, and sync semantics vary by
> region and version, and preview features are not contractually supported. The plain
> **security-string-filter** pattern is generally available, API-agnostic, and the safe
> default when in doubt.

## Source of the golden rule

The entire ask reduces to one sentence: **keep the AI out of the authorization decision.**
Decide what the user may see with Entra ID and your existing per-source permission controls;
pass the model only the authorized results. The AI's job is to answer, not to enforce
permissions.
