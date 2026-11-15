---
name: tracker
description: Live progress page for long jobs. Use this whenever the user types /tracker, and also, without being asked, at the start of any request you expect to take more than 15 minutes (multi-step research, reports, data clean-ups, building or fixing code across several files, anything with many tool calls). Adds a run with steps tailored to the request to the owner's private "Claude Run Tracker" page and keeps it updated as steps start, finish or fail, or as the plan changes, so the owner can follow along from any device.
---

# Tracker

The owner follows long jobs from their phone or another computer. The tracker page is their only view of what you are doing, so keep it true and current.

- **Page:** https://claude.ai/artifact/RJKSUfXkHnFywKtyeYbrJK
- **Tool:** `ArtifactData`, always with `url` = the page link and `action: "batch"`.
- **Data:** collection `runs`, one document per run. The page refreshes by itself; never republish it.

## When to track

- The user types `/tracker`: track the task they gave. If there is none, share the link and ask what to track.
- Any request you expect to take more than 15 minutes: track it without being asked.
- Quick questions: don't track.

## Data model

```
runs/<runId>                    runId: YYYYMMDD-HHMM-short-slug, e.g. 20261114-0905-q3-sales-summary
  title    short plain name of the job
  started  UTC time, e.g. "2026-11-14T09:05:00Z" (date only "2026-11-14" if you can't tell the time)
  updated  UTC time of this write
  status   in_progress | waiting | done | failed | stopped
  note     one line: interruptions, questions for the owner, plan changes ("" if none)
  steps    list, in display order, of { name, what, status, result }
           step status: waiting | in_progress | done | failed | skipped
```

Write for a non-technical reader: `name` is 1 to 3 words, `what` is one short plain sentence, `result` is one line filled when a step ends (what came out, or why it failed). Plan 3 to 8 steps that fit this request. Keep one step `in_progress` while working. Get the time with `date -u +%Y-%m-%dT%H:%M:%SZ` when you have a shell; otherwise use the time you know.

Run `status`: `waiting` when you need the owner (say what in `note`), `done` when every step is done or skipped, `failed` when the job cannot finish, `stopped` when the owner cancels.

## Writing

**Start** (creates the run, so no `if_version`):

```json
{"action": "batch", "url": "https://claude.ai/artifact/RJKSUfXkHnFywKtyeYbrJK", "writes": [
  {"op": "set", "collection": "runs", "doc_id": "20261114-0905-q3-sales-summary", "data": {
    "title": "Q3 sales summary", "started": "2026-11-14T09:05:00Z", "updated": "2026-11-14T09:05:00Z",
    "status": "in_progress", "note": "",
    "steps": [
      {"name": "Collect figures", "what": "Pull Q3 sales from the spreadsheet", "status": "in_progress", "result": ""},
      {"name": "Check totals", "what": "Compare with last quarter", "status": "waiting", "result": ""},
      {"name": "Write summary", "what": "One-page summary in plain words", "status": "waiting", "result": ""}]}}]}
```

**Update** (step 1 done, step 2 started). Use `update`, with `if_version` set to the `version` the previous write returned:

```json
{"action": "batch", "url": "https://claude.ai/artifact/RJKSUfXkHnFywKtyeYbrJK", "writes": [
  {"op": "update", "collection": "runs", "doc_id": "20261114-0905-q3-sales-summary", "if_version": 1, "data": {
    "updated": "2026-11-14T09:20:00Z",
    "steps": [
      {"name": "Collect figures", "what": "Pull Q3 sales from the spreadsheet", "status": "done", "result": "1,240 rows, 3 regions"},
      {"name": "Check totals", "what": "Compare with last quarter", "status": "in_progress", "result": ""},
      {"name": "Write summary", "what": "One-page summary in plain words", "status": "waiting", "result": ""}]}}]}
```

- **Always send the whole `steps` list.** Lists are replaced, not merged. That is also how you add, rename, re-order or remove a step: send the new list, keep earlier results, and say why in `note`.
- Other fields can be sent alone, e.g. `"data": {"updated": "...", "note": "Paused: waiting for the owner to confirm regions"}`.
- If a write is refused as out of date, or you lost the version, read the run (`"action": "get", "collection": "runs", "doc_id": ...`) and redo the write on top of it.
- If one change touches several runs, put them all in the same batch.

## When to write

1. At the start, before the real work begins.
2. When a step starts, finishes, fails or is skipped. Finishing one step and starting the next is one write.
3. When the plan changes or you are interrupted or blocked (update `note`).
4. At the end: final run status and every step's final status and result.

Write only on real changes. Writing on a timer just adds noise.

## Share the link once per run

Right after the start write, tell the owner once: "Live progress: https://claude.ai/artifact/RJKSUfXkHnFywKtyeYbrJK". Don't repeat it later in the same run.

## If tracking is unavailable

Some apps expose the same writes as the `Artifact` tool with `action: "write_db"` and `db_op: "batch"` (other fields unchanged). If neither tool exists, or writes keep failing, tell the owner once that live tracking is unavailable here and carry on with the job. Tracking must never hold up the work. Never put passwords, keys or other secrets on the page.
