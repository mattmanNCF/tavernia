# Feature Landscape

**Domain:** Hospitality back-office — invoice processing and AP automation for independent restaurants
**Researched:** 2026-04-17

---

## Dependency Chain

Every feature in this domain maps to one of five pipeline stages. This is the spine of the module:

```
Capture → Extract → Code → Approve → Export
(photo/email/EDI)  (OCR/line-item)  (GL mapping)  (review gate)  (QBO Bill API / CSV)
```

The entire invoice-processing module is this pipeline. Features that don't fit here belong in a separate module.

---

## Table Stakes

Features operators expect. Missing = product feels incomplete or broken. Derived from MarginEdge, xtraCHEF/Toast, and Craftable feature sets; confirmed by user review patterns.

| Feature | Why Expected | Stage | Complexity | Notes |
|---------|--------------|-------|------------|-------|
| Photo invoice capture (mobile) | Staff photograph paper invoices on delivery — this is the primary capture path today | Capture | Low | Web-responsive camera upload; no native app required |
| Email-to-invoice ingestion | Vendors email PDFs directly; must parse without human intervention | Capture | Medium | SMTP listener or forwarding address; parse PDF attachment |
| Line-item OCR extraction | Vendors, quantities, units, pack sizes, prices per line — not just totals | Extract | **High** | 90-95% accuracy on clean PDFs (xtraCHEF benchmark); degrades on handwritten; no human fallback in self-hosted — see stack note below |
| Vendor auto-recognition | Map vendor name on invoice to known vendor record automatically | Extract | Medium | String matching + fuzzy match; improves over time |
| GL code assignment | Each line item needs a GL account mapped before export | Code | Medium | Rule-based per vendor+item; user-trainable |
| Invoice review queue | Human review before export for exceptions and low-confidence extractions | Approve | Low | Simple approve/reject UI; not a SLA — operator-owned |
| Multi-role approval workflow | Chef confirms received goods; GM approves cost; accountant exports | Approve | Medium | Three-role permission model; sequential or parallel |
| QBO Bill API export | Push approved invoices as Bills to QuickBooks Online (with VendorRef, line items, GL codes) | Export | Medium | OAuth2 + entity mapping; API entities: Bill, Vendor, VendorCredit, JournalEntry |
| CSV/IIF fallback export | Non-QBO shops need a generic export path | Export | Low | CSV with configurable column mapping; IIF for QBO Desktop |
| Vendor management | Named vendor records, contact info, payment terms | Code | Low | Core lookup table; drives GL coding rules |
| Price history per item | Track what was paid per SKU over time per vendor | Extract | Low | Derived from invoice line items; no extra capture needed |
| Duplicate invoice detection | Prevent double-entry of the same invoice | Approve | Low | Hash on vendor + invoice number + amount + date |

**OCR complexity note:** Line-item extraction accuracy is the hardest technical problem in this module. It depends entirely on OCR/LLM choice — Tesseract struggles with non-standard layouts; modern vision LLMs (local or API) significantly improve accuracy on varied invoice formats. This is a stack decision, not just a feature. Flag for deeper research in STACK.md.

---

## Differentiators

Features that set Tavernia apart. Not universally expected, but valued — especially against the security-first positioning.

| Feature | Value Proposition | Stage | Complexity | Notes |
|---------|-------------------|-------|------------|-------|
| Local-first data sovereignty | Invoice data never leaves operator hardware; no shared analyst pool; auditable code | All | Low (arch) | Core premise — not a feature to build, but a constraint that makes all features more valuable |
| Raw data portability | MarginEdge users explicitly complain they cannot extract raw data from the platform; self-hosted DB ownership eliminates this | Export | Low | Postgres/SQLite directly accessible; no export gate |
| Confidence scoring with per-field review | Show extraction confidence per field; highlight uncertain fields for human review rather than processing everything or nothing | Extract | Medium | Forces transparency on OCR quality; builds trust faster than black-box processing |
| Price deviation alerts | Flag when a vendor charges more than the historical price for an item — prompt for manual review | Extract/Approve | Low | Computed from price history; valuable for catching distributor price creep |
| Vendor price comparison across vendors | Show price paid for same item across multiple vendors (e.g., tomatoes from Sysco vs. local distributor) | Extract | Medium | Requires item matching across vendors; MarginEdge calls this "Price Movers" |
| Plugin/module architecture | Operators install only what they need; community can add modules without forking | All | Medium | Isolated packages with defined API contracts; installable via CLI (Medusa/Mercur pattern); ships with 90% of needs but 100% extensible |
| Role-specific UX (three distinct views) | Chef/receiver sees delivery confirmation UI; GM sees cost dashboard; accountant sees export queue — not one-size-fits-all | Approve/Export | Medium | Three modes derived from distinct job functions; reduces training friction |
| Offline-capable capture | Delivery dock may have poor connectivity; capture should queue locally and sync later | Capture | Medium | PWA with service worker + local queue; sync on reconnect |
| Audit trail on every invoice | Every extraction, edit, approval, and export logged with user + timestamp | All | Low | Append-only event log per invoice; critical for AP compliance |

---

## Anti-Features

