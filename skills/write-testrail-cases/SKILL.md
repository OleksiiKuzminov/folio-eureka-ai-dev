---
name: write-testrail-cases
description: Use when writing, generating, or adding manual test cases to TestRail. Produces structured cases with preconditions, step-by-step actions with expected results, and all required metadata fields. Also use when asked to cover a User Story with test cases, prepare cases for a new feature, or post cases to a TestRail section via API.
---

# Write TestRail Cases

> **Core principle:** A test case exists to verify a business rule, not to document a click path. Navigation is the means; assertions about the resulting state are the point. A case whose expected results could be true even if the feature were broken is a failed case.

## Mandatory workflow

1. **Detect the area** (rules below) and **read the matching context file** in `references/context/`. Never skip. If the area cannot be determined, ask.
2. **Assess context quality.** Enrich (see below) if ANY holds: file missing; header says DRAFT; fewer than 10 Key Business Rules; no "Exact UI Texts" section or fewer than 3 confirmed toasts; more than 5 "Known Gaps" items.
3. **Read `references/examples.md`** — golden cases from the team's TestRail. Match their verification depth and style.
4. **Read the story** (fetch via Jira MCP if given an ID). Extract Summary, Description, Acceptance Criteria, Development Team (→ `custom_dev_team`), and the ticket key(s) for `refs`.
5. **Map the story to the context file's Key Business Rules.** Every touched rule is a scenario candidate; so is every acceptance criterion.
6. **Present Scenario Analysis** and wait for confirmation.
7. **Generate cases**, one per confirmed scenario.
8. **Run the self-review gate** on every case before showing it. Fix violations silently.
9. **Ask for the Section ID**, show the posting preview, wait for explicit confirmation, then post. See `references/testrail-api.md`.

### `refs` is what the case verifies

Not "the story key". Very often it is a Bug the case regression-covers, a Tech-Debt/automation ticket, or several related tickets across projects (`UIF-525, UIF-526, MODFIN-391, MODFIN-421`). Include every ticket the user gave and every ticket the case directly verifies. Do not auto-add unrelated links Jira happens to show (parent epics, sibling subtasks, review tasks pulled in by enrichment). If the case exists to cover a bug and there is no story, the bug key is correct — never invent a story-style key. When a case genuinely exercises sibling FE/BE tickets, list all of them.

## Area detection

Read `references/areas.md` for the area table and per-area house style.

Determine the area from, in priority order: **the apps and entities described in the story text**, the Development Team field, then the Jira project prefix as a hint only. **If story content contradicts the prefix mapping, trust the story content.** Confirm in one line ("Story describes X — loading `<file>` context") rather than asking, unless detection is genuinely ambiguous.

**The story is the source of truth.** If the user's instructions contradict it (e.g. "this is not an ECS story" while the story carries an `ecs` label, mentions tenants/affiliations, or lives in a consortia module), do not silently pick a side — flag the conflict in one line and ask. Same for ECS Enabled: derive it from the story's labels and text.

If no context file exists for the area, tell the user: _"I don't have a context file for this area yet. I'll generate cases based on the User Story alone — results may be less accurate for domain-specific preconditions."_

### Using the context file

- **Key Business Rules is the scenario source.** For each rule the story touches, ask: does this story change it, depend on it, or risk breaking it?
- Use the domain model for realistic preconditions — which records must exist, in what status, with what values (an encumbrance needs a Fund, a Budget in an active Fiscal Year, an Open PO with a POL fund distribution).
- Use documented side effects to build verification steps: if an action touches N entities, the case verifies all N.
- Use exact terminology from the context file — never invent UI labels.
- **Context files are read-only for this skill.** Never modify, overwrite, or delete them. If one looks outdated, say so and suggest running `build-app-context`.

### Lightweight context enrichment

Fast and targeted to the story — not a `build-app-context` run. Writes nothing to disk.

**Fetch, max 3 GitHub files in priority order:** the area's `translations/<ui-module>/en_US.json` (values for keys containing the story's feature keywords); `package.json` → `stripes.permissionSets` (candidate Capability Set names); one Cypress fragment from `stripes-testing/cypress/support/fragments/<area>/` matching the feature (UI labels, selector names).

