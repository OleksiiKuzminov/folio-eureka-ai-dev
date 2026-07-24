# Area Detection and House Style

Two lookup tables. Read this file at workflow step 1 (to pick the context file) and again before Scenario Analysis (to pick case shape, Type, and size).

## Area detection table

Context files live in `references/context/`. A story can span two areas (e.g. closing an order affects Finance) — read both context files.

| Area | Context file | Jira prefixes / keywords |
|---|---|---|
| Agreements | `references/context/agreements.md` | ERM, UIAG; "agreement", "agreement line", "SAS" |
| Bulk Edit | `references/context/bulk-edit.md` | UIBULKED, MODBULKOPS; "bulk edit", "identifier", "in-app", "commit changes", "preview of records matched", "errors accordion", "reason for error", "bulk edit profile", "query builder", "matched records", "suppress from discovery" |
| Check In | `references/context/check-in.md` | UICHKIN, CIRC; "check in", "return", "backdate" |
| Check Out | `references/context/check-out.md` | UICHKOUT, CIRC; "check out", "loan", "patron block" |
| Orders | `references/context/orders.md` | UIOR, MODORDERS; "purchase order", "POL", "PO line", "encumbrance", "fund distribution", "order template", "open order", "unopen", "reopen", "claiming", "donor information", "bindery active", "routing list", "acquisitions unit" |
| Invoices | `references/context/invoices.md` | UINV, MODINVOICE, MODINVOSTO; "invoice", "invoice line", "voucher", "approve invoice", "pay invoice", "cancel invoice", "pending payment", "lock total", "release encumbrance", "batch group", "adjustment", "pro rate", "export to accounting", "vendor invoice number", "approve and pay in one click", "update order status", "duplicate invoice" |
| Receiving | `references/context/receiving.md` | UIREC; "piece", "receive", "receiving title" |
| Finance | `references/context/finance.md` | UIF, MODFIN; "fund", "budget", "ledger", "fiscal year", "allocation", "encumbrance", "transfer", "rollover", "expense class", "acquisition unit" |
| Inventory | `references/context/inventory.md` | UIIN, MODINV; "instance", "holdings", "item" |
| Requests | `references/context/requests.md` | UIREQ; "request", "hold", "page", "recall", "pickup" |
| Mediated Requests | `references/context/mediated-requests.md` | UIREQMED, MODREQMED; "mediated request", "secure tenant", "secure request", "interim service point" |
| Users | `references/context/users.md` | UIU, MODUSERS; "patron", "user record", "custom fields", "proxy", "service points", "profile picture" |
| Fees/Fines | `references/context/fees-fines.md` | UIU, MODFEE, MODFEESFINES; "fee", "fine", "fee/fine", "charge", "pay", "waive", "refund", "transfer account", "manual charge", "overdue fine", "lost item fee", "fee/fine owner", "payment method", "bursar" |
| Loans | `references/context/loans.md` | UIU, CIRC; "loan", "loan details", "renew", "change due date", "declare lost", "declared lost", "claim returned", "claimed returned", "anonymization", "loan comments" |
| Organizations | `references/context/organizations.md` | UIORG, MODORGS; "organization", "vendor", "vendor record", "interface", "integration", "EDI", "account number", "accounting code", "contact people", "donor", "banking information" |
| Circulation Settings | `references/context/circulation-settings.md` | UICIRC, CIRCSET; "loan policy", "request policy", "overdue fine policy", "lost item fee policy", "circulation rules", "fixed due date schedule", "staff slips", "loan anonymization", "title level request", "TLR", "cancellation reasons", "closed library due date" |
| Patron Notices | `references/context/patron-notices.md` | UICIRC; "patron notice", "notice template", "notice policy", "triggering event", "loan notice", "request notice", "fee/fine notice", "notice token" |
| Circulation Log | `references/context/circulation-log.md` | UICIRCLOG, MODAUD; "circulation log", "circ log", "circ action", "log event", "logged" |
| Course Reserves | `references/context/course-reserves.md` | UICR, MODCOURSE; "course", "course reserve", "reserve item", "instructor", "crosslist", "cross-listed", "registrar id" |
| Data Import | `references/context/data-import.md` | UIDATIMP, MODDATAIMP, MODDICORE, MODSOURMAN, MODSOURCE, MODELINKS; "import", "job profile", "match profile", "action profile", "field mapping profile", "data import", "MARC import", "EDIFACT import", "LDR 05", "MARC Holdings", "file extension" |
| Data Export | `references/context/data-export.md` | UIDEXP, MDEXP; "export", "mapping profile", ".mrc" |
| MARC Authority | `references/context/marc-authority.md` | UIMARCAUTH, MODELINKS; "authority", "quickMARC" |
| MARC Bib (quickMARC) | `references/context/marc-bib-quickmarc.md` | UIQM, MODSOURCE; "quickMARC", "MARC bib", "MARC bibliographic", "LDR", "leader", "008 field", "derive", "controlled field", "linking authority" |
| eHoldings | `references/context/eholdings.md` | UIEH; "package", "title", "provider", "KB", "EBSCO" |
| Licenses | `references/context/licenses.md` | UILIC, MODLIC, ERM; "license", "amendment", "term", "core document", "supplementary document" |
| Lists | `references/context/lists.md` | UILISTS, MODLISTS, MODFQM; "list", "FQM", "query" |
| OAI-PMH | `references/context/oai-pmh.md` | MODOAIPMH; "harvest", "OAI" |
| Consortium Manager | `references/context/consortium-manager.md` | UICONSET (ui-consortia-settings), MODCON; "consortium", "ECS", "affiliation", "shared setting", "central tenant", "member tenant", "select members", "confirm share to all", "confirm member libraries", "authorization roles", "authorization policies", "data import logs", "data export logs" |

