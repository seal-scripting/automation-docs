# SEAL Onboarding Automation — System Overview

_This file is the plain-language explanation of what this system is and how it currently
operates. It is meant to be read on its own — by a person (e.g. Professor Alex) or by a
Claude session — and answer "how does this work?" without needing to read code first._

**Maintenance rule: whenever the automation's actual behavior changes — a new stage, a
changed rule, a new external system touched, a status change like the GitHub live-edit
setup below — update this file in the same change.** It should never describe a version
of the system that no longer exists. If you (a Claude session) make a change and are not
sure whether it belongs here, err on the side of updating this file.

**Also update the public mirror:** this repo is private, so
`https://github.com/seal-scripting/automation-docs/blob/main/onboarding-v1.md` exists as
a no-auth-required copy of this file for sessions without GitHub read access. Copy any
change made here into that file too, in the same work — it's a read-only mirror, and it
drifts stale the moment this file changes without it being updated alongside.

**Also update the Dropbox copy:** a plain-text copy of this file lives at
`[MASTER]ClaudeMain/[3]Tools/Automations/docs/onboarding-v1.md`, so a Claude session
with Dropbox access but no GitHub credentials or working URL-fetch tool can read it
directly — no fetch, no search, no asking a person to paste content back. Copy any
change made here into that `[MASTER]` file too, in the same work. **Do not** also write
the full content into the `[ACTIVE]` tree — its `docs/onboarding-v1.md` is intentionally
just a short pointer stub to the `[MASTER]` copy (2026-07-28: real content used to live
in both trees; now it's `[MASTER]`-only so there's one file to update, not two).

_Last updated: 2026-08-31 (Run Log fix)._

---

## 1. What this system does, in one paragraph

Every hour, five scripts run in sequence to keep SEAL's membership records, Google Groups,
and Slack workspace access in sync with each other. New applicants get reviewed and
routed; students who finish onboarding get promoted into full membership; members who
leave or change status get evicted from active rosters and have their access revoked;
and a running audit makes sure nobody's Slack access has drifted out of step with who is
actually still an active member. A separate daily job lifts time-based holds. The whole
thing exists so that a human doesn't have to manually cross-reference five different
Google Sheets and two other systems (Slack, Google Groups) every time someone joins,
advances, or leaves.

## 2. Design philosophy (why it's built this way)

The system deliberately separates **intent** from **execution**:

- **`directives/*.md`** — one Markdown file per script, written like an SOP for a person.
  Each says what the script is for, what it reads/writes, and documents every hard-won
  lesson from real incidents. **These are the source of truth for intent** — read the
  relevant directive before touching a script's logic.
- **`execution/*.py`** — deterministic Python. No judgment calls at runtime; all the
  judgment calls were made in advance and encoded as rules.

The reasoning: an LLM making the same judgment call fresh every hour would eventually get
it wrong (even at 99% accuracy per decision, that compounds across many decisions and many
runs). Encoding the decision once as code, and reserving the LLM for the parts that
actually need language understanding or one-off orchestration, is what makes this reliable
enough to run unsupervised, hourly, against real people's access and real emails.

Every time something has broken in production, the fix followed the same loop: fix the
script → update the directive with what was learned → the system is a little more robust
than before. The "Learnings / Known Constraints" sections at the bottom of each directive
are not historical trivia — they are why the code has the shape it has today. Skim them
before assuming a script's behavior is a bug rather than a deliberate scar from a past
incident.

## 3. The hourly pipeline (`run_all.sh`)

Runs via cron, once an hour. Order is load-bearing — each stage depends on the previous
one having already updated the roster:

