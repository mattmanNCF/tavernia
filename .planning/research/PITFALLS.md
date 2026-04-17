# Domain Pitfalls — Tavernia

**Domain:** Modular open-source hospitality operating system
**Researched:** 2026-04-17
**Confidence:** HIGH (QBO API specifics verified against official Intuit developer docs; OCR pitfalls verified against restaurant-specific sources; architecture pitfalls cross-referenced across multiple implementation sources)

---

## 1. Invoice OCR / Processing

### CRITICAL: Catch-Weight Items Break Every Naive Line-Item Pipeline

**What goes wrong:** Produce and meat invoices frequently contain "catch-weight" items — items sold by the actual weighed unit at delivery, not a fixed per-case price. A salmon fillet case invoiced at $8.72/lb × 14.3 lbs = $124.70 looks nothing like a dry-goods line item of qty × unit-price. Naive OCR extracts a number that looks like a price but is actually a weight, or vice versa. The extended total may be correct while every individual field is wrong.

**Why it happens:** Most OCR pipelines are designed around structured invoice formats (B2B SaaS, e-commerce, utilities) where qty × unit_price = line_total is invariant. Catch-weight breaks this invariant. No standard invoice schema captures "priced_by_weight" as a field type.

**Consequences:** Line items import into QBO at wrong unit costs. Food cost % calculations are silently wrong — the most important metric for any restaurant. Errors compound over weeks before anyone notices because the totals reconcile (the dollar amount is correct; only the cost-per-unit is wrong).

**Warning signs:**
- Any produce or meat vendor in the vendor list
- Invoices from Sysco, US Foods, or local farms with columns labeled "Weight," "Net Wt," or "Catch Wt"
- Line items where unit price × quantity does not equal extended price within a small tolerance

**Prevention:**
- Model the line-item schema with an explicit `pricing_type` field: `unit`, `catch_weight`, `case_price`
- After OCR extraction, run a validation pass: if `unit_price × qty ≠ extended_total` within 2%, flag as catch-weight candidate and surface for human confirmation, do not silently accept
- Store raw OCR output alongside parsed fields so reviewers can see what the engine read
- Build a per-vendor "invoice profile" that remembers which line items are catch-weight — most vendors repeat the same items

**Phase:** Invoice Processing module (Phase 2 or equivalent). Model the data schema before writing any OCR extraction code. Retrofitting catch-weight support after line-item schema is locked requires a migration.

---

### CRITICAL: Handwritten Annotations Are a Second Document Overlaid on the First

**What goes wrong:** Restaurant receivers write on invoices at the dock: crossed-out quantities, weight corrections, "short 2 cs," credit request codes. These annotations are semantically critical (they change what was actually received vs. invoiced) but OCR engines treat them as noise or misread them as part of the printed text.

**Why it happens:** OCR is trained on clean documents. Handwritten ink over printed text is an adversarial input to character recognition. The annotation and the original value compete in the same bounding box.

**Consequences:** System records invoice quantity as delivered quantity. Variance between ordered/invoiced and received is the entire purpose of a receiving workflow. If the system can't capture that variance, the chef has no tool for chargeback disputes with vendors — eliminating a core use case.

**Warning signs:**
- Any photo-capture workflow (camera photos from dock, not PDFs)
- Confidence scores on OCR output dropping below 80% on numeric fields
- Testing with real invoice photos, not clean PDF test fixtures

**Prevention:**
- Design the receiving confirmation step as a required human action, not an optional correction step. The model: OCR extracts a draft; the receiver confirms or corrects before saving. Never auto-approve photo-captured invoices.
- Display the original photo alongside extracted fields during confirmation so discrepancies are visually obvious
- Log all human corrections to build a per-vendor training dataset

**Phase:** Invoice Processing module. Must be baked into the UX flow design before building the confirmation screen, not bolted on after.

---

### MODERATE: Vendor Format Diversity Is Larger Than It Appears

