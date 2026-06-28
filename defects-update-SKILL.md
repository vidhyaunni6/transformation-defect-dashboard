---
name: defects-update
description: Sync the UAT defect list from the "Transformation Project" Google Drive folder into the Transformation Project (SCRUM) Jira project as Bugs — mapping each defect to its user story by label, resolving ambiguous rows with the user, and creating or updating Bugs idempotently with no duplicates. Invoke with /defects-update.
user-invocable: true
---

# Defects Update

Sync UAT defects (a versioned Google Sheet) into Jira as Bugs under the right user stories,
idempotently. Follow the steps **in order**. Do not skip the confirmation and verification gates.

## Fixed facts about this environment

- **Jira site / cloudId:** `vidhyaunni6.atlassian.net` → `3215a260-ee4c-42fd-8255-0c4dec1fb617`
- **Jira project "Transformation Project" = key `SCRUM`** (team-managed). Bug issuetype id `10007`, Story id `10004`.
- **Drive folder "Transformation Project"** id `1-uTFK62qH0fTkvx3HLWgLYP2DBvTW1y-` — **the only in-scope source.**
- **Story↔label rule:** every SCRUM Story carries a label equal to its GSMR key. Build this map live in Step 3
  (do not hardcode). Static fallback: `jira-key-mapping.csv` in the project root.
- **Helper script:** `scripts/read_defects.mjs` (Drive-only; Jira is done by you via the `atlassian` MCP tools).

In every `atlassian` tool call pass `cloudId: "3215a260-ee4c-42fd-8255-0c4dec1fb617"`.

---

## Step 1 — Locate the latest version (folder-scoped) and confirm

Run from the project root:

```bash
node .claude/skills/defects-update/scripts/read_defects.mjs versions
```

This returns the defect-list files in the Transformation Project folder, newest-first (by version, then
modified time). Scope is **strictly this folder** — do not search elsewhere in Drive.

- Take `files[0]` as the candidate latest. Tell the user its `name`, `version`, and `modifiedTime`.
- **If more than one version exists**, explicitly say which is newer and **ask the user to confirm it is indeed
  the latest** before continuing.
- If only one exists, still confirm it with the user.
- If `count` is 0, stop and report that no matching file was found in the folder.

Do not proceed without the user's confirmation. Record the chosen `fileId`.

## Step 2 — Load the defects

```bash
node .claude/skills/defects-update/scripts/read_defects.mjs rows <fileId>
```

Returns `{ header, rows, rowCount, uniqueKeyCount }`. Each row is an object keyed by column name:
`Key, Summary, Defect Classification, Component/s, Priority, Fix Version, Assignee, Status,
Labels/User Story, Description, Sprint, Date Created, Remarks`.

Report `rowCount` and `uniqueKeyCount`. The `Key` (e.g. `GEMPOS-266`) is the unique defect id and the
**dedup key**. If `rowCount != uniqueKeyCount`, list the duplicate Keys and ask how to handle them before continuing.

## Step 3 — Build the Story label index (source of truth)

```
searchJiraIssuesUsingJql(cloudId, jql="project = SCRUM AND issuetype = Story ORDER BY key ASC",
                         fields=["summary","labels","customfield_10020"], maxResults=100)
```

For each Story, for every label matching `GSMR-\d+`, map `label -> { storyKey, summary, sprintId }`, where
`sprintId` is the Story's current sprint — the `state=="active"` entry in `customfield_10020` if present,
otherwise the last (most recent) entry; `null` if the Story has no sprint. This sprint is what new Bugs inherit
in Step 7. If the query returns no stories, fall back to `jira-key-mapping.csv` (`snapshot_key` = GSMR label,
`jira_key` = Story) — in which case `sprintId` is unavailable and new Bugs are created without a sprint.

## Step 4 — Map each defect to a Story

For each row, extract the set of `GSMR-\d+` tokens found in **both** `Labels/User Story` and `Summary`
(note the spreadsheet uses suffixed forms like `GSMR-5_Income`; the regex still extracts `GSMR-5`). Dedupe the tokens:

- **Exactly one** token that exists in the Story index → **mapped** to that Story.
- **More than one distinct** known token → **ambiguous (overlap)**.
- **Zero** tokens, or only tokens absent from the Story index → **ambiguous (unknown)**.

Keep a running map `Key -> storyKey` for mapped rows, and a list of ambiguous rows with their candidate stories.

## Step 5 — Resolve ambiguous rows (one consolidated review)

Present **all** ambiguous rows together in a single table: `Key`, `Summary` (truncated), detected tokens, and
candidate stories. Ask the user to assign each ambiguous row to one Story key (or to **skip** it). Apply the
answers to the `Key -> storyKey` map. Do not prompt row-by-row.