**Fetch from Jira:** issues directly linked to the story (subtasks, related bugs, parent epic — one call each); extract acceptance criteria as extra business-rule candidates. Issues fetched for enrichment are a *context source* — they do not enter `refs`.

**Never fetch:** all closed stories or all bugs for the component (that is `build-app-context` scope); more than 3 GitHub files; any file over 150 KB.

Add extracted strings to in-memory context only, and note it in Scenario Analysis:
> ⚠️ Context was enriched from GitHub/Jira for this session. Run `build-app-context` to make the enrichment permanent.

If enrichment returns nothing useful, say so and proceed on the context file alone.

## Scenario Analysis

```
I've read the story (refs: XXX-123) and the <area> context. Business rules touched: <rule numbers/short names>.

Here are the scenarios I plan to cover:

Happy path:
1. User can successfully [main action] with valid data → Critical Path

Business-rule verification:
2. [action] releases/creates/updates [entity] with [exact state] → Critical Path

UI verification:
3. [page/modal] shows all required elements on open → Extended

Capability boundaries:
4. User without [Capability Set] cannot [action] → Critical Path

Negative / edge cases:
5. [action] fails when [condition] → Extended

Does this coverage look complete? Any scenarios to add or remove before I write the cases?
```

Wait for confirmation or corrections before generating.

**Think like a senior QA — propose integration scenarios the ACs don't spell out** (fiscal-year rollover, create-from-template, sibling-gating-off, record-open side effects), derived from the context file's domain knowledge, in their own group so the user can confirm scope:

```
Inferred integration journeys (not in the ACs — from domain knowledge, confirm scope):
J1. Create + edit an order carrying the feature across a fiscal-year rollover → Functional / Critical Path / Journey
J2. Create an order from a template that has the feature preconfigured → Functional / Critical Path / Journey
J3. Feature behavior when a new fiscal year begins → Functional / Extended / Journey
```

### Case shape: match the area, not a global rule

The corpus shows Type, case size, and journey-flag usage vary **strongly by area** (`references/areas.md`). There is no universal "journey-first" or "atomic" rule.

- **Size follows the area's median.** Fees&Fines ~2 steps, Circulation Settings / Course Reserves ~3, acquisitions 7–8, Data Import ~10, Patron Notices ~12, Bulk Edit ~14. Don't emit 14 one-AC stubs where the area writes bundled workflow cases; don't force a giant journey where the area writes tight atomic checks. Real example: for UIOR-1530 the team wrote 4 large bundled cases (C1385639/C1395029/C1404901/C1404902), not 13 one-AC stubs.
- **Type is per-area.** Use the dominant Type from the area's row; resolve IDs via `get_case_types`.
- **`User Journey` defaults to `No`.** It is 0–3% in most areas. Do not set it `Yes` just because a case walks several steps — the team writes large bundled cases and still leaves it `No`. Set `Yes` only in areas that actually use the flag (Check-in ~18%, Organizations ~15%, Circulation Settings ~10%) or for an explicit end-to-end lifecycle in a circulation area.
- **Lifecycle / workflow stories** (request lifecycle with notices, order open→receive→pay, import job stages) → one journey case per flow variant, walking the whole lifecycle and verifying every checkpoint. Flow variants (Page vs Hold, item-level vs title-level) become separate journey cases, not separate per-checkpoint cases. **Never split one lifecycle into per-checkpoint stubs** — an executor would rebuild the same preconditions N times and intermediate transitions would go unverified.
- **Feature / element stories** (a new column, modal, validation, setting) → atomic cases, `User Journey = No`, sized to the area's median.

**Terse titles can hide a journey.** "Item with at least one open request" often means *walk every outcome that condition produces in one case*. The real case C7148 is a single journey: check in at a non-pickup service point (→ In transit, transit slip, reprint), then at the pickup service point (→ Awaiting pickup, hold slip, reprint). When a condition resolves to multiple end states, default to a journey covering all of them and confirm in Scenario Analysis.

