---
name: build-app-context
description: Use when you need to build or refresh a context file for a FOLIO application area before writing test cases. This skill gathers live data from TestRail (existing cases), GitHub (README, docs, feature files, capability/permission definitions), and Jira (closed stories and bugs for the component), then distills everything into a structured context .md file and saves it to references/context/. Run this skill once per application area, then use write-testrail-cases normally.
license: Apache-2.0
metadata:
  author: folio-org
  version: 1.0.0
---

# Build App Context

Builds or refreshes `write-testrail-cases/references/context/<area>.md` from live TestRail, GitHub, and Jira data. That file is the primary domain knowledge source for `write-testrail-cases`.

**Runtime:** ~2–5 min per area. **Output:** the context file + an updated area row in `write-testrail-cases/references/areas.md`.

Read `references/operations.md` for error handling, known Jira access limits, refresh mode, and the recency weight table. Read `references/context-file-template.md` before Phase 4.

## Credentials

Try in order, stop at first hit. Never ask the user — try all three silently. If all three fail, report which keys are missing and stop.

1. `.env` in the repository root — parse KEY=VALUE pairs
2. `~/.folio-credentials` — legacy local file
3. Environment variables — for CI/Codespaces

```
TESTRAIL_URL, TESTRAIL_USER, TESTRAIL_API_KEY
JIRA_BASE_URL, JIRA_USER_EMAIL, JIRA_API_TOKEN
GITHUB_TOKEN (optional)
```

## Phase 0 — Setup

### 0.1 Auto-resolve module metadata first

Before asking anything, silently fetch the FOLIO responsibility matrix:

```
GET https://folio-org.atlassian.net/wiki/spaces/REL/pages/5210256/
    FOLIO+Module+JIRA+project-Team-PO-Dev+Lead+responsibility+matrix
```

Build a lookup by area/module name and extract for the named area:

| Field | Matrix column | Use for |
|---|---|---|
| `jira_prefix` | JIRA project | Jira JQL filter, area detection row |
| `team` | Team | `custom_dev_team` default |
| `ui_repo` | Module starting `ui-` | GitHub, Phase 2 |
| `be_repo` | Module starting `mod-` | GitHub, Phase 2 |
| `tests_repo` | Tests repository | GitHub, Phase 2 |
| `ecs_scope` | ECS (Central/Member/All)/NonECS/Both | `ECS Enabled` default |
| `eureka_app` | Application Trillium | Context file header |

An area typically has 2–3 rows (one per module) — collect all matches. If the area name maps to multiple distinct teams (e.g. ERM is K-Int), note it in the questions. If the page needs auth, skip this step silently and use the fallback questions.

### 0.2 Confirm with the user

**When metadata resolved** — skip the repo and Jira-component questions, show pre-filled values:

```
I found the following metadata for "<area>" in the FOLIO module matrix:

  JIRA prefix:    <prefix>
  Team:           <team>
  GitHub repos:   <ui_repo>, <be_repo>
  Tests repo:     <tests_repo> (if present)
  ECS scope:      <Central/Member/All | NonECS | Both>
  Eureka app:     <app name>

I still need from you:

1. TestRail section ID(s) for existing manual cases
   (Find it in the URL: ?/cases/view/14&section_id=XXXXX)

2. Jira component name (if different from "<prefix>") — or leave blank
   to use the JIRA project prefix directly.

3. Releases to focus on [default: Umbrellaleaf, Trillium, Sunflower]:

Confirm or correct the pre-filled values, then I'll start building.
```

**Fallback** — ask all five in one message:

```
I'll build a context file for a FOLIO app area by pulling data from TestRail,
GitHub, and Jira. A few questions:

1. **App area name** — what should the context file be called?
   (e.g. "orders", "data-export", "bulk-edit")

2. **TestRail section ID(s)** — the section(s) containing existing manual
   test cases for this area. One ID or a comma-separated list.
   (Find it in the TestRail URL: ?/cases/view/14&section_id=XXXXX)

3. **GitHub repositories** — which repos cover this area?
   Typically ui-<name> and mod-<name>. (Leave blank to skip GitHub.)

4. **Jira component or epic** — component name or epic key.
   Examples: component = "Orders", epic = "UXPROD-1234"
   (Leave blank to skip Jira.)

5. **Releases to focus on** [default: Umbrellaleaf, Trillium, Sunflower]
```