| # | Script | Directive | What it does |
|---|---|---|---|
| 0 | `token_health_check.py` | — | Pre-flight: confirms every OAuth token can still refresh. If any token is dead, the whole run aborts before touching anything (nothing worse than a partial run against half-working credentials). |
| 1 | `process_clan_cleanup.py` | [`process_clan_cleanup.md`](directives/process_clan_cleanup.md) | Reads the **Associates** tab of **SEAL Clan Life**. Routes departing/reclassified members out: evictions (GameOver/Ex-Communicado, Ex-Associate) move the row to the **Clan Life AAD** sheet and pull the person out of Google Groups + Slack — GameOver/Ex-Communicado additionally sends the student a one-time removal-notice email (as of 2026-08-11) explaining why and that they may reapply in six months; Affiliate moves the row within Clan Life; **Alumni** (a specific grade-column status) does a full access teardown (all groups) plus a one-time Discord-invite email. Also runs a **Sandbox swap** — a grade-driven, read-only Google Group membership toggle (in/out of the sandbox group) that never touches the sheet rows. Runs first because later stages assume departures have already been removed. |
| 2 | `process_applicants.py` | [`process_applicants.md`](directives/process_applicants.md) | Reads new rows in the **SEAL Applicants** sheet's "Current Applicants" tab, classifies each Approved/Rejected based on a status column, files them into the right tab, adds approved applicants to the onboarding Google Group, and sends the approval/rejection email (tracked so it's never sent twice). The field-of-interest/waitlist/referral system (2026-08-10) is present in code but disabled as of 2026-08-31 — see §3a below. §3b covers the 2026-08-24 stuck-applicant fix and reprocessing gate, which still applies. Supports `--dry-run`. |
| 3 | `process_challenge.py` | [`process_challenge.md`](directives/process_challenge.md) | Watches the **SEAL Applicant Challenge** sheet for applicants who've hit "stage 3" (finished Step 1 of onboarding). Promotes them into the **Associates** tab of Clan Life, adds them to the active Google Group, and gets them into the Slack workspace (invite if new, reactivate if returning). Runs after cleanup so a promoted student is never mistaken for someone who already left. |
| 4 | `process_slack_audit.py` | [`process_slack_audit.md`](directives/process_slack_audit.md) | Compares every current Associate against the Slack workspace member list and fixes any drift (missing → invite, deactivated → reactivate). Deliberately runs **last** among the first four — it trusts the Associates tab completely, so anyone who should have already been evicted needs to be gone from Associates *before* this runs, or it will incorrectly restore their Slack access. |
| 5 | `process_onboarding_cleanup.py` | [`process_onboarding_cleanup.md`](directives/process_onboarding_cleanup.md) | Removes departed members from the `onboarding@maxalton.com` Google Group (the one applicants get added to before they're full members), by cross-referencing the AAD sheet's departure tabs against current Associates so returning members are never wrongly removed. Also does routine housekeeping: deletes stale (>7-day, no check-in) rows from the Applicant Challenge sheet. |
| (daily, once/day) | `process_aad_expiry.py` | — (bash-level utility, no directive yet) | Once a day only: lifts the 6-month Ex-Communicado hold — anyone whose Expiry Tracker date has passed is removed from both Ex-Communicado and the tracker, so they become eligible to reapply. |

If any stage errors or hangs past its timeout, `run_all.sh` keeps going with the remaining
stages (it does not abort the whole run for one bad stage) and emails
`harrisnakajima@gmail.com` a summary of what failed (`error_notify.py` — labeled TEMPORARY
in the code, meaning it's a deliberate stopgap that can be removed once the pipeline has a
longer trust record, not a sign something is wrong).

There's also a **daily, separate cron line** for `monitor_fx_associates.py` — not part of
`run_all.sh`. It watches for a specific historical spreadsheet-formula bug (wrong-row
references in an FX formula, fixed 2026-04-28) recurring, and self-disables after 7
consecutive clean days.

### 3a. The waitlist + referral system (added 2026-08-10, SUNSET 2026-08-31)

**Disabled as of 2026-08-31, per Harris.** `waitlist_enabled: false` in `config.yaml`
turns off everything described in this section — the field-of-interest gate, the
referral resolution, manual promotion, and pending-referral expiry sweep — without
deleting any code, so it can come back with a one-line config change. While
disabled, every non-blank, non-rejected applicant goes straight to Approved, same
as before 2026-08-10. A git revert to before this feature was considered and
rejected: 6 unrelated commits (an OAuth-scope fix, the 2026-08-24 stuck-applicant
fix in §3b below, a removal-notice email feature, a Pending Referrals crash fix)
landed in the same files afterward, and a revert would have destroyed those too.
As part of the sunset, the 13 real applicants then sitting on the Waitlist tab were
migrated to Approved (sheet row + both Google Groups), and the ~300 rows that had
piled up in Pending Referrals were cleared — both confirmed with Harris first.
The rest of this section describes what re-enabling the flag turns back on.

Professor Alex asked for a way to stop the applicant pool skewing so heavily toward
engineers — the lab needs more business/entrepreneurship/marketing/finance-minded
students. The fix layers on top of the existing Approved/Rejected classification,
without changing it for anyone who was already going to be Rejected:

- An applicant who would otherwise be **Approved** (reviewer set a non-blank,
  non-rejection status) now also has their "Positions you are applying for" answer
  (and, as of 2026-08-26, their "Interest" free-text answer too — students tend to
  elaborate more there) checked against an accept-keyword list (business, marketing,
  entrepreneurship, finance, strategy, and similar — the live list is column B of the
  **Keywords** tab in the SEAL Applicants sheet, kept separate from that tab's
  unrelated pre-existing column A). A match in either field → still Approved. No
  match in either → **Waitlisted** instead, into a new **Waitlist** tab that mirrors
  Approved's layout. They get a waitlist email explaining why, with a way back in:
  recruit someone.
- **Referral path back in:** a new applicant can name their referrer in "How Did You
  Hear of Us?". If that name matches someone currently on the Waitlist tab, that
  person is promoted straight to Approved (same group-add + email as any normal
  approval) and removed from the Waitlist. If the named person **isn't** on the
  Waitlist yet (they haven't applied themselves), the referral is held in a new
  **Pending Referrals** tab for 7 days — if that person applies within the window
  and would otherwise be waitlisted, they're approved directly instead (the pending
  credit auto-redeems); if the week passes with no application, the entry expires
  and is deleted automatically on the next run.
- Deletions in this flow (removing a promoted person from Waitlist, removing a
  resolved/expired Pending Referrals row) always re-confirm the row's identity
  immediately before deleting and batch all deletes for a tab into one call in
  descending row order — the same discipline adopted after the 2026-06-06
  wrong-row-deletion incident below, applied here from the start.
- **Manual leader override (added 2026-08-29):** a lab leader can also promote a
  waitlisted applicant to Approved directly, without a referral, by checking a
  checkbox in column P ("Manual Approve") on that person's row in the Waitlist
  tab itself — no need to go through Claude Code. On the next hourly run that row
  is promoted through the exact same write/group-add/email pipeline as a referral
  promotion, and removed from Waitlist the same identity-reverified way. A row
  flagged this way is excluded from that run's referral matching so it can't be
  promoted/deleted twice in one cycle.
- **Quality-check exception for Waitlist promotions (added 2026-08-29):** a
  cross-tab quality check normally blocks the approval email for an address
  that also appears in Rejected or Waitlist (usually a sign of misclassification).
  A row promoted off Waitlist this run — referral or column P — is a deliberate
  decision, so it always gets the approval email regardless, even against an
  older, unrelated Rejected record for the same person from a different
  application cycle. Scoped to this run's promotions only; logged as an
  `OVERRIDE` rather than silently skipped.

### 3b. The stuck-applicant fix and reprocessing gate (2026-08-24)

Applicant review silently stalled starting 2026-08-10 — the same day the waitlist feature
above shipped. Column N ("Status") in "Current Applicants" is meant to be set before a row
is classified; nothing had been setting it for two weeks, so every applicant who applied
after 2026-08-10 (and a large pre-existing backlog going back further) sat permanently in
"awaiting review" — zero errors anywhere in the logs, because the script was working exactly
as designed. Diagnosed and fixed same-day:

- **Blank status is no longer skipped.** It's now treated identically to any other
  non-rejected status — it flows through the same field-of-interest gate as everyone else
  (Approved or Waitlisted, never Rejected, since rejection requires an explicit keyword
  match).
- **A new gate on column O** ("Current Applicants") throttles how much of the backlog gets
  processed per cycle. Column O there is a pre-existing, reviewer-maintained "I've already
  contacted this person" marker — a different field from the same-lettered "Email Sent"
  column O on the Approved/Rejected/Waitlist tabs, despite sharing an index. A non-blank
  value means this row's own classification is not reconsidered this cycle. This is what let
  the two-week backlog come back online gradually (13 real applicants in the first live run)
  instead of firing hundreds of emails in one shot.
- **Referral resolution bypasses that gate.** A person currently on the Waitlist can still be
  promoted to Approved via a new applicant's referral even when that new applicant's own row
  hasn't been cleared by the column-O gate — otherwise the referral system would functionally
  break for the entire backlog. Verified with a real test row (create + dry run + cleanup):
  both the referring applicant and the referred Waitlist person came out Approved.
- **The automation now writes "Y" back to that same column O** after actually sending an
  approval/rejection/waitlist email, traced from the destination-tab row back to its source
  row via the same (email, date) key used for dedup. Previously only a human ever wrote to
  that column, so the automation's own sends left it looking untouched — a reviewer scanning
  the sheet had no way to tell "the bot already emailed this person" from "still needs
  outreach."
- **`process_applicants.py` now supports `--dry-run`** — full classification and logging with
  zero sheet writes, group changes, or emails — used to safely preview the backlog-sized first
  run before it went live.

## 4. The systems this touches

| System | Role |
|---|---|
| **SEAL Applicants** (Google Sheet) | Intake form responses; "Current Applicants" → "Approved"/"Rejected" |
| **SEAL Applicant Challenge** (Google Sheet, `1tVHLoybyghVJo5w93UmeG5dBSpHqMSeq7ce1t7hnBYo`) | Tracks onboarding progress (stages) before someone becomes a full Associate |
| **SEAL Clan Life** (Google Sheet, `1k19sS9NfwlVfG7GCCf18LO69pr4reSOTN6v1lY2nykQ`) | The live roster — "Associates" tab is the authoritative list of current active members |
| **Clan Life AAD** (Google Sheet, `1HJmG3VZs0Z-r-aU0hBFg3kjazYimtkJP3tuWky1JRyM`) | Departure records — Ex-Communicado, Ex-Associate, Alumni, Affiliates, Expiry Tracker |
| **Automation Log** (Google Sheet, `1KS6JXbZVw3sSOQV17iNTtWdALLLueWJIHWeA5jCdX7Y`) | Every script's run log + error log, written by every stage |
| **Google Groups** | `seal-active@maxalton.com` (active members), `sandbox@maxalton.com` (sandboxed members), `onboarding@maxalton.com` (in-progress applicants) — membership is added/removed as people move through the pipeline |
| **Slack workspace** (`sealuw.slack.com`) | Access is invited/reactivated/deactivated in step with Associates membership. No official reliable admin API exists for this on the current plan, so most of the invite/reactivate/deactivate logic drives the Slack **web admin panel** via Playwright browser automation, with an API attempt tried first where possible |
| **Gmail** (`admin@maxalton.com` and a dedicated send-only credential) | Sends approval/waitlist/rejection emails to applicants, the one-time Alumni Discord-invite email, and (as of 2026-08-11) the one-time GameOver/Ex-Communicado removal-notice email |

## 5. Where things live on disk

```
run_all.sh              the hourly orchestrator — read this first for the real run order
directives/*.md         one SOP per script — the "why" and the incident history
execution/*.py          the actual scripts run_all.sh calls
config.yaml             every sheet ID, tab name, group email, column index — no code
                         changes needed to retune most behavior
.env                    secrets (API tokens, Slack credentials) — never in Dropbox, never in git
credentials.json /
token_*.json            cached OAuth tokens per Google/Slack credential in use
.tmp/                   logs and intermediate state — always safe to delete/regenerate
```

For a **technical, auto-generated dependency map** (every file, every read/write, every
external system, as a diagram) see [`docs/ONBOARDING_AUTOMATION.md`](docs/ONBOARDING_AUTOMATION.md).
That document regenerates itself automatically whenever the code changes (a hash-guarded
build step, `characterize/`) and is the ground truth for exact file-level wiring. **This
file (`SYSTEM_OVERVIEW.md`) is the human-readable companion to it** — read this one first
for "what does this do and why," go to the generated one for "exactly which file touches
which cell."

## 6. Current status: GitHub is the only source of truth for this project's code

This project's code lives **only** in a real GitHub repo
(`seal-scripting/Seal-Onboarding-Automation`), pushed to via an HTTPS remote with a
fine-grained Personal Access Token embedded in the remote URL (Contents: Read/write,
scoped to this repo only — regenerate at
`https://github.com/settings/tokens?type=beta` if it ever expires). Hosted under the
`seal-scripting` account rather than a personal one specifically so no single person is
a bottleneck for repo/editor access.

**This project's code has no presence in Dropbox at all.** A single source of truth for
code was a deliberate choice, not an oversight — if you're a Claude session and you find
scripts, config, or credentials mirrored in a Dropbox folder for this project, that's
stale/wrong, and you should come back to this GitHub repo instead. The one deliberate
exception is *this document itself* (not code): a read-only copy also lives in Dropbox,
specifically so a session with no GitHub credentials or working URL-fetch tool can still
read it — see the maintenance rule above for the exact path.

Mechanically, it's just `git pull` — no LLM, no external service in the loop at all:
- **`execution/git_pull_check.py` pulls once/day, on its own dedicated cron line at 4am
  PST (as of 2026-08-03).** `run_all.sh` (hourly) and `monitor_fx_associates.py` (daily
  6:35) no longer touch git at all — they always run on whatever code is currently
  checked out. This replaced an earlier design where `run_all.sh` pulled first and
  aborted the entire hourly cycle on a pull failure; see the incident below for why that
  changed. A failed daily pull still emails `harrisnakajima@gmail.com` via
  `error_notify.py`, and the repo just stays on yesterday's code until the next
  successful pull — the hourly pipeline keeps running normally in the meantime (fail
  *open* on code staleness, not fail-closed on the whole pipeline).
- Secrets (`.env`, `credentials.json`, `token_*.json`) are gitignored and have never been
  pushed to GitHub — same discipline as before, just enforced by `.gitignore` instead of
  sync-tooling logic. Runtime/state files (`alumni_email_sent.json`,
  `onboarding_cleanup_processed.json`, log files) are gitignored too — they're
  regenerated, not source of truth.
- **Editing this project now means:** push a change to `master` on GitHub (or open a PR
  and merge it) — the next hourly/daily cron cycle picks it up automatically via the
  pull above. This repo is edited by technical staff directly; this file exists so that
  a non-technical reader (or a Claude session) can still understand what the system does
  without needing git or Python literacy.

## 7. Notable incidents worth knowing about (full detail in the directives)

These shaped why several parts of the code look the way they do — see the linked
directive's "Learnings" section for the complete account:

- **2026-06-06** — a sheet that reorders itself mid-run caused a deletion step to delete
  7 wrong (still-active) people instead of the intended departures. Fixed by resolving
  each deletion target by identity (email), re-checked immediately before each delete,
  instead of by row position captured earlier in the run. ([`process_clan_cleanup.md`](directives/process_clan_cleanup.md))
- **2026-07-03** — a Slack session-detection bug caused `process_clan_cleanup.py` and
  `process_slack_audit.py` to fight each other for 16+ hours, alternately deactivating and
  reactivating the same accounts. Fixed the detection bug and changed cleanup to delete
  each row immediately after its own work completes, rather than in one batch at the end.
- **2026-07-20** — the first version of the Alumni Discord-invite email scanned the
  *entire* Alumni tab every run instead of just newly-evicted rows, and mass-emailed 235
  people (65 of them twice, from an accidental concurrent run). Fixed by scoping the email
  strictly to rows evicted in the current run. **General lesson applied project-wide:**
  before wiring up any side-effecting step that iterates over a collection, check and log
  how many items it will actually touch — ideally via a dry run — before it goes live.
- **2026-08-01/02 — silent 57-hour pipeline blackout.** A machine-wide DNS resolution
  failure (`Could not resolve host`, affecting both `github.com` and
  `oauth2.googleapis.com` — this machine resolves DNS primarily through Tailscale's
  MagicDNS, root cause not yet found) meant `run_all.sh`'s git-pull-first, fail-closed
  preflight aborted every single hourly run from 2026-08-01 00:00 PDT through
  2026-08-03 09:00 PDT — **the entire pipeline didn't run for 57 hours.** Worse: the
  `error_notify.py` alert meant to warn about exactly this also failed silently every
  time, since sending it requires resolving `oauth2.googleapis.com` too — same root
  cause, so zero alerts went out for the whole blackout. It self-recovered and caught up
  on the backlog automatically (idempotent design), no data lost. Fixed by decoupling
  git pull into its own daily job (see section 6) so a GitHub-specific outage can no
  longer block the pipeline — note this specific incident was a *general* DNS failure,
  so this fix alone would not have prevented it; the Tailscale/DNS root cause is a
  separate open question.
- **2026-08-11/12 — a same-repo change silently downgraded a token shared with a
  different script.** The GameOver/Ex-Communicado removal-notice email added
  `process_clan_cleanup.py` code that reused `token_applicants.json` (already
  authorized for `spreadsheets` + `gmail.send`, since `process_applicants.py` needs
  both) but requested only `gmail.send`. `get_credentials()` rewrites a token file
  with whatever scope list it's called with on refresh, so this silently stripped
  the `spreadsheets` scope — `process_applicants.py` then 403'd
  (`ACCESS_TOKEN_SCOPE_INSUFFICIENT`) on every hourly run for ~31 hours before it
  was caught, even though `process_clan_cleanup.py` itself looked completely
  healthy the whole time (it doesn't need that scope). Fixed by always requesting
  the union of scopes any shared token needs, everywhere it's loaded, so no single
  caller can narrow what another caller depends on. **General lesson:** when two
  scripts share one OAuth token file, grep for every other place that token path is
  used before changing the scope list passed to `get_credentials()` for it.
- **2026-08-10 through 2026-08-24 — applicant review silently stalled for two weeks,
  zero errors logged.** See §3b above for the full fix. The root cause was a data/process
  gap, not a code bug — the classification logic did exactly what it was told (skip until
  a status is set); nothing was setting the status. Worth remembering generally: a script
  reporting "SUCCESS" every run with 0 errors does not mean it's accomplishing anything —
  check the actual outcome counts, not just the exit status. ([`process_applicants.md`](directives/process_applicants.md))
- **2026-08-31 — a run doing real work could vanish from the Run Log.** Found during a
  full-pipeline dry-run audit: `process_applicants.py`'s self-healing email backfill (see
  "Column O — Email Sent tracker" in its directive) can send real emails on a run that
  classifies zero new applicants — exactly what happened that afternoon, sending 13 real
  approval emails left over from the same-day Waitlist migration (§3a). But the Run Log
  summary only ever checked that cycle's classification counts, never whether the backfill
  actually sent anything, so the run's own audit-trail entry silently vanished as "no
  actions or errors" despite 13 real sends and 13 real column-O writes. Same class of blind
  spot as the row above, inverted: real work, zero record, instead of zero work, a clean-
  looking log. Fixed by tracking emails-sent separately from classification counts and
  always giving it a Run Log entry when non-zero. ([`process_applicants.md`](directives/process_applicants.md))

## 8. How to check on it right now

- **Is it running / did the last run succeed?** `.tmp/cron.log` (hourly pipeline) and
  `.tmp/aad_expiry.log` (daily job) have a timestamped entry for every run.
- **Did anything fail?** Failures trigger an email to `harrisnakajima@gmail.com` via
  `error_notify.py`; per-script logs are in `.tmp/process_<name>.log`.
- **What does a given script actually do, in detail?** Read its directive in
  `directives/` first — it's written for exactly this question.
- **Is a change safe to make?** Most tunable values (sheet IDs, tab names, column indices,
  group emails, email templates, thresholds) live in `config.yaml` and don't require code
  changes. Anything else should go through a script's `--dry-run` mode (most of the
  eviction-related scripts support one, and `process_applicants.py` gained one 2026-08-24)
  before a live run.