### Verify effects at their real destination

- Notices → the **mailbox** from preconditions: exact subject + body contains the configured tokens. A Circulation log "Notice sent" entry is an additional check, never the only one.
- Exports → download and check the produced file, not only the log row.
- Finance effects → the transaction and budget values, not only a toast.

### Scenario coverage checklist

- [ ] Primary happy path (valid data, expected flow, correct end state)
- [ ] **Every Key Business Rule the story touches — verified by explicit state assertions**
- [ ] UI element verification (buttons, labels, columns, counts — especially for new pages)
- [ ] Edit and duplicate flows; delete flow including Cancel, confirm, and removal from list
- [ ] Capability boundaries: restricted user cannot act outside their Capability Sets — and the case also asserts what they still CAN see/do
- [ ] Negative: invalid input, wrong profile, empty files, referenced records blocking deletion
- [ ] Edge: repeated UUIDs, records referenced in multiple places, boundary values
- [ ] Status field changes (exact before/after values); toasts with exact text
- [ ] Modal elements verified inline on first open; full inventory only when the modal is under test
- [ ] Interactive modal buttons (Swap, Recalculate) get a dedicated scenario — fields, errors, and button states after clicking
- [ ] Inline edit: Save/Cancel appear inline, Save disabled until change, row returns read-only on Cancel
- [ ] Multi-select dropdowns: all options shown, stays open after selection, Save enables/disables correctly
- [ ] Search & filter pane: all accordions with initial expanded/collapsed state; Search/Reset disabled by default
- [ ] Pagination; date range filters
- [ ] Drag and drop / reordering (if record order matters); multi-tab (only if the story covers concurrent editing)
- [ ] Multi-tenant / ECS behavior (only if applicable)
- [ ] Load/performance for bulk operations (mark Extended)
- [ ] **Every entry point for the touched action, and every selection state.** When an action is reachable from more than one place (an `Actions` menu on several tabs, a per-row ellipsis, a detail-page menu, a bulk-selection toolbar), the team tests it from **all** of them — different entry points expose different bugs. Likewise each selection state is its own candidate: none selected (control disabled), single valid, single invalid (disabled or blocked), mixed valid+invalid (alert / "deselect to continue"), multiple valid (applied independently, or split/aggregated across them). Check the context file for a documented entry-point list before assuming one flow covers the feature. (Learned from validating Fees/Fines Waive against real case C465.)

## Case structure

```
Title: [Actor/Subject] [verb] [object] [condition]

Type:               [per area — see references/areas.md]
Priority:           [Critical | High | Medium | Low]
Release:            Umbrellaleaf
Test Group:         [Smoke | Critical Path | Extended — required]
User Journey:       No   [Yes only for journey/lifecycle cases in areas that use the flag]
Multi-Tenant:       No
Bug Created:        No
Unstable:           No
ECS Enabled:        No   [Yes only when the story explicitly tests ECS/cross-tenant behavior]
ECS Unsupported:    No   [fixed default — never ask]
Capabilities Ready: Yes
Execution Type:     Manual   [required]
Dev Team:           [from "Development Team" in the story]
Customer Name:      [All | MOBIUS/GALILEO | LOC — only if the story targets one]
Labels:             AI   [always]
References:         [ticket IDs the case verifies, comma-separated]

Preconditions:
1. User with following Capability Sets is logged in:
   - Data - [Module] [Resource] - [Action]
2. [Records with exact values; closely related facts grouped into one entry]
3. User is on [starting app/page]

Steps:
1. Action:   [verb] [object] on [location]
   Expected: [observable UI or system result]
```

Field IDs, API keys, and posting: `references/testrail-api.md`. Step shapes to copy: `references/step-patterns.md`.

## Title rules

Plain descriptive sentence, no prefixes, no labels: `[Actor/Subject] [verb] [object] [condition]`.

**Do ✓**
- `User with Edit capability set can create locked mapping profile`
- `Mapping profile Status column shows Locked after lock checkbox is enabled`
- `Delete mapping profile modal shows all required elements`
- `Holdings export fails when submitted file contains invalid UUIDs`
- `User can create a multi-select custom field` ← always include the actor
- `User can check out requested item in Central tenant` ← ECS context lives in the sentence