## Step 6 — Verify mapping completeness (gate)

Before any Jira writes, assert that **every unique `Key`** is either mapped to a Story or explicitly skipped by
the user. Print: per-Story defect counts, the total mapped, and the skip list. If anything is unaccounted for,
return to Step 5. Only continue once this holds.

## Step 7 — Create new Bugs / update status of existing Bugs (idempotent; dedup on the Key label)

First, read Bug field metadata once to learn whether `priority` is on the Bug screen and its allowed values:
`getJiraIssueTypeMetaWithFields(cloudId, projectIdOrKey="SCRUM", issueTypeId="10007")`.

Process defects **in small batches** (e.g. 10–20). For each mapped defect:

1. **Find existing:** `searchJiraIssuesUsingJql(cloudId, jql='project = SCRUM AND issuetype = Bug AND labels = "<Key>"', fields=["summary","labels","status","issuelinks"])`.

2. **If NOT found → create (new entries only):**
   Build the full field set and call `createJiraIssue`:
   - `summary` = the defect `Summary`.
   - `labels` = `["<Key>", "<Story GSMR label>"]`.
   - `description` (markdown) = the source `Description`, followed by a metadata block:
     `Source Key`, `Assignee (source)`, `Defect Classification`, `Component/s`, `Fix Version`, `Sprint`,
     `Date Created`, `Status (source)`, `Remarks`.
   - `priority` — only if metadata shows priority is available: map `Critical→Highest, Major→High, Minor→Low`
     (snap to the project's actual allowed names; omit if unavailable).
   - `assignee` — try `lookupJiraAccountId(cloudId, searchString=<Assignee>)`; if a unique account is found, set
     `assignee_account_id`; otherwise leave unassigned (name preserved in description).
   - `Sprint` (`customfield_10020`) — set to the mapped Story's `sprintId` from the Step 3 index (pass the sprint
     **id** as an integer in `additional_fields`, e.g. `{"customfield_10020": <sprintId>}`), so the Bug lands in the
     **same sprint as the user story it relates to**. Omit only if the Story has no sprint (`sprintId == null`).
   - After creating, apply the status transition (see below) and create the `Relates` link to the Story.

3. **If found → check for status change first:**
   - Compare the current Jira status to what the source-sheet status maps to (see mapping below).
   - **No status change** → skip entirely. Do not call `editJiraIssue`, do not transition, do not touch the link.
     Count as **unchanged**.
   - **Status change detected** → update all fields via `editJiraIssue` (summary, description, labels, priority,
     assignee, **Sprint** — same field set as a new Bug, including `customfield_10020` = the Story's `sprintId`)
     **and** apply the status transition. Also ensure the `Relates` link to the mapped Story exists (create it if
     missing). Count as **status-updated**.

4. **Status transition mapping** (applies to both new and existing Bugs):
   Call `getTransitionsForJiraIssue` at runtime to get available transitions — do not hard-code ids.
   Map source `Status` column value:
   - `Open`, `Reopened`, `Pending Information` → leave **Open / To Do** (no transition for new; no transition
     for existing unless it was previously moved forward and needs to go back — in that case, leave it as-is).
   - `In Progress`, `In Verification`, `Resolved`, `Pending Deployment` → transition to **In Progress** (id ~41).
   - `Closed` → transition to **Released** (id ~51).

Track counts as you go: created, status-updated, unchanged, skipped (user-skipped), failed (Key + error).

## Step 8 — Verify no duplicates and full coverage (gate)

Re-query all Bugs: `searchJiraIssuesUsingJql(cloudId, jql="project = SCRUM AND issuetype = Bug", fields=["labels"], maxResults=100)` (page through if needed). Then assert:

- Every processed `Key` appears in **exactly one** Bug's labels (no duplicate Keys).
- The number of distinct Key-labelled Bugs equals (mapped defects − user-skipped).

Report a final summary: total defects, created (new), status-updated (existing), unchanged (existing, no action needed), skipped (user-skipped), failures, and any Key that is missing or duplicated. If duplicates exist, list them for the user to reconcile — do not auto-delete.

---

## Notes
- The `atlassian` MCP currently points at the SSE endpoint being retired after 30 Jun 2026; if Jira calls start
  failing, the endpoint in `.mcp.json` needs updating to the Streamable HTTP URL.
- `read_defects.mjs` is read-only against Drive (`drive.readonly`). It cannot write to the sheet.
- Re-running this skill is safe: existing Bugs are never duplicated and their fields are not overwritten — only
  their status is touched if the source sheet shows a change. New rows in the sheet are created as new Bugs.
