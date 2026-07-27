> **Public mirror.** Source of truth: https://github.com/seal-scripting/Sector-Account-Sharing-Automation/blob/main/SYSTEM_OVERVIEW.md (private repo). This copy may lag behind if it was not updated in the same push — if in doubt, and you have access, check the private repo directly.

---

# Sector Account Sharing Automation — System Overview

_This file is the plain-language explanation of what this system is and how it currently
operates. It is meant to be read on its own — by a person (e.g. Professor Alex) or by a
Claude session — and answer "how does this work?" without needing to read code first._

**Maintenance rule: whenever the automation's actual behavior changes — a new rule, a
new external system touched, a status change — update this file in the same change.**
It should never describe a version of the system that no longer exists.

_Last updated: 2026-07-27._

---

## 1. What this system does, in one paragraph

Ten shared Claude/computer accounts each have a Google Doc holding that account's
password. This system keeps each doc's sharing list in sync with a master spreadsheet:
whoever a spreadsheet says is authorized for an account gets Viewer access to that
account's password doc; whoever isn't authorized (anymore) gets removed. It runs
automatically, event-driven, whenever someone edits the spreadsheet — no one has to
remember to update sharing by hand when a person joins, leaves, or changes teams.

## 2. Design philosophy (why it's built this way)

Same 3-layer separation as the other SEAL automations:

- **`directives/share_accounts.md`** — the SOP: what to read, how to reconcile, every
  hard-won edge case from real incidents. Source of truth for intent.
- **`execution/*.py`** — deterministic Python that does the actual reconciling.

## 3. How a run works

1. **Read the access matrix** — the "SEAL Claude Account Sharing" spreadsheet, `Users`
   tab. Columns are accounts (Artintel1110, Artintel1120, SEAL 215J, ArtSensitive, Seal
   Sector 1–6, ARC); each row is a person; a `✓ <Name>` cell means that person is
   authorized for that account.
2. **Resolve names to emails** — look up each authorized nickname in the SEAL Clan Life
   spreadsheet (`Associates` tab first, then `Affiliates`), exact-normalized match only
   (no fuzzy matching — similar names must never collide).
3. **Find each account's password doc** — one Google Doc per account, named
   `"<Account> Password"`, in the SEAL Passwords folder.
4. **Reconcile, idempotently** — add anyone authorized-but-missing (silent, Viewer-only,
   no notification email); remove anyone with access who's no longer authorized;
   downgrade anyone holding more than Viewer back to Viewer. A clean run makes zero
   changes.
5. **Delete unresolved rows** — a Users-tab row whose nickname has no valid email in
   Clan Life is deleted entirely from the sheet (not just skipped), since it was never
   shared and never will be until the nickname is fixed to match Clan Life.
6. **Log everything** — every add/remove/downgrade/unresolved/warning is appended to a
   dedicated audit-log spreadsheet.

**Safelist (never touched):** the doc owner (`sealscripting@gmail.com`), the
`seal-manager` service account (other automation depends on it), and **KaviP**
(kept as Editor/admin, not downgraded).

**Never auto-removed:** broad "anyone with the link" or whole-domain shares — those are
logged as `WARN` for a human to review, not silently revoked.

## 4. Trigger architecture (event-driven, not polling-heavy)

This does **not** run on a simple fixed schedule. Instead:

1. A Google Apps Script `onEdit` trigger fires whenever anyone edits the `Users` tab
   and writes a timestamp into a `Trigger` tab cell.
2. `poll_and_trigger.py` runs every 2 minutes via cron. If the trigger timestamp is
   newer than the last successful reconcile, it runs `reconcile_sharing.py --live`.
3. On success, it clears the trigger (with a 12-second-delayed double-check, to dodge a
   feedback loop where the reconcile's own row-deletions re-fire the edit trigger).
4. On failure, the trigger is **left set** — the next poll (≤2 min later) retries
   automatically, no separate alerting needed for transient failures.
5. `reconcile_sharing.py` also runs directly once a day via its own cron line, as a
   backstop in case the event-driven path is ever missed.

## 5. The systems this touches

| System | Role |
|---|---|
| **SEAL Claude Account Sharing** (Google Sheet) | `Users` tab = the access matrix; `Trigger` tab = the edit-trigger cell |
| **SEAL Clan Life** (Google Sheet) | `Associates`/`Affiliates` tabs — nickname → email lookup |
| **SEAL Passwords folder** (Google Drive) | Holds the 10 password docs, one per account |
| **Audit Log** (Google Sheet) | Every action this system takes, one row each |
| **Google Apps Script** (bound to the Account Sharing sheet) | Fires the edit trigger |

## 6. Where things live on disk

```
execution/poll_and_trigger.py     the 2-minute cron entry point — checks for a pending trigger
execution/reconcile_sharing.py    the actual reconcile logic (--dry-run or --live)
directives/share_accounts.md      the SOP — the "why" and the incident history
.env                              sheet IDs, folder ID, safelist, audit-log ID — never in git
token_drive.json / credentials.json   cached OAuth token / client secret — never in git
.tmp/                              last_run.txt and logs — always safe to delete/regenerate
```

## 7. Current status: GitHub-based live-edit

This project's code lives in its own GitHub repo (pushed via an HTTPS remote with a
fine-grained Personal Access Token embedded in the URL, Contents: Read/write scoped to
this repo only — same pattern as SEAL Onboarding Automation V1). Both cron-invoked
entry points (`poll_and_trigger.py` and `reconcile_sharing.py`) do `git pull --ff-only
origin main` as the very first thing they do; if the pull fails, that run aborts
(fail-closed) rather than proceeding on a possibly-stale tree, and — since
`poll_and_trigger.py` runs every 2 minutes — the next cycle simply retries. Secrets
(`.env`, `credentials.json`, `token_drive.json`) are gitignored and have never been
pushed. This repo is edited by technical staff directly; this file exists so a
non-technical reader (or a Claude session) can understand what the system does without
reading Python.

## 8. Notable incidents worth knowing about (full detail in the directive)

- **The "Vivian loop"** — the reconcile script's own row-deletions were asynchronously
  re-firing the Apps Script edit trigger, causing an infinite re-reconcile loop. Fixed
  with a sealscripting-authorship guard in the Apps Script plus a delayed double-clear
  on the Python side.
- **Non-Google email crash** — a person with no linked Google account crashed the entire
  run when the Drive API rejected a silent-share request for their address. Fixed with
  a per-permission try/except so one bad address just logs a `WARN` and the run
  continues.
- **ARC doc orphaned** — an account column was removed from the spreadsheet, leaving its
  password doc unmanaged. The reconciler now logs a `WARN` for any doc in the folder
  that can't be matched to a current account column, so this is caught automatically.

## 9. How to check on it right now

- **Did the last run succeed?** `.tmp/last_run.txt` (updated only on success) and the
  Audit Log spreadsheet (a `RUN` summary row every time).
- **What does a given step actually do, in detail?**
  `directives/share_accounts.md` — written for exactly this question.
- **Is a change safe to make?** Run `python execution/reconcile_sharing.py --dry-run`
  first — it reports the full add/remove/downgrade plan and makes zero changes.