**What goes wrong:** Testing with 3-5 vendor PDFs gives false confidence. Real operators deal with Sysco (structured EDI-derived PDF), a local farm (QuickBooks invoice printed to PDF, completely different layout), a specialty meat supplier (dot-matrix printer output, sometimes photographed rather than scanned), and a restaurant equipment repair invoice (generic Word template). These are not variations of the same format — they are different documents.

**Prevention:**
- Collect 20+ real invoice samples across vendor types before building extraction templates
- Design around AI-assisted extraction (vision model or document AI) rather than template-matching — template systems require per-vendor maintenance and break on any format change
- Build explicit confidence thresholds: below threshold → queue for human review, never silently drop or guess

**Phase:** Invoice Processing module. Validate sample diversity before committing to an extraction approach.

---

## 2. QuickBooks Online API Integration

### CRITICAL: Pushing Journal Entries Instead of Bills Breaks AP Entirely

**What goes wrong:** The QBO API offers a `JournalEntry` endpoint and a `Bill` endpoint. Teams reaching for accounting primitives often push journal entries (debit expense, credit AP) because it feels "correct" from a double-entry accounting perspective. It is wrong for AP.

**Why it happens:** Journal entries are general-purpose accounting tools. The `Bill` entity is QBO's AP workflow object. Bills appear in the "Pay Bills" queue, generate AP aging reports, and can be marked paid against a `BillPayment`. Journal entries that debit/credit AP do not appear in Pay Bills, do not age, and cannot be cleared by a bill payment — they create orphan balances in the AP account that accountants must manually remove.

**Consequences:** The accountant's QBO environment fills with phantom AP balances. The "Pay Bills" queue does not reflect actual outstanding vendor invoices. Month-end close requires manual reconciliation to untangle JE-based entries from real Bills. This is typically discovered at the end of the first accounting period, requiring a full reimport.

**Warning signs:**
- Any code that instantiates `JournalEntry` for vendor payables
- Accountant reports that "Pay Bills shows nothing" or "AP balance is doubled"
- Reconciliation errors on AP accounts in QBO

**Prevention:**
- Use the `Bill` endpoint for every vendor invoice. `Bill` requires `VendorRef` (vendor ID) and at least one `Line` with `AccountBasedExpenseLineDetail`. Ensure vendor records exist in QBO before pushing bills — create or match vendors first.
- `JournalEntry` is appropriate only for period-end accruals and corrections, not for AP transactions
- Test the full AP cycle: create Bill → run Pay Bills in QBO → verify it clears — before declaring the integration complete

**Phase:** QBO Integration module. This is a design constraint, not an implementation detail — it drives the vendor-matching data model.

---

### CRITICAL: OAuth Refresh Token Expiry Silently Disconnects the Integration

**What goes wrong:** QBO OAuth 2.0 issues access tokens (expire 1 hour) and refresh tokens. As of November 2025, Intuit announced that refresh tokens now have a maximum validity of 5 years (previously effectively permanent if used within 100 days). More importantly, each time a refresh token is used, QBO returns a new refresh token and the old one expires immediately. If the application fails to persist the new refresh token (crash, race condition, storage failure), the integration disconnects permanently and requires the user to reauthorize.

**Why it happens:** The rolling-token pattern is non-obvious. Developers persist the initial refresh token and assume it is long-lived. It is — until the first use, after which it is a single-use token.

**Consequences:** Integration silently stops working. No invoices push to QBO. The restaurant's accountant notices at month-end that QBO has no data from the past N weeks. For a non-technical operator, "reconnect to QuickBooks" is a confusing multi-step OAuth flow.

**Warning signs:**
- Refresh token stored once at authorization and never updated
- No error handling for `invalid_grant` errors on token refresh
- No monitoring or alerting for failed QBO API calls

