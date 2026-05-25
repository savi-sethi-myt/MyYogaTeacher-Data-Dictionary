# MyYogaTeacher Data Dictionary

_Last updated: 2026-05-25_
_Owner: Savi Sethi (savi@myyogateacher.com)_
_For: SQL agent on LibreChat, querying ClickHouse_

This file documents the curated views in `myt_views.*`. For each view: purpose, grain, joins to other views, default filters already applied, and field reference.

## How the agent should use this document

- Prefer views in `myt_views.*` over raw `myt.*` tables.
- Each view has standard filters and timezone conversions baked in — **do NOT re-add them**.
- Field categories used below:
  - **Standard fields** — self-explanatory, use as-is.
  - **Business-defined fields** — need internal definition to use correctly.
  - **ML / inferred fields** — probabilistic, NOT user-entered; treat as estimates, not ground truth.
  - **Calculated fields** — already computed in the view definition.

---

## View Index

| View | Grain | Use for |
|---|---|---|
| `v_student` | one row per student | demographics, signup, trial info, attribution |
| `v_session` _(coming)_ | one row per (student, session) | class bookings, attendance, teacher info |
| `v_transaction` _(coming)_ | one row per student (first transaction only) | first-membership analysis |
| `v_transaction_history` _(coming)_ | one row per transaction | full payment history, upgrades/downgrades |

---

## VIEW: `myt_views.v_student`

### What it is
One row per registered student (customer). The primary view for any student-level question — demographics, signup attribution, trial status, contact details, and concierge-entered metadata.

### Grain
One row per `student_id` (also unique on `student_uuid`).

### Underlying tables
- `myt.myt_student` (main)
- `myt.myt_student_meta` (1:1 extension)
- `myt.myt_cms_user` (internal staff lookup)

All three are accessed with `FINAL` to return the latest version of each row (ReplacingMergeTree deduplication).

### Joins to other views
- `v_student.student_id = v_session.student_id`
- `v_student.student_id = v_transaction.student_id`
- `v_student.student_id = v_transaction_history.student_id`

### Default filters (ALREADY APPLIED — do NOT re-add)
- Excludes internal/test accounts: `email NOT LIKE '%@myyogateacher.com'` AND `email NOT LIKE '%@myt.com'`
- Excludes test users: `test_user <> 1`
- Filters to `client_domain = 'lifestyle'` (so every row in this view is lifestyle)
- All timestamps converted from UTC to `America/Los_Angeles` (US/Pacific)
- `FINAL` applied on all source tables for latest-row semantics

### When NOT to use this view
- If you need a non-lifestyle `client_domain` — query `myt.myt_student` directly (subject to read permissions).
- If you need a field not listed below (e.g., `phone_number_verified`, `has_attempted_to_refer`, `join_prediction_score`) — those exist on the source tables but are not currently exposed here. See "Notes & gotchas" below.

---

### Standard fields (self-explanatory)

| Field | Notes |
|---|---|
| `student_id` | Internal numeric ID. Primary join key for related views. |
| `student_uuid` | UUID. Alternative join key. |
| `student_email` | Student's email address. |
| `email_verified` | Whether the email has been verified. |
| `iana_timezone` | Student's local IANA timezone string (e.g., `America/New_York`, `Asia/Kolkata`). |
| `age` | Self-reported age. |
| `date_of_birth` | Self-reported DOB. |
| `gender` | Self-reported gender (free-text or controlled list). |
| `phone_personal` | Personal phone number. |
| `phone_no` | Phone number (alternate field). |
| `phone_cc` | Phone country code. |
| `country`, `city`, `state` | Self-reported location. |
| `ip_address` | IP at signup. |
| `user_agent` | Browser/device user-agent at signup. |
| `utm_medium`, `utm_campaign`, `utm_content`, `utm_source`, `utm_term` | Standard URL-based marketing attribution captured at signup. |
| `is_banned`, `banned_reason` | Account ban status and reason. |
| `zendesk_id` | Identifier linking this student to the Zendesk support system. |
| `onboarding_completed` | Whether the student finished onboarding. |
| `how_did_you_hear_about_us` | Self-reported attribution. Free text. |

---

### Business-defined fields

