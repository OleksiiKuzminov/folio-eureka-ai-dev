# Live Investigation with Chrome DevTools MCP

Optional. Use **only** when the failure details the user gave are insufficient or unclear. If they described the failure, skip this entirely and write the report.

**Requires:** Chrome DevTools MCP server configured; test credentials available from the case's `custom_preconds` or notes; the test flow understood from the retrieved case.

## Sequence

1. `mcp_chrome-devtoo_new_page()`
2. `mcp_chrome-devtoo_navigate_page({ url: "<baseUrl>" })`
3. **Log in.** Credentials come from `custom_preconds` or TestRail project settings — look for "Login as user with permissions..." in the steps. Drive the form with `mcp_chrome-devtoo_fill` and `mcp_chrome-devtoo_click`.
4. **Walk the test steps** from `custom_steps_separated`, using `mcp_chrome-devtoo_take_snapshot` to capture page state and `click` / `fill` / `type_text` to interact. Use `mcp_chrome-devtoo_wait_for` where elements load asynchronously.
5. **Capture the failure:**
   - `mcp_chrome-devtoo_take_screenshot` at the failure point
   - `mcp_chrome-devtoo_list_console_messages` for console errors
   - `mcp_chrome-devtoo_list_network_requests` for failed calls and status codes
   - `mcp_chrome-devtoo_evaluate_script` for JavaScript state

## What to record

Exact error message text, which UI element misbehaved, network failures with status codes, and console warnings/errors. These become the Actual Result.

## Outcomes

| Result | Effect on the report |
|---|---|
| Failure reproduced | Document the exact observed behavior as Actual Result |
| Cannot reproduce | Note it as an intermittent issue |
| Investigation blocked (no credentials, page won't load) | State the blocker and fall back to test case information |

When blocked, tell the user what stopped you and offer: provide failure details directly, attach screenshots/logs to the Jira ticket afterwards, or reproduce manually to gather evidence.
