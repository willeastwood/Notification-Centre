# Notifications Centre — Definition Spec

Status: **draft, pending open items in §9.** Source of requirements: `Notifications_Centre.md`
plus the definition Q&A on 2026-09-08.

Sales leadership and operations subscribe to business-event notifications. Preferences are set
by each person on a z2h page; notifications are delivered as Slack DMs to that person.

- **14 notifications** across 5 categories.
- **All notifications are opt-in. Default is OFF for every toggle.**
- Two ARR threshold sliders (one for Big deals, one for Renewals).

---

## 1. Identity and scope

### Identity
The z2h app resolves the viewer server-side:

```ts
const { id, email, name, photo, permissions, roles } = getBigBrainAPI().contextService.currentUser;
```

`email` is the canonical key. It joins to:

| Purpose | Join |
| --- | --- |
| Preferences row | `preferences.user_email` |
| Salesforce user (scope, hierarchy) | `bigbrain.l3.dim_salesforce_users.email` |
| Slack DM target | `users.lookupByEmail` → Slack user ID |

Because identity comes from the session, nobody can read or write anyone else's preferences.

### Scope — whose records a person is notified about
Reporting line, resolved recursively through the Salesforce `Manager` field.

```
in_scope(recipient) = { recipient } ∪ { all direct and indirect reports of recipient }
```

An event is in a recipient's scope when the opportunity / account / lead **owner** is in that set.
A rep with no reports therefore sees only their own book; a manager sees their whole downline
including themselves.

### Channel and segment
No filter. Both channels (direct sales and partner) and all segments are in scope.

---

## 2. Data sources and change detection

| Cadence | Grain | Layer |
| --- | --- | --- |
| Big deals (hourly) | Hourly comparison against previous hourly read | **L2** opportunity table |
| Daily + weekly notifications | Day-over-day snapshot comparison | **L4** daily snapshot tables |

Change detection is **snapshot diffing**, not field history. Two consequences, both accepted:

- Big deals: multiple changes to the same field inside one hour collapse into a single
  from→to transition (the net move over the hour).
- Daily/weekly: same collapse over a 24h window.

---

## 3. Schedules

| Job | Cadence | Time |
| --- | --- | --- |
| Big deals | Hourly | On the hour |
| Daily notifications | Daily | 09:00 recipient-local |
| Weekly notifications | Weekly, Thursday | 10:00 recipient-local |

Runner: z2h scheduled job. Recipient-local time requires a per-user timezone — see §9.

---

## 4. Message shape

- **One separate Slack DM per notification type.** Never a combined digest across types.
- Every message links to the Salesforce record(s).
- **Zero results ⇒ send nothing.** No "nothing to report" messages.
- Digest-style notifications (renewals, expansion lists) attach a file — see the individual
  definitions.

---

## 5. Deduplication

### 5.1 Within-run precedence (Big deals)
Per opportunity, per run, **at most one** Big-deal notification fires. Highest-priority
event wins and suppresses the rest:

```
1. Closed Won / Closed Lost   (highest)
2. Forecast category change
3. Stage change
4. Close date change          (lowest)
```

So an opportunity moving to Closed Won sends only the Closed Won notification, even though its
stage and forecast category also changed in that same hour.

### 5.2 Across runs — the notification log
Every send writes to a `notification_log` table. Before sending, the job checks it.

| Notification | Dedup key | Effect |
| --- | --- | --- |
| `BD_CLOSED_WON` | `(user_email, notification_id, opportunity_id)` | Sent at most **once ever** per opportunity — a reopened-and-re-won deal does not notify twice. |
| `BD_CLOSED_LOST` | `(user_email, notification_id, opportunity_id)` | Once ever per opportunity. |
| `BD_STAGE_CHANGE`, `BD_CLOSE_DATE_CHANGE`, `BD_FORECAST_CATEGORY_CHANGE` | `(user_email, notification_id, opportunity_id, from_value, to_value, detected_at_hour)` | Each distinct transition sends once; a later, different transition on the same opportunity sends again. |
| Digest notifications | `(user_email, notification_id, run_date)` | One send per scheduled run. |

---

## 6. Preferences data model

Stored in z2h's own storage.

