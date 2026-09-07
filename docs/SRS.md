# Word Talent — Software Requirements Specification (SRS)

## Document Information

| Field | Value |
|---|---|
| Project | **Word Talent** — an English vocabulary trainer for middle- and high-school students |
| Document type | Software Requirements Specification (SRS) |
| Version | 0.3 (draft) |
| Status | For review |
| Owner | Project owner / lead teacher |
| Related systems | Prior "Daily Talent" project (same stack, **separate** system; optional future integration) |

### Revision history

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | — | Owner | Initial Korean draft |
| 0.2 | 2026-09-06 | — | Full English rewrite: restructured, added interface contract, precise SRS/points rules, acceptance criteria, quota analysis |
| 0.3 | 2026-09-06 | — | Simplified per owner: SM-2 replaced by a fixed 1/3/7-day interval ladder; single 50-item/day cap (reviews first); explicit study-card-before-test for new words; flat +1 Talent per correct answer; streak & perfect-day point bonuses removed (streak kept as a display counter) |

---

## 1. Introduction

### 1.1 Purpose

This document specifies the requirements for **Word Talent**, a mobile-first web application that helps Korean middle- and high-school students memorize English vocabulary. It combines **spaced repetition (SRS)** for long-term retention with a **reward-point ("Talent") economy** for motivation. The document is the reference for design, implementation, and acceptance testing of version 1 (v1).

### 1.2 Scope

Word Talent lets a student log in with a lightweight credential, study new words and clear a review queue in short sessions (about 50 items a day), earn one Talent point for every correct answer, and spend those points in a Shop on real-world rewards that a teacher fulfils offline. Teachers manage all content (words, policy settings, reward catalog) by editing a Google Sheet directly; no code deploy is required for content changes.

The entire system runs on **Google Sheets** (data + configuration) and **Google Apps Script** (backend logic + server-rendered front end via `HtmlService`). There is no external database, no external hosting, and no build pipeline.

### 1.3 Background and rationale

Rote repetition without scheduling wastes effort: students re-study words they already know and forget words they never revisit. Word Talent schedules each word per student on a **fixed expanding-interval ladder** — a word studied today is re-tested 1 day later, then 3 days after that, then 7 days after that, and is then considered mastered. This tracks the forgetting curve with no per-word tuning and is easy for a teacher to reason about. The "Talent" economy — a point currency named after the parable of the talents, already familiar to the target community — supplies extrinsic motivation during the weeks before intrinsic habit forms.

The stack is deliberately minimal so that a single teacher can own and operate the system: content lives in a spreadsheet they already know how to edit, and there is nothing to host or patch.

### 1.4 Goals and success metrics

| Goal | Metric (v1 target) |
|---|---|
| Students study consistently | ≥ 60% of enrolled students have a ≥ 3-day streak within the first 2 weeks |
| Reviews actually reduce forgetting | Review-item accuracy trends upward week over week for the median student |
| Zero-code content operations | Teacher adds/edits words and changes policy with **no** script deploy |
| Stable under classroom load | 40 concurrent students in a 10-minute window with no lost writes and p95 action latency ≤ 2.5 s (excluding cold start) |
| Low operational burden | Teacher spends ≤ 15 min/week on redemption approval and monitoring |

### 1.5 Out of scope (v1)

- Speech recognition or pronunciation scoring.
- Real-time multiplayer / head-to-head competition.
- Native mobile apps (a responsive web UI covers mobile).
- Automatic reward fulfilment or payment (redemption is a manual teacher step).
- Parent/guardian accounts and automated reports (listed under future enhancements).
- Cross-device account sync beyond what a PIN login provides.
- Integration with the prior "Daily Talent" system.

### 1.6 Definitions, acronyms, and glossary

| Term | Meaning |
|---|---|
| **Talent** | The in-app reward-point currency. "Points" and "Talent" are used interchangeably. |
| **SRS** | Spaced Repetition System — scheduling reviews at expanding intervals. |
| **Interval ladder** | The fixed sequence of gaps between reviews: `srs_intervals`, default `1, 3, 7` days. |
| **SRS step** | Index into the interval ladder for a (student, word) pair: `0` = just learned today, `1` = passed the day-1 review, `2` = passed the day-4 review, then **mastered**. |
| **Mastered** | A (student, word) pair that has passed every rung of the ladder; it leaves the review rotation and its `next_review_date` is blank. |
| **Interval** | Days until the next scheduled review of a (student, word) pair. |
| **Due item** | A `learning` (student, word) pair whose `next_review_date` is on or before "today". |
| **New item** | A word the student has never studied (no row in `UserWordProgress`). |
| **Study card** | The ungraded first exposure to a new word (word, meaning, example, audio) shown before its first test. Earns no Talent. |
| **Session** | One uninterrupted run through a queue of review items and/or new words. |
| **Streak** | Count of consecutive calendar days with ≥ 1 completed item. **Display only in v1** — it earns no bonus Talent. |
| **Attempt** | One answered test item within a session, producing one `ReviewLog` row. |
| **Quality (q)** | Integer 0–5 grading of an attempt, fed into the SRS update. |
| **Admin / Teacher** | The person who manages content in Sheets and approves redemptions. |
| **Web App** | The Apps Script deployment exposing `doGet`/`doPost` over HTTPS. |
| **Script timezone** | The timezone configured in the Apps Script project; defines "today". Default **Asia/Seoul**. |

### 1.7 References

- Leitner system / expanding-interval spacing (background for the fixed interval ladder).
- Google Apps Script quotas and limitations.
- Google Sheets service (`SpreadsheetApp`) behavior and batching guidance.
- Korea Personal Information Protection Act (PIPA) — informs §11.

---

## 2. Stakeholders and user roles

| Role | Description | Access |
|---|---|---|
| **Student** | Studies new words, clears review queue, earns Talent, redeems rewards, views own history. | Web App only, authenticated by name/ID + 4-digit PIN. |
| **Teacher / Admin** | Owns and edits the Google Sheet: words, `Settings`, reward catalog. Approves redemptions. Monitors progress via an admin view. | Editor on the Google Sheet; admin pages in the Web App gated by Google account. |
| **System (Apps Script)** | Scheduling, grading, point calculation, data-integrity enforcement, nightly batch jobs. | Runs as the deploying account. |

### 2.1 Authentication approach

Requiring a Google sign-in for every student is a poor fit for classroom use (shared devices, students without managed accounts). v1 uses a **lightweight credential**:

- The teacher pre-creates one `Users` row per student (name / student number, class, grade).
- On first login the student sets a **4-digit PIN**; thereafter they select their name and enter the PIN.
- PINs are stored **salted and hashed** (§11), never in clear text.
- Admin pages are separately protected by `Session.getEffectiveUser()` against an allow-list of teacher emails in `Settings`.

This is explicitly a **low-assurance** scheme suitable only for a closed school context. No sensitive personal data is stored (§11).

---

## 3. System overview

### 3.1 Context diagram

```
        edit content                     HTTPS (doGet / doPost)
Teacher ───────────────►  Google Sheet  ◄─────────────────────────►  Apps Script Web App  ◄──►  Student / Teacher browser
                          (data + config)     read / batched write        (HtmlService UI +          (mobile-first,
                                                                           SRS, grading, points,      responsive)
                                                                           LockService, CacheService,
                                                                           time-based triggers)
```

### 3.2 Technology stack

| Layer | Technology |
|---|---|
| Data & configuration | Google Sheets (one spreadsheet, multiple sheets/tabs) |
| Backend logic | Google Apps Script (V8 runtime) |
| Front end | Server-rendered HTML via `HtmlService` (`IFRAME` sandbox), vanilla JS + CSS; no framework, no bundler |
| Concurrency control | `LockService.getScriptLock()` |
| Caching | `CacheService` (script cache) + `PropertiesService` (script properties) for cache-version pointers |
| Scheduling | Time-based installable triggers |
| Client persistence | `localStorage` for the session token and last-used name |