Features to explicitly NOT build in v1. Each has a deliberate reason.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Bill pay / payment execution | Adds PCI scope, bank integration complexity, and regulatory surface; QBO already handles payment execution | Export Bills to QBO and let QBO handle payment |
| Recipe costing module | Different domain (kitchen operations, not AP); high complexity; creates coupling between invoice module and menu database | Defer to a separate plugin module; not in v1 |
| Inventory counting | Same domain problem as recipe costing — separate concern from invoice-to-accounting flow | Defer to inventory plugin module |
| Demand forecasting / purchasing optimization | Valuable but complex; requires historical sales data and POS integration; MarginEdge users flag it as missing but it's a v2+ feature | Defer; design data model to support it later |
| Native mobile apps | Web-responsive handles capture well enough in v1; native app is expensive to maintain and blocks shipping | PWA with camera access covers the use case |
| EDI integrations | Enterprise distributor EDI (GS1/X12) is complex, expensive to certify, and used by chains, not independents | Email PDF parsing covers independent restaurant reality |
| Human review SLA (MarginEdge model) | MarginEdge's moat is a staffed analyst team that reviews extractions in 24-48 hours. Replicating this defeats the self-hosted premise and creates a scaling bottleneck | Confidence scoring + operator review queue is the answer |
| Multi-tenant SaaS mode | Defeats the core security premise; a breach of one customer exposes all | Single-tenant installs only; managed hosting = one instance per customer |
| Native GL / full accounting | High complexity; QuickBooks covers most restaurants; building a GL is a multi-year project | QBO API bridge + CSV/IIF fallback |
| Menu engineering / profitability analytics | Useful but requires POS sales data integration and recipe database; scope creep for v1 | Defer to a later analytics plugin |

---

## Feature Dependencies

Dependencies within the invoice-processing module pipeline:

```
Photo capture / Email ingestion
    └── Line-item OCR extraction
            └── Vendor auto-recognition
                    └── GL code assignment (requires vendor rules)
                            └── Confidence scoring (requires extraction output)
                                    └── Invoice review queue
                                            └── Multi-role approval
                                                    └── QBO Bill export (requires approved + coded invoice)
                                                    └── CSV/IIF export (same gate)

Price history ← depends on: Line-item extraction (running over time)
Price deviation alerts ← depends on: Price history
Vendor price comparison ← depends on: Price history + item matching across vendors
Duplicate detection ← depends on: Vendor recognition + invoice metadata
```

Plugin architecture is a dependency for everything: it is the delivery mechanism. Every module is an isolated package; the core platform provides auth, multi-role permissions, and the plugin registry.

---

## MVP Recommendation

The minimum viable product is the complete pipeline for one invoice path (photo capture preferred; email ingestion second), ending in a working QBO export.

**Prioritize for v1:**
1. Photo + email invoice capture
2. Line-item OCR extraction with confidence scoring
3. Vendor recognition and GL code assignment (rule-based, user-trainable)
4. Invoice review queue with three-role permission model
5. QBO Bill API export (OAuth2 + Bill entity)
6. CSV/IIF fallback export
7. Price history tracking (derived, low-cost)
8. Audit trail on every invoice
9. Duplicate invoice detection

**Defer to v2+:**
- Vendor price comparison / Price Movers
- Price deviation alerts
- Offline-capable capture (implement if field testing reveals connectivity issues)
- Recipe costing module
- Inventory counting module
- Forecasting and menu analytics

**The one thing that will make or break adoption:** QBO export accuracy. If Bills land in QBO with wrong GL codes or missing line items, accountants reject the tool immediately. Invest disproportionately in the Code → Export stage.

---

## Competitive Pricing Context

Understanding incumbent pricing informs positioning:

| Product | Price | Model |
|---------|-------|-------|
| MarginEdge | ~$350/month/location | SaaS, multi-tenant |
| xtraCHEF (Toast add-on) | ~$149+/month/module | SaaS, multi-tenant |
| Restaurant365 | $400+/month/location | SaaS, multi-tenant |
| CrunchTime | $5,000+/month (enterprise) | SaaS, multi-tenant |
| Tavernia (self-hosted) | Infrastructure cost only | Open-source, single-tenant |
| Tavernia (managed) | TBD — single-tenant instance | Managed hosting, per-customer |

Independent restaurants paying $350/month are price-sensitive. Self-hosted at infrastructure cost is a significant differentiator for cost-conscious operators who have minimal IT staff but someone willing to run Docker.

---

## Sources

- MarginEdge feature set: [MarginEdge How It Works](https://www.marginedge.com/how-it-works), [MarginEdge Inventory](https://www.marginedge.com/inventory-management)
- MarginEdge user complaints: [Capterra Reviews](https://www.capterra.com/p/187718/MarginEdge/reviews/)
- xtraCHEF/Toast AP automation: [xtraCHEF AP Automation](https://xtrachef.com/features/ap-automation/), [GetApp xtraCHEF](https://www.getapp.com/retail-consumer-services-software/a/xtrachef/)
- BirchStreet Invoice Management: [BirchStreet Invoice Management](https://birchstreetsystems.com/products/invoice-management/)
- QBO API surface: [Bill API reference](https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/bill), [BillPayment](https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/billpayment), [Knit QBO Integration Guide](https://www.getknit.dev/blog/quickbooks-online-api-integration-guide-in-depth)
- OCR accuracy benchmark: xtraCHEF Essential Guide to AP Automation (90-95% line-item extraction)
- Plugin architecture patterns: [Medusa/Mercur modular architecture](https://www.mercurjs.com/), [dotCMS Plugin Architecture](https://www.dotcms.com/blog/plugin-achitecture)
- Restaurant pain points: [Over Easy Office OCR Guide](https://www.overeasyoffice.com/blog/the-definitive-guide-to-ocr-invoice-processing-in-restaurants), [Altametrics AP Automation Guide](https://altametrics.com/blog/the-ultimate-guide-to-accounts-payable-automation-for-restaurant-owners.html)
