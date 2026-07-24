# TestRail API and Metadata Fields

Read this file at workflow step 8, before filling metadata or posting.

## Credentials

Try in order, stop at first hit. Never ask the user for credentials. If all three fail, report which keys are missing and stop.

1. `.env` in the repository root — parse KEY=VALUE pairs
2. `~/.folio-credentials` — legacy local file
3. Environment variables — for CI/Codespaces

```
TESTRAIL_URL, TESTRAIL_USER, TESTRAIL_API_KEY
JIRA_BASE_URL, JIRA_USER_EMAIL, JIRA_API_TOKEN
GITHUB_TOKEN (optional)
```

## Metadata fields reference

**Resolve IDs at runtime.** Dropdown fields (Type, Priority, Release, Test Group, Execution Type, Dev Team, Customer Name, Automation Scope/Team/For) take a numeric option ID, not the label string — sending a label returns `400 Field :<name> is not a valid natural number`. Look up current IDs with `get_priorities`, `get_case_types`, `get_case_fields`. Checkbox fields take a JSON boolean. The IDs below are a snapshot from `foliotest.testrail.io` and may change.

**Required fields:** `custom_test_group` and `custom_automation_type` ("Execution Type"). A case will not post without them.

**Standing defaults — do not ask the user:** `ECS Unsupported = false` on every case; every case gets the "AI" label. Set `ECS Enabled = true` only when the story explicitly tests cross-tenant / ECS behavior.

| Field | TestRail API key | Type | Default | Options / Notes (current IDs) |
|---|---|---|---|---|
| Title | `title` | string | — | Plain descriptive sentence; no prefixes ("Verify", "Negative:", "ECS \|", "Load testing -") |
| Type | `type_id` | dropdown (id) | **Per area — see `areas.md`** | Resolve via `get_case_types`. Current: **Functional=6**, Other=7 (also Acceptance=1, Regression=9, Smoke & Sanity=11). Acquisitions (Orders/Invoices/Finance/Organizations/Mediated) skew Functional; most circulation/ERM/export areas skew Other; several are ~50/50. |
| Priority | `priority_id` | dropdown (id) | — | Resolve via `get_priorities`. Current: Low=1, Medium=2, High=3, Critical=4. **Calibration:** core daily-workflow path whose failure breaks the feature (main check-in/check-out fulfillment, opening/paying an order, posting an invoice) → **Critical**. Important-but-not-blocking (edit/duplicate, capability boundaries, secondary flows) → **High**. Search/filter, UI-element checks, edge/negative → **Medium**. Rarely **Low**. Don't default everything to High. |
| Release | `custom_release` | dropdown (id) | latest release | Current latest: **R2 2026 Umbrellaleaf=21**, R1 2026 Trillium=20, R1 2025 Sunflower=19 |
| Test Group | `custom_test_group` | dropdown (id) | — | **Required.** Smoke=1, Critical Path=2, Extended=3, Obsolete=4, Draft=5, Backend=6, Edge Cases=7 |
| Execution Type | `custom_automation_type` | dropdown (id) | `Manual` (2) | **Required.** Automated=1, Manual=2, Karate=3, Unit=4, Backend Component=5 |
| User Journey | `custom_user_journey` | checkbox (bool) | `false` | `true` for a journey/lifecycle case — one case walking a multi-step flow through several checkpoints or outcomes. Must agree with the case shape chosen in Scenario Analysis; a journey case with `false` is a self-review failure. |
| Multi-Tenant | `custom_multi_tenant` | checkbox (bool) | `false` | `true` / `false` |
| Bug Created | `custom_bug_created` | checkbox (bool) | `false` | `true` / `false` |
| Unstable | `custom_unstable` | checkbox (bool) | `false` | `true` / `false` |
| ECS Enabled | `custom_ecs_enabled` | checkbox (bool) | `false` | `true` only when the story explicitly tests ECS/cross-tenant behavior |
| ECS Unsupported | `custom_ecs_unsupported` | checkbox (bool) | `false` | **Always `false`. Never ask the user.** |
| Capabilities Ready | `custom_capabilities_ready` | checkbox (bool) | `true` | `true` / `false` |
| Dev Team | `custom_dev_team` | dropdown (id) | — | From "Development Team" in the story. Concorde=1, Firebird=3, Folijet=4, Spitfire=6, Thor=7, **Thunderjet=8**, Vega=9, Volaris=13, Citation=18, Eureka=21 (full list via `get_case_fields`) |
| Customer Name | `custom_customer_name` | dropdown (id) | — | Optional. All=1, MOBIUS/GALILEO=2, LOC=3 — only when the story targets a specific customer |
| Automation Scope | `custom_automation_scope` | dropdown (id) | — | Optional; omit if N/A. Review by MQA=1, Review by PO=2, Automation Ready=3, Not an automation candidate=4, Mriya scope=5, Karate Approved=6, Karate Not Applicable=7, FE scope=8 (no "None" option) |
| Automated For | `custom_case_automated_in` | dropdown (id) | — | Optional; omit if N/A. Old releases=1, R2 2024 Ramsons=2, R1 2025 Sunflower=3, R2 2025 Trillium=4 |
| Automation Team | `custom_automation_team` | dropdown (id) | — | Optional; omit if N/A. TaaS=1, Mriya=2, FE team=3 (no "None" option) |
| References | `refs` | string | — | The work item(s) this case verifies, comma-separated. Often a Bug or Tech-Debt ticket rather than a story, and may span several projects (e.g. `UIF-525, MODFIN-391`). Do not invent a story key when none exists; do not auto-add unrelated links pulled in via enrichment. |
| Labels | `labels` | array | `["AI"]` | **Always add "AI" to every case.** Pass label ID(s) — resolve via `get_labels/{project_id}` ("AI" = id **67** on `foliotest.testrail.io`) — or the title string `"AI"`. |