### API/protocol areas

OAI-PMH, and API-flagged cases in MARC validation (`API | ...` sections), assert HTTP requests and XML/JSON response fields rather than toasts, modals, and panes. Same rigor, different surface: "steps" are request URLs/params (e.g. `verb=ListRecords&metadataPrefix=marc21_withholdings&from=<date>`) and "expected results" are response contents and exact field mappings (e.g. holdings → MARC `952` subfields). Do not force a UI navigation/toast shape onto these. Execution Type may be `Karate` or `Backend Component` rather than `Manual`.

---

## House style by area

Measured from the real TestRail corpus, 2026-07-23. Match the target area's row: use the dominant `Type`, aim near the median step count, and set `User Journey = Yes` only where the flag column is non-trivial. Percentages are share of that area's cases. Resolve Type IDs via `get_case_types` (Functional=6, Other=7).

| Area | Dominant Type | Median steps | Journey flag | Shape note |
|---|---|---|---|---|
| Orders | Functional (94%) | 8 | ~0% | Functional, mid-size; bundle workflow ACs |
| Invoices | Functional (95%) | 8 | ~1% | Functional, mid-size |
| Finance | Functional (94%) | 7 | ~2% | Functional, exact money values |
| Organizations | Functional (85%) | 8 | ~15% | Functional; journey flag sometimes used |
| Mediated Requests | Functional (94%) | 6 | ~0% | Functional, ECS-by-default |
| eHoldings | Functional (62%) | 6 | ~9% | Lean Functional |
| Loans | Other (85%) | 6 | ~0% | Other, short/atomic |
| Fees&Fines | Other (79%) | **2** | ~1% | Other, very short atomic cases |
| Bulk Edit | Other (94%) | **14** | ~0% | Other, large multi-step cases |
| Agreements | Other (89%) | 8 | ~9% | Other |
| License | Other (98%) | 7 | ~3% | Other |
| Data Export | Other (93%) | 6 | ~0% | Other |
| Lists | Other (65%) | 6 | ~0% | Other/Functional mix, lean Other |
| Circulation Settings | Other (69%) | **3** | ~10% | Other, short |
| Circulation Log | Other (82%) | 5 | ~8% | Other |
| Course Reserves | Other (61%) | **3** | ~0% | Other, short |
| OAI-PMH | Other (84%) | 8 | ~0% | Other, API/protocol assertions |
| Check-in | ~50/50 | 6 | **~18%** | Mixed Type; journey flag used |
| Check-out | Func 51% / Other 43% | 6 | ~6% | Mixed Type |
| Data Import | Func 52% / Other 47% | 10 | ~1% | Mixed Type, large cases |
| Users | Func 52% / Other 46% | 6 | ~3% | Mixed Type |
| Requests | Func 53% / Other 45% | 5 | ~1% | Mixed Type |
| Patron Notices | Other 53% / Func 40% | **12** | ~2% | Lean Other, large cases |
| Inventory | Functional (67%) | 5 | ~0% | Functional, compact atomic; per-variant (record type / OS) |
| Receiving | Functional (99%) | 8 | ~0% | Almost pure Functional; piece flow + Inventory/Finance side effects |

For any area not in this table, sample the area's TestRail section directly and match what you see.
