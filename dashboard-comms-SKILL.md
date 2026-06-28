---
name: dashboard-comms
description: Build a self-contained HTML dashboard of defects across all ACTIVE sprints in the Transformation Project (SCRUM) Jira — broken down by severity (from Priority) and status, with drill-down/filter by sprint and full summary+description per defect — plus a "changes since last refresh" panel. When a refresh detects changes, draft a status-update communication tailored to Executive Leadership, the Engineering Team, and the External Customer (each with an avatar and a copy button) embedded in the same HTML, prompting the user for non-Jira inputs and for review edits. Use this whenever the user wants a defect dashboard, a sprint/defect status report, a stakeholder status update, release-risk comms, or asks to summarize defects or blockers for leadership/engineering/customers. Invoke with /dashboard-comms.
user-invocable: true
---

# Dashboard + Comms

Build an interactive HTML defect dashboard for the **active sprints** of the Transformation
Project, detect what changed since the last refresh, and — only when there are changes — draft
stakeholder-tailored status communications inside the same HTML. Follow the steps **in order** and
do not skip the gates (Step 3 decides whether comms are drafted; Step 5 is a human review loop).

## Fixed facts about this environment

- **Jira site / cloudId:** `vidhyaunni6.atlassian.net` → `3215a260-ee4c-42fd-8255-0c4dec1fb617`
- **Project "Transformation Project" = key `SCRUM`** (team-managed). Bug issuetype id `10007`.
- **Active sprints:** the Sprint field is `customfield_10020` (array of `{id,name,state,startDate,endDate}`).
  Active = `state == "active"`. Use the Jira board sprint **name** (e.g. `Sprint 4`) for grouping/filtering.
- **Severity = Jira Priority** (no native severity field): `Highest→Critical, High→Major,
  Medium→Moderate, Low→Minor`, anything else → `Unknown`. **Critical & major = Highest + High.**
- **Helper scripts:** `scripts/distill.mjs` (turns the large MCP search dumps into a compact
  `defects.json`) and `scripts/build_dashboard.mjs` (renders the HTML and computes the refresh diff).
  Jira reads are done by you via the `atlassian` MCP tools.

In every `atlassian` tool call pass `cloudId: "3215a260-ee4c-42fd-8255-0c4dec1fb617"`.

Working files live in `dashboard-comms/` under the project root: `defects.json` (current pull),
`snapshot.json` (previous pull, for diffing), `comms.json` (stakeholder content). The dashboard is
written to `defects-dashboard.html` in the project root, and timestamped PDF snapshots to
`dashboard-snapshots/`. **Run all node commands from the project root.**

---

## Step 1 — Pull active-sprint defects (Jira MCP) and distill to `defects.json`

Query open (active) sprints' bugs, paging until `isLast`:

```
searchJiraIssuesUsingJql(
  cloudId,
  jql="project = SCRUM AND issuetype = Bug AND sprint IN openSprints() ORDER BY priority DESC, updated DESC",
  fields=["summary","status","priority","labels","customfield_10020","description","created","updated","statuscategorychangedate"],
  maxResults=100, nextPageToken=<token>)
```

Results are large (~80 bugs ≈ 220K chars) — request **only** those fields, use the **default**
(structured JSON) response format (not `markdown`), and **do not** echo the raw payload into the
conversation. The MCP saves big results to a file shaped like `[{type,text},…]` (one element is the
deprecation notice, another's `text` is the `{issues,nextPageToken,isLast}` payload). Note the saved
file path(s) — one per page — and hand them to the distiller instead of hand-parsing:

```bash
node .claude/skills/dashboard-comms/scripts/distill.mjs <savedResultPage1.txt> [<savedResultPage2.txt> ...]
```

`distill.mjs` writes `dashboard-comms/defects.json` (deduped by key) and prints
`{totalDefects, activeSprints, perSprint, perSeverity}`. It derives, per issue: `severity` from
`priority.name` (`Highest→Critical`, …); `sprint` = the `state=="active"` entry in `customfield_10020`
(keeping its `endDate` as `sprintEndDate`); `sourceKey` from the non-`GSMR` label; `url`; and keeps
`summary`, full `description`, `status`, `statusCategory`, `created`, `updated`. Pass `--site <host>` if
the site ever differs from the default.

If you ever need to build it by hand instead, write `dashboard-comms/defects.json` in this shape:

```json
{
  "generatedAt": "<ISO now>",
  "site": "vidhyaunni6.atlassian.net",
  "activeSprints": [{"id":2,"name":"Sprint 4","state":"active","startDate":"...","endDate":"..."}],
  "defects": [{
    "key":"SCRUM-96","sourceKey":"GEMPOS-455","url":"https://vidhyaunni6.atlassian.net/browse/SCRUM-96",
    "summary":"...","description":"...","severity":"Minor","priority":"Low",
    "status":"In Progress","statusCategory":"In Progress","sprint":"Sprint 4","sprintEndDate":"2026-06-30T...",
    "created":"...","updated":"..."
  }]
}
```

Report the total defect count and a per-active-sprint count.

## Step 2 — Build the dashboard and compute the diff

```bash
node .claude/skills/dashboard-comms/scripts/build_dashboard.mjs
```

This writes `defects-dashboard.html` (severity×status matrix, **multi-select** sprint/severity/status
chip filters, defect drill-down with summary+description, and a "Changes since last refresh" panel) and
prints a JSON diff summary to stdout: `{ hasPrevSnapshot, hasChanges,
counts:{new,closed,statusChanged,removed}, criticalMajorChanges:[...] }`. **Read that stdout.** If an
approved `comms.json` already exists from a previous run, it is automatically re-embedded (it persists).

## Step 3 — Decide whether to draft comms (gate)

- **`hasPrevSnapshot == false` (first run):** the dashboard shows a "baseline established" note.
  **Do not draft comms.** Skip to Step 6.
- **`hasPrevSnapshot == true` and `hasChanges == false`:** dashboard shows "no changes since last
  refresh". **Do not draft new comms.** Any previously approved comms stay on the dashboard
  automatically (see persistence note). Skip to Step 6.
- **`hasPrevSnapshot == true` and `hasChanges == true`:** continue to Step 4.

Comms exist to communicate *change*, so a clean refresh (or the very first baseline) drafts no new ones.
**Persistence:** once a stakeholder update is approved (Step 5), it is **not** regenerated on a no-change
refresh — the last approved `comms.json` remains embedded as the latest iteration until the next refresh
that detects changes, which replaces it.

## Step 4 — Gather non-Jira inputs, then draft tailored comms

First, in a **single consolidated prompt**, ask the user for the two things Jira can't tell you, scoped
to the critical/major defects that changed or are blocking the active-sprint release:

1. **Mitigation / recovery plan** — workarounds, recovery actions, owners, revised ETAs.
2. **Decisions / asks / escalations** — what's needed from leadership/engineering/the customer
   (approvals, resources, go/no-go, sign-off on customer messaging).

Then read `references/comms-guidance.md` and draft **three** concise summaries — one per stakeholder
(Executive Leadership, Engineering Team, External Customer). Ground them in the dashboard data:
critical+major defect counts/keys, the changes since last refresh, and **delay/blocking risk** derived
from each active sprint's `endDate` vs. unresolved status (do **not** ask the user for a release date —
infer risk from sprint end date and open critical/major work). Tailor tone per `comms-guidance.md`.