### Test Group rules

- **Smoke** — core feature works at all; the single most critical happy path. Typically 1 case per feature area, rarely 2.
- **Critical Path** — essential daily functionality: create, edit, delete, key capability boundary checks, export/import flows, status transitions, business-rule verifications.
- **Extended** — search and filtering, UI element verification, page title checks, edge cases, negative cases, load/performance, complex multi-step workflows.

Most cases are Critical Path or Extended.

## Jira MCP integration

When the user gives a Jira ticket ID instead of pasting a story:

1. Fetch the issue by ID. Extract Summary, Description, Acceptance Criteria, Development Team (→ `custom_dev_team`), Fix Version.
2. If Jira MCP is not available, ask the user to paste the story manually.

## Optional: read existing cases from a section

If the user provides a Section ID for the area, fetch up to 20 existing cases as an *additional* style reference (the primary references stay the context file and `examples.md`):

```
GET {TESTRAIL_URL}/index.php?/api/v2/get_cases/14&section_id={section_id}&limit=20
Authorization: Basic base64(TESTRAIL_USER:TESTRAIL_API_KEY)
```

Extract from `custom_preconds` and `custom_steps_separated`: exact UI labels, toast texts, precondition depth, step granularity. Match the depth and format you find. If no Section ID is provided, do not ask for one until the posting step.

## Posting preview

Ask: _"Please provide the TestRail Section (folder) ID where the cases should be posted."_ Then show:

```
Ready to post 5 test cases to Section ID: XXXXX

1. User with Edit capability set can create unlocked mapping profile — Other / High / Critical Path
2. User with Edit capability set can edit unlocked mapping profile — Other / High / Critical Path
3. User with Edit capability set cannot delete mapping profile referenced in job profile — Other / High / Critical Path
4. Mapping profile "Status" column shows "Locked" after lock checkbox is enabled — Other / Medium / Extended
5. User with View-only capability set cannot see "New" or "Actions" buttons — Other / High / Critical Path

Confirm? (yes / no / edit)
```

Wait for explicit confirmation, then post.

## Add case request

```
POST /index.php?/api/v2/add_case/{section_id}
Content-Type: application/json
Authorization: Basic <base64(email:api_key)>
```

```json
{
  "title": "User with Edit capability set can delete unlocked mapping profile not referenced in job profile",
  "type_id": 7,
  "priority_id": 3,
  "refs": "MODFIN-273",
  "custom_preconds": "1. User with following Capability Sets is logged in:\n  - Data - UI-Data-Export Settings - Edit\n2. Unlocked mapping profile NOT referenced in any job profile exists\n3. User is on Settings > Data export > Field mapping profiles",
  "custom_steps_separated": [
    {
      "content": "Click on the row with the unlocked mapping profile from Preconditions #2",
      "expected": "Mapping profile view form is displayed; \"Lock profile\" checkbox is disabled, unchecked; \"Actions\" menu is enabled"
    },
    {
      "content": "Click \"Actions\" menu",
      "expected": "Menu expands and displays the following options: Edit, Duplicate, Delete"
    },
    {
      "content": "Click \"Delete\" option",
      "expected": "\"Delete mapping profile\" modal opens with: \"The mapping profile <name> will be deleted.\" text, \"Cancel\" button (enabled), \"Delete\" button (enabled, focused)"
    },
    {
      "content": "Click \"Cancel\" button",
      "expected": "\"Delete mapping profile\" modal closes; mapping profile view form is displayed"
    },
    {
      "content": "Click \"Actions\" menu > \"Delete\" > \"Delete\" in modal",
      "expected": "Modal closes; toast message \"Mapping profile <name> has been successfully deleted\" is displayed; \"Field mapping profiles\" pane shows list without deleted profile"
    }
  ],
  "custom_release": 21,
  "custom_test_group": 2,
  "custom_automation_type": 2,
  "custom_dev_team": 4,
  "custom_customer_name": 1,
  "custom_multi_tenant": false,
  "custom_user_journey": false,
  "custom_bug_created": false,
  "custom_unstable": false,
  "custom_ecs_enabled": false,
  "custom_ecs_unsupported": false,
  "custom_capabilities_ready": true,
  "labels": [67]
}
```

When a step uses bullets, put each bullet on its own line in the `content` / `expected` string (`\n- ...`) so it renders as a list, not a wall of text.

## Success output

```
✅ Posted 5 test cases to Section 1042:
  C123456 — User with Edit capability set can create unlocked mapping profile
  ...
```

## Error handling

| HTTP status | Action |
|---|---|
| `401` | Credentials error — check `.env` or `~/.folio-credentials` |
| `400` | Show the full error body and identify which field caused it |
| `404` | Confirm the Section ID exists and is accessible |
| Any failure | Do not skip silently — report the failed case title and full error message |

## Fallback output (no API access)

If posting via API fails entirely, output each case in this format:

```
---
Title:          [title]
Type:           Other
Priority:       [High | Medium | ...]
Release:        Umbrellaleaf
Test Group:     [Smoke | Critical Path | Extended]
Execution Type: Manual
Dev Team:       [from story]
References:     [jira_ids]

Preconditions:
1. User with following Capability Sets is logged in:
   - [Capability Set 1]
2. [Required data with exact values]
3. User is on [app/page]

Steps:
1. Action:   [step]
   Expected: [result]
---
```

Capability Sets format: `Data / Procedural / Settings / Module - <Module> <Resource> - <Action>`.