### 3.3 Design principles

1. **Content is data, not code.** Every word, policy value, and reward is a spreadsheet row. Changing content never requires editing or redeploying script.
2. **Read rarely, cache aggressively.** Word list and settings are cached; a session should perform a small, bounded number of Sheet reads.
3. **Write in batches, under a lock.** Point mutations and progress updates take the script lock and use `setValues` batch writes. Client attempts carry an idempotency key so retries never double-award.
4. **Degrade visibly.** Cold starts and network drops surface as skeletons, spinners, and retry prompts — never as silent failure or a frozen screen.
5. **Stay inside the platform envelope.** No single execution approaches the 6-minute limit; heavy work (streak sweep, cache warm) runs in nightly triggers.

### 3.4 Deployment and environments

| Environment | Purpose | Notes |
|---|---|---|
| **Dev** | Development and testing | Separate copy of the spreadsheet; Web App deployed as "test deployment" (`/dev` URL). |
| **Prod** | Classroom use | Versioned deployment (`/exec` URL). Deploy runs as the **teacher/owner account**; access set to "Anyone" (students are not Google-authenticated) — this makes PIN auth and the teacher-email allow-list mandatory. |

Configuration that must not live in the sheet (teacher email allow-list bootstrap, script timezone) is set in the Apps Script project and/or Script Properties.

### 3.5 Platform constraints (must design within these)

Apps Script / Sheets limits vary by account type (consumer Gmail vs. Google Workspace). Design to the **consumer** tier and treat headroom on Workspace as a bonus.

| Constraint | Representative limit | Design implication |
|---|---|---|
| Script runtime per execution | 6 minutes | No synchronous bulk operation; paginate/queue. |
| Simultaneous executions | ~30 | Sessions must be short; the lock is held for milliseconds, not seconds. |
| Trigger total runtime per day | ~90 min (consumer) | Nightly batch must be O(students), bounded and resumable. |
| `CacheService` value size | 100 KB per key | Chunk the word list across keys; store a manifest. |
| `CacheService` max TTL | 6 hours | Word cache TTL ≤ 6 h; explicit version bump on content change. |
| `PropertiesService` | ~500 KB total, ~9 KB per value | Store only small pointers (cache version, schema version). |
| `LockService` wait | Bounded wait, then throw | Wait ≤ 30 s, then return a friendly "busy, retry" error. |
| Sheets calls | No hard per-day quota via `SpreadsheetApp`, but each call is slow (~0.1–1 s) | Batch: one `getValues` / `setValues` per region, never per-cell. |
| `UrlFetchApp` | ~20k calls/day | Not used in v1 core; audio URLs are played by the browser, not fetched by script. |

---

## 4. Data model (Google Sheets)

One spreadsheet contains the sheets below. Row 1 of every sheet is a header row with the exact column names given. The backend addresses columns **by header name**, not position, so teachers may reorder or add columns without breaking the app.

### 4.1 Conventions

- **IDs**: string, stable, never reused. Recommended format `w_000123` (words), `u_000042` (users), `rw_007` (rewards). Generated by the app on insert where the app inserts; teacher-entered IDs must be unique and non-empty.
- **Dates**: ISO `YYYY-MM-DD` **in script timezone**. "Today" is `Utilities.formatDate(new Date(), scriptTz, 'yyyy-MM-dd')`.
- **Timestamps**: ISO 8601 with offset, e.g. `2026-09-06T14:03:11+09:00`.
- **Booleans**: `Y` / `N` (case-insensitive on read; written as `Y`/`N`).
- **Empty cell** = "not set"; the backend applies documented defaults.
- **Enumerations** are validated on read; unknown values are logged and treated as the default.

### 4.2 `Words`

Master vocabulary list. Teacher-maintained.

| Column | Type | Required | Notes |
|---|---|---|---|
| `word_id` | ID | yes | Unique. |
| `word` | string | yes | The English headword. |
| `meaning` | string | yes | Korean gloss. Multiple senses separated by `;`. |
| `pos` | enum | no | Part of speech: `n`, `v`, `adj`, `adv`, `prep`, `conj`, `pron`, `phr`, `idiom`. |
| `example_sentence` | string | no | English example. |
| `example_translation` | string | no | Korean translation of the example. |
| `level` | enum | yes | Difficulty tag: `ms_basic`, `ms_core`, `hs_basic`, `hs_core`, `csat`. (Displayed with Korean labels.) |
| `grade_hint` | int 1–6 | no | Recommended grade: 1=중1 … 6=고3. Empty = "any". |
| `tags` | string | no | Comma-separated topic tags (e.g. `current-affairs,literature`). |
| `audio_url` | URL | no | Pronunciation audio; played by the browser `<audio>` element. |
| `active` | `Y`/`N` | no | `N` hides the word from all new-word selection. Default `Y`. |

**Ordering key** for new-word selection: `level` rank (`ms_basic` < `ms_core` < `hs_basic` < `hs_core` < `csat`), then `grade_hint` ascending (nulls last), then `word` ascending. See §5.5.

### 4.3 `Settings`

Key–value policy store. Teacher-maintained. The backend reads this (through cache) on every session and **never hard-codes** these values.

| `key` | Type | Default | Meaning |
|---|---|---|---|
| `daily_item_limit` | int | `50` | Max test items offered per student per day, **combined** across reviews and new words. Reviews are offered first; remaining slots go to new words. |
| `points_per_correct` | int | `1` | Talent awarded for every correct answer (`q ≥ 3`), for both the initial new-word test and scheduled reviews. Wrong answers and study cards award `0`. |
| `srs_intervals` | csv of ints | `1,3,7` | The interval ladder. After each correct review the gap advances to the next value; after the last value the word is `mastered`. |
| `srs_lapse_interval_days` | int | `1` | Days until re-test after a wrong answer (`q < 3`). |
| `srs_lapse_behavior` | enum | `retry` | `retry` = keep the current SRS step, re-test after `srs_lapse_interval_days`. `reset` = drop back to step 0 and re-ladder from the start. |
| `new_word_daily_cap` | int | `50` | Optional sub-cap on **brand-new** words per day (≤ `daily_item_limit`). Set lower to smooth the first week when there are no reviews yet. |
| `quiz_mode_default` | enum | `mixed` | `flashcard` \| `multiple_choice` \| `letter_tiles` \| `spelling` \| `mixed`. |
| `mc_slow_threshold_ms` | int | `8000` | Response time above which a correct MC / letter-tile answer scores q=4 instead of 5. |
| `letter_tiles_distractors` | int | `2` | Decoy letters shown alongside the word's own letters in `letter_tiles` mode (`0` = exact letters only). |
| `session_ttl_minutes` | int | `360` | Session token lifetime. |
| `pin_max_attempts` | int | `5` | Failed PIN attempts before lockout. |
| `pin_lockout_minutes` | int | `15` | Lockout duration after `pin_max_attempts`. |
| `ranking_enabled` | `Y`/`N` | `N` | Show the class ranking screen (§8.1 FR-S-11). |
| `timezone` | string | `Asia/Seoul` | Informational mirror of the script timezone (script setting is authoritative). |
| `teacher_emails` | string | — | Comma-separated allow-list for admin pages. |
| `cache_version` | int | `1` | Bumping this invalidates the word cache (also settable via admin action). |

Unknown keys are ignored. A missing key uses the default above.

### 4.4 `Users`

One row per student. Teacher pre-creates identity columns; the app fills the rest.