**Prevention:**
- Store refresh tokens in a persistent, transactional store (database row with write confirmation, not a file or env var)
- Every token refresh must atomically: (1) call QBO refresh endpoint, (2) store the new refresh token, (3) invalidate the old one. Use a database transaction or optimistic locking.
- Implement a "QBO connection health" indicator visible to the GM role — green/red — so disconnection is noticed immediately, not at month-end
- Handle `invalid_grant` with a specific "reconnect to QuickBooks" user flow, not a generic error screen
- Since Intuit now sends in-app and email warnings 30 days / 7 days before token expiry, surface these in the UI as well

**Phase:** QBO Integration module. Token persistence and health monitoring must be built before any Bill-push logic.

---

### MODERATE: Rate Limits on Bulk Historical Import

**What goes wrong:** QBO Online enforces 500 requests/minute per company, 10 concurrent requests, 40/minute for batch operations. During initial import of historical invoices (onboarding a new restaurant with months of backlog), naive sequential pushing can hit rate limits immediately.

**Prevention:**
- Use QBO batch API (up to 30 operations per batch request) for bulk imports
- Implement exponential backoff with jitter on 429 responses
- Queue historical imports as background jobs with configurable rate — do not run them synchronously in a web request
- Warn users that historical import takes time; show progress, not a spinner

**Phase:** QBO Integration module. Rate limit handling required before any bulk operations.

---

### MODERATE: Minor Version API Drift

**What goes wrong:** QBO API uses "minor versions" (e.g., `minorversion=70`) to gate new features and occasionally deprecate fields. Omitting the minor version parameter returns the oldest compatible behavior, which may be missing fields needed for correct Bill creation.

**Prevention:**
- Pin a specific `minorversion` in all API calls
- Watch Intuit developer changelog for deprecations
- Integration tests against a QBO sandbox with the pinned version

**Phase:** QBO Integration module.

---

## 3. Plugin / Module Architecture

### CRITICAL: Plugins That Write to Core Tables Couple the Entire System

**What goes wrong:** The most common coupling failure in plugin architectures is not event bus coupling — it is schema coupling. A plugin adds a column to the `invoices` table ("needed just one extra field"), another plugin adds a join table against `vendors`, a third embeds plugin state in a `metadata` JSON blob on a core entity. Now every migration must account for plugin-owned columns in core tables. Removing a plugin requires schema surgery. Two plugins can conflict on the same column name.

**Why it happens:** It is faster to add a column than to design an extension table. The architecture feels clean until the third plugin arrives.

**Consequences:** The "installable module" promise breaks. Uninstalling a module is unsafe because data is entangled in core tables. The database migration graph becomes a directed acyclic graph of plugin dependencies. Community plugins can corrupt the core schema.

**Warning signs:**
- Any plugin migration that references a core table with ALTER TABLE ADD COLUMN
- Plugin code that imports core table models directly rather than via a published API
- Hotfixes that add `plugin_data` JSON columns to core tables

**Prevention:**
- Core tables are owned exclusively by the core platform. Plugins may not add columns to core tables under any circumstances.
- Each plugin owns its own tables, prefixed with the plugin name (e.g., `invoice_ocr_extractions`, not `invoices.ocr_result`)
- Core tables expose a stable typed API (service layer or repository pattern) — plugins call the API, never query core tables directly via SQL
- Plugin-to-core associations use foreign keys into core tables from plugin tables (many plugin rows → one core row), never the reverse
- Core events (invoice created, vendor updated) are published to an event bus; plugins subscribe and write to their own tables

**Phase:** Core Platform (Phase 1). This constraint must be encoded in the plugin contract before any plugin is written — including the first-party invoice module. If the first-party module violates it, all future community plugins will follow suit.

---

### CRITICAL: Event Bus Without Versioning Becomes a Coupling Vector

**What goes wrong:** An event bus solves the "no direct imports" problem but creates a new coupling: event schema coupling. If the `invoice.created` event changes its payload shape (adds a field, renames one), every plugin that consumes it must update simultaneously. Without versioning, a core upgrade silently breaks plugins written against the previous event schema.

