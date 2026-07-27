> **Public mirror.** Source of truth: https://github.com/seal-scripting/Battlestation-Bruce-Diagnostic/blob/main/SYSTEM_OVERVIEW.md (private repo). This copy may lag behind if it was not updated in the same push — if in doubt, and you have access, check the private repo directly.

---

# Battlestation Bruce Diagnostic — System Overview

_This file is the plain-language explanation of what this system is and how it currently
operates. It is meant to be read on its own — by a person (e.g. Professor Alex) or by a
Claude session — and answer "how does this work?" without needing to read code first._

**Maintenance rule: whenever the automation's actual behavior changes — a new stage, a
changed rule, a new external system touched — update this file in the same change.** It
should never describe a version of the system that no longer exists.

**Also update the public mirror:** this repo is private, so
`https://github.com/seal-scripting/automation-docs/blob/main/battlestation-bruce.md`
exists as a no-auth-required copy of this file for sessions without GitHub read access.
Copy any change made here into that file too, in the same work — it's a read-only
mirror, and it drifts stale the moment this file changes without it being updated
alongside.

_Last updated: 2026-07-27._

> Note: an older `BRUCE_SYSTEM_OVERVIEW.md` exists in a sibling directory,
> `Battlestation Bruce Automation/`, which is a stale/superseded predecessor to this
> project (`Battlestation Bruce Diagnostic/`, the live one). This file describes the
> code actually running today, verified against it directly — treat the other one as
> historical only.

---

## 1. What this system does, in one paragraph

Once a day, "Bruce" — a persona built for evaluating Battlestation quest progress —
scans every group's live "Battlestation Q" tracking sheet, has an LLM evaluate each
quest against Bruce's rubric, publishes a plain-language diagnostic report, and writes
any finding that needs a human's attention directly onto that quest's own Kanban board
as a task. The goal: catch quests that are stalling, stuck, or need help *before* a
human would otherwise notice, without anyone having to manually read five groups' worth
of spreadsheets every day.

## 2. Current status

**Currently disabled** — a local `.disabled` sentinel file sits in this project's
directory, paused pending bug fixes (since 2026-07-22). `run.sh` checks for this file
and skips the run entirely (exit 0, no-op) if present. Delete the file to re-enable.
This is deliberately a **local-only** file — it is gitignored, so re-enabling/disabling
is always a hands-on action on this machine, never something a GitHub push can change.

## 3. The daily pipeline (`run.sh`)

Runs via cron once a day. Each stage's output feeds the next:

| # | Script | What it does |
|---|---|---|
| 1 | `bruce_fetch.py` | For each of 5 groups (ITAC, Embedded, Plasma, Sudoku, BizTech), reads that group's "Battlestation Q" tab and writes a compact snapshot: `work/<group>.md` (one row per quest — status, title, leads, last-edit, deadline, milestone, score%, weekly delta) and `work/<group>.questmeta.json` (quest → its own per-quest Kanban sheet ID, read from a "YBR Link" row). Authenticates via a Google service-account key in `credentials/` (no human-owned OAuth token involved). |
| 2 | `bruce_diagnose.py` | Reads each group's snapshot and runs a headless `claude -p` call with Bruce's full persona spec as the system prompt — the actual "AI-level" evaluation. Bruce reads each quest and, where something's wrong, produces a flag: a severity (`act`/`watch`), a route (who should see it — a human channel or another persona), a `finding`, a ready-to-write `action` (task body text in Bruce's established voice), and a `due_date` (computed in Python, not by the model, for reliability). Writes `work/<group>.diagnosis.json`. |
| 3 | `bruce_render.py` | Turns the diagnosis JSON into a published Markdown report, written to two places in the shared Dropbox tree (`sealadmin`'s synced folder, `[5]Misc`): `bruce-diagnostic-latest.md` (always overwritten) and a dated copy per day under `Bruce Diagnostics/`, so there's a running history. |
| 4 | `watchdog_kanban_task.py --commit` | For every flag with severity `act` routed to an eligible human channel (not persona-routed, not `watch`-tier), writes a new row directly onto that quest's own per-quest Kanban tab — six specific cells only (From/To/Description/Due Date/Date Added/Last Modified), never touching status or existing rows. Gated by the `WATCHDOG_KANBAN_PUSH_ENABLED` env var: unset means dry-run (no real writes); set to `1` (as it is in the live cron line) means real writes happen. A dedup ledger (`work/_watchdog_kanban_pushed.json`) stops the same finding from becoming a second task on a re-run. |

If `bruce_fetch.py`, `bruce_diagnose.py`, or `bruce_render.py` fails, the whole run
aborts (the report is more important than the Kanban step, so those three are hard
gates). If only the Kanban-writing step fails, the run still counts as a success — the
diagnostic report has already been published, so a Kanban hiccup shouldn't block that.

## 4. Important cross-account dependency

Two parts of this pipeline reach into `sealadmin`'s Dropbox-synced folder tree on this
same machine (not this project's own account) via a world-readable path, because that's
where the shared Dropbox content actually lives:
- `bruce_diagnose.py` reads Bruce's live, human-maintained persona spec from there
  (`.../Sector-Personas/P2. Persona - Battlestation Bruce 2026/battlestation-bruce.assembled.md`).
- `bruce_render.py` writes the published report there, so it syncs out to everyone via
  Dropbox.

This is a **read/write dependency on Dropbox as the delivery/consumption channel for
Bruce's persona and its output report** — unrelated to how this project's own *code* is
versioned. This project's code now lives in GitHub (see below); Dropbox is still how
Bruce's spec is authored and how its report reaches readers, and that hasn't changed.

## 5. Where things live on disk

```
run.sh                        the daily cron entry point — runs the 4 stages in order
bruce_fetch.py                stage 1 — snapshot each group's Battlestation Q
bruce_diagnose.py             stage 2 — the actual LLM evaluation
bruce_render.py                stage 3 — publish the report to Dropbox
watchdog_kanban_task.py        stage 4 — write findings to each quest's Kanban board
run_logger.py / sheets_retry.py   shared logging + retry-on-429 helpers
credentials/                  Google service-account keys — never in git
work/                          per-run snapshots/diagnoses/ledger — regenerated, never in git
.disabled                     local kill-switch — never in git, always a hands-on toggle
.tmp/run.log                  cron output log
```

## 6. Current status: GitHub-based live-edit

This project's code lives in its own GitHub repo (pushed via an HTTPS remote with a
fine-grained Personal Access Token embedded in the URL, Contents: Read/write scoped to
this repo only — same pattern as the other SEAL automations). `run.sh` does
`git pull --ff-only origin main` as its first action (before even checking `.disabled`),
so a code fix lands in the working tree the moment it's pushed, whether or not the
daemon is currently paused. If the pull fails, that day's run aborts (fail-closed)
rather than running on a possibly-stale tree. Secrets (`credentials/`) and regenerated
runtime output (`work/`) are gitignored and have never been pushed. This repo is edited
by technical staff directly; this file exists so a non-technical reader (or a Claude
session) can understand what the system does without reading Python.

## 7. How to check on it right now

- **Is it currently enabled?** Check whether `.disabled` exists in this directory.
- **Did the last run succeed?** `.tmp/run.log` has a timestamped entry for every run.
- **What did Bruce find?** The published report in the shared Dropbox `[5]Misc` folder
  (`bruce-diagnostic-latest.md`, or a specific day's dated copy).
- **Structured run history / errors?** The "Bruce Automation Log" Google Sheet
  (`run_logger.py`'s "Run Log" and "Error Log" tabs).
