# Operations: Errors, Access Limits, Refresh Mode

## Error handling

| Situation | Action |
|---|---|
| TestRail section returns 0 cases | Warn: "Section <ID> returned no cases — check the section ID and project access." Still proceed with GitHub and Jira. |
| GitHub rate limit hit (403) | No token: ask the user to add `GITHUB_TOKEN`. Token present: wait 60s and retry once. |
| GitHub repo not found (404) | Skip that repo; note in report: "Repo <name> not found or not accessible." |
| Jira component returns 0 results | Retry without the `component` filter, project key only. If still 0, warn and skip. |
| Jira v2 returns empty results | Switch to `/rest/api/3/` — v2 is deprecated in this environment. |
| Jira API returns 401 | Report credentials error for `JIRA_API_TOKEN`. |
| File > 200 KB on GitHub | Skip and note the filename in the report. |
| Translation file missing | Note "No translation file found — UI texts rely on TestRail cases and feature files only." |
| Context file > 400 lines | Trim Verification Patterns to the 3 most representative; trim Toast and Modal tables to top 20 each by weighted frequency. Context files must stay readable — quality over quantity. |

## Known Jira access limitations

These projects return 0 results with the current API token. Run Phase 1 (TestRail) + Phase 2 (GitHub) only, log "Jira inaccessible" in the context file header, and do not retry Phase 3.

| Area | Jira projects |
|---|---|
| OAI-PMH | MODOAIPMH, UIOAIPMH, EDGOAIPMH |
| MARC Authority | UIMARCAUTH, MODELINKS |
| Licenses | ERM |
| Agreements | ERM |
| eHoldings | ERM |

## Refresh mode

When the user says "refresh" or "update context for <area>" and the file exists:

1. Pull only cases updated since the file's generation date (TestRail `updated_after` filter).
2. Pull only Jira issues updated in the last 90 days.
3. Fetch GitHub only if the user explicitly says "also refresh GitHub".
4. Merge: new toasts/modals are added; old ones with no recent case support move to "Known Gaps".
5. Update the generation date in the header.
6. Report: "Refreshed <area>.md: +<N> new signals, <N> items moved to Known Gaps."

## Recency weights

Assign by `custom_release`. Use weights when voting on whether a string is a real current UI element — high-weight occurrences beat low-weight ones.

| Release label | Weight |
|---|---|
| Umbrellaleaf (R2 2026) | 1.0 |
| Trillium (R1 2026) | 0.9 |
| Sunflower (R1 2025) | 0.8 |
| Ramsons (R2 2024) | 0.6 |
| Quesnelia (R1 2024) | 0.5 |
| Older / blank | 0.3 |
