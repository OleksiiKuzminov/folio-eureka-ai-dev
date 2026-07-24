---
name: testrail-bug-report
description: 'Generate Jira-ready bug reports from failed FOLIO test cases. Takes TestRail test case ID, investigates failure context, and creates structured bug report with Title, Description, Environment, Steps to Reproduce, Expected Result, and Actual Result. Optionally uses Chrome DevTools MCP to investigate live application behavior.'
argument-hint: 'Test case ID (e.g., C389500) and optional failure details'
---

# TestRail Bug Report Generator

Turns a failed TestRail case into a structured bug report ready to paste into Jira.

**Requires:** TestRail MCP server; a case ID (e.g. C389500); access to `cypress.config.js`.

## Procedure

### 1. Retrieve the case

```javascript
mcp_testrail_getCase({ caseId: 389500 })  // numeric ID, no 'C' prefix
```

Extract `title`, `custom_preconds`, `custom_steps_separated` (array of `content` / `expected`), `custom_automation_status`, `section_id`, and any referenced attachments (MARC records, screenshots).

If the case is not found, report which of these to check — ID format (numeric, no `C`), MCP connection, case access — and offer to build the report from failure details the user provides manually. If the case is not automated, ask whether a manual test still needs a bug report.

### 2. Determine the environment

Read `baseUrl` from `cypress.config.js`; check for environment-specific configs (e.g. `cypress.config.BF.R.Eureka.js`). Default: `https://folio-etesting-cypress-diku.ci.folio.org`.

A user-specified environment always wins. If several configs exist and the user didn't say, ask which one was used. If none is found, state the default you're assuming and ask the user to confirm. Include the tenant when applicable: `https://folio-etesting-cypress-diku.ci.folio.org (diku tenant)`.

### 3. Analyze the failure

Gather which step failed, the error messages, and the observed behavior — from the user's description first, then from the case's expected-vs-actual and preconditions.

- **User gave detailed failure info** → go to step 5.
- **Details vague or missing** → offer live investigation: _"To create a comprehensive bug report I need to know what step failed or what unexpected behavior occurred, and any error messages displayed. Want me to investigate the application live using Chrome DevTools?"_ If they accept, follow `references/devtools-investigation.md`.

### 4. Construct the report

**Title** — `[Module/Feature] Brief description of defect`, under 100 characters.

Name the module from the case context or user input (`[MARC Bibliographic]`, `[Inventory]`, `[Circulation]`); if it isn't clear, ask rather than guess. Describe **what is broken**, not the test case name, and keep the case number out of the title — it belongs in Description. Use module names matching FOLIO's project structure.

- ✓ `[MARC Bibliographic] Unable to save new field when scope is local`
- ✓ `[Inventory] Holdings record not displaying after creation`
- ✓ `[Circulation] Check-in fails for items with pending requests`

**Steps to Reproduce** — preconditions from `custom_preconds`, then numbered steps from `custom_steps_separated[].content`. Strip HTML tags and convert markdown left over from TestRail. Stop at the failing step when it is known. Write for someone unfamiliar with the test: include navigation ("from Settings > Inventory"), exact field names and button labels, and any timing considerations ("wait for record to load").

**Expected Result** — from the failing step's `expected` field, combining relevant steps where the failure spans several.

**Actual Result** — the observed behavior, verbatim error message text, and any evidence reference. Sources in priority order: the user's description, DevTools findings, console/network errors, and the location of Cypress screenshots or logs if the user mentions it.

**Never include credentials.** Generalize to "Login with valid user credentials" or "user created in preconditions"; redact tenant-specific data. No passwords, no API keys.

### 5. Deliver

Present the report in a code block for copy-paste, then offer to adjust any section or investigate further.

```
**Copy to Jira:**

**Title:**
[MARC Bibliographic] Unable to save local field with scope "local"

**Description:**
System fails to persist MARC field configuration when scope is set to "local".
Related test case: C389500

**Environment:**
URL: https://folio-etesting-cypress-diku.ci.folio.org
Browser: Chrome
Date Tested: <current date>
Test Case: C389500

**Steps to Reproduce:**

Preconditions:
- User has permission X
- Data Y exists

Steps:
1. Navigate to module Z
2. Click on button A
3. Enter value B in field C
4. Submit the form

**Expected Result:**
The system should save the record and display success message.

**Actual Result:**
The system displays error "Invalid field configuration" and does not save the record.
Console shows error: "Cannot read property 'id' of undefined"
```

Optionally append links to the test execution results and the TestRail case, attachment references, and any workaround discovered.

## Self-review gate

- [ ] Title names the module and describes the defect, under 100 chars, no case number
- [ ] Environment includes URL and browser
- [ ] Steps are reproducible by someone who has never run this test
- [ ] Expected result is specific
- [ ] Actual result carries the exact error message and any evidence
- [ ] Test case ID referenced in Description
- [ ] No credentials or sensitive data
- [ ] No HTML artifacts left from TestRail content

## Related skills

- **testrail-marc-attachments** — when the case references MARC file attachments; mention them in preconditions if relevant to the failure.
- **test-review-agent** — when it's unclear whether the failure is an application bug or a test defect, suggest reviewing the test implementation first.