**Don't ✗**
- ~~`Verify mapping profile creation`~~ — vague, no subject or condition
- ~~`Create a text field custom field`~~ — missing actor
- ~~`Verify that final custom field can be removed`~~ / ~~`Test that user can create profile`~~ — no "Verify that" / "Test that" openers
- ~~`Negative: Holdings export with invalid UUIDs`~~ / ~~`Load testing - Export 500k records`~~ / ~~`ECS | Delete locked mapping profile`~~ — no prefixes, no `|`

Exception: `[ECS <Area>]` prefix only when the test is exclusively ECS-specific *and* the area label adds essential disambiguation. Use sparingly.

The title is the one place actor naming ("User can...") appears. Steps use imperative style.

## Writing rules

> Write every case as if a QA engineer who has never seen the feature will execute it. Any team member — Firebird, Spitfire, Vega, Volaris, Thunderjet — must execute it without guessing. Same detail regardless of team or area.

### Preconditions

- **Capability Sets are the source of truth** on Eureka — always include them. Many real cases list **both** the legacy Okapi permission and the Eureka capability set, often as a two-column table (`Permission name for Okapi env.` / `Capabilities/Sets for Eureka env.`). That dual listing is an accepted transition convention — preserve it if the source material or the section's existing cases use it. Generating from scratch with no precedent, a clean Eureka-only list is fine. Never use a legacy permission *instead of* a capability set.
- **Number every precondition** so steps can reference "Preconditions #N". Before finalizing, check every referenced #N resolves to a real top-level item.
- **The list must be flat.** Each user, each data record, and the starting page are separate top-level items. Never nest a record or login state under another item. Sub-bullets list Capability Sets only.
- **Grouping related facts in one entry is expected, not merely tolerated.** Real preconditions pack several related facts into one item ("Two Fiscal years exist. One has an assigned Acquisition unit"; "Fund A-FY-current: Allocated = 200.00, Encumbered = 50.00, Available = 150.00"). Split only when facts belong to genuinely different records, or when a step must reference one independently.
- **Exact values for everything a step later asserts as a number or fixed status** — amounts, statuses, counts, relationships. If a step verifies `Available = 200.00`, the precondition establishes `Available = 150.00 (Allocated 200.00 − Encumbered 50.00)`.
- **Preconditions hold only what exists before the test starts.** Any in-test UI action — selecting members, switching tabs, opening panes, applying filters — is a step with its own expected result. "User is on <page> with all 3 members selected" is wrong: the selection is a step, and the selection modal itself gets verified.
- **Default to a single, unnamed user** ("User with following Capability Sets is logged in..."). Most real cases never name an actor. Introduce named actors only when two distinct roles are active at once (proxy and sponsor; requester and processing staff; admin setting up for a restricted user) — and name them by role ("Admin user" / "Restricted user") rather than letters.
- **ECS cases:** name the tenant for every data record, the configured service points, and the user's starting affiliation.
  ```
  1. Active Ledger exists in Central tenant for the current fiscal year; Fund A (Allocated = 200.00) exists in member tenant (College), related to the Ledger above
  2. User is logged in with affiliation set to Central tenant
  3. Item with barcode 12345 exists in member tenant, status Available; pickup service point SP-Central is configured in Central tenant
  ```

### Step sequence

1. **Entry** — grouped navigation from the app entry point, each with a real verification (pane opened, exact counts like "3 members selected", button states).
2. **First-open context** — on first landing on the target pane/modal, verify the elements relevant to the scenario. A full element-by-element inventory only when that pane/modal is itself under test.
3. **Business assertions** — per-row / per-field verification. The core of the case.
4. **Interaction + re-verification** — when state changes, perform the change then re-assert the new exact state.

Size to the area's median (`references/areas.md`); overall team median is 9 steps. **A case with 3 or fewer steps is a self-review failure** unless the scenario is genuinely one trivial assertion — usually the navigation and context verification are missing, or it should merge with a sibling.