**Prevention:**
- Version all event schemas from day one: `invoice.created.v1`, `invoice.created.v2`
- Old versions are kept alive for a deprecation period (2-3 releases)
- Event consumers declare which version they handle; the bus routes accordingly
- Treat event schema changes as a breaking API change — require changelog entry and migration guide

**Phase:** Core Platform. Design event versioning into the bus before the first event is emitted.

---

### MODERATE: Plugin Discovery Becoming a Hidden Dependency Graph

**What goes wrong:** Plugin A depends on Plugin B's vendor normalization service. Plugin C depends on Plugin A's cost calculation. The plugin system now has undeclared runtime dependencies. Installing Plugin C without A fails mysteriously. The marketplace lists plugins without surfacing these dependencies.

**Prevention:**
- Plugins declare explicit `requires` and `conflicts` in their manifest
- The install flow validates the dependency graph before installation
- No inter-plugin function calls — cross-plugin communication only via the event bus or a shared service layer owned by core

**Phase:** Core Platform. Build the manifest schema before publishing any plugin.

---

## 4. Self-Hosted Deployment for Non-Technical Operators

### CRITICAL: No Update Path = The System Rots on First Install

**What goes wrong:** V1 ships with a Docker Compose file and a "run this command" setup guide. The operator installs it, it works, and then nothing is ever updated. Security patches do not get applied. New features do not reach users. The operator discovers an exploit or a broken feature 6 months later and has no path to update without potentially breaking their data.

**Why it happens:** Updates are a deployment concern, not a feature concern. They get deferred until "after launch" and never built.

**Consequences:** The entire security value proposition ("you can audit the code running on your system") becomes "you are running 6-month-old unaudited code." More concretely: a bug in invoice parsing that over-charges for produce costs real money and is not fixed until the operator notices and someone walks them through a manual update.

**Warning signs:**
- No `./tavernia update` command in the tooling
- Docker images tagged `latest` instead of pinned to release versions
- No in-app notification of available updates

**Prevention:**
- Ship an update mechanism with v1. Options in order of operator-friendliness: (a) Watchtower or similar auto-pull (opt-in), (b) in-app "update available" banner with one-click update script, (c) versioned release manifest the app polls
- Pin all Docker image versions to semver releases — `latest` is a footgun in production
- Separate data volumes from application containers explicitly in the Compose file so `docker compose pull && docker compose up -d` is safe and does not touch data
- Document the update path in the setup guide, not as an afterthought

**Phase:** Core Platform / Deployment (Phase 1). An update mechanism must ship with v1. Do not defer.

---

### CRITICAL: Backup Is Not a Feature — Until Data Is Lost

**What goes wrong:** Operators run the system on a NUC or a VPS. No automated backups. A disk fails, or the VPS provider terminates the instance, or someone accidentally runs `docker compose down -v`. Three months of invoice data is gone. The operator's accountant has no records for the last quarter.

**Why it happens:** Backups feel like infrastructure, not product. They are always deferred.

**Consequences:** Permanent data loss. Regulatory exposure (food costs and invoices are tax records). Loss of trust and likely churn.

**Warning signs:**
- Persistent data volumes not explicitly documented in Compose file
- No `./tavernia backup` command
- Setup guide does not mention backups

**Prevention:**
- Ship a `./tavernia backup` command that dumps the database to a timestamped file and optionally syncs to an S3-compatible store or a remote path
- Make backup configuration a required step during initial setup — not optional, not skippable
- Display "last backup: [date]" in the admin UI. If last backup > 7 days, show a warning.
- Clearly document volume names in the Compose file so operators know what to protect

**Phase:** Core Platform / Deployment (Phase 1). Required before go-live.

---

### MODERATE: "Self-Hosted" Assumes Linux; Most Restaurant Operators Have Windows PCs or Macs

