# Notifications Centre — Definition Spec & Kremer Session Brief

Sales leadership and operations subscribe to business-event notifications. Preferences are set
by each person on a Zero to Hero (z2h) page; notifications are delivered as Slack DMs to that
same person.

- **14 notifications** across 5 categories.
- **All notifications are opt-in. Default is OFF for every toggle.**
- Two ARR threshold sliders — one for Big deals, one for Renewals.

**Source of requirements**: the `Notifications_Centre.md` category list, plus a full definition
Q&A completed 2026-09-08. Every definition below is agreed unless it appears in §9 Open items.

---

## 0. How to use this document

This document is self-contained and is written to be handed to a **Kremer-enabled session**,
which is where the build happens. The session that produced it had no Kremer, Slack or z2h
access, so all business definitions are settled but **no table or field name has been
verified against the warehouse**.

Work in this order:

1. Read §1–§7 — the agreed behaviour. Do not re-litigate these; they were decided with the
   requester.
2. Work §10 — the data discovery checklist. Resolve every placeholder to a real
   table and column. This is the main unblocking task.
3. Use §11 as the per-notification field requirement list when writing queries.
4. Run §12 validation before anything is wired to Slack. These notifications are DMs to
   leadership; a bad first send is expensive.
5. Bring §9 open items back to the requester (Will Eastwood, willea@monday.com).

Placeholders appear as `<ANGLE_BRACKETS>` throughout and are collected in §10.

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
| Preferences row | `notification_preferences.user_email` |
| Salesforce user — scope, hierarchy, timezone | `bigbrain.l3.dim_salesforce_users.<EMAIL_COL>` |
| Slack DM target | `users.lookupByEmail` → Slack user ID |

Identity comes from the session, so nobody can read or write anyone else's preferences, and
nobody can subscribe a colleague to DMs.

### Scope — whose records a person is notified about
Reporting line, resolved **recursively** through the Salesforce `Manager` field.

```
in_scope(recipient) = { recipient } ∪ { all direct and indirect reports of recipient }
```

An event is in a recipient's scope when the **owner** of the opportunity, account or lead is in
that set. A rep with no reports therefore sees only their own book; a manager sees their whole
downline including themselves.

Implement as a recursive CTE over `dim_salesforce_users` on the manager relationship. Guard
against cycles (self-referencing or circular manager records do occur in Salesforce) with a
depth cap.

### Channel and segment
No filter. Both channels — direct sales and partner — and all segments are in scope.

---

## 2. Data sources and change detection

| Cadence | Grain | Layer |
| --- | --- | --- |
| Big deals — hourly | Compare against the previous hourly read | **L2** opportunity table |
| Daily and weekly notifications | Day-over-day snapshot comparison | **L4** daily snapshot tables |

Change detection is **snapshot diffing**, not Salesforce field history. Two consequences, both
explicitly accepted by the requester:

- Big deals: multiple changes to the same field inside one hour collapse into a single from→to
  transition — the net move over that hour.
- Daily and weekly: the same collapse over a 24-hour window.

The hourly job needs the prior hour's values. Either read two consecutive L2 loads, or persist a
per-opportunity watermark of the last-seen `stage`, `close_date` and `forecast_category` in z2h
storage and diff against that. **The watermark approach is preferred** — it is resilient to a
skipped or late job run, whereas comparing only against the immediately previous load silently
drops events whenever a run is missed.

---

## 3. Schedules

| Job | Cadence | Time |
| --- | --- | --- |
| Big deals | Hourly | On the hour |
| Daily notifications | Daily | 09:00 recipient-local |
| Weekly notifications | Weekly, Thursday | 10:00 recipient-local |

Runner: z2h scheduled job.

### Recipient timezone
1. Salesforce `User.<TIMEZONE_COL>` (`TimeZoneSidKey`, e.g. `Europe/London`, `America/New_York`).
2. If absent or unresolvable, fall back to **UK time — `Europe/London`**, which correctly
   observes GMT/BST.