### Business-logic verification

- **Verify the business-rule outcome, not the UI mechanics of triggering it.** Compress the trigger flow (menus, modals, Submit); detail the state verification.
- **Quantitative assertions are absolute, never relative.** Write `Encumbered = 0.00`, `Available = 200.00 (restored to full allocation)` — never "increased by 50.00" or "decreased accordingly". The executor compares a number on screen to a number in the case. Preconditions carry concrete amounts too (real case C648502: "Fund A having current budget with money allocation $100 … Allowable expenditure percentage 110%").
- **But be concrete only where the number is the point.** When a threshold or direction is what's under test, the team writes it relatively — real case C380517 (insufficient funds) says *"enter any value exceeding money allocation for Fund B"* because the exact figure is irrelevant. Match the number's role.
- **Circulation-style flows use symbolic placeholders — do not invent fake concrete values.** Check in, Check out, Requests, Loans, Inventory use abstract identifiers the executor fills in: `service point S` / `S1`, `<barcode>`, `<title>`, `<material type>`, "Item with at least one open request". Real C7148 preconditions: *"User with Check In permissions … with service points S and S1 assigned. Item with at least one open request, with top request with pickup service point S."* Fabricating `12345` or `SP-A` just adds noise the executor must ignore. Use a concrete value only when that value is itself under test (a barcode with a trailing space, a specific fee amount). Don't mix invented concrete values into an otherwise symbolic case.
- **Verify every entity the action touches**, per the context file. Closing a PO must assert: encumbrance transaction status, budget values, PO status and "Reason for closure", POL receipt/payment statuses.
- **Business-critical tables get column-level verification.** Never "transaction is displayed" — assert `Type = "Encumbrance"`, `Source = <PO number>`, `Amount = $50.00`, `Status = "Released"`. A concise summary is acceptable only for navigation/lookup tables where it still proves the behavior.
- **Reference preconditions by number**: "the unlocked mapping profile from Preconditions #3".
- **Toasts verbatim, with placeholders**: `"<amount> was successfully allocated to the budget <Fund-FY>"`. Never "success toast is displayed". If the exact text is unknown and absent from the context file and examples, write the most likely text and flag it: `(verify exact wording on first execution)`.
- Cover scenario types in order: happy path → business-rule verification → edge cases → negative → capability boundaries.

### Granularity and format

- **Group navigation/setup actions into one step** when they share a single verification point. **Never group assertion steps** — one assertion target per step.
- **Multi-part actions and multi-fact results get a lead-in line + bullets — never a run-on sentence.** When a step fills several form fields, sets several options, or chains sub-actions, write a lead-in clause ending in a colon, then one bullet per field/option/sub-action. Same for an expected result confirming several field values.

  ```
  Action:
  Navigate to Settings > Data Import > Match profiles, click "New" to create a match profile:
  - Existing record type = MARC Authority
  - Existing record field = "Authority: 001"
  - Incoming record field = "MARC Authority: 001"
  - Match criteria = "Exactly matches" (the only available option)
  - Click "Save as profile & Close"
  Expected:
  Match profile is saved and appears in the Match profiles list with:
  - Name = <profile name>
  - Existing record field = "Authority: 001"
  - Incoming record field = "MARC Authority: 001"
  ```

- **HARD RULE — 2+ discrete `field = value` pairs (or any field whose value is a long phrase/sentence) is ALWAYS a lead-in line + bullets, never a semicolon run-on.** This overrides "prose first". A record/detail-view check that reads a list of fields is an *inventory*, not one observation.

  ❌ `Expected: Detail view shows Name = "X"; Description = "…"; Action = "Delete"; FOLIO record type = "Authority"`

  ✅
  ```
  Expected: Action profile detail view shows:
  - Name = "X"
  - Description = "…"
  - Action = "Delete"
  - FOLIO record type = "Authority"
  ```

  A chain of `A = …; B = …; C = …` is the "сплошняк" the tester keeps rejecting. **If you typed a second `; <Field> =`, convert the whole thing to bullets.**