```
notification_preferences
  user_email            string   PK
  updated_at            timestamp
  -- category sliders (USD)
  big_deals_arr_threshold     integer  default 10000
  renewals_arr_threshold      integer  default 10000
  -- per-notification toggles, all default FALSE
  bd_closed_won                       boolean
  bd_closed_lost                      boolean
  bd_stage_change                     boolean
  bd_close_date_change                boolean
  bd_forecast_category_change         boolean
  rn_upcoming_renewals                boolean
  hy_close_date_in_past               boolean
  hy_early_stage_close_date_upcoming  boolean
  hy_no_activity_last_week            boolean
  ex_crm_trial_activated              boolean
  ex_ai_usage_indication              boolean
  ex_members_exceeding_seats          boolean
  ld_leads_received_summary           boolean
  ld_uncontacted_open_leads           boolean
```

### Sliders
- **Big deals**: opportunity ARR. Range **$10,000 – $500,000**, step **$5,000**, default **$10,000**.
- **Renewals**: account-level total current ARR. Same range, step and default (see §9).
- One slider per category, applied to every notification in that category.
- A slider has no effect while every toggle in its category is off.

---

## 7. Notification definitions

### Category: Big deals
Hourly, L2 opportunity table. **All five are filtered by `opportunity_arr >= big_deals_arr_threshold`.**

#### BD_CLOSED_WON
- **Trigger**: stage transitions to `Closed Won` since the previous hourly read.
- **Filters**: `opportunity_arr >= threshold`, `recognized = true`.
- **Dedup**: once ever per opportunity (§5.2).
- **Message**: account name, opportunity name, ARR, owner, close date, Salesforce link.

#### BD_CLOSED_LOST
- **Trigger**: stage transitions to any closed-lost stage since the previous hourly read.
- **Filters**: `opportunity_arr >= threshold`. All closed-lost opportunities included.
- **Message**: account, opportunity, ARR, owner, **loss reason**, Salesforce link.

#### BD_STAGE_CHANGE
- **Trigger**: any stage change where the resulting stage is **beyond `Qualified`**, plus **any
  backward slip** (a move to an earlier stage in the funnel order).
- **Excludes**: transitions to Closed Won / Closed Lost — those are covered by BD_CLOSED_WON /
  BD_CLOSED_LOST and suppressed by §5.1.
- **Message**: account, opportunity, ARR, owner, `from stage → to stage`, flagged as a slip
  where backward, Salesforce link.

#### BD_CLOSE_DATE_CHANGE
- **Trigger**: any change to close date. No minimum shift; pushes and pull-ins both notify.
- **Message**: account, opportunity, ARR, owner, `old date → new date`, days moved,
  Salesforce link.

#### BD_FORECAST_CATEGORY_CHANGE
- **Trigger**: any forecast category change involving **Best Case** or **Commit** — i.e. the
  prior or the new value is one of those two. Movement between other categories does not notify.
- **Message**: account, opportunity, ARR, owner, `from category → to category`, Salesforce link.

---

### Category: Renewals

#### RN_UPCOMING_RENEWALS
- **Cadence**: weekly, Thursday 10:00 local.
- **Population**: opportunities where the **Source Type (Auto)** field contains `renewal`.
- **Filters**:
  - Renewal date falls inside a **rolling forward window** (length pending — §9).
  - `account_current_arr >= renewals_arr_threshold` (account-level total ARR).
  - Excludes renewals already closed, already renewed, or churned.
- **Rolling window** means a renewal reappears in each weekly digest until it leaves the window
  or is excluded.
- **Message**: count plus a digest listing account, account current ARR, renewal date, owner,
  Salesforce link — **sorted by ARR descending**. A **spreadsheet export** is attached to the DM.

---

### Category: Hygiene
**No ARR slider applies to this category.**

**Recipient**: the **manager**. Each notification is a per-rep digest — one message per offence
category, listing each report and their offending records, grouped by rep. A subscriber with no
reports receives their own records only.

#### HY_CLOSE_DATE_IN_PAST
- **Cadence**: weekly, Thursday 10:00 local.
- **Population**: open opportunities with `close_date < today`.
- **Grace period**: ignore anything **≤ 1 day** overdue.
- **Message**: grouped by rep — opportunity, ARR, stage, close date, days overdue, Salesforce link.

#### HY_EARLY_STAGE_CLOSE_DATE_UPCOMING
- **Cadence**: daily, 09:00 local.
- **Population**: open opportunities in an early stage — **Pre-Qualified, Qualified, Evaluation** —
  with `close_date` within the next **14 days**.
- **Message**: grouped by rep — opportunity, ARR, stage, close date, Salesforce link.

#### HY_NO_ACTIVITY_LAST_WEEK
- **Cadence**: weekly, Thursday 10:00 local.
- **Population**: all open opportunities with `close_date` in the **current fiscal quarter**.
- **Condition**: no activity in the last 7 days, where activity is **either** a Gong-recorded call
  **or** any logged activity (email, meeting, task), matched at **either the opportunity or the
  account** level. Activity at either level clears the flag.