Implementation note: run the daily and weekly jobs hourly and, on each pass, send to the set of
recipients whose local time has just reached 09:00 (daily) or Thursday 10:00 (weekly). Use the
`notification_log` (§5.2) to guarantee one send per recipient per period regardless of DST
transitions — on the day a clock shifts, a naive scheduler will otherwise double-send or skip.

---

## 4. Message shape

- **One separate Slack DM per notification type.** Never a combined digest across types.
- Every message links to the Salesforce record(s).
- **Zero results ⇒ send nothing.** No "nothing to report" messages.
- Digest notifications attach a file — spreadsheet for renewals, CSV for the expansion lists.
- Currency: ARR values are shown in USD. Confirm during discovery whether the ARR columns are
  already USD-normalised or need conversion.

---

## 5. Deduplication

The requester called this out specifically: an opportunity moving to Closed Won must notify
**only** that it is Closed Won, not additionally that its stage changed.

### 5.1 Within-run precedence — Big deals
Per opportunity, per run, **at most one** Big-deal notification fires. The highest-priority
event wins and suppresses the others:

```
1. Closed Won / Closed Lost   (highest)
2. Forecast category change
3. Stage change
4. Close date change           (lowest)
```

This matters more than it looks: stage and forecast category very often move together, and a
close date is frequently edited in the same save as a stage advance.

### 5.2 Across runs — the notification log
Every send writes a row to `notification_log`. The job checks it before sending.

| Notification | Dedup key | Effect |
| --- | --- | --- |
| `BD_CLOSED_WON` | `(user_email, notification_id, opportunity_id)` | Sent at most **once ever** per opportunity — a reopened-and-re-won deal does not notify twice. |
| `BD_CLOSED_LOST` | `(user_email, notification_id, opportunity_id)` | Once ever per opportunity. |
| `BD_STAGE_CHANGE`, `BD_CLOSE_DATE_CHANGE`, `BD_FORECAST_CATEGORY_CHANGE` | `(user_email, notification_id, opportunity_id, from_value, to_value, detected_at_hour)` | Each distinct transition sends once; a later, different transition on the same opportunity sends again. |
| All digest notifications | `(user_email, notification_id, run_date)` | One send per scheduled run. |

```
notification_log
  user_email        string
  notification_id   string
  entity_id         string     -- opportunity / account / lead id, or run date for digests
  from_value        string     nullable
  to_value          string     nullable
  detected_at_hour  timestamp  nullable
  run_date          date       nullable
  sent_at           timestamp
  PRIMARY KEY (user_email, notification_id, entity_id, coalesce(from_value,''), coalesce(to_value,''), coalesce(detected_at_hour, run_date))
```

---

## 6. Preferences data model

Stored in z2h's own storage.