| Column | Type | Written by | Notes |
|---|---|---|---|
| `user_id` | ID | teacher or app | Unique. |
| `name` | string | teacher | Display name or student number. Unique within a class. |
| `class` | string | teacher | e.g. `2-3`. |
| `grade` | int 1–6 | teacher | Drives new-word selection. |
| `pin_hash` | string | app | Base64 SHA-256 of `pin_salt + pin`. Empty until first login. |
| `pin_salt` | string | app | Per-user random salt (UUID). |
| `pin_set_at` | timestamp | app | When the PIN was set. |
| `failed_attempts` | int | app | Consecutive failed PIN attempts. |
| `lockout_until` | timestamp | app | If in the future, login is refused. |
| `total_points` | int | app | Current Talent balance (earned − spent). Authoritative balance. |
| `lifetime_points` | int | app | Total ever earned (never decremented). |
| `current_streak` | int | app | Consecutive active days. Dashboard display only — earns no Talent in v1. |
| `longest_streak` | int | app | Max streak ever. Display only. |
| `last_active_date` | date | app | Last calendar day with ≥ 1 completed item. Drives the streak counter. |
| `created_at` | timestamp | teacher/app | — |
| `status` | enum | teacher | `active` \| `disabled`. `disabled` blocks login. Default `active`. |

`total_points` is the single source of truth for spendable balance; it is recomputed as `lifetime_points − SUM(RedemptionLog.points_spent for non-cancelled rows)` during the nightly job as a consistency check.

### 4.5 `UserWordProgress`

The SRS state table. One row per (student, word) the student has encountered. **This is the hot table** — nightly compaction keeps it sorted by `user_id` so a student's rows form a contiguous block.

| Column | Type | Notes |
|---|---|---|
| `user_id` | ID | — |
| `word_id` | ID | — |
| `status` | enum | `learning` \| `mastered`. `mastered` = passed the whole ladder; excluded from the review queue. |
| `srs_step` | int | Ladder index: `0` after the initial test, `1` after the day-1 review passes, `2` after the day-4 review passes. Passing the final step sets `status = mastered`. |
| `interval_days` | int | The gap (from `srs_intervals`) that produced the current `next_review_date`. Convenience/audit column; blank when mastered. |
| `next_review_date` | date | Due when ≤ today. Blank when `status = mastered`. |
| `last_result` | enum | `pass` \| `lapse`. |
| `last_quality` | int 0–5 | Last attempt's quality. |
| `last_reviewed_at` | timestamp | — |
| `first_learned_at` | timestamp | Set once, on the study card / first test. |
| `total_reviews` | int | Lifetime attempts for this pair. |
| `total_lapses` | int | Lifetime lapses; drives the "hard words" list (§8.1 FR-S-08). |

Composite key: (`user_id`, `word_id`). The app enforces uniqueness on insert.

### 4.6 `ReviewLog`

Append-only attempt history for analytics and idempotency.

| Column | Type | Notes |
|---|---|---|
| `log_id` | ID | App-generated. |
| `attempt_id` | UUID | **Client-generated idempotency key.** A repeat with the same `attempt_id` is a no-op that returns the original result. |
| `user_id` | ID | — |
| `word_id` | ID | — |
| `session_id` | UUID | Groups attempts in one session. |
| `timestamp` | timestamp | Server time. |
| `mode` | enum | `flashcard` \| `multiple_choice` \| `letter_tiles` \| `spelling`. |
| `item_type` | enum | `new` \| `review`. |
| `quality` | int 0–5 | Graded quality. |
| `result` | enum | `pass` (q ≥ 3) \| `fail` (q < 3). |
| `response_ms` | int | Client-measured response time. |
| `points_earned` | int | Talent awarded for this attempt: `points_per_correct` when `result = pass`, else `0`. |

### 4.7 `RewardCatalog`

Teacher-maintained store of redeemable items.

| Column | Type | Notes |
|---|---|---|
| `reward_id` | ID | Unique. |
| `name` | string | Shown in the shop. |
| `cost_points` | int | Talent price. |
| `stock` | int | Remaining units; `-1` = unlimited. |
| `active` | `Y`/`N` | `N` hides the item. |
| `description` | string | Optional detail. |
| `sort_order` | int | Ascending display order; ties broken by `name`. |

### 4.8 `RedemptionLog`

Redemption requests and fulfilment state.

| Column | Type | Notes |
|---|---|---|
| `log_id` | ID | App-generated. |
| `request_id` | UUID | Client idempotency key for the redeem action. |
| `user_id` | ID | — |
| `reward_id` | ID | — |
| `reward_name` | string | Snapshot at request time (catalog may change later). |
| `timestamp` | timestamp | Request time. |
| `points_spent` | int | Snapshot of `cost_points` at request time. |
| `status` | enum | `pending` \| `fulfilled` \| `cancelled`. |
| `decided_by` | string | Teacher email who fulfilled/cancelled. |
| `decided_at` | timestamp | — |
| `note` | string | Optional teacher note. |

**Cancellation** refunds `points_spent` to `Users.total_points` and, if `stock` ≥ 0, returns one unit to stock. This is the only way points are refunded.

### 4.9 `SchemaMeta` (app-managed, do not edit)

Single-row sheet holding `schema_version` and `last_nightly_run_at`. Used for migrations and to detect a missed nightly job.

---

## 5. Spaced repetition algorithm

### 5.1 Overview

Scheduling uses a **fixed expanding-interval ladder**, not SM-2. The ladder is `srs_intervals` (default `1, 3, 7`).

- A new word is first shown as an ungraded **study card**, then immediately given its **initial test**. Passing (or failing) the initial test creates the `UserWordProgress` row with `srs_step = 0` and schedules the first review for `today + srs_intervals[0]` (= tomorrow).
- Each later review that the student **passes** (`q ≥ 3`) advances `srs_step` by one and schedules the next review at `today + srs_intervals[srs_step]`. So a word learned on day 0 is reviewed on day 1, then day 4, then day 11.
- Passing the **final** rung sets `status = mastered`; the word leaves the review rotation and `next_review_date` is cleared.
- A **lapse** (`q < 3`) applies `srs_lapse_behavior`: `retry` (default) keeps `srs_step` and re-tests after `srs_lapse_interval_days` (tomorrow); `reset` sets `srs_step = 0` and re-ladders.

There is no ease factor and no per-word difficulty term. Every knob is a `Settings` value the teacher can change without a deploy.

### 5.2 Mapping an attempt to quality

The client reports `mode`, correctness details, and `response_ms`; the **server** computes `q` (never trust a client-supplied quality).

**Flashcard** (student self-reveals, then taps "I knew it" / "I didn't"):

| Outcome | q |
|---|---|
| "I knew it" | 5 |
| "I knew it" but `response_ms > mc_slow_threshold_ms` | 4 |
| "I didn't" | 2 |

**Multiple choice** (4 options, one retry allowed):

| Outcome | q |
|---|---|
| Correct on 1st try, `response_ms ≤ mc_slow_threshold_ms` | 5 |
| Correct on 1st try, `response_ms > mc_slow_threshold_ms` | 4 |
| Correct on 2nd try | 3 |
| Wrong on 2nd try (or gave up) | 2 |

**Letter tiles** (the prompt is the meaning; the student taps scrambled letter buttons to build the English word; `letter_tiles_distractors` decoy letters are mixed in; tiles can be removed and re-placed freely; one hint locks the next correct letter):

| Outcome | q |
|---|---|
| Correct word on the 1st submission, no hint, `response_ms ≤ mc_slow_threshold_ms` | 5 |
| Correct on the 1st submission, no hint, `response_ms > mc_slow_threshold_ms` | 4 |
| Correct after a wrong submission or after using the hint | 3 |
| Wrong on the final submission (or gave up) | 2 |

Letters are supplied, so this mode scaffolds recall (easier than free typing). See Q-06 on whether its ceiling should be capped at q=4.