- **Message**: grouped by rep — opportunity, ARR, stage, close date, days since last activity,
  Salesforce link.

---

### Category: Expansion indications

#### EX_CRM_TRIAL_ACTIVATED
- **Cadence**: daily, 09:00 local.
- **Trigger**: CRM product trial-start event in the product-usage table.
- **Look-back**: activations since the previous send.
- **Excludes**: accounts that already hold paid CRM; accounts with an open expansion opportunity.
- **Message**: "*N* accounts activated a CRM trial" plus an attached **CSV** listing account,
  current ARR, owner, activation date, Salesforce link.

#### EX_AI_USAGE_INDICATION
- **Cadence**: daily, 09:00 local.
- **Definition**: **pending — see §9.** Options tabled for selection.
- **Message**: "*N* accounts showing AI usage indication" plus an attached **CSV**.

#### EX_MEMBERS_EXCEEDING_SEATS
- **Cadence**: weekly, Thursday 10:00 local.
- **Condition**: `active_members > licensed_seats`, by any amount.
- **Message**: "*N* accounts exceeding seats" plus an attached **CSV** listing account, active
  members, licensed seats, overage, current ARR, owner, Salesforce link.

---

### Category: Leads
Both notifications: **inbound leads only**.

#### LD_LEADS_RECEIVED_SUMMARY
- **Cadence**: weekly, Thursday 10:00 local.
- **Metric**: count of leads received **per rep, for the completed week**, shown against that
  rep's **baseline = mean weekly count over the prior 4 weeks**.
- **Filters**: inbound only; `lead_source <> 'tools'`.
- **Message**: per rep — leads this week, 4-week baseline, variance vs baseline.

#### LD_UNCONTACTED_OPEN_LEADS
- **Cadence**: weekly, Thursday 10:00 local.
- **Condition**: **no activity ever on the lead by its current owner**, and `status` in
  (**Received**, **Attempting**).
- **Filters**: inbound only.
- **Message**: grouped by rep — count and the lead list with name, company, status, lead source,
  received date, days held, Salesforce link.

---

## 8. Build order

1. z2h preferences page — 14 toggles (default off), 2 sliders, save/load keyed on session email.
2. Preferences storage + read API for the schedulers.
3. Scope resolver — recursive `Manager` walk over `dim_salesforce_users`.
4. Slack delivery layer — email → Slack user ID, DM send, file attachment, `notification_log` write.
5. Hourly Big-deals job (L2 diff + §5 dedup) — the highest-value and highest-risk piece.
6. Daily job (L4 snapshots): HY_EARLY_STAGE_CLOSE_DATE_UPCOMING, EX_CRM_TRIAL_ACTIVATED,
   EX_AI_USAGE_INDICATION.
7. Weekly job (L4 snapshots): RN_UPCOMING_RENEWALS, HY_CLOSE_DATE_IN_PAST,
   HY_NO_ACTIVITY_LAST_WEEK, EX_MEMBERS_EXCEEDING_SEATS, both Leads notifications.

---

## 9. Open items

Each blocks the item named. Everything else in this spec is settled.

| # | Item | Blocks |
| --- | --- | --- |
| O1 | **Renewal window length** — 30, 60 or 90 days. Confirmed as rolling; the number was not chosen. | RN_UPCOMING_RENEWALS |
| O2 | **AI usage indication definition** — options tabled, selection needed. | EX_AI_USAGE_INDICATION |
| O3 | **Per-user timezone source** for "local system time". `currentUser` returns no timezone. Proposed: Salesforce `User.TimeZoneSidKey`, falling back to the Slack profile timezone. | All daily/weekly schedules |
| O4 | **Ordered stage list** — needed to evaluate "beyond Qualified" and to detect backward slips. | BD_STAGE_CHANGE |
| O5 | **Table and field names** in Kremer for: L2 opportunity, L4 opportunity snapshot, account ARR, renewal source type, Gong/activity, product usage (CRM trial, AI), seats vs members, leads. | All queries |
| O6 | **Renewals slider range** — assumed identical to Big deals ($10k–$500k / $5k / $10k default). Account total ARR may warrant a different range. | Preferences page |
| O7 | **Slack app scopes** — `chat:write`, `users:read.email`, and `files:write` for the spreadsheet/CSV attachments. | Delivery layer |
| O8 | **Closed-lost threshold** — the Big-deals slider is applied to BD_CLOSED_LOST as a category-wide filter. Confirm losses should be size-filtered rather than all-inclusive. | BD_CLOSED_LOST |