```
notification_preferences
  user_email            string   PK
  updated_at            timestamp
  -- category sliders, USD
  big_deals_arr_threshold     integer  default 10000
  renewals_arr_threshold      integer  default 10000
  -- per-notification toggles, ALL default FALSE
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
- **Big deals** — opportunity ARR. Range **$10,000 – $500,000**, step **$5,000**, default **$10,000**.
- **Renewals** — account-level total current ARR. Same range, step and default (see §9 O6).
- One slider per category, applied to every notification in that category.
- A slider is inert while every toggle in its category is off. The page should visually
  de-emphasise it in that state.

### z2h page requirements
- Five category sections matching §7, each listing its notifications as individual toggles.
- Sliders shown at category level for Big deals and Renewals only.
- Loads the current user's saved row on mount; creates a defaults row (everything off) on first
  visit.
- Shows the resolved identity — name and email — so the user can see whose preferences they are
  editing.
- Save is explicit, with a confirmation state.

---

## 7. Notification definitions

### Category: Big deals
Hourly, L2 opportunity table. **All five are filtered by
`opportunity_arr >= big_deals_arr_threshold`** (see §9 O8 for the closed-lost caveat).

#### BD_CLOSED_WON
- **Trigger** — stage transitions to `Closed Won` since the previous read.
- **Filters** — `opportunity_arr >= threshold`; `recognized = true`.
- **Dedup** — once ever per opportunity.
- **Message** — account name, opportunity name, ARR, owner, close date, Salesforce link.

#### BD_CLOSED_LOST
- **Trigger** — stage transitions to any closed-lost stage since the previous read.
- **Filters** — `opportunity_arr >= threshold`. All closed-lost opportunities qualify; no
  exclusion of admin-style lost stages such as duplicate or disqualified.
- **Message** — account, opportunity, ARR, owner, **loss reason**, Salesforce link.

#### BD_STAGE_CHANGE
Evaluated against the **full funnel** stage order (§10 D4).

- **Trigger**, either of:
  - **Forward move** where the resulting stage is **beyond `Qualified`** in funnel order, i.e.
    `order(new_stage) > order('Qualified')`.
  - **Backward slip** anywhere in the full funnel, i.e. `order(new_stage) < order(old_stage)` —
    including slips between stages at or below Qualified.
- **Excludes** — transitions to Closed Won or Closed Lost; those are BD_CLOSED_WON /
  BD_CLOSED_LOST and are suppressed here by §5.1.
- **Message** — account, opportunity, ARR, owner, `from stage → to stage`, explicitly flagged as
  a slip when backward, Salesforce link.

#### BD_CLOSE_DATE_CHANGE
- **Trigger** — any change to close date. No minimum shift; pushes and pull-ins both notify.
- **Message** — account, opportunity, ARR, owner, `old date → new date`, days moved and
  direction, Salesforce link.

#### BD_FORECAST_CATEGORY_CHANGE
- **Trigger** — any forecast category change **involving Best Case or Commit**: the prior value
  or the new value is one of those two. A change between two other categories does not notify.
- **Message** — account, opportunity, ARR, owner, `from category → to category`, Salesforce link.

---

### Category: Renewals

#### RN_UPCOMING_RENEWALS
- **Cadence** — weekly, Thursday 10:00 local.
- **Population** — opportunities where the **Source Type (Auto)** field contains `renewal`
  (substring match, case-insensitive).
- **Filters**
  - Renewal date within a **rolling 60-day forward window** from the run date.
  - `account_current_arr >= renewals_arr_threshold`, using **account-level total ARR**.
  - Excludes renewals already closed, already renewed, or churned.
- **Rolling window** — a renewal reappears in each weekly digest until it leaves the window or
  becomes excluded. This is intended, not a duplication bug: it is a standing worklist.
- **Message** — count, plus a digest listing account, account current ARR, renewal date, owner
  and Salesforce link, **sorted by ARR descending**. A **spreadsheet export** is attached to the DM.

---

### Category: Hygiene
**No ARR slider applies to this category.** Every open opportunity qualifies regardless of size.

**Recipient — the manager.** Each of the three is a per-rep digest: one message per offence
category, listing each report and their offending records grouped by rep. A subscriber with no
reports receives their own records only.

#### HY_CLOSE_DATE_IN_PAST
- **Cadence** — weekly, Thursday 10:00 local.
- **Population** — open opportunities with `close_date < today`.
- **Grace period** — ignore anything **≤ 1 day** overdue, i.e. require `close_date < today - 1 day`.
- **Message** — grouped by rep: opportunity, ARR, stage, close date, days overdue, Salesforce link.

#### HY_EARLY_STAGE_CLOSE_DATE_UPCOMING
- **Cadence** — daily, 09:00 local.
- **Population** — open opportunities in an early stage — **Pre-Qualified, Qualified,
  Evaluation** — with `close_date` within the next **14 days**.
- **Message** — grouped by rep: opportunity, ARR, stage, close date, Salesforce link.

#### HY_NO_ACTIVITY_LAST_WEEK
- **Cadence** — weekly, Thursday 10:00 local.
- **Population** — all open opportunities with `close_date` in the **current fiscal quarter**.
- **Condition** — no activity in the last 7 days, where activity is **either** a Gong-recorded
  call **or** any logged activity (email, meeting, task), matched at **either the opportunity or
  the account** level. Activity found at either level clears the flag.
- **Message** — grouped by rep: opportunity, ARR, stage, close date, days since last activity,
  Salesforce link.

---

### Category: Expansion indications

#### EX_CRM_TRIAL_ACTIVATED
- **Cadence** — daily, 09:00 local.
- **Trigger** — a CRM product trial-start event in the product-usage table.
- **Look-back** — activations since the previous send for that recipient.
- **Excludes** — accounts that already hold paid CRM; accounts with an open expansion opportunity.
- **Message** — "*N* accounts activated a CRM trial", plus an attached **CSV** listing account,
  current ARR, owner, activation date and Salesforce link.

#### EX_AI_USAGE_INDICATION
- **Cadence** — daily, 09:00 local.
- **Definition — NOT YET DECIDED. See §9 O2 for the six options tabled.** Build the toggle and
  the delivery path, but do not implement a metric until the requester selects one.
- **Message** — "*N* accounts showing AI usage indication", plus an attached **CSV**.

#### EX_MEMBERS_EXCEEDING_SEATS
- **Cadence** — weekly, Thursday 10:00 local.
- **Condition** — `active_members > licensed_seats`, by any amount.
- **Message** — "*N* accounts exceeding seats", plus an attached **CSV** listing account, active
  members, licensed seats, overage, current ARR, owner and Salesforce link.

---

### Category: Leads
Both notifications: **inbound leads only**.

#### LD_LEADS_RECEIVED_SUMMARY
- **Cadence** — weekly, Thursday 10:00 local.
- **Metric** — count of leads received **per rep for the completed week**, shown against that
  rep's **baseline = mean weekly count over the prior 4 weeks**.
- **Filters** — inbound only; `lead_source <> 'tools'`.
- **Message** — per rep: leads this week, 4-week baseline, variance against baseline.

#### LD_UNCONTACTED_OPEN_LEADS
- **Cadence** — weekly, Thursday 10:00 local.
- **Condition** — **no activity ever on the lead by its current owner**, and `status` in
  (**Received**, **Attempting**). Note the owner qualifier: activity logged by a previous owner
  does not clear the flag.
- **Filters** — inbound only.
- **Message** — grouped by rep: count, then lead name, company, status, lead source, received
  date, days held and Salesforce link.

---

## 8. Build order

1. **z2h preferences page** — 14 toggles (default off), 2 sliders, save/load keyed on session
   email. Independent of all Kremer discovery, so it can start immediately.
2. **Preferences storage** plus a read API the schedulers can call.
3. **Scope resolver** — recursive `Manager` walk over `dim_salesforce_users`, with cycle guard.
4. **Slack delivery layer** — email → Slack user ID, DM send, file attachment, `notification_log`
   write. Build the log before the first real send.
5. **Hourly Big-deals job** — L2 diff plus §5 dedup. Highest value and highest risk; the dedup
   rules are the whole point of this piece.
6. **Daily job**, L4 snapshots — HY_EARLY_STAGE_CLOSE_DATE_UPCOMING, EX_CRM_TRIAL_ACTIVATED,
   EX_AI_USAGE_INDICATION (pending O2).
7. **Weekly job**, L4 snapshots — RN_UPCOMING_RENEWALS, HY_CLOSE_DATE_IN_PAST,
   HY_NO_ACTIVITY_LAST_WEEK, EX_MEMBERS_EXCEEDING_SEATS, LD_LEADS_RECEIVED_SUMMARY,
   LD_UNCONTACTED_OPEN_LEADS.

---

## 9. Open items

| # | Item | Blocks | Status |
| --- | --- | --- | --- |
| O1 | Renewal window length | RN_UPCOMING_RENEWALS | **Resolved — 60 days, rolling.** |
| O2 | AI usage indication definition | EX_AI_USAGE_INDICATION | **Open** — options below. |
| O3 | Per-user timezone source | All daily/weekly schedules | **Resolved** — Salesforce timezone field, fallback `Europe/London`. |
| O4 | Ordered stage list | BD_STAGE_CHANGE | **Resolved in principle** — full funnel order; actual stage names to be read from the warehouse (§10 D4). |
| O5 | Kremer table and field names | All queries | **Open** — this is §10, the main task for the receiving session. |
| O6 | Renewals slider range | Preferences page | **Open, assumption in place** — same as Big deals ($10k–$500k, $5k step, $10k default). Account total ARR may warrant a higher ceiling; revisit once real ARR distribution is visible. |
| O7 | Slack app scopes | Delivery layer | **Resolved** — `chat:write`, `users:read.email`, `files:write` approved. |
| O8 | Closed-lost threshold | BD_CLOSED_LOST | **Open, assumption in place** — the Big-deals slider is applied to closed-lost as a category-wide filter. The requester's wording ("all closed-lost opportunities") may have meant unfiltered. |

### O2 — options tabled for the AI usage indication
Ordered by strength of commercial signal. Which are buildable depends on §10 D8.

1. **Credit exhaustion** — account has consumed ≥80% (or 100%) of its monthly AI credit
   allowance. Strongest of the set: there is a hard wall the customer is about to hit, so the
   expansion conversation is concrete rather than speculative. Needs both allocation and
   consumption data.
2. **First AI activation** — account's first-ever use of any AI feature. Clean, unambiguous,
   genuinely new information. Fires once per account ever, so volume decays as the base adopts.
3. **Adoption breadth** — distinct users in the account using an AI feature in the last 7 days
   crosses a threshold (≥5 users, or ≥20% of active members). Signals org-wide adoption rather
   than one power user experimenting; a better predictor of willingness to pay than raw volume.
   Needs per-user event grain.
4. **Gated-feature use** — use of AI features behind a higher tier or paid add-on. Direct upsell
   path, but only as good as current packaging, so it needs re-checking whenever packaging changes.
5. **Absolute consumption threshold** — AI actions or credits in the last 7 days above a fixed
   number. Simple, but the threshold needs calibrating per segment or it floods for enterprise
   and never fires for SMB.
6. **Growth trend** — AI actions in the last 7 days ≥2× the trailing 4-week average, with a
   minimum floor so 1→2 does not register as a doubling. Catches acceleration early, produces
   the most false positives.

**Recommendation: 1 as the primary signal, with 2 as a second separately-toggled notification.**
They answer different questions — "this account is hitting a ceiling" versus "this account just
started" — and bundling them into one message means neither gets an obvious next action. Add 3
later once volumes are observable.

---

## 10. Data discovery checklist

Resolve each placeholder to a real table and column, then record the answer inline in this
document so it stops being a handoff and becomes the build spec. Layer hints (L2/L3/L4) come
from the requester; the only table name carried over from prior work is
`bigbrain.l3.dim_salesforce_users`, and even its column names are unverified.

| # | Placeholder | Needed for | Notes |
| --- | --- | --- | --- |
| D1 | `<L2_OPPORTUNITY_TABLE>` | All Big deals | Must support hourly freshness. Confirm actual load cadence — if it lands less often than hourly, the hourly schedule is not achievable and the requester needs to know. |
| D2 | `<L4_OPPORTUNITY_SNAPSHOT>` | Hygiene, Renewals | Daily snapshot grain. Confirm one row per opportunity per day and the snapshot date column. |
| D3 | Opportunity columns | All Big deals, Hygiene | `opportunity_id`, `account_id`, `name`, `stage`, `close_date`, `forecast_category`, `opportunity_arr`, `owner_id`, `recognized`, loss reason, `source_type_auto`, `is_closed`/`is_won`. |
| D4 | **Ordered funnel stage list** | BD_STAGE_CHANGE, HY_EARLY_STAGE_CLOSE_DATE_UPCOMING | The full ordered stage sequence with a sort key. Confirm the exact strings for `Pre-Qualified`, `Qualified`, `Evaluation`, `Closed Won`, and every closed-lost variant. Without a numeric sort key, derive one from the stage dimension rather than hardcoding. |
| D5 | Forecast category values | BD_FORECAST_CATEGORY_CHANGE | Confirm the exact strings for `Best Case` and `Commit`, and enumerate the others so "involving either" can be evaluated. |
| D6 | `<ACCOUNT_ARR_TABLE>` + column | Renewals, all expansion CSVs | Account-level **total current ARR**. Confirm whether it rolls up to parent/ultimate-parent, and whether it is USD-normalised. |
| D7 | `<ACTIVITY_TABLE>` / `<GONG_TABLE>` | HY_NO_ACTIVITY_LAST_WEEK | Needs Gong calls **and** logged emails/meetings/tasks, each joinable at opportunity **and** account level. May be two or more sources unioned. |
| D8 | `<PRODUCT_USAGE_TABLE>` | EX_CRM_TRIAL_ACTIVATED, EX_AI_USAGE_INDICATION | For CRM: a trial-start event and a paid-CRM entitlement flag. For AI: report back which of the O2 options are supported — specifically whether **credit allocation** exists alongside consumption (needed for options 1 and 5) and whether AI events carry **per-user grain** (needed for option 3). |
| D9 | `<SEATS_TABLE>` | EX_MEMBERS_EXCEEDING_SEATS | `active_members` and `licensed_seats` per account. Confirm whether these are per-product or account-wide. |
| D10 | `<LEADS_TABLE>` | Both Leads notifications | `lead_id`, `owner_id`, `status`, `lead_source`, received/created date, plus lead activity joinable **by owner** (LD_UNCONTACTED_OPEN_LEADS needs "no activity by the *current* owner"). Confirm the values that mean inbound, the exact `tools` lead-source string, and that `Received` and `Attempting` are the literal status values. |
| D11 | `dim_salesforce_users` columns | Identity, scope, timezone | Email column, manager column (id or email), timezone column, active/inactive flag. Inactive users must be excluded from digests but must not break the hierarchy walk — a departed manager between two active levels will otherwise sever a reporting line. |
| D12 | Fiscal calendar | HY_NO_ACTIVITY_LAST_WEEK | Current fiscal quarter boundaries. monday.com's fiscal year may not align to calendar quarters — do not assume. |
| D13 | Salesforce instance URL | All messages | For building record deep links. |

---

## 11. Per-notification field requirements

Minimum select list per notification, for query writing. Owner name and Salesforce link are
required on every one and are not repeated.

| Notification | Fields |
| --- | --- |
| BD_CLOSED_WON | account name, opportunity name, ARR, close date |
| BD_CLOSED_LOST | account name, opportunity name, ARR, loss reason |
| BD_STAGE_CHANGE | account name, opportunity name, ARR, old stage, new stage, slip flag |
| BD_CLOSE_DATE_CHANGE | account name, opportunity name, ARR, old close date, new close date, days moved |
| BD_FORECAST_CATEGORY_CHANGE | account name, opportunity name, ARR, old category, new category |
| RN_UPCOMING_RENEWALS | account name, account current ARR, renewal date — sorted ARR desc |
| HY_CLOSE_DATE_IN_PAST | rep, opportunity name, ARR, stage, close date, days overdue |
| HY_EARLY_STAGE_CLOSE_DATE_UPCOMING | rep, opportunity name, ARR, stage, close date |
| HY_NO_ACTIVITY_LAST_WEEK | rep, opportunity name, ARR, stage, close date, days since last activity |
| EX_CRM_TRIAL_ACTIVATED | account name, current ARR, activation date |
| EX_MEMBERS_EXCEEDING_SEATS | account name, active members, licensed seats, overage, current ARR |
| LD_LEADS_RECEIVED_SUMMARY | rep, leads this week, 4-week baseline, variance |
| LD_UNCONTACTED_OPEN_LEADS | rep, lead name, company, status, lead source, received date, days held |

---

## 12. Validation before first live send

These are DMs to sales leadership. Validate against real data with delivery disabled, then dry-run,
then enable.

1. **Scope resolver** — pick three users at different levels (rep, front-line manager, director)
   and confirm the resolved downline matches the org chart. Check that an inactive manager
   mid-hierarchy does not sever the line (D11).
2. **Dedup** — replay a day of real opportunity history through the Big-deals job. Assert that
   **every** opportunity reaching Closed Won produced exactly one notification and no
   accompanying stage, close-date or forecast-category message. This is the requester's headline
   requirement; test it explicitly rather than by inspection.
3. **Once-ever keys** — replay an opportunity that was won, reopened and re-won. Assert one
   BD_CLOSED_WON.
4. **Volume sanity** — for each notification, count what a real run would send per recipient at
   the default $10k threshold. Anything sending dozens of DMs per day per person needs to go back
   to the requester before launch, not after. Big deals at a $10k floor is the likeliest offender.
5. **Threshold behaviour** — confirm an opportunity just under the slider value sends nothing and
   just over sends once.
6. **Timezone** — verify a user with no Salesforce timezone falls back to `Europe/London`, and
   that no recipient double-sends or is skipped across a BST transition.
7. **Zero-result suppression** — confirm a recipient with an empty result set receives no message
   at all.
8. **Opt-in default** — confirm a brand-new user with no preferences row receives nothing.
9. **Attachments** — confirm the renewals spreadsheet and expansion CSVs open cleanly and that
   `files:write` is functioning in the target workspace.

---

## 13. Traceability to the source document

Every line of the original `Notifications_Centre.md` maps to exactly one notification. No line was
dropped or merged.

| Source line | Notification |
| --- | --- |
| Big deals — size slider | `big_deals_arr_threshold` |
| Big deals — Closed won | BD_CLOSED_WON |
| Big deals — Closed lost | BD_CLOSED_LOST |
| Big deals — Stage change | BD_STAGE_CHANGE |
| Big deals — Close date change | BD_CLOSE_DATE_CHANGE |
| Big deals — Forecast category change | BD_FORECAST_CATEGORY_CHANGE |
| Renewals — size slider | `renewals_arr_threshold` |
| Renewals — Upcoming renewals, weekly | RN_UPCOMING_RENEWALS |
| Hygiene — Weekly, close dates in past | HY_CLOSE_DATE_IN_PAST |
| Hygiene — Daily, early stage close date upcoming | HY_EARLY_STAGE_CLOSE_DATE_UPCOMING |
| Hygiene — Weekly, no gong activity in last week | HY_NO_ACTIVITY_LAST_WEEK |
| Expansion — Daily, account CRM trial activated | EX_CRM_TRIAL_ACTIVATED |
| Expansion — Daily, account AI usage indication | EX_AI_USAGE_INDICATION |
| Expansion — Weekly, members exceeding seats | EX_MEMBERS_EXCEEDING_SEATS |
| Leads — Weekly summary of numbers | LD_LEADS_RECEIVED_SUMMARY |
| Leads — Weekly summary of uncontacted open leads by rep | LD_UNCONTACTED_OPEN_LEADS |

The three expansion "x many accounts — click to see list" lines are satisfied by the CSV
attachment on each expansion notification (§4, §7).