- **Prose first only for a single coherent observation.** *"Modal appears with message Route <title> (<material type>) (Barcode: <barcode>) to <Service point S>. Print slip is checked by default."* is one modal message plus its default state — prose. Prose covers: a toast, a modal message, a status change, a single field value, or 2–3 tightly-related facts that read as one natural sentence. Don't explode one coherent observation into bullets just because it mentions three nouns; don't cram a real field list into one line either.
- **Imperative style, no actor prefix** — "Click ...", "Navigate to ...", "Select ...". Not "User clicks". The actor is defined once in Preconditions.
- **Don't attach status adjectives to records that have no such status.** Write "A MARC Authority record exists with heading X", not "An **Active** MARC Authority record" — MARC Authority / Instance / Holdings / Item have no "Active" state. Reserve status words for records where the status is a real named field value (an **Active** Budget/Fund/Ledger, an **Active** Organization, an order in **Pending/Open/Closed**). When unsure, check the context file rather than adding a default adjective.
- **Expected results are terse observable states — never padded with rationale, ticket-scope reasoning, or hedging.** State what the executor should see (`"Edit" and "Delete" are absent from the Actions menu`), not why. Do NOT write meta-commentary like "these restrictions are explicitly in scope for MODDICONV-436 (not gated by UIDATIMP-1768)…". That reasoning belongs in the story or PR discussion. If an expected result reads like an explanation or an argument, cut it to the single checkable state.
- **`NOTE:` is one short sentence, for a real caveat that changes how the executor judges pass/fail** (`NOTE: known display bug MODX-123 may show "Updated" — verify at the record`). Never a place to explain ticket scope or hedge about unshipped work. If you're writing two or more sentences of NOTE, delete it.
- For list/table verifications, list the exact columns expected.
- **ECS cases:** after an action in one tenant, add a verification step in the other tenant when the story requires cross-tenant consistency. Name the tenant in both action and expected result. Tenant switching is its own step with its own expected result.

### Don'ts

- Vague expected results: "works correctly", "is successful", "confirming X is displayed"
- Relative value assertions: "increased by", "reduced accordingly"
- Padding expected results or NOTE lines with rationale or ticket-scope reasoning
- Skipping expected results for intermediate steps
- Bundling multiple user roles into one case — one role per case
- Omitting capability boundary scenarios when the story involves roles or restrictions
- Legacy permission names in place of Capability Sets
- Combining UI element verification with action steps; grouping assertion steps
- `→` inside a step to chain actions across verification points; `|` as a title separator
- Inventing UI labels, statuses, or toast texts that contradict the context file or examples

## Self-review gate

Run on every case before showing it.

- [ ] Has a navigation/entry step, business assertion step(s), and — if state changes — a re-verification step? A 2–3 step case fails by default unless genuinely trivial.
- [ ] Preconditions are data/state only, with no in-test UI actions hidden in them?
- [ ] At least one step asserts the business-rule outcome with exact values? If every expected result could pass on a broken feature, rewrite.
- [ ] All entities the context file says the action touches are verified?
- [ ] Quantitative values absolute; toasts verbatim?
- [ ] No invented concrete values in a circulation/inventory case that should use `S` / `<barcode>`?
- [ ] Business-critical tables verified column by column?
- [ ] Preconditions numbered and flat, with starting values for everything asserted later, and every "Preconditions #N" resolving?
- [ ] `User Journey` agrees with the case shape, and with whether the area uses the flag at all?
- [ ] Multiple entry points and selection states enumerated as separate scenarios rather than tested from one path?
- [ ] Expected results prose unless a true inventory justifies bullets — and no semicolon-joined field lists?
- [ ] Every expected result a terse observable state, with no rationale or multi-sentence NOTE?
- [ ] Multi-field actions and multi-fact results broken into lead-in + bullets?
- [ ] Steps imperative, no actor prefix, single unnamed user unless two roles are genuinely required?
- [ ] `refs` = the item(s) this case actually verifies, no invented story key?
- [ ] Type, size, and journey flag match the area's row in `references/areas.md`?
- [ ] Verification density comparable to `references/examples.md`?