Echo one line of confirmation, then run Phases 1–4 without further interruption:
> Building context for **<area>** from: TestRail section(s) <IDs> · GitHub: <repos> · Jira: <component/epic>

## Phase 1 — Gather: TestRail

```
GET {TESTRAIL_URL}/index.php?/api/v2/get_cases/{project_id}&section_id={id}&limit=250&offset=0
Authorization: Basic base64(TESTRAIL_USER:TESTRAIL_API_KEY)
```

Paginate (offset += 250) until `_links.next` is null, across every provided section ID. Extract per case: `id`, `title`, `custom_preconds`, `custom_steps_separated`, `custom_test_group`, `custom_release`, `custom_ecs_enabled`, `updated_on`, `refs`.

Weight every case by `custom_release` (table in `references/operations.md`) and use weighted frequency for all thresholds below. Process steps + preconditions as plain text after stripping HTML.

| Signal | Match | Keep |
|---|---|---|
| **A. Toast messages** | `Toast message "..."`, `"..." toast message appears`, `System notifies that "..."`, `success.*"..."`, `"...successfully..."` | weighted freq ≥ 3, or top 30 if fewer |
| **B. Modal/dialog titles** | `"..." modal`, `"..." dialog`, `"..." popup` | weighted freq ≥ 2 |
| **C. Pane names** | `"..." pane` | weighted freq ≥ 3 |
| **D. Accordion names** | `"..." accordion` | weighted freq ≥ 3 |
| **E. Button labels** | `"..." button`, `click "..."`, `select "..."` | weighted freq ≥ 5 **and** ≤ 40 chars (drops sentence-length matches) |
| **F. Status values** | `status.*"..."`, `"..." status`, `= "..."` next to a known status field (Workflow, Payment, Receipt, Request status) | dedupe; keep per-field groupings |
| **G. Capability Sets** | preconditions lines matching `Data - ...`, `Procedural - ...`, `Settings - ...`, `Module - ...` | all unique, exact dedupe |
| **H. Verification patterns** | step sequences where the action names a field and the expected result contains `=` or `displays` with a value | 3–5 most representative, verbatim |
| **I. Navigation paths** | `navigate to ... > ... > ...`, `Settings > ... > ...` | 10 most common |
| **J. ECS signals** | cases with `custom_ecs_enabled = true` | tenant names in preconditions (Central, member-1, member-2), cross-tenant step patterns, affiliation switching steps |

For toast texts, canonicalize placeholders (`<name>`, `<number>`, `<PO number>`) by looking at what fills the slot in the most common variant.

**Business rules from titles.** Group titles: "cannot" / "not able" / "without permission" → negative and capability-boundary rules; "successfully" / "can create/edit/delete" → positive rules; status words → lifecycle rules. Extract noun phrases repeating across ≥ 3 titles in the same category and format as `"<subject> <verb> <object> when <condition>"`.

## Phase 2 — Gather: GitHub

```
GET https://api.github.com/repos/{owner}/{repo}/git/trees/HEAD?recursive=1
Authorization: Bearer {GITHUB_TOKEN}   (omit header if no token)
```

Fetch at most **20 files** total across all repos, in this priority order. Skip files > 200 KB.

