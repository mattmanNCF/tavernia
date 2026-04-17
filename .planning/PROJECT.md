# Tavernia

## What This Is

Tavernia is a modular, open-source hospitality operating system that lets restaurants and catering operations replace commercial SaaS tools (QuickBooks, Toast, Kickfin, MarginEdge, Paycom, etc.) with self-hostable, bespoke alternatives. It is structured as a core platform with installable plugin modules, so operators can adopt one tool at a time without any disruption to the rest of their existing stack. The security value proposition is explicit: single-tenant installs and local-first data storage mean a breach of one customer can never expose another, and sensitive data (vendor pricing, margins, payroll, event inventory) never lives in a shared multi-tenant cloud.

Tavernia is also designed as a **natively agentic platform**. Each domain (accounting, invoicing, inventory, PoS, payroll, etc.) carries its own scoped context files (markdown) that orient any AI agent working in that domain — analogous to CLAUDE.md in software projects. The platform is model-agnostic: operators can swap AI providers (Anthropic, OpenAI, Gemini, local Ollama) without disrupting agent behavior. A shared wiki knowledge base accumulates task history and outcomes; agents read and write to it continuously, enabling recursive self-improvement and full continuity across model switches.

**The multiplicative enhancement principle:** Each module Tavernia adds does not merely provide its own functionality — it makes every other installed module smarter. A payroll module that knows labor cost feeds the invoice module's food cost % in a way no standalone tool can. A rental inventory module that tracks event load feeds the purchasing module's par level suggestions. The agent KB accumulates cross-domain context, so adding module N doesn't add N units of value — it multiplies the intelligence across all N modules already installed.

## Core Value

A hospitality operator's data — invoices, costs, tips, sales — stays on their own infrastructure, visible only to them, and the tools that process it are open-source and auditable.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Core platform with plugin/module system
- [ ] Invoice processing module (photo OCR + vendor PDF email → line-item extraction → QBO export / CSV)
- [ ] Role-specific views: chef/receiver, GM/ops, accountant
- [ ] QuickBooks Online API integration for AP/journal entry push
- [ ] Generic CSV/IIF accounting export as fallback
- [ ] Self-hosted deployment path (Docker or similar)
- [ ] Optional managed hosting path (single-tenant cloud instance per customer)
- [ ] Agentic framework: domain-scoped context files (one per module/domain) that orient agents to their working area
- [ ] Model-agnostic agent abstraction layer — swap provider (Anthropic, OpenAI, Gemini, Ollama) via config with no behavior regression
- [ ] Wiki knowledge base: persistent, structured markdown KB that agents read from and write to after each task
- [ ] KB self-improvement loop: agent task outcomes update the KB, which improves future agent performance in that domain
- [ ] KB continuity guarantee: any agent, on any model, bootstrapped from the KB reaches equivalent performance without manual reorientation

### Out of Scope

- Multi-tenant SaaS — defeats the security premise
- Native mobile apps (v1) — web-responsive first
- Real-time PoS (Toast replacement) — deferred to later module
- Tip reconciliation module (Kickfin replacement) — deferred to later module
- Native GL/accounting module (QuickBooks replacement) — deferred; QBO integration bridges gap for v1

### Future Modules (roadmapped, not v1)

- **Paycom integration module** — Payroll sync: labor costs feed food cost % calculations; tip reconciliation feeds payroll; schedule data feeds staffing cost projections. Paycom API integration for bi-directional data exchange.
- **Rental inventory management module** — Photo-based inventory tracking for catering/event equipment (chairs, tables, glassware, chafing dishes, chargers, linens, etc.). Photo capture at load-out and return; discrepancy detection; depreciation tracking; cross-event utilization analytics. OCR pipeline reused from invoice module.
- **Industry standards bridge layer** — Adapters for common hospitality data formats and protocols so each module slots into the operator's existing stack without disruption: MICROS/Opera POS export formats, ADP/Paychex payroll interchange, broadline distributor EDI (Sysco, US Foods), POS receipt formats (Toast, Square, Clover). An operator using only the invoice module still benefits from its ability to receive vendor data in whatever format their distributor uses.

## Context

- Target customer: Independent restaurants and small multi-unit operators — not chains with dedicated IT
- Pain point: SaaS tools like MarginEdge share customer data across a large, valuable target surface; AI-assisted attacks increasingly exploit this
- The bespoke/small-clientele angle is a genuine security differentiation, not just marketing — operators can self-audit the code
- Invoice flow today (with MarginEdge): staff photograph paper invoices (OCR) + vendors email PDFs; both paths need to be handled
- Day-to-day users span three roles with distinct needs:
  - **Chef / kitchen manager** — receives goods, reconciles received vs invoiced
  - **GM / ops manager** — monitors food cost %, approves payments
  - **Accountant / bookkeeper** — exports to accounting system, closes AP
- Accounting export is the #1 output — drives adoption because it replaces the most painful manual step

## Constraints

- **Architecture**: Plugin/marketplace model from day 1 — modules are installable add-ons, not monolithic features
- **Security**: Single-tenant (one DB per install) + local-first (data on operator's hardware) — non-negotiable
- **Deployment**: Must ship a working self-hosted path; managed hosting is an option, not a requirement
- **Accounting**: QBO API integration is the primary export; CSV/IIF is the fallback for non-QBO shops
- **AI**: Model-agnostic from day 1 — no provider SDK hardcoded into core; agent behavior driven by context files + KB, not model-specific prompts
- **KB design**: Knowledge base must be portable markdown (not a vector DB or proprietary format) so it survives model/provider switches and is human-readable/auditable
- **Industry compatibility**: Every module must work standalone against the operator's existing tools — Tavernia is never a walled garden. A single installed module should provide more interconnectivity with the operator's existing stack than any competing SaaS tool.
- **Multiplicative principle**: Module-to-module data sharing via the event bus and shared KB is mandatory, not optional. No module may be designed as if it's the only one installed. Cross-domain signals (labor cost from payroll → food cost % in invoicing; event rental load → purchasing par levels) must be first-class features, not post-hoc integrations.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Plugin architecture from day 1 | Prevents module coupling debt; marketplace model enables community contributions | — Pending |
| Invoice processing as first module | Highest operator pain, clearest data flow, bridges to accounting which is universal | — Pending |
| QBO API + CSV/IIF (not own GL) | Own GL is high complexity; QBO covers most restaurants; generic CSV covers the rest | — Pending |
| Local-first + single-tenant | Core security premise; no multi-tenant architecture to be introduced later | — Pending |
| Agentic framework as first-class architecture | Domain-scoped context files + model-agnostic abstraction + KB layer must be in core, not bolted on later | — Pending |
| KB in portable markdown, not vector DB | Human-readable, model-agnostic, survives any infrastructure change; performance via structured conventions not embeddings | — Pending |
| Multiplicative not additive module design | Modules must share cross-domain signals via event bus + KB; value compounds with each module added, not just stacks | — Pending |
| Industry standards compatibility first | Every module must work against existing operator stack (Paycom, QBO, Toast, ADP, distributor EDI) before Tavernia-native paths | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-17 after Paycom/rental inventory modules and multiplicative design principle added*