Write `dashboard-comms/comms.json`:

```json
{
  "generatedAt":"<ISO now>",
  "context":{"mitigation":"<user input>","decisions":"<user input>"},
  "stakeholders":[
    {"id":"exec","role":"Executive Leadership","initials":"EL","color":"#6C5CE7","content":"..."},
    {"id":"eng","role":"Engineering Team","initials":"ENG","color":"#00B894","content":"..."},
    {"id":"cust","role":"External Customer","initials":"EXT","color":"#0984E3","content":"..."}
  ]
}
```

Embed the comms into the HTML (re-render — an existing `comms.json` is always embedded):

```bash
node .claude/skills/dashboard-comms/scripts/build_dashboard.mjs
```

Each stakeholder renders as a card with an inline SVG avatar (initials + color) and a copy-to-clipboard
button for its tailored content.

## Step 5 — Review loop (gate)

Present the three drafts to the user and ask for edits **per stakeholder** (and any change to the
mitigation / decisions context). Apply the edits to `dashboard-comms/comms.json` and re-run
`build_dashboard.mjs`. Repeat until the user confirms they're satisfied. Do not finalize
until the user approves.

## Step 6 — Finalize (persist snapshot + save a PDF snapshot)

```bash
node .claude/skills/dashboard-comms/scripts/build_dashboard.mjs --finalize --pdf
```

This re-renders the HTML, writes the current pull to `dashboard-comms/snapshot.json` (so the next
invocation can diff against it), and saves a **PDF snapshot** of the rendered dashboard to
`dashboard-snapshots/` in the project folder. (`--finalize` keeps any approved comms embedded; add
`--no-comms` only if you intentionally want to drop them.)

`--pdf` is self-gating: it writes a PDF **only for the baseline run** (first run, no prior snapshot —
filename `…_baseline.pdf`) and **for each refresh that has a diff** (`…_update.pdf`). A no-change refresh
writes no PDF. The PDF is produced with headless Chrome so the JS-rendered view (matrix, defects, and any
embedded comms) is captured faithfully; read the `pdf` field in the command's stdout to confirm the path
(or `pdf.error` if no Chrome was found — set `CHROME_PATH` and re-run `--finalize --pdf` in that case).

Report: the output path (`defects-dashboard.html`), the PDF snapshot path (if one was written),
total/per-sprint counts, the change summary, and whether comms were generated.

---

## Notes
- Re-running is safe and idempotent: `defects.json`/`defects-dashboard.html` are regenerated each run;
  `snapshot.json` only advances on `--finalize`, so the diff is stable across Steps 2–5 of one run.
  `comms.json` is only overwritten when a change-bearing refresh drafts a new update, so it always holds
  the latest approved iteration and persists through any number of no-change refreshes.
- The dashboard HTML is fully self-contained (no network, no external assets) — it can be emailed or
  opened directly in a browser. Avatars are inline SVG; copy buttons use the Clipboard API with a
  `document.execCommand` fallback.
- PDF snapshots accumulate in `dashboard-snapshots/` (timestamped, `_baseline` or `_update`), giving a
  point-in-time history: one for the baseline and one per change-bearing refresh. They require headless
  Chrome (`/Applications/Google Chrome.app/...` by default, or `CHROME_PATH`).
- The `atlassian` MCP SSE endpoint is being retired after 30 Jun 2026; if Jira calls start failing, the
  endpoint in `.mcp.json` needs updating to the Streamable HTTP URL (same caveat as `defects-update`).