**What goes wrong:** Docker Compose setup guides written for Linux VPS deployments confuse operators who want to run this on a Windows machine in the back office. Docker Desktop on Windows has networking differences, file path differences (`C:\path` vs `/path`), and permission model differences that cause subtle failures.

**Prevention:**
- Test the Compose setup on Docker Desktop for Windows and Mac before declaring self-hosted support
- Provide platform-specific setup guides (Linux server, Windows, Mac), not a single generic guide
- Consider a one-click installer (e.g., a shell script or a small Electron wrapper) for the Windows/Mac path — operators should not be copy-pasting commands

**Phase:** Deployment hardening phase.

---

### MODERATE: Secrets in Compose Files Get Committed to Git

**What goes wrong:** Operators copy the example `docker-compose.yml`, add their QBO client secret and database password directly into the file, and commit it to a shared folder or git repository.

**Prevention:**
- Use `.env` files for all secrets, with `.env.example` containing placeholder values
- `.env` is in `.gitignore` in the project template
- Setup script generates a strong random secret for the app secret key on first run
- Document clearly: "Never put real credentials in docker-compose.yml"

**Phase:** Core Platform (Phase 1).

---

## 5. Multi-Role UX

### CRITICAL: Building Views Before Validating the Data Model Means Rebuilding Views

**What goes wrong:** The chef view needs "received quantity vs. invoiced quantity." The accountant view needs "extended cost per GL account." The GM view needs "food cost % by category this week." These look like view problems, but they are data model problems. If the invoice line item schema does not capture `received_qty` (separate from `invoiced_qty`), `gl_account_assignment`, and `category`, the views cannot be built — or they are built with stubbed data that breaks when real invoices arrive.

**Why it happens:** UX design focuses on screens, not on what fields underlie them. The data model is specified after the wireframes, rather than the wireframes being derived from the data model.

**Consequences:** First build of the chef receiving screen has to be thrown out when it becomes clear there is no `received_qty` field. The accountant view requires a schema migration to add GL codes. Each view drives a separate retrofit.

**Warning signs:**
- Wireframes completed before data model is specified
- "We'll add that field later" during data model review
- Any role's core workflow represented as a TODO in the schema

**Prevention:**
- For each role, list the 5 questions they must be able to answer from the system, then work backward to required data fields. Do this before designing any screens.
- Chef: Did I receive what was invoiced? (requires `invoiced_qty`, `received_qty`, `unit`, per line item)
- GM: What is my food cost % by category this week? (requires `category`, `extended_cost`, date, and sales data linkage)
- Accountant: Which invoices are unpaid and what GL accounts are they coded to? (requires `gl_account`, `payment_status`, QBO sync status)
- Build the data model first, validate it covers all three roles, then design views

**Phase:** Invoice Processing module (Phase 2). Data model must be signed off before any front-end work begins.

---

### CRITICAL: Permissions Modeled as Role Booleans Cannot Handle Real Restaurant Org Charts

**What goes wrong:** `is_chef`, `is_gm`, `is_accountant` as boolean columns on the user. Works until the first restaurant where the GM is also the bookkeeper, or a multi-unit operator where the accountant sees all locations but each GM sees only their location. Boolean role flags do not compose.

**Why it happens:** RBAC seems like over-engineering for a small tool. It is not.

**Consequences:** Operators cannot configure the system to match their org. The accountant who is also a part-time GM sees one view and loses the other. Multi-location operators hit this immediately.

**Prevention:**
- Model roles as a many-to-many: `user_roles` join table with `(user_id, role_id, location_id?)`. A user can have multiple roles, optionally scoped to a location.
- Permissions are derived from roles at query time, not stored as boolean flags
- Even if multi-location is out of scope for v1, the schema must not prevent it — use `location_id NULLABLE` rather than omitting the column entirely

**Phase:** Core Platform (Phase 1). This must be in the user/auth schema before the first role-based view is built.

---

### MODERATE: Role-Specific Terminology Creates Comprehension Gaps