| Priority | Pattern | Extract |
|---|---|---|
| High | `README.md`, `doc/*.md`, `docs/*.md`, `**/CHANGELOG.md` | Feature descriptions, business rules, known limitations, version-specific behavior ("Since Ramsons", "Deprecated in Sunflower") |
| High | `**/*.feature`, `**/test/**/*.feature` | `Scenario:` / `Scenario Outline:` titles; Given/When/Then steps describing UI interactions or business assertions; UI text candidates from `I should see "..."` steps |
| Medium | `**/permissions.json`, `**/moduleDescriptor.json`, `**/descriptors/*.json` | All `permissionName` / `id` values under `provides` / `permissionSets`; explicit Eureka mappings via `replaces` / `eureka` keys |
| Medium | `src/main/resources/swagger.api/*.yaml`, `**/*openapi*.yaml` | API endpoints and schemas (record types, field names) |
| Low | `translations/ui-*/en_US.json`, `translations/ui-*/en.json` | **Values** (not keys) for keys containing: modal, button, label, toast, message, title, header, pane, accordion, action, success, error, confirm |

Translation values are the exact rendered UI strings — highest reliability of any source. Where a permission has no confirmed Eureka mapping, flag it: `(verify Capability Set name — legacy permission: <name>)`.

**Reconcile against Phase 1.** Both sources → **[confirmed]**. GitHub only → **[from source, verify in env]**. TestRail only, weighted freq ≥ 5 → **[from cases]**.

## Phase 3 — Gather: Jira

Check `references/operations.md` for projects known to return nothing before querying.

```
POST {JIRA_BASE_URL}/rest/api/3/issue/search
Authorization: Basic base64(JIRA_USER_EMAIL:JIRA_API_TOKEN)
Content-Type: application/json

{
  "jql": "project = FOLIO AND component = \"<component>\" AND issuetype in (Story, \"New Feature\") AND status = Done ORDER BY updated DESC",
  "maxResults": 100,
  "fields": ["summary", "description", "customfield_10014", "fixVersions", "labels", "status", "issuetype"]
}
```

For an epic key instead of a component: `"jql": "\"Epic Link\" = <epic_key> AND issuetype in (Story) AND status = Done ORDER BY updated DESC"`.

Paginate with `startAt` up to 500 stories; stop earlier once results go older than 2 years. Repeat with `issuetype = Bug` and fields `summary, description, fixVersions, labels`; keep the latest 100 bugs.

Extract:
- **Story descriptions** → Acceptance Criteria sections (`h3. Acceptance Criteria`, `*Acceptance Criteria*`, `**Scenarios:**`). Each criterion is a candidate business rule.
- **Story summaries** → group by verb (View, Create, Edit, Delete, Share, Export, Import, Open, Close, Cancel, Approve, Reorder) into `"User can <verb> <object> [when <condition>]"`.
- **Bug summaries** → what should NOT happen. Format as `"<object> must not <behaviour> when <condition>"` or `"<action> should fail when <condition>"`.
- **Fix versions** → date-stamp each rule with its FOLIO release.

## Phase 4 — Distill

Assemble the signals using `references/context-file-template.md`. Fill every section; mark gaps explicitly rather than omitting them.

## Phase 5 — Save and report

Write to `write-testrail-cases/references/context/<area>.md`.

> **The only valid output path is inside the `write-testrail-cases` skill directory.** From the repo root that is `skills/write-testrail-cases/references/context/<area>.md`. Never write to a `references/context/` at the repo root.

If the file exists, overwrite it and note the previous version date in the header.

Then update the area detection table in `write-testrail-cases/references/areas.md` — update the Jira prefixes/keywords for an existing area, or add a row:

```
| <Area Name> | `references/context/<area>.md` | <Jira prefixes>; "<keywords from story summaries>" |
```

Report:

```
✅ Context file built: references/context/<area>.md

Summary:
  TestRail: <N> cases processed (<N> recent, <N> older)
  GitHub:   <N> files read from <repos>
  Jira:     <N> stories + <N> bugs processed

Extracted:
  Toast messages:         <N> (confirmed: <N>, from cases only: <N>)
  Modal titles:           <N>
  Capability Sets:        <N> unique
  Business rules:         <N>
  Verification patterns:  <N>

⚠️  Items needing env verification: <N> (listed in "Known Gaps" section)

The context file is ready. You can now run write-testrail-cases for <area>.
```