- **`yoga_level`** — Student's self-reported experience level. Common values: `beginner`, `intermediate`, `advanced`. _[Confirm exact value set.]_
- **`health_history`** — Student-entered health background (free text).
- **`goals`** — Student-entered goals (free text or tag list).
- **`class_preference`** — Type/style of classes the student prefers.
- **`primary_goal`** — Student's main reason for joining (controlled list). _[Confirm exact value set.]_
- **`preferred_practice_timebands`** — Time-of-day buckets the student prefers to practice (e.g., morning / afternoon / evening).
- **`funnel_type`** — Marketing funnel category that brought the student in. _[Confirm valid values.]_
- **`funnel_url`** — Specific funnel landing URL.
- **`offer_type`** — Promotional offer category at signup.
- **`slug`** — Specific offer slug/identifier.
- **`concierge_entered_ethnicity`** — Ethnicity manually entered by a concierge agent (NOT self-reported by the student). The internal "concierge" team enters this during account review.
- **`entered_ethnicity_by_cms_user_uuid`** — UUID of the CMS staff user who entered `concierge_entered_ethnicity`. Joins to the underlying `cms_user` table.

---

### ML / inferred fields (probabilistic — NOT user-entered)

These are model predictions, not ground truth. Do not present them to end users as fact.

- **`ml_gender`** — ML-predicted gender.
- **`ml_ethnicity`** — ML-predicted ethnicity (v1 model).
- **`ml_ethnicity_v2`** — ML-predicted ethnicity (v2, newer model). **Prefer `ml_ethnicity_v2` over `ml_ethnicity`** when both are available.
- **`ml_health_tags`** — ML-inferred health-related tags about the student.

---

### Calculated fields (already computed in the view)

- **`student_name`** = `concat(first_name, ' ', last_name)` — full name string.
- **`signup_date`** = `toTimeZone(student.created_date, 'America/Los_Angeles')` — when the student account was created, in US/Pacific.
- **`trial_start_date`** = `toTimeZone(student.trial_start_date, 'America/Los_Angeles')` — when the trial was created.
- **`trial_start_date_active`** = `toTimeZone(student.trial_start_date_active, 'America/Los_Angeles')` — when the trial actually became active. **Use this for trial-conversion analysis, NOT `trial_start_date`.** The two can differ.
- **`trial_end_date`** = `toTimeZone(student.trial_end_date, 'America/Los_Angeles')` — when the trial expires / expired.
- **`cms_user_entered_ethnicity`** = `arrayElement(splitByChar(' ', cms_user.name), 1)` — first name of the CMS staff user who entered the ethnicity (for human-readable attribution).

---

### Common query patterns

```sql
-- Single student by email
SELECT *
FROM myt_views.v_student
WHERE student_email = 'jane@example.com';

-- Students who signed up in the last 7 days
SELECT count() AS signups
FROM myt_views.v_student
WHERE signup_date >= now() - INTERVAL 7 DAY;

-- Active trials (currently within the trial window)
SELECT student_id, student_email, trial_start_date_active, trial_end_date
FROM myt_views.v_student
WHERE trial_start_date_active IS NOT NULL
  AND trial_end_date >= now();

-- Signups by UTM source, last 30 days
SELECT utm_source, count() AS signups
FROM myt_views.v_student
WHERE signup_date >= now() - INTERVAL 30 DAY
GROUP BY utm_source
ORDER BY signups DESC;

-- Signups by funnel and country, this month
SELECT funnel_type, country, count() AS signups
FROM myt_views.v_student
WHERE toStartOfMonth(signup_date) = toStartOfMonth(now())
GROUP BY funnel_type, country
ORDER BY signups DESC;
```

---

### Notes & gotchas

- **`client_domain` is filtered to `'lifestyle'` in WHERE but NOT exposed in SELECT.** Every row in this view is implicitly lifestyle. The agent should not attempt to filter by `client_domain` against this view — it's already done.
- **`FINAL` has a performance cost.** The view uses `FINAL` on all three source tables to deduplicate `ReplacingMergeTree` rows. Queries against this view will be slower than against plain MergeTree views. Filter on indexed columns (e.g., `student_id`, `signup_date`) wherever possible.
- **Fields present in the original Tableau/Redshift query but NOT in this view** (yet — add to the `CREATE VIEW` if the agent needs them):
  - `signup_date_utc` (raw UTC version of `created_date`)
  - `signup_date_plus30` (signup + 30 days)
  - `phone_number_given_1`, `phone_number_given_2`, `phone_number_given`
  - `phone_number_verified`
  - `has_attempted_to_refer`
  - `join_prediction_score`
  - `show_2_weeks_remaining_banner_in_high_volume_plan`
  - `hv_billing_banner_close_date`
  - `app_clips_control_split_test_v2`, `app_clips_test_split_test_v2`
- **`ml_ethnicity` vs `ml_ethnicity_v2`** — both are exposed. Prefer v2.
- **`trial_start_date` vs `trial_start_date_active`** — these are different fields and the difference matters. Active is what counts for conversion analysis.