**Spelling** (type the English word from the keyboard; one hint allowed):

| Outcome | q |
|---|---|
| Exact match | 5 |
| Match ignoring case / surrounding whitespace | 4 |
| Correct after using the hint, or within edit distance 1 of the answer | 3 |
| Otherwise | 1 |

**Mixed**: the engine picks a mode per item to ramp from recognition toward unassisted production. Default rotation — new word initial test: `multiple_choice` (the study card just before it already served the recognition exposure). Reviews: `multiple_choice` at `srs_step 0`, `letter_tiles` at `srs_step 1`, `spelling` at `srs_step 2`. Grading uses the chosen mode's table.

### 5.3 State update (pseudocode)

```
ladder = parseInts(S.srs_intervals)      # e.g. [1, 3, 7]

function initialTest(word_id, user_id, q, S):     # new word, after the study card
    p = newRow(user_id, word_id)
    p.status          = 'learning'
    p.srs_step        = 0
    p.first_learned_at = now()
    if q >= 3:
        p.interval_days    = ladder[0]
        p.next_review_date = today() + ladder[0]
        p.last_result      = 'pass'
    else:
        p.interval_days    = S.srs_lapse_interval_days
        p.next_review_date = today() + S.srs_lapse_interval_days
        p.last_result      = 'lapse'
        p.total_lapses    += 1
    p.last_quality = q; p.last_reviewed_at = now(); p.total_reviews = 1
    return p

function reviewUpdate(p, q, S):                   # existing learning row
    if q >= 3:                                    # PASS
        if p.srs_step >= len(ladder) - 1:
            p.status           = 'mastered'
            p.srs_step        += 1
            p.interval_days    = null
            p.next_review_date = null             # leaves the queue
        else:
            p.srs_step        += 1
            p.interval_days    = ladder[p.srs_step]
            p.next_review_date = today() + ladder[p.srs_step]
        p.last_result = 'pass'
    else:                                         # LAPSE
        if S.srs_lapse_behavior == 'reset':
            p.srs_step = 0
        p.interval_days    = S.srs_lapse_interval_days
        p.next_review_date = today() + S.srs_lapse_interval_days
        p.last_result      = 'lapse'
        p.total_lapses    += 1
    p.last_quality = q; p.last_reviewed_at = now(); p.total_reviews += 1
    return p
```

Notes:
- All date arithmetic is in the script timezone (§5.6).
- With the default ladder `[1, 3, 7]`: learn day 0 → review day 1 (`srs_step 0→1`) → review day 4 (`1→2`) → review day 11 (`2→mastered`).
- `mastered` rows are kept for stats (words learned, hard-words list) but are never queued for review and never earn Talent again.
- Changing `srs_intervals` later affects only reviews scheduled after the change; already-scheduled `next_review_date` values are not retroactively moved.

### 5.4 Building the daily queues

There is **one** daily budget: `daily_item_limit` (default 50) test items per student per day, counted from `ReviewLog` rows dated today. **Reviews are always served before new words.** Let `done_today` = items already completed today and `remaining = max(0, daily_item_limit − done_today)`.

1. **Review queue** = `learning` rows for this `user_id` with `next_review_date ≤ today`, ordered by `next_review_date` ascending, then `srs_step` ascending (earlier-stage words are more fragile), then `total_lapses` descending. Take up to `remaining`.
2. **New queue** = active words with **no** `UserWordProgress` row for this user, ordered per §5.5. Take up to `min(remaining − reviews_taken, new_word_daily_cap − new_done_today)`.
3. A session serves all queued reviews first, then new words. Each new word is delivered as a **study card followed by an initial test** (§8.1 FR-S-05). The student may scope a session to "review only" or "new only"; both remain bounded by `remaining`.
4. When `remaining = 0`, `startSession` returns no items (`done = true`).

### 5.5 New-word selection priority

Among active words the student has never seen, order by:

1. `grade_hint` is null **or** `grade_hint ≤ Users.grade` (words above the student's grade are deprioritised, not excluded — they sort last).
2. `level` rank ascending (`ms_basic` → `csat`).
3. `grade_hint` ascending (nulls last).
4. `word` ascending (stable tiebreak).

### 5.6 Timezone and daily rollover

- "Today" is computed in the **script timezone** (default `Asia/Seoul`), not the browser's.
- Day boundary is local midnight. A session that spans midnight keeps its already-issued items but counts completions against the day in which each attempt's server `timestamp` falls.
- `last_active_date` and the streak counter use script-timezone calendar days.

---

## 6. Talent (points) system

### 6.1 Earning

One flat rule: **+`points_per_correct` Talent (default 1) for every correct answer** (`q ≥ 3`).

| Event | Award |
|---|---|
| Initial test of a new word, `q ≥ 3` | `points_per_correct` |
| Scheduled review, `q ≥ 3` | `points_per_correct` |
| Any answer with `q < 3` | `0` — a lapse never deducts points |
| Study card (ungraded first exposure) | `0` — it is not an attempt |
| Any attempt on a `mastered` word | not possible — mastered words are never queued |

Per-attempt points are written to `ReviewLog.points_earned` and added to `Users.total_points` and `Users.lifetime_points` **in the same locked write** as the progress update.

There are **no** streak or perfect-day point bonuses in v1. The streak counter (§6.2) is displayed for motivation but carries no Talent.

### 6.2 Streak counter (display only)

Maintained for the dashboard; **awards nothing**. On the student's first completed item of a calendar day:

- If today is exactly `last_active_date + 1 day`: `current_streak += 1`.
- Else if today `> last_active_date + 1 day`: `current_streak = 1`.
- `longest_streak = max(longest_streak, current_streak)`; `last_active_date = today`.

The nightly job (FR-SYS-06) resets `current_streak` to `0` for any student whose `last_active_date` is more than one day in the past, so a stale streak never shows.

### 6.3 Spending — redemption workflow

```
Student picks a reward
   → server (under lock):
        validate reward.active == Y
        validate reward.stock == -1 OR reward.stock > 0
        validate Users.total_points >= reward.cost_points
        validate request_id not already used   # idempotency
        → deduct cost_points from Users.total_points
        → if stock >= 0: stock -= 1
        → append RedemptionLog row status='pending' (snapshot name + cost)
        → return success + new balance
Teacher (admin page or sheet):
        sets status='fulfilled' after handing over the item
        OR status='cancelled' → server refunds points + restores stock
```

- Redemption is **never** auto-fulfilled; a real item changes hands offline.
- A `pending` request already debited the student's balance (so they cannot double-spend); cancellation is the refund path.
- The student's shop view shows `pending` requests and their status.

### 6.4 Idempotency and anti-abuse

| Concern | Mitigation |
|---|---|
| Network retry double-submits an attempt | `attempt_id` unique check in `ReviewLog`; duplicate returns the stored result. |
| Network retry double-submits a redemption | `request_id` unique check in `RedemptionLog`. |
| Farming points by repeating the same word | A word is only queued on its scheduled ladder days (plus lapses); once `mastered` it is never queued again, so it cannot be re-earned. |
| Rapid-fire guessing | Points require `q ≥ 3`; MC wrong-then-right caps at q=3; spelling/letter-tiles require a real match. |
| Concurrent sessions on two devices | All point/progress writes serialize on the script lock; balance is read inside the lock. |
| Clock manipulation on the client | Server timestamps and server-computed quality; client `response_ms` only nudges q between 4 and 5. |

---

## 7. External interface requirements

### 7.1 Web App routing — `doGet(e)`

Serves HTML pages. `e.parameter.page` selects the view; unknown or missing → `home` (student) or the login gate.

| `page` | View | Auth |
|---|---|---|
| _(none)_ / `login` | Name picker + PIN entry (or PIN set on first login) | none |
| `home` | Student dashboard | student session |
| `study` | Session runner (new + review) | student session |
| `shop` | Reward catalog + redemption | student session |
| `me` | History, hard-words list, streak | student session |
| `ranking` | Class ranking (only if `ranking_enabled = Y`) | student session |
| `admin` | Admin dashboard (student progress, redemption queue) | teacher email allow-list |

The page shell loads immediately with a skeleton; data arrives via a `doPost` call from the page's JS (keeps `doGet` fast and cacheable).

### 7.2 Action API — `doPost(e)`

Single endpoint. Body is JSON: `{ action, token, payload }`. Response is JSON: `{ ok, data?, error? }` where `error = { code, message, retryable }`.

| `action` | Payload | Response `data` | Notes |
|---|---|---|---|
| `login` | `{ name, pin }` | `{ token, user: {name, grade, class, total_points, current_streak} }` | Rejects on lockout; increments `failed_attempts`. |
| `setPin` | `{ name, pin }` | `{ token, user }` | Only if `pin_hash` empty. |
| `getHome` | `{}` | `{ items_remaining, review_due, new_available, total_points, current_streak, longest_streak, pending_redemptions }` | `items_remaining = max(0, daily_item_limit − done_today)`; `review_due`/`new_available` are the un-capped counts. Cheap: one `UserWordProgress` block read + counts. |
| `startSession` | `{ scope: 'mixed'\|'new'\|'review' }` | `{ session_id, items: [Item...], done: bool }` | Items batched (≤ 20); more via `nextBatch`. `done = true` with no items when the daily budget is spent. |
| `nextBatch` | `{ session_id }` | `{ items: [Item...], done: bool }` | — |
| `submitAttempt` | `{ session_id, attempt_id, word_id, mode, item_type, correctness, response_ms }` | `{ quality, result, points_earned, new_balance, srs_step, status, next_review_date }` | `item_type` is `new` (initial test) or `review`. `next_review_date` is `null` when `status = 'mastered'`. Idempotent on `attempt_id`; server computes quality. |
| `endSession` | `{ session_id }` | `{ attempted, correct, points_earned, mastered_count, reviews_due_next }` | Session summary. `points_earned` equals `correct` (× `points_per_correct`). |
| `getShop` | `{}` | `{ balance, rewards: [Reward...], pending: [Redemption...] }` | — |
| `redeem` | `{ request_id, reward_id }` | `{ ok, new_balance, redemption }` | Idempotent on `request_id`. |
| `getHistory` | `{}` | `{ words_learned, accuracy_7d, hard_words: [...], recent_sessions: [...] }` | — |
| `getRanking` | `{}` | `{ rows: [{rank, name_masked, points_or_count}] }` | Only if enabled; names partially masked. |
| `adminOverview` | `{ class? }` | `{ students: [...], redemption_queue: [...] }` | Teacher only. |
| `adminDecideRedemption` | `{ log_id, decision: 'fulfilled'\|'cancelled', note? }` | `{ ok }` | Teacher only; cancellation refunds. |
| `adminRefreshCache` | `{}` | `{ ok, cache_version }` | Teacher only; bumps `cache_version`. |

**`Item`** = `{ word_id, item_type, mode, prompt, options?, letters?, audio_url?, study? }`:
- `item_type` — `new` or `review`.
- `mode` — the test mode for this item (`flashcard` \| `multiple_choice` \| `letter_tiles` \| `spelling`).
- `prompt` — the English word for `flashcard`/`spelling` (meaning hidden); the Korean meaning for `multiple_choice`/`letter_tiles` ("which word means…").
- `options` — 4 choices, only for `multiple_choice`.
- `letters` — the word's letters plus `letter_tiles_distractors` decoys, shuffled; only for `letter_tiles`.
- `study` — present **only when `item_type = new`**: `{ word, meaning, pos, example_sentence, example_translation, audio_url }`. The client shows this as an ungraded study card first, then the test. One `submitAttempt` is sent per item (the study card is not reported).

### 7.3 Error model

| `code` | Meaning | `retryable` | Client behavior |
|---|---|---|---|
| `AUTH_REQUIRED` | Missing/expired token | no | Redirect to login. |
| `AUTH_LOCKED` | PIN lockout active | no | Show lockout message + minutes remaining. |
| `RATE_BUSY` | Could not acquire lock in time | yes | Backoff + retry (≤ 3×) with the same idempotency key. |
| `VALIDATION` | Bad payload | no | Show inline error. |
| `INSUFFICIENT_POINTS` | Redeem cost > balance | no | Disable button, show needed amount. |
| `OUT_OF_STOCK` | Reward stock 0 | no | Mark item unavailable. |
| `NOT_FOUND` | Unknown id | no | Refresh the list. |
| `FORBIDDEN` | Non-teacher hit an admin action | no | — |
| `SERVER` | Unexpected | yes | Generic retry prompt; log for the teacher. |

### 7.4 Sessions

- Token = `Utilities.getUuid()`, stored in `CacheService` under `session:<token>` → `{ user_id, issued_at, last_seen }` with TTL `session_ttl_minutes`; each authenticated action refreshes `last_seen` and extends the TTL (sliding window).
- Client stores the token in `localStorage`; there is no server-side session sheet (cache loss just forces re-login, which is cheap).
- Logout clears the cache key and `localStorage`.

---

## 8. Functional requirements

IDs: `FR-S-*` student, `FR-A-*` admin/teacher, `FR-SYS-*` system/backend. Each requirement is testable (see §14).

### 8.1 Student-facing

| ID | Requirement |
|---|---|
| FR-S-01 | The student logs in by selecting their name and entering a 4-digit PIN. On first login (empty `pin_hash`) the student sets the PIN (entered twice, must match). |
| FR-S-02 | After `pin_max_attempts` consecutive wrong PINs, login is refused for `pin_lockout_minutes`; the UI shows remaining lockout time. |
| FR-S-03 | The home dashboard shows: items remaining today (against `daily_item_limit`), reviews due, new words available, current Talent balance, current streak, longest streak, and any pending redemptions. |
| FR-S-04 | The student can start a session scoped to "mixed", "new only", or "review only". Every scope is bounded by the remaining daily budget. |
| FR-S-05 | The session runner presents items one at a time. **A new word is always presented as a study card first** (word, meaning, part of speech, example sentence + translation, audio) with a "test me" control, then its initial test. **Review items go straight to the test.** Test modes: flashcard (self-reveal), 4-option multiple choice (pick the word from a list), letter tiles (tap scrambled letter buttons to build the word), spelling (type the word); the mode follows `quiz_mode_default` or the mixed-mode rotation. Letter-tile and spelling modes offer one hint. |
| FR-S-06 | On each answered test the student sees immediate feedback: correct/incorrect, the correct meaning, and the example sentence + translation when present. Audio plays on demand when `audio_url` is set. |
| FR-S-07 | After a session the student sees a summary: items attempted, accuracy, Talent earned (= number of correct answers), words mastered this session, and the number of reviews due next. |
| FR-S-08 | "My record" shows total distinct words learned, words mastered, 7-day accuracy trend, and a "hard words" list ranked by `total_lapses` (then most recent `last_result = lapse`). |
| FR-S-09 | The shop lists active rewards the student can afford (and those they cannot, greyed with the shortfall). Redeeming debits the balance immediately and creates a `pending` request. |
| FR-S-10 | The student can view the status of their past redemptions (`pending` / `fulfilled` / `cancelled`). |
| FR-S-11 | When `ranking_enabled = Y`, the student can view a class ranking by Talent earned this period and by items completed; other students' names are partially masked (e.g. `김○○`). When disabled, the ranking page is not reachable. |
| FR-S-12 | The total test items offered per day never exceed `daily_item_limit` (default 50), combined across reviews and new words; reviews are offered before new words; brand-new words additionally never exceed `new_word_daily_cap`. Completed items reduce the remaining count. |
| FR-S-13 | The UI shows a skeleton/spinner during loads and, on a failed request, a clear retry affordance; no action silently fails. |
| FR-S-14 | The student can log out; the session token is invalidated. |
| FR-S-15 | The study card is ungraded and awards no Talent; only the test that follows it can. The student cannot skip straight past the study card to the test result (the test is a separate step). |
| FR-S-16 | The Shop is reachable as its own tab from the home dashboard at all times (not only after a session). |

### 8.2 Admin / teacher

| ID | Requirement |
|---|---|
| FR-A-01 | The teacher adds/edits/deactivates words by editing the `Words` sheet, with no code change or redeploy. Changes take effect within the word-cache TTL or immediately after `adminRefreshCache`. |
| FR-A-02 | The teacher changes any policy value by editing the `Settings` sheet; the new value takes effect within the settings-cache TTL (≤ 1 h) or on cache refresh. |
| FR-A-03 | The teacher manages rewards (add, price, stock, activate/deactivate, order) by editing `RewardCatalog`. |
| FR-A-04 | The admin dashboard lists pending redemptions; the teacher marks each `fulfilled` or `cancelled` (cancellation refunds points and restores stock). The same can be done by editing `RedemptionLog.status` directly. |
| FR-A-05 | The admin dashboard shows per-student progress: distinct words learned, due count, 7-day accuracy, current streak, Talent balance — filterable by class. |
| FR-A-06 | Admin pages are accessible only to Google accounts whose email is in `Settings.teacher_emails`; others get `FORBIDDEN`. |
| FR-A-07 | The teacher can trigger a cache refresh (`adminRefreshCache`) to force-publish content/policy changes. |

### 8.3 System / backend

| ID | Requirement |
|---|---|
| FR-SYS-01 | On every session the backend reads `Settings` and the word list **through `CacheService`**, not raw Sheet reads, except on a cache miss or version change. |
| FR-SYS-02 | Every mutation of `Users.total_points`, `UserWordProgress`, `ReviewLog`, `RewardCatalog.stock`, or `RedemptionLog` executes inside `LockService.getScriptLock()` with a ≤ 30 s wait; on timeout the action returns `RATE_BUSY`. |
| FR-SYS-03 | `submitAttempt` is idempotent on `attempt_id`; `redeem` is idempotent on `request_id`. A duplicate returns the original result without a second side effect. |
| FR-SYS-04 | Quality `q` is computed on the server from mode + correctness + `response_ms`; a client-supplied quality is ignored. |
| FR-SYS-05 | Point awards follow §6.1 exactly: a flat `points_per_correct` for every attempt with `q ≥ 3` (initial new-word test and scheduled reviews alike); study cards, lapses, and mastered words award nothing. No streak or perfect-day bonuses. |
| FR-SYS-06 | A time-based trigger runs nightly to: reset `current_streak` to 0 for students inactive for more than one day, warm the word cache, compact `UserWordProgress` sort order, and reconcile `total_points` against the ledger. The job is bounded (O(students)) and resumable within the trigger runtime budget. |
| FR-SYS-07 | All Sheet writes for a region use a single batched `setValues`; no per-cell writes in request paths. |
| FR-SYS-08 | "Today" and all calendar-day logic use the script timezone. |
| FR-SYS-09 | PINs are stored only as `base64(SHA-256(pin_salt + pin))` with a per-user random `pin_salt`; the raw PIN is never written to any sheet, log, or cache. |
| FR-SYS-10 | Malformed enum/date/number cells are logged and treated as the documented default rather than throwing. |
| FR-SYS-11 | If the nightly job did not run (`SchemaMeta.last_nightly_run_at` older than ~26 h), the next student session performs a lightweight inline streak check for that student. |
| FR-SYS-12 | Review scheduling uses the fixed ladder `srs_intervals` (default `1,3,7`): a passed review advances `srs_step` by one and sets `next_review_date = today + srs_intervals[srs_step]`; passing the final rung sets `status = 'mastered'` and clears `next_review_date`; a lapse applies `srs_lapse_behavior`. `mastered` rows are never placed in the review queue. |
| FR-SYS-13 | A new word enters the schedule only after its study card **and** initial test are completed; the study card alone creates no `UserWordProgress` row and no `ReviewLog` row. |

---

## 9. Non-functional requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-01 | Performance | Excluding cold start, p95 latency ≤ 2.5 s for `getHome`, `startSession`, `submitAttempt`; p95 ≤ 4 s for `adminOverview`. |
| NFR-02 | Performance | A full session (up to `daily_item_limit` ≈ 50 items) performs ≤ 6 Sheet read operations total and ≤ 1 batched write per attempt. |
| NFR-03 | Scalability | Correct behavior with 300 students, 5,000 words, and 200k `ReviewLog` rows. `ReviewLog`/`RedemptionLog` are archived per school year to a dated sheet when they exceed ~150k rows. |
| NFR-04 | Concurrency | 40 concurrent students over 10 minutes: no lost point writes, no double awards, no corrupted `UserWordProgress` rows. |
| NFR-05 | Reliability | Any request may be safely retried; idempotency keys guarantee at-most-once side effects. |
| NFR-06 | Availability | Depends on Google infrastructure; the app adds no single point of failure beyond the spreadsheet. Cold-start slowness is masked by skeleton UI (NFR-08). |
| NFR-07 | Usability | Mobile-first responsive layout; usable one-handed on a 360 px-wide viewport; tap targets ≥ 44 px. |
| NFR-08 | Usability | First contentful paint of the page shell ≤ 1 s after HTML arrives; data panels show skeletons until loaded. |
| NFR-09 | Security / privacy | See §11. No real names required; PIN hashed; sheet shared only with teacher editors; admin actions gated by email allow-list. |
| NFR-10 | Maintainability | Backend organized into modules with single responsibilities (`auth`, `srs`, `points`, `sheets`, `cache`, `sessionApi`, `admin`, `nightly`); each independently testable with a mockable Sheets layer. |
| NFR-11 | Maintainability | No policy constant is hard-coded; all live in `Settings`. Column access is by header name. |
| NFR-12 | Observability | Errors and quota exceptions are written to a `SystemLog` sheet (capped, ring-buffer) with timestamp, action, user_id (if known), and message, so the teacher can diagnose without opening the script editor. |
| NFR-13 | Portability | The spreadsheet can be copied to create a new class instance; only `teacher_emails` and the timezone need re-setting. |
| NFR-14 | Compliance | Data handling conforms to §11 and avoids storing sensitive personal information (PIPA "고유식별정보" and "민감정보" excluded). |
| NFR-15 | Offline/degraded | On network failure the client shows a non-blocking banner and offers retry; a session in progress is not lost (unsent attempts are queued in memory and retried). |

---

## 10. UX and screen flow

```
                 ┌─────────────────────────┐
                 │ Login (name + PIN)      │  first login → set PIN
                 └───────────┬─────────────┘
                             ▼
                 ┌─────────────────────────┐
                 │ Home dashboard          │  items remaining (≤50) · Talent · streak
                 └───┬───────────────┬─────┘
        start study  │               │ Shop tab / My record
                     ▼               ▼
     ┌────────────────────────────┐   ┌──────────────────┐
     │ Session runner             │   │ Shop  ── redeem ─┼──► pending
     │ reviews first, then new    │   ├──────────────────┤
     │ new word: study card ──►   │   │ My record        │
     │   initial test ──► feedback│   │  words mastered, │
     │ review: test ──► feedback  │   │  hard words,     │
     └──────────┬─────────────────┘   │  accuracy, streak│
                ▼                     └──────────────────┘
     ┌────────────────────────────┐
     │ Session summary            │
     │ accuracy · Talent earned · │
     │ mastered · reviews due next│
     └────────────────────────────┘

     (Admin, teacher only): Admin dashboard → student progress · redemption queue → decide
```

Key UX rules: one primary action per screen; the session runner never blocks on network (queue + retry); every list refreshes on pull/again; cold start shows a branded skeleton, not a blank page.

---

## 11. Security and privacy

| Area | Requirement |
|---|---|
| Identifiers | Prefer student number or nickname over full legal name. `Users.name` is display-only. |
| Credentials | 4-digit PIN, stored as `base64(SHA-256(salt + pin))` with a per-user UUID salt. Raw PIN never persisted or logged. |
| Brute force | `pin_max_attempts` failures → `pin_lockout_minutes` lockout, tracked in `Users`. |
| Transport | HTTPS only (Apps Script Web App default). |
| Authorization | Student actions require a valid session token. Admin actions require the caller's Google email to be in `Settings.teacher_emails`; this is checked server-side on every admin action, not just at page load. |
| Sheet access | The spreadsheet is shared only with teacher editors. Students never have Sheet access; they only reach the Web App. |
| Data minimization | No addresses, phone numbers, resident registration numbers, health, or other sensitive data. Only: display name, class, grade, learning stats, PIN hash. |
| Logs | `ReviewLog` / `SystemLog` contain no credential material. `SystemLog` is a capped ring buffer. |
| Retention | Learning logs retained for the school year, then archived or deleted at the teacher's discretion. A student can be set `status = disabled` to block access without deleting history. |
| Deployment identity | The Web App runs as the teacher/owner account; students are anonymous to Google. This is acceptable only because no sensitive data is handled and the context is a closed classroom. |

The PIN scheme is **explicitly low-assurance**. It is acceptable for a closed school deployment with no sensitive data and must not be reused for anything else.

---

## 12. Constraints, assumptions, dependencies

### 12.1 Constraints

- Backend is Google Apps Script; front end is `HtmlService`. No external servers, databases, or npm build step.
- Must operate within the quotas in §3.5.
- Content and policy are edited in Google Sheets by non-programmers.

### 12.2 Assumptions

- Most students use a smartphone browser.
- A single spreadsheet per class (or per teacher) is sufficient for v1 scale (§NFR-03).
- The teacher can manage Google Sheet sharing and a one-time Apps Script deployment.
- Students are online while studying; brief drops are tolerated (§NFR-15) but true offline is not supported.
- The class roster is known in advance and entered by the teacher.

### 12.3 Dependencies

- Google Workspace / Google account with Apps Script and Sheets enabled.
- Browser support for `localStorage`, `fetch`, and `<audio>` (any current mobile browser).

---

## 13. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Cold-start latency on first hit of the day | Student sees a slow blank screen | Skeleton UI; keep `doGet` payload minimal; optional nightly "warm-up" trigger that pings `getHome`. |
| Sheets slowness / contention under concurrent load | Timeouts, `RATE_BUSY` | Cache reads; short critical sections; batched writes; exponential backoff with idempotency keys; interleave nightly heavy work off-peak. |
| Trigger daily runtime budget exceeded | Nightly job doesn't finish | Bounded O(students) work; resumable cursor in `SchemaMeta`; inline per-student fallback (FR-SYS-11). |
| PIN guessing / shared devices | Account takeover within a class | Lockout policy; no sensitive data at stake; teacher can reset a PIN by clearing `pin_hash`. |
| Teacher edits break a sheet (renamed header, bad enum) | App errors for everyone | Access columns by header name; validate + default on read (FR-SYS-10); `SystemLog` surfaces problems. |
| `ReviewLog` growth slows reads | Latency creep over a semester | Only read `UserWordProgress` (small per student) in request paths; archive logs per §NFR-03. |
| Points economy imbalance (rewards too cheap/expensive) | Motivation flattens or rewards run out | Earning is a fixed +1/correct; teacher tunes `RewardCatalog` prices without a deploy; `lifetime_points` vs `total_points` lets the teacher audit. |
| Day-1 load: no reviews exist yet, so up to `daily_item_limit` items are all brand-new words + study cards | A heavy, monotonous first session | `new_word_daily_cap` lets the teacher throttle new words for the first week; study-card→test pacing breaks up the effort; the load self-balances once reviews accumulate. |
| Ladder too short/long for a given cohort | Words forgotten after "mastered", or busywork | `srs_intervals` is a `Settings` value; teacher can set e.g. `1,3,7,16` or `2,5,10`; see Q-05 (final-check review). |
| `CacheService` eviction mid-day | Extra Sheet reads, brief slowness | Version pointer in Script Properties; graceful rebuild on miss. |
| Ranking screen causes social friction | Student stress | Off by default (`ranking_enabled = N`); names masked when on. |

---

## 14. Acceptance criteria and test scenarios

Representative, testable scenarios. Each maps to functional requirements.

| # | Scenario | Expected result | Covers |
|---|---|---|---|
| T-01 | New student logs in, sets PIN `1234` twice | Session issued; `pin_hash`/`pin_salt`/`pin_set_at` written; raw PIN nowhere in the sheet | FR-S-01, FR-SYS-09 |
| T-02 | Wrong PIN entered `pin_max_attempts` times, then correct PIN | 6th attempt refused with `AUTH_LOCKED` and minutes remaining; correct PIN works only after lockout expires | FR-S-02 |
| T-03 | Student with 0 progress starts a "new only" session | Each new word arrives with a `study` block; the client shows the study card, then the test; no review items; total items ≤ `min(daily_item_limit, new_word_daily_cap)`, ordered per §5.5 | FR-S-04, FR-S-05, FR-S-12, FR-SYS-13 |
| T-04 | Complete a new word's study card, then answer its initial test correctly and fast in MC mode | `quality = 5`, `result = pass`, `points_earned = points_per_correct` (1); `UserWordProgress` row created `status = 'learning'`, `srs_step = 0`, `interval_days = 1`, `next_review_date = today + 1`; the study card produced no `ReviewLog` row | FR-S-05, FR-SYS-04, FR-SYS-05, FR-SYS-13 |
| T-05 | Pass that word's review on day 1, then day 4, then day 11 (default ladder `1,3,7`) | Day 1: `srs_step = 1`, `next_review_date = +3` (day 4). Day 4: `srs_step = 2`, `next = +7` (day 11). Day 11: `status = 'mastered'`, `next_review_date` blank, word no longer offered for review. Each pass awards 1 Talent | §5.3, §6.1, FR-SYS-12 |
| T-06 | Lapse (`q ≤ 2`) on a `srs_step = 2` learning word, `srs_lapse_behavior = retry` | `srs_step` unchanged (2), `next_review_date = today + srs_lapse_interval_days` (today + 1), `total_lapses += 1`, 0 points | §5.3, §6.1 |
| T-06b | Same lapse with `srs_lapse_behavior = reset` | `srs_step = 0`, `next_review_date = today + 1`, word re-ladders from the start | §5.3 |
| T-07 | `submitAttempt` sent twice with the same `attempt_id` (simulated retry) | Exactly one `ReviewLog` row; identical response both times; balance changed once | FR-SYS-03 |
| T-08 | Study card shown for a new word, then the student closes the app before the test | No `UserWordProgress` row, no `ReviewLog` row, no Talent; the word is still "new" next session | FR-S-15, FR-SYS-13 |
| T-09 | Student completes ≥ 1 item on consecutive days, then skips 2 days | `current_streak` increments by 1 each active day (shown on dashboard); **no** Talent awarded for the streak; after the 2-day gap the nightly job sets `current_streak = 0` | §6.2, FR-SYS-06 |
| T-10 | Student who has already completed 50 items today opens home and starts a session | `items_remaining = 0`; `startSession` returns `done = true` with no items | §5.4, FR-S-12 |
| T-11 | Redeem a reward priced at exactly the current balance | Balance → 0; `RedemptionLog` row `pending`; stock decremented (if ≥ 0) | FR-S-09, §6.3 |
| T-12 | Redeem when cost > balance | `INSUFFICIENT_POINTS`; no ledger or stock change | §7.3 |
| T-13 | Two near-simultaneous redemptions of the last unit in stock | One succeeds, the other gets `OUT_OF_STOCK`; stock never goes negative | FR-SYS-02, §6.3 |
| T-14 | Teacher cancels a `pending` redemption | Status → `cancelled`; `points_spent` refunded to `total_points`; one unit returned to stock | FR-A-04, §6.3 |
| T-15 | Teacher adds a word, then calls `adminRefreshCache` | New word appears in new-word selection on the next session without a redeploy | FR-A-01, FR-A-07, FR-SYS-01 |
| T-16 | Teacher changes `points_per_correct` from 1 to 2 | Within the settings-cache TTL (or after refresh), every correct answer pays 2 | FR-A-02, FR-SYS-01 |
| T-16b | Teacher changes `srs_intervals` from `1,3,7` to `1,3,7,16` | Words already scheduled keep their dates; the next passed review of a `srs_step = 2` word now schedules `+16` instead of mastering it | §5.3, FR-A-02 |
| T-17 | Non-teacher Google account opens `?page=admin` or calls `adminOverview` | `FORBIDDEN`; no data returned | FR-A-06, §11 |
| T-18 | 40 simulated students each submit up to 50 attempts within a 10-minute window | No lost point writes; no double awards; no `RATE_BUSY` unresolved after ≤ 3 backoff retries | NFR-01, NFR-04, FR-SYS-02 |
| T-25 | Student has 30 reviews due and `daily_item_limit = 50` | The session serves all 30 reviews first, then at most 20 new words; a 31st item onward is new | §5.4, FR-S-12 |
| T-19 | Session request crosses local midnight | Items already issued remain; completions count toward the day of each attempt's server timestamp | §5.6 |
| T-20 | `Words` sheet has a row with `level = "banana"` | Row is treated as the default level, logged to `SystemLog`, app does not crash | FR-SYS-10, NFR-12 |
| T-21 | Nightly trigger skipped (simulated) | Next session runs an inline streak check for that student; `SchemaMeta` staleness detected | FR-SYS-11 |
| T-22 | Network fails mid-session on `submitAttempt` | Client shows retry banner; attempt is re-sent with the same `attempt_id` on reconnect; no data loss | FR-S-13, NFR-15 |
| T-23 | Review item served in `letter_tiles` mode; student builds the correct word on the first submission, no hint, quickly | `quality = 5` (or `4` if Q-06 caps the mode), `result = pass`, `points_earned = points_per_correct`, `srs_step` advances one rung; `ReviewLog.mode = 'letter_tiles'` | FR-S-05, FR-SYS-04 |
| T-24 | `letter_tiles` item with `letter_tiles_distractors = 2` | `Item.letters` contains the word's letters plus exactly 2 decoys, shuffled; submitting a word using a decoy letter is graded wrong | FR-S-05, §7.2 |

---

## 15. Release plan (v1)

| Milestone | Contents | Exit criteria |
|---|---|---|
| **M0 — Skeleton** | Sheet templates (all sheets + headers + sample data), Apps Script project, `doGet` shell, Sheets access module with header-name mapping, `CacheService` wrapper | Login page renders; word list loads from cache; unit tests for the Sheets/cache layer pass |
| **M1 — Auth + Home** | `login` / `setPin`, session tokens, lockout, `getHome` | T-01, T-02 pass; home dashboard shows real counts |
| **M2 — Study loop** | `startSession` / `nextBatch` / `submitAttempt` / `endSession`; study-card→test flow for new words; all four test modes; server-side quality; fixed-ladder SRS update; flat +1 points; idempotency; single 50-item daily budget | T-03–T-08, T-19, T-22, T-25 pass |
| **M3 — Shop** | `getShop` / `redeem`, redemption workflow, stock control, Shop tab, streak counter on the dashboard | T-09–T-14 pass |
| **M4 — Admin + nightly** | Admin dashboard, `adminDecideRedemption`, `adminRefreshCache`, nightly trigger (streak reset, cache warm, compaction, reconcile), `SystemLog` | T-09, T-15–T-17, T-21 pass; nightly job runs within budget on seeded data |
| **M5 — Hardening** | Load test, cold-start UX, error model polish, log archiving, docs for the teacher | T-18, T-20 pass; NFR-01/04 measured and met |

---

## 16. Future enhancements

- Additional item types: cloze (fill-in-the-blank from `example_sentence`), listening (audio → meaning).
- Weekly progress email to guardians / homeroom teacher (Gmail service + trigger).
- Auto-generate a word set: paste a textbook or mock-exam passage, extract high-frequency vocabulary (optional AI integration).
- Optional integration with the "Daily Talent" system (shared balance or SSO).
- Adaptive scheduling (SM-2 / FSRS) as an opt-in alternative to the fixed ladder, once there is enough per-student data.
- A final "mastered" check review (e.g. 30 days out) before a word is retired for good.
- Per-student adaptation of `daily_item_limit`.
- Teacher-authored custom quizzes / assignments with due dates.
- Streak or perfect-day bonuses reintroduced as an opt-in `Settings` toggle.

---

## 17. Open questions

| # | Question | Owner | Needed by |
|---|---|---|---|
| Q-01 | One spreadsheet per class or one per teacher for all classes? Affects `Users`/`UserWordProgress` size and admin filtering. | Owner | M0 |
| Q-02 | Default `new_word_daily_cap` — leave at 50 (a full first day of new words) or lower it (e.g. 20) so the first week is gentler? | Owner | M2 |
| Q-03 | Is a 4-digit PIN acceptable to the school, or is a 6-digit / name-only scheme preferred? | Owner + school | M1 |
| Q-04 | Retention period for `ReviewLog` after the school year — archive or delete? | Owner | M5 |
| Q-05 | Should a `mastered` word ever resurface for a final check (e.g. a 4th rung at ~30 days), or is 1/3/7 the whole lifecycle in v1? | Owner | M2 |
| Q-06 | Should `letter_tiles` cap at `q = 4` (letters are supplied — assisted recall), or reach `q = 5` like free typing? | Owner | M2 |
| Q-07 | Do we need a teacher-facing "adjust points / progress" manual override, or is editing the sheet enough? | Owner | M4 |
| Q-08 | Default `srs_lapse_behavior` — `retry` (gentle, keeps the step) as speced, or `reset` (stricter, re-ladders)? | Owner | M2 |
| Q-09 | Ranking period if `ranking_enabled` — rolling 7 days, calendar week, or all-time? | Owner | M3 |

---

## Appendix A — `Settings` quick reference

| key | default | unit |
|---|---|---|
| `daily_item_limit` | 50 | items/day (reviews + new, combined) |
| `new_word_daily_cap` | 50 | brand-new words/day |
| `points_per_correct` | 1 | Talent per correct answer |
| `srs_intervals` | 1,3,7 | days (interval ladder) |
| `srs_lapse_interval_days` | 1 | days |
| `srs_lapse_behavior` | retry | retry / reset |
| `quiz_mode_default` | mixed | enum |
| `mc_slow_threshold_ms` | 8000 | ms |
| `letter_tiles_distractors` | 2 | letters |
| `session_ttl_minutes` | 360 | minutes |
| `pin_max_attempts` | 5 | attempts |
| `pin_lockout_minutes` | 15 | minutes |
| `ranking_enabled` | N | Y/N |
| `timezone` | Asia/Seoul | IANA tz |
| `teacher_emails` | — | csv |
| `cache_version` | 1 | integer |