**What goes wrong:** The accountant calls it a "Bill." The chef calls it a "delivery" or "invoice." The GM calls it a "vendor order." If the UI uses one vocabulary throughout ("invoice" everywhere), the chef thinks they are looking at a billing system, not a receiving tool. If the UI adapts terminology per role without a shared mental model, the same record looks like three different things and the team cannot talk to each other about it.

**Prevention:**
- Define a canonical internal vocabulary (entity names in code) and a per-role display vocabulary (labels in the UI)
- Code: `vendor_invoice`, `line_item`, `receipt_confirmation`
- Chef UI: "Delivery," "Items Received," "Confirm Delivery"
- Accountant UI: "Bill," "Line Items," "AP Status"
- Do user interviews (even informal ones — a 15-minute call with a working chef and a restaurant accountant) before finalizing UI copy

**Phase:** Invoice Processing module front-end phase.

---

### MODERATE: Notification Model Assumes Everyone Checks the Same Dashboard

**What goes wrong:** "Invoice approved by chef — accountant needs to export to QBO" is a workflow handoff. If the accountant only logs in when they feel like it, the invoice sits in limbo. Teams build notifications last, or not at all, and discover that the multi-role workflow breaks down at handoff points.

**Prevention:**
- Map every role handoff as an explicit workflow event: chef confirms delivery → system creates a task in accountant queue
- Build a simple notification/task queue (not email, just an in-app badge count) in Phase 1 — even if it is primitive, it closes the loop
- Email notifications for pending accountant actions are optional but high-value for non-daily users

**Phase:** Invoice Processing module. Model handoff events alongside data model design.

---

## Phase-Specific Warning Summary

| Build Phase | Key Pitfall to Preempt | Critical Action |
|-------------|------------------------|-----------------|
| Core Platform | Plugin schema coupling | Define and enforce plugin contract (no core table mutations) before first plugin |
| Core Platform | User permission model | Build role-as-join-table before any role-specific view |
| Core Platform | Deployment / update | Ship update mechanism and backup tooling with v1 |
| Invoice OCR module | Catch-weight schema | Model `pricing_type` field before writing extraction code |
| Invoice OCR module | Annotation capture | Design human confirmation as required step, not optional |
| QBO Integration | Bill vs JE | Use Bill endpoint exclusively; test full Pay Bills cycle |
| QBO Integration | Token persistence | Build transactional token store and health indicator before Bill-push |
| Role-Based UX | View-driven data model | Derive schema from per-role question list before any wireframes |
| Deployment hardening | Windows/Mac gaps | Test Docker Desktop path before announcing self-hosted support |

---

## Sources

- QuickBooks Online OAuth 2.0 FAQ: https://developer.intuit.com/app/developer/qbo/docs/develop/authentication-and-authorization/faq
- Intuit refresh token policy changes (Nov 2025): https://medium.com/intuitdev/important-changes-to-refresh-token-policy-8443779d40db
- QBO API rate limits: https://truto.one/blog/how-much-does-the-quickbooks-api-cost-2026-pricing-rate-limits
- QBO Bill API reference: https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/bill
- QBO community: JE vs Bill AP discussion: https://quickbooks.intuit.com/learn-support/en-us/reports-and-accounting/accounts-payable-in-general-journal/00/204131
- Restaurant OCR guide: https://www.overeasyoffice.com/blog/the-definitive-guide-to-ocr-invoice-processing-in-restaurants
- OCR limitations: https://docuclipper.com/blog/ocr-limitations/
- Catch weight invoicing: https://fsiedi.com/help/loblaws/en/dc/varweight.html
- Plugin architecture coupling: https://arjancodes.com/blog/best-practices-for-decoupling-software-using-plugins/
- Self-hosted Docker pitfalls: https://dev.to/danieljglover/docker-compose-self-hosted-services-guide-4hj4
- Docker in production: https://docs.docker.com/compose/how-tos/production/
