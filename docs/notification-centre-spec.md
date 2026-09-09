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

This document is self-contained. Business definitions were settled on 2026-09-08, and the
**data discovery in §10 was completed against the live warehouse on the same day** — every table
and column below is verified unless explicitly marked open.

Work in this order:

1. Read §1–§7 — the agreed behaviour. Do not re-litigate these; they were decided with the
   requester. Note that §7 has since been corrected in four places where the warehouse
   contradicted an assumption — those corrections are listed in §10.10.
2. Build in the §8 order. §10 gives you the real table and column for every query.
3. Use §11 as the per-notification field requirement list when writing queries.
4. Run §12 validation before anything is wired to Slack. These notifications are DMs to
   leadership; a bad first send is expensive. §12 now carries measured volumes.
5. Bring any remaining **Open** rows in §9 back to the requester as they come up during the
   build. As of 2026-09-08 the only ones left are O6, O8, O11, O12 and O14 — all "assumption in
   place," none blocking.

**Nothing is blocking the build any more.** Every §10 discovery item and every product decision
that had a toggle depending on it (O2, O9, O10, O13) is resolved.

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
| Salesforce user — scope filters | `bigbrain.l3.dim_salesforce_users.email` |
| Salesforce user — timezone | `bigbrain.l1.salesforce_user.DATA:"TimeZoneSidKey"` (§10.9) |
| Slack DM target | `users.lookupByEmail` → Slack user ID |

Identity comes from the session, so nobody can read or write anyone else's preferences, and
nobody can subscribe a colleague to DMs.

### Scope — whose records a person is notified about
**Superseded 2026-09-08.** The original design resolved scope automatically from the recipient's
own position in the reporting line — you saw yourself and everyone below you, full stop. The
requester replaced this: **scope is now four explicit multi-select filters on the z2h
preferences page — region, sub-region, role and manager — set independently of who the
recipient is or where they sit in the org chart.** Nothing about scope runs off the recipient's
own identity or hierarchy position any more; every recipient, including a manager, must
explicitly choose what they want to see.

```
in_scope(recipient) = { owner : owner.region     ∈ recipient.scope_regions     ∨ scope_regions     = ∅ }
                     ∩ { owner : owner.sub_region ∈ recipient.scope_sub_regions ∨ scope_sub_regions = ∅ }
                     ∩ { owner : owner.role_bucket ∈ recipient.scope_roles      ∨ scope_roles       = ∅ }
                     ∩ { owner : owner.manager_id ∈ recipient.scope_managers    ∨ scope_managers    = ∅ }
```

An event is in scope when the **owner** of the opportunity, account or lead — resolved via that
owner's own row on `dim_salesforce_users` — passes all four filters. Within one filter, multiple
selections are OR'd (any match qualifies). Across the four filters it's AND. **An empty filter
places no restriction on that dimension** — leaving all four empty (the default, §6) means
every owner qualifies, which is intentional and harmless on its own: every notification toggle
also defaults off, so an unconfigured new user still receives nothing.

Real values, verified against the warehouse (owners of currently-open opportunities,
`bigbrain.l4.dim_opportunities`, 2026-09-08):

- **Region** (`business_region`) — `NAM`, `EMEA`, `APJ`, `LATAM`, `Global`.
- **Sub-region** (`business_sub_region`) — 27 values including `US`, `UK&I`, `ANZ`, `FR`,
  `Brazil`, `DACH`, `Benelux`, `Nordics`, `IL`, `Mexico`, `SEA`, `India`, `ME`, `CEE/CIS`,
  `Africa`, `Japan`, and several `<Region> - Multiple` / `Rest of the world` catch-alls.
- **Role** (`monday_owner_business_role`, bucketed to **exactly three options — the only three
  the requester wants shown**, revised 2026-09-08):

  | Selectable option | Raw values it collapses |
  | --- | --- |
  | `AE` | `AE` |
  | `AM` | `AM`, `Commercial AM`, `Scale AM`, `Territory AE` |
  | `Partner` | `Partner`, `CPM` |

  Every other raw role (`People Manager`, `Overlay`, `SDR`, `High-Touch CSM`, and the rest of
  the long tail in §10.11) is **excluded from the Role picklist entirely** — not grouped into a
  fourth "Other" option, dropped. It still participates internally so cascading (below) stays
  correct for the other three filters, it just never appears as something to pick.
- **Manager** — **195 distinct front-line managers** of a current open-opportunity owner
  (`dim_salesforce_users` self-joined on `manager_id`), all with a populated `full_name`. This
  is a flat filter on the owner's **direct** manager, not a recursive downline from the manager
  you pick — see §9 O16.

**The four filters cascade against each other.** Selecting `EMEA` in Region immediately narrows
Sub-region to only EMEA's values (`UK&I`, `FR`, `DACH`, …) — and the same narrowing runs in
every direction at once: picking a Role also narrows which Managers and Regions can still be
picked, and so on. Mechanically this means the z2h page doesn't ask the warehouse for four
independent distinct-value lists; it asks for one list of every `(region, sub_region, role,
manager)` combination that actually owns an open opportunity, and derives each picklist's
visible options by filtering that combined list against whatever is already selected in the
*other* three. If narrowing one filter invalidates a value already selected in another, that
stale value is silently dropped rather than saved invisibly.

Full derivation, including the exact queries, is in §10.11.

Two things this change removes, now that scope no longer depends on the org chart at all:

- The recursive CTE, its cycle guard and its depth cap (§1 previously required these; they are
  gone, not just unused).
- The D11 concern about an inactive manager severing the reporting line mid-walk — there is no
  walk to sever any more. D11's `is_active` finding is still used, just for a different purpose:
  see the next section.

### Active reps only — per-rep groupings
**Requester instruction, 2026-09-08: every per-rep breakdown considers active reps only.**

This is unrelated to the scope filters above — it governs which rows appear *inside* a digest a
recipient already qualifies for. When a notification lists or groups by rep — the three Hygiene
digests and both Leads notifications, all of which are explicitly "grouped by rep" in §7 —
filter the rep set to `dim_salesforce_users.is_active = 1` (confirmed available, §10.9) before
grouping. A departed rep's records don't get their own row; they simply don't appear.

Big-deal notifications are per-opportunity, not per-rep groupings, so this rule doesn't change
them — an opportunity still owned by a departed rep still fires its Big-deal DM to that rep's
former manager, which is correct: the deal itself hasn't gone anywhere.

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

A useful source the original draft did not know about: **`bigbrain.l3.scd_opportunities`** is a
dbt SCD2 history of every opportunity (`start_date`, `end_date`, `is_open`, 8.3M rows). Its grain
is daily, so it cannot drive the hourly job — but it is real change history, which makes it the
right source for the §12 replay tests and for validating the dedup rules before go-live.

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

**Hourly is achievable but not real-time.** The L2 opportunity table refreshes at least hourly
yet trails Salesforce by 60–75 minutes, so each run sees the CRM as it was up to ~75 minutes ago
and a Big-deal DM can lag the actual event by close to two hours. Measured 2026-09-08; see §10.3.

### Recipient timezone
1. `bigbrain.l1.salesforce_user.DATA:"TimeZoneSidKey"::string`, joined on
   `salesforce_id = dim_salesforce_users.user_id`. Populated for all 3,120 active users.
2. **Treat the literal value `GMT` as unset** — it is the Salesforce org default, it is held by
   55% of active users, and it does not observe BST. Where `country` or `office_region` is
   populated on `dim_salesforce_users`, derive the zone from that instead.
3. Otherwise fall back to **UK time — `Europe/London`**, which correctly observes GMT/BST.

See §10.9 for the measured distribution and the residual risk this leaves. Getting this wrong
sends every UK recipient their 09:00 DM at 10:00 for seven months of the year.

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
  -- scope filters, added 2026-09-08 — see §1. Arrays of strings, ALL default empty (= unrestricted)
  scope_regions          string[]
  scope_sub_regions      string[]
  scope_roles            string[]   -- bucketed values, e.g. 'AM' represents 4 raw roles — §1, §9 O16
  scope_managers         string[]   -- each entry is the manager's dim_salesforce_users.user_id
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
- **A Scope section at the top of the page** (§1), above the notification categories: four
  multi-select filters — Region, Sub-region, Role, Manager — each searchable, each backed by
  live option lists read from the warehouse at page load (not hardcoded), since role and manager
  names in particular will drift as the org changes. An empty selection is explained inline as
  "everyone" rather than left to look broken or unset.
- Five category sections matching §7, each listing its notifications as individual toggles.
- Sliders shown at category level for Big deals and Renewals only.
- Loads the current user's saved row on mount; creates a defaults row (everything off, no scope
  filters set) on first visit.
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
- **Filters** — `arr >= threshold`; recognised, which has no boolean — use `recoginzed_arr > 0`
  (note the warehouse's typo). This is a heavier filter than it looks: it excludes about 41% of
  closed-won opportunities. See §9 O14.
- **Dedup** — once ever per opportunity.
- **Message** — account name, opportunity name, ARR, owner, close date, Salesforce link.

#### BD_CLOSED_LOST
- **Trigger** — stage transitions to any closed-lost stage since the previous read.
- **Filters** — `opportunity_arr >= threshold`. All closed-lost opportunities qualify; no
  exclusion of admin-style lost stages such as duplicate or disqualified.
- **Message** — account, opportunity, ARR, owner, **loss reason**, Salesforce link.

#### BD_STAGE_CHANGE
Evaluated against the **full funnel** stage order, now resolved in §10.4:
`Pre Qualified` < `Qualified` < `Evaluation` < `Validation` < `Buying process` < closed.

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
- **Field** — `DATA:"ForecastCategoryName"::string`, matched against `'Best Case'` and `'Commit'`.
  There is no first-class forecast-category column. **Do not use `DATA:"ForecastCategory"`** — in
  that key Commit is stored as `Forecast`, so the filter silently never fires. See §10.5.
- **Message** — account, opportunity, ARR, owner, `from category → to category`, Salesforce link.

---

### Category: Renewals

#### RN_UPCOMING_RENEWALS
- **Cadence** — weekly, Thursday 10:00 local.
- **Population** — renewal opportunities, identified by `is_renewal_opportunity_by_type = 1`
  (equivalently `logic_type IN ('Flat Renewal','Expansion On Renewal','Downgrade On Renewal')`).

  > **Corrected 2026-09-08.** The original rule — "the Source Type (Auto) field contains
  > `renewal`" — is not implementable: that field does not exist in the warehouse, and the real
  > `source_type` column holds only `Inbound` / `Outbound`. See §10.10.
- **Filters**
  - **Renewal date (= the renewal opportunity's `close_date`) — either within the rolling 60-day
    forward window, or already in the past.** Extended 2026-09-08 at the requester's instruction
    to also surface **late renewals**: `close_date <= today + 60 days`, with no lower bound. A
    renewal one day overdue and a renewal a year overdue are both included — there is no cutoff
    on how late is too late to still appear here, deliberately.
  - `account_current_arr >= renewals_arr_threshold`, using **account-level total ARR**.
  - Excludes renewals already closed, already renewed, or churned:
    `renewal_stage NOT IN ('Closed Won','Churn') AND is_closed = 0`. This is what actually caps
    the late-renewal tail in practice — a renewal only keeps appearing here for as long as it
    stays open with an unresolved `renewal_stage`; the "no lower bound on lateness" point above
    only matters for renewals nobody has closed out yet.
- **Rolling window** — a renewal reappears in each weekly digest until it leaves the window or
  becomes excluded. This is intended, not a duplication bug: it is a standing worklist, and that
  is now true on both sides of today, not just the forward side.
- **Message** — count, plus a digest listing account, account current ARR, renewal date, owner
  and Salesforce link, **sorted by ARR descending**. A **spreadsheet export** is attached to the DM.

---

### Category: Hygiene
**No ARR slider applies to this category.** Every open opportunity qualifies regardless of size.

**Recipient — the manager.** Each of the three is a per-rep digest: one message per offence
category, listing each report and their offending records grouped by rep. A subscriber with no
reports receives their own records only. **Active reps only** — filter the rep set to
`dim_salesforce_users.is_active = 1` before grouping (§1).

#### HY_CLOSE_DATE_IN_PAST
- **Cadence** — weekly, Thursday 10:00 local.
- **Population** — open opportunities with `close_date < today`.
- **Grace period** — ignore anything **≤ 1 day** overdue, i.e. require `close_date < today - 1 day`.
- **Message** — grouped by rep: opportunity, ARR, stage, close date, days overdue, Salesforce link.

#### HY_EARLY_STAGE_CLOSE_DATE_UPCOMING
- **Cadence** — daily, 09:00 local.
- **Population** — open opportunities in an early stage — exact strings **`Pre Qualified`,
  `Qualified`, `Evaluation`** (see §10.4; note `Pre Qualified` has a space, not a hyphen) — with
  `close_date` within the next **14 days**.
- **Message** — grouped by rep: opportunity, ARR, stage, close date, Salesforce link.

#### HY_NO_ACTIVITY_LAST_WEEK
**Redefined 2026-09-08 — replaces the fixed-7-days-for-everyone rule with a stage-dependent SLA
and a future-meeting exemption.**

- **Cadence** — weekly, Thursday 10:00 local.
- **Population** — open opportunities with `close_date` within a **rolling 60-day forward
  window** from the run date. (No longer "current fiscal quarter" — see §9 O17, which also
  means D12's fiscal-calendar resolution is no longer used by anything in this spec; it's kept
  in §10 as a resolved fact, not removed, in case a future notification needs it.)
- **Future-meeting exemption** — excluded entirely, regardless of the SLA below, if there is a
  **future-dated meeting already on the calendar**: `dim_activities.activity_type = 'meeting'
  AND activity_timestamp > now`, matched at either the opportunity or the account level (§10.6),
  consistent with how "activity" is matched everywhere else below. A meeting already booked
  means the rep doesn't need chasing, even if the deal has gone quiet up to today.
- **Activity SLA — stage-dependent, replacing the old flat 7-day rule for everyone:**

  | Stage | Required activity cadence | Included in this notification when |
  | --- | --- | --- |
  | `Qualified`, `Evaluation` | every **14 days** | most recent activity is more than 14 days old, or there has never been one |
  | `Validation`, `Buying process` | every **7 days** | most recent activity is more than 7 days old, or there has never been one |

  "Activity" keeps its existing definition unchanged: **either** a Gong-recorded call **or**
  any logged activity (email, meeting, task), matched at **either the opportunity or the
  account** level — activity found at either level, of either kind, resets the clock.

  **Open question — `Pre Qualified` stage isn't named in either bucket.** The instruction gave
  cadences for exactly four stages; the full open-stage set (§10.4) has five. Implemented here
  as folding `Pre Qualified` into the 14-day bucket, matching how HY_EARLY_STAGE_CLOSE_DATE_UPCOMING
  already treats `Pre Qualified` as "early stage" alongside `Qualified` and `Evaluation` — but
  this is this session's assumption, not a stated instruction. See §9 O17.
- **Message** — grouped by rep: opportunity, ARR, stage, close date, days since last activity,
  which SLA bucket applied (7-day or 14-day), Salesforce link.

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
- **Trigger — resolved 2026-09-08: credit exhaustion (§9 O2 option 1).** An account's AI credit
  usage crosses **≥80% of its current billing-cycle allowance**:
  `bigbrain.l3.fact_accounts_ai_daily.pct_cap_used >= 0.80 AND is_active_cycle = 1` (§10.7).
  80% is the primary number named in the option description, not the "or 100%" alternative also
  mentioned there — confirm if 100% (i.e. `is_over_cap = 1`) was actually intended.
- **Look-back** — accounts crossing the threshold since the previous send for that recipient, the
  same "since previous send" semantics as EX_CRM_TRIAL_ACTIVATED.
- **Message** — "*N* accounts showing AI usage indication", plus an attached **CSV** listing
  account, current ARR, owner, `pct_cap_used`, and Salesforce link.

**Not built this round**: option 2 (first AI activation) was recommended as a second, separately
toggled notification. Out of scope for this pass — no toggle exists for it yet; revisit if the
requester wants it.

#### EX_MEMBERS_EXCEEDING_SEATS
- **Cadence** — weekly, Thursday 10:00 local.
- **Condition** — `active_members > licensed_seats`, by any amount.
- **Message** — "*N* accounts exceeding seats", plus an attached **CSV** listing account, active
  members, licensed seats, overage, current ARR, owner and Salesforce link.

---

### Category: Leads
Both notifications: **inbound leads only**, and both are per-rep groupings — **active reps
only** (§1).

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
3. **Scope filter matching** — apply the four §1 preference filters (region, sub-region, role,
   manager) as a flat equality/membership check against each record's owner. No recursion, no
   cycle guard — those were needed only under the superseded hierarchy model.
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
| O2 | AI usage indication definition | EX_AI_USAGE_INDICATION | **Resolved 2026-09-08 — option 1, credit exhaustion.** Implemented at the ≥80% threshold named in the option (§7). |
| O3 | Per-user timezone source | All daily/weekly schedules | **Resolved and located** — `l1.salesforce_user.DATA:"TimeZoneSidKey"`, fallback `Europe/London`. Follow-on risk in O11. |
| O4 | Ordered stage list | BD_STAGE_CHANGE | **Resolved in principle** — full funnel order; actual stage names to be read from the warehouse (§10 D4). |
| O5 | Kremer table and field names | All queries | **Resolved 2026-09-08** — §10 complete. D13 alone remains, as O9. |
| O6 | Renewals slider range | Preferences page | **Open, assumption in place** — same as Big deals ($10k–$500k, $5k step, $10k default). Account total ARR may warrant a higher ceiling; revisit once real ARR distribution is visible. |
| O7 | Slack app scopes | Delivery layer | **Resolved** — `chat:write`, `users:read.email`, `files:write` approved. |
| O8 | Closed-lost threshold | BD_CLOSED_LOST | **Open, assumption in place** — the Big-deals slider is applied to closed-lost as a category-wide filter. The requester's wording ("all closed-lost opportunities") may have meant unfiltered. |
| O9 | Salesforce instance URL | Links in every message | **Resolved 2026-09-08** — `https://monday.lightning.force.com/`. Deep-link pattern: `https://monday.lightning.force.com/lightning/r/Opportunity/<id>/view` (swap the object name for Account / Lead). |
| O10 | CRM trial-start event source + paid-CRM entitlement flag | EX_CRM_TRIAL_ACTIVATED | See §10.11 — resolved 2026-09-08. |
| O11 | Non-UK users defaulted to `GMT` | Daily/weekly send times | **Open, mitigation in place** — `GMT` is treated as unset and falls back to `Europe/London` (§3). A minority of GMT-defaulted users are demonstrably not in the UK (9 APAC, 7 Brazil, 5 Mexico, 4 US, 4 Australia, long tail) and would get UK-time DMs. Prefer `country`/`office_region` where populated. |
| O12 | Renewals threshold basis | RN_UPCOMING_RENEWALS | **Open** — §7 filters on the account's total ARR. `arr_to_renew` on the renewal opportunity is the ARR actually at stake and is arguably the better filter. Requester's call. |
| O13 | Close-date notification volume | BD_CLOSE_DATE_CHANGE | **Resolved 2026-09-08 — accepted as-is.** The requester confirmed the ~124/day global figure is fine since delivery is per-recipient, not global (§12.4). No threshold or digest change made. |
| O14 | `recognized = true` on wins | BD_CLOSED_WON | **Open, assumption in place** — implemented as `recoginzed_arr > 0`, which excludes about 41% of closed-won opportunities (16,519 of 28,074 over 90 days have it populated). Confirm that suppressing unrecognised wins is intended. |
| O15 | Active reps only in per-rep groupings | Hygiene (3), Leads (2) | **Resolved 2026-09-08** — requester instruction. `dim_salesforce_users.is_active = 1` filters the rep set before grouping (§1). |
| O16 | Manager filter — direct report vs. whole downline | The scope model, all 14 notifications | **Open, assumption in place.** §1 replaced the recursive hierarchy walk with four flat picklist filters, one of them "manager." Implemented as a **flat filter on the owner's direct `manager_id`** — picking a manager shows only people who report to them directly, not their whole downline. The instruction that prompted this change ("rather than running based on your hierarchy") reads most naturally as dropping recursion everywhere, including here, but the alternative reading — pick a manager, see their entire downline, i.e. the old recursive behaviour with an explicit instead of an automatic root — is plausible enough to flag rather than silently assume away. If the recursive reading is what's wanted, the fix is confined to how the manager filter is *applied* at send-time (a CTE instead of an equality check); the picklist itself, and everything else in §1, is unaffected either way. |
| O17 | `Pre Qualified` stage not named in the HY_NO_ACTIVITY_LAST_WEEK SLA | HY_NO_ACTIVITY_LAST_WEEK | **Open, assumption in place.** The 2026-09-08 redefinition gave a 14-day cadence for `Qualified`/`Evaluation` and a 7-day cadence for `Validation`/`Buying process` — four of the five open stages. `Pre Qualified` was left out of the instruction. Folded into the 14-day bucket here, matching how HY_EARLY_STAGE_CLOSE_DATE_UPCOMING already treats it as "early stage." Confirm, or state the intended cadence (including "no SLA at all for Pre Qualified" as a valid answer). |

### O2 — options tabled for the AI usage indication
**Resolved 2026-09-08 — the requester chose option 1, credit exhaustion.** Implemented in §7
EX_AI_USAGE_INDICATION. Kept here for the record of what was considered and why.

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

## 10. Data discovery — RESOLVED 2026-09-08

Completed against the live warehouse (Snowflake, `BIGBRAIN` database) by a Kremer-enabled
session on 2026-09-08. Every placeholder is now a verified table and column.
Row counts, freshness and volume figures were measured on that date.

### 10.1 Resolved sources

| # | Placeholder | Resolved to | Notes |
| --- | --- | --- | --- |
| D1 | `<L2_OPPORTUNITY_TABLE>` | `bigbrain.l2.raw_salesforce_opportunities` | 708,239 rows. Refreshes ~hourly but **trails Salesforce by 60–75 min**. See §10.3. |
| D2 | `<L4_OPPORTUNITY_SNAPSHOT>` | `bigbrain.l3.fact_opportunities_daily` | **L3, not L4.** One row per opportunity per day; snapshot date column is `day`. 76.7M rows, rebuilt daily ~02:45 UTC. |
| D3 | Opportunity columns | `bigbrain.l4.dim_opportunities` | Field map in §10.2. |
| D4 | Ordered funnel stage list | Derived — **no stage dimension table exists** | §10.4. |
| D5 | Forecast category values | `DATA:"ForecastCategoryName"` | **Not a first-class column.** §10.5. |
| D6 | `<ACCOUNT_ARR_TABLE>` | `bigbrain.l4.dim_accounts.arr` | USD, annualised, account-level total. 327,552 populated; 263,486 paying accounts. |
| D7 | `<ACTIVITY_TABLE>` / `<GONG_TABLE>` | `bigbrain.l4.dim_activities` | One unified table — already contains Gong, SF email, SF events, Salesloft, deduplicated. §10.6. |
| D8 | `<PRODUCT_USAGE_TABLE>` | `bigbrain.l3.fact_accounts_ai_daily`, `bigbrain.l3.fact_users_ai_daily` | Both O2 option 1 and option 3 are buildable. §10.7. |
| D9 | `<SEATS_TABLE>` | `bigbrain.l4.dim_accounts` — `members_count` vs `seats` | Account-wide. Per-product alternative: `bigbrain.l4.dim_accounts_products_metrics`. |
| D10 | `<LEADS_TABLE>` | `bigbrain.l4.dim_leads` | 5.35M rows. §10.8. |
| D11 | `dim_salesforce_users` columns | `bigbrain.l3.dim_salesforce_users` **+** `bigbrain.l1.salesforce_user` | Timezone is **not** on the dim. §10.9. |
| D12 | Fiscal calendar | **Fiscal quarter = calendar quarter** | Verified on four dates spanning four quarters. Use `LAST_DAY(day,'quarter')`, or `days_till_end_of_quarter` on `fact_opportunities_daily`. |
| D13 | Salesforce instance URL | **Resolved 2026-09-08** | `https://monday.lightning.force.com/` — not in the warehouse, given directly by the requester. See §9 O9. |

### 10.2 Opportunity field map (D3)

Canonical table: **`bigbrain.l4.dim_opportunities`** (708,148 rows, one row per opportunity).

| Spec field | Column | Notes |
| --- | --- | --- |
| `opportunity_id` | `opportunity_id` | |
| `account_id` | `pulse_account_id` (NUMBER) / `monday_account_id` (TEXT, SF 18-char) | Join key to `dim_accounts` is `monday_account_id`. |
| `name` | `opportunity_name` | |
| `stage` | `stage` | §10.4 for values. |
| `close_date` | `close_date` (DATE) | On L2 this is `close_date` **TIMESTAMP_NTZ** — cast when diffing L2 against L3/L4. |
| `forecast_category` | `DATA:"ForecastCategoryName"::string` | §10.5. |
| `opportunity_arr` | `arr` (FLOAT) | Populated for all stages. USD. `recoginzed_arr` (note the typo) is closed-won realised ARR. |
| `owner_id` | `owner_id` / `monday_owner_id`; name in `owner_name` / `monday_owner_name` | |
| `recognized` | **no boolean** — use `recoginzed_arr > 0` | See warning in §10.10. |
| loss reason | `lost_reason` | |
| `source_type_auto` | **does not exist** | See §10.10 — renewals are identified differently. |
| `is_closed` / `is_won` | `is_closed`, `is_won`, `is_lost` (NUMBER 0/1) | |
| governed-pipeline gate | `is_gb = 1` | Apply to every query, or results include test/excluded opportunities. |

**`bigbrain.l3.scd_opportunities`** (8.3M rows) is a dbt SCD2 history of the same record —
`start_date`, `end_date`, `is_open`. Daily grain, so it cannot drive the hourly job, but it is
the right source for the §12 replay tests.

### 10.3 Hourly freshness (D1) — schedule caveat

`raw_salesforce_opportunities` covers every hour with no gaps, so an hourly job is viable.
But the newest record in the table was 73 minutes old when measured, so **each hourly run sees
Salesforce as it was 60–75 minutes ago**. A deal marked Closed Won at 11:50 is not visible until
roughly the 13:00 run.

This does not break the design — the §2 watermark approach tolerates it — but the requester
should know that Big-deal DMs lag reality by up to about two hours. If near-real-time matters,
the source has to change and that is a separate piece of work.

L3 and L4 are rebuilt once daily (~02:45 UTC), which is fine for the daily and weekly jobs and
unusable for the hourly one.

### 10.4 Funnel stage order (D4)

No stage dimension table exists anywhere in the warehouse, so the order below was derived from
the observed transition graph in `scd_opportunities` over the last 6 months and must be
hardcoded. Forward:backward transition ratios were 3557:12, 3906:184, 2307:168 and 2958:425
respectively — unambiguous at every step.

| Order | Exact stage string |
| --- | --- |
| 1 | `Pre Qualified` |
| 2 | `Qualified` |
| 3 | `Evaluation` |
| 4 | `Validation` |
| 5 | `Buying process` |
| 6 | `Closed Won` / `Closed Lost` (terminal) |

Exact-string corrections against §7:

- It is **`Pre Qualified`** — one space, no hyphen. `Pre-Qualified` matches nothing.
- **`Buying process`** has a lowercase `p`.
- `Validation` and `Buying process` were not in the spec's list at all, and both sit **after**
  `Evaluation`. `Buying process` sounds early but is the last stage before close.
- There is exactly **one** closed-lost stage, `Closed Lost` — no duplicate/disqualified variants.
  This retires the §7 concern about enumerating them.
- A single legacy `Qualification` row exists (n=1). Treat any unrecognised stage as unordered and
  skip the ordering test rather than crashing.

`Buying process` after `Validation` means the §7 BD_STAGE_CHANGE rule `order(new) > order('Qualified')`
notifies on moves into Evaluation, Validation and Buying process, which is the intent.

Reopens are real: in 6 months, 380 opportunities went `Closed Won → Buying process`, 210 went
`Closed Lost → Buying process` and 171 went `Closed Lost → Closed Won`. The §5.2 "once ever"
dedup key for BD_CLOSED_WON is therefore load-bearing, not theoretical.

### 10.5 Forecast category (D5) — naming trap

There is **no `forecast_category` column** on `dim_opportunities`, `fact_opportunities_daily` or
`scd_opportunities`. It lives only in the `DATA` variant, under two keys that disagree:

| `DATA:"ForecastCategoryName"` (label) | `DATA:"ForecastCategory"` (API value) | rows, 6 mo |
| --- | --- | --- |
| `Omitted` | `Omitted` | 68,355 |
| `Closed` | `Closed` | 40,541 |
| `Pipeline` | `Pipeline` | 12,665 |
| **`Best Case`** | `BestCase` | 1,902 |
| **`Commit`** | **`Forecast`** | 1,473 |

**Use `DATA:"ForecastCategoryName"::string` and match on `('Best Case','Commit')`.** Filtering the
API-value key for `'Commit'` returns zero rows forever — in that field Commit is stored as
`Forecast`. This is the single easiest way to ship a notification that silently never fires.

### 10.6 Activity and Gong (D7)

Use **`bigbrain.l4.dim_activities`** alone. It is a deduplicated union of Gong calls, Salesforce
emails, Salesforce events, Salesloft and several CS sources — so the spec's "either a Gong call or
any logged activity" is one table, not a union you assemble.

| Need | Column |
| --- | --- |
| Activity date | `activity_timestamp` (TIMESTAMP_NTZ), or `day` (DATE, partition key) |
| Kind | `activity_type` — `meeting`, `phone_call`, `email`; `activity_source` for finer detail |
| Opportunity-level join | `sf_opportunity_id` |
| Account-level join | `sf_account_id`, or `pulse_account_id` |

Both join levels exist on the one table, which is exactly what HY_NO_ACTIVITY_LAST_WEEK needs
("activity found at either level clears the flag").

**Do not use `fact_opportunities_daily.gong_calls_count`.** It is a point-in-time count of
currently-linked Gong records, not a cumulative total, and it goes *down* when records are
unlinked — observed dropping 20 → 19 → 18 on a single opportunity across three snapshots.
Diffing it would produce both false positives and false negatives.

`bigbrain.l3.dim_gong_calls` (569,225 rows) remains available if Gong needs to be isolated from
other activity, with `opportunity_id`, `pulse_account_id` and `effective_start_time`.

### 10.7 AI usage and CRM trials (D8) — O2 is unblocked

**Credit allowance and consumption both exist**, so O2 option 1 — the recommended primary signal
— is buildable today.

`bigbrain.l3.fact_accounts_ai_daily`, grain `day` × `pulse_account_id`:

| Need | Column |
| --- | --- |
| Cycle credit allowance | `cycle_renewable_cap` |
| Credits used this cycle | `cycle_renewable_used` |
| **Pre-computed ratio** | `pct_cap_used` — can exceed 1.0 |
| Pre-computed bucket | `cap_utilization_bucket` — `0-50`, `50-75`, `75-90`, `90-100`, `100+` |
| Over cap | `is_over_cap` |
| Guard | `is_active_cycle = 1` — cap columns are NULL otherwise |

`pct_cap_used >= 0.80` on the latest `day` implements option 1 directly.

**Per-user AI grain also exists**, so option 3 (adoption breadth) is buildable:
`bigbrain.l3.fact_users_ai_daily`, grain `day` × `pulse_account_id` × `pulse_user_id`, with
`daily_credits_consumed`, `daily_consumed_tokens`, `active_feature_count`. The table is **dense** —
every eligible user has a row every day — so any "using AI" count must filter
`daily_credits_consumed > 0 OR daily_consumed_tokens > 0`, or every user in the account counts.

Per-feature detail: `bigbrain.l3.fact_users_ai_features_daily`. Raw billing cycles:
`bigbrain.l2.accounts_ai_billing_cycles`.

The recommendation in §9 O2 therefore stands unchanged and is now known to be implementable. It
still needs the requester's decision, not more discovery.

CRM trial activation for EX_CRM_TRIAL_ACTIVATED was **not** separately confirmed and is the one
remaining data gap in this category — see §9 O10.

### 10.8 Leads (D10)

**`bigbrain.l4.dim_leads`** (5,353,453 rows).

| Need | Column |
| --- | --- |
| `lead_id`, `lead_name` | `lead_id`, `lead_name` |
| Current owner | `lead_current_owner_id`, `lead_current_owner_full_name` |
| Status | `lead_status` |
| Source | `lead_source`, `lead_source_type`, `first_lead_source`, `lead_channel` |
| Received date | `received_timestamp`, `initial_received_timestamp`; flag `is_received` |
| Attempting | `attempting_timestamp`, `is_attempting` |
| Contacted | `contacted_timestamp`, `is_contacted`, `first_contacted_owner_id` |
| Account ARR | `account_current_arr` |

Status values confirmed — **`Received` and `Attempting` are literal values** (28,254 and 72,554 in
the last 3 months). The full set is `Unqualified`, `Attempting`, `Drop-Off`, `Received`, `New`,
`Eligible for Distribution`, `Qualified`, `Contacted`, `Meeting Scheduled`, `Nurturing`.

For LD_UNCONTACTED_OPEN_LEADS the "no activity **by the current owner**" qualifier is satisfiable:
compare `lead_current_owner_id` against `first_contacted_owner_id` / `contacted_owner_details`,
or join `dim_activities` filtered to the current owner. Note `dim_leads` has no plain company-name
column — use `domain`, or pull `Company` from the `DATA` variant.

Watch out for `is_ai_agent` / `ai_agent_name`: AI SDR agents own leads and will appear as "reps" in
any per-rep grouping unless excluded.

### 10.9 Users, hierarchy and timezone (D11) — timezone is not where §3 assumed

`bigbrain.l3.dim_salesforce_users` (a pre-filtered view over `scd_salesforce_users`) gives:
`user_id`, `email`, `full_name`, `is_active`, `manager_id`, `business_role`, `sub_region`,
`business_region`, `team`, `segment`, `office`, `office_region`, `country`.

`manager_id` self-joins to `user_id` to build the §1 manager picklist (every manager of a
current opportunity owner) and to evaluate the manager filter itself — both flat lookups now,
not a walk. `is_active` still matters for the §1 active-reps-only grouping rule.

**It has no timezone column.** Timezone is only in the raw object:

```sql
bigbrain.l1.salesforce_user.DATA:"TimeZoneSidKey"::string
  -- join: l1.salesforce_user.salesforce_id = l3.dim_salesforce_users.user_id
```

Populated for 3,120 of 3,120 active users. `ManagerId` is present for 3,090 of 3,120 — the 30
without one are the top of the tree (no manager to filter on, which is fine now that scope
doesn't walk the chain — see §1).

**The DST problem §3 anticipated is real and bigger than expected.** The distribution of
`TimeZoneSidKey` for active users:

| Value | Users |
| --- | --- |
| `GMT` | 1,726 |
| `America/New_York` | 1,101 |
| `Europe/Dublin` | 81 |
| `Asia/Jerusalem` | 75 |
| `Australia/Sydney` | 27 |
| `Europe/London` | 26 |
| everything else | < 10 each |

`GMT` is 55% of all active users and is a **fixed UTC+0 offset that does not observe BST**. Only 26
users have explicitly chosen `Europe/London`. Taken literally, every GMT user would receive their
09:00 daily DM at 10:00 local for the seven months of British Summer Time.

`GMT` is plainly the Salesforce org default rather than a deliberate choice: 1,660 of those 1,726
users have neither `office_region` nor `country` populated. **Recommendation — treat `GMT` as
"unset" and apply the §3 `Europe/London` fallback to it.** That is consistent with the rule §3
already specifies and fixes BST for the UK majority.

The residual risk is small but real: a minority of GMT-defaulted users are demonstrably not in the
UK (9 APAC, 7 Brazil, 5 Mexico, 4 US, 4 Australia and a long tail). Where `country` or
`office_region` is populated, prefer a timezone derived from it before falling back. Logged as O11.

### 10.10 Corrections to earlier sections

Four things stated in §1–§7 do not survive contact with the warehouse.

1. **`Source Type (Auto)` does not exist.** §7's RN_UPCOMING_RENEWALS population rule — "the Source
   Type (Auto) field contains `renewal`" — cannot be implemented. `DATA:"Source_Type_Auto__c"` is
   NULL on every row, and the real `source_type` column holds only `Inbound` / `Outbound`.
   Renewals are identified instead by `is_renewal_opportunity_by_type = 1`, equivalently
   `logic_type IN ('Flat Renewal','Expansion On Renewal','Downgrade On Renewal')`. §7 has been
   corrected in place.

2. **`renewal_stage` gives the exclusion §7 asked for.** The full value set is `Pre Renewal`,
   `Renewal Kickoff`, `Renewal Proposal`, `Renewal Negotiation`, `Renewal Execution`, `Closed Won`,
   `Churn`. "Excludes renewals already closed, already renewed, or churned" is therefore
   `renewal_stage NOT IN ('Closed Won','Churn') AND is_closed = 0`.

   Open renewals today: 63,678 `Pre Renewal` ($241.0M `arr_to_renew`), 1,582 `Renewal Execution`,
   1,542 `Renewal Kickoff`, 700 `Renewal Negotiation`, 568 `Renewal Proposal`.

3. **Stage strings differ from §7** — see §10.4.

4. **Forecast category `Commit` is stored as `Forecast`** in the API-value key — see §10.5.

One further judgement call for the requester rather than a correction: `arr_to_renew` on the
renewal opportunity is the ARR actually at stake in that renewal, and is arguably a better filter
for RN_UPCOMING_RENEWALS than the account's total ARR that §7 specifies. Raised as O12.

### 10.11 Scope filter data (added 2026-09-08 — supersedes the §1 hierarchy model)

All four picklists are derived from open opportunities on `bigbrain.l4.dim_opportunities`
(`is_closed = 0`), joined to `bigbrain.l3.dim_salesforce_users` twice — once for the owner, once
self-joined on `manager_id` for the owner's manager.

**Region and sub-region come straight off the opportunity, no owner join needed.**
`business_region` and `business_sub_region` on `dim_opportunities` already match the resolved
owner's region in 99.7%+ of rows (spot-checked against `monday_owner_business_region`, which
exists as a separate column and agrees with `business_region` in every case checked bar two rare
edge combinations) — so there was no need to go back to `dim_salesforce_users` for these two.

Real region → sub-region pairs on currently-open opportunities:

| Region | Sub-regions |
| --- | --- |
| `NAM` | `US` |
| `EMEA` | `UK&I`, `FR`, `DACH`, `Benelux`, `Nordics`, `IL`, `ME`, `CEE/CIS`, `Africa`, `Iberia`, `IT`, `IL (&GR)`, `Greece`, `EMEA - Multiple` |
| `APJ` | `ANZ`, `SEA`, `India`, `North Asia`, `Japan` |
| `LATAM` | `Brazil`, `Mexico`, `SoLA`, `NoLA`, `MCLA`, `Rest of the world` |
| `Global` | `Multiple` |

**Role bucketing.** `monday_owner_business_role` on the same table, joined to real open-opp
owner counts, 2026-09-08:

| Raw role | Open opps | Distinct owners | Bucket |
| --- | --- | --- | --- |
| `CPM` | 33,938 | 86 | `Partner` |
| `AE` | 20,741 | 222 | `AE` |
| `Territory AE` | 10,123 | 97 | `AM` |
| `AM` | 7,745 | 104 | `AM` |
| `People Manager` | 7,002 | 105 | excluded |
| `Scale AM` | 6,045 | 32 | `AM` |
| `Commercial AM` | 5,577 | 46 | `AM` |
| `Partner` | 3 | 3 | `Partner` |
| everything else (Hybrid Overlay, Renewal, SDR, Overlay, High-Touch CSM, GTM/RevOps/Program Manager, and ~20 more single-to-double-digit-owner roles) | — | — | excluded |

The `AE`/`AM`/`Partner` bucketing is the requester's explicit instruction, given twice — first
naming `Commercial AM`, `Scale AM` and `Territory AE` as folding into `AM`, then naming `CPM` as
folding into `Partner` alongside the literal `Partner` role, and confirming the Role picklist
should show **only** those three options. `People Manager` — the largest single excluded role by
owner count (105, i.e. opportunities still sitting on a manager's own book) — is deliberately not
a fourth option; it is simply not selectable, though it, like every excluded role, still
participates in cascading so selecting a Region or Manager isn't skewed by silently dropping
those rows first.

**Manager.** Self-join `dim_salesforce_users` on `manager_id`, restricted to managers of a
current open-opportunity owner:

```sql
SELECT DISTINCT m.user_id AS manager_id, m.full_name AS manager_name
FROM bigbrain.l4.dim_opportunities o
JOIN bigbrain.l3.dim_salesforce_users u ON u.user_id = o.monday_owner_id
JOIN bigbrain.l3.dim_salesforce_users m ON m.user_id = u.manager_id
WHERE o.is_closed = 0
ORDER BY m.full_name
```

**195 distinct managers**, all with `full_name` populated. Every one carries the role
`People Manager` — expected, and irrelevant to the Role picklist above since Manager and Role
are independent filter dimensions (a `People Manager`'s own book, if they have one, would be
excluded by the Role filter regardless of who manages them).

**The combined query the app actually runs** (`backend/handlers/get_scope_options.js`) is a
single pass returning one row per distinct `(region, sub_region, bucketed_role, manager_id,
manager_name)` combination, bucketing role with a `CASE` and using `LEFT JOIN` on the manager so
an owner with no manager still contributes a region/sub-region/role row (with a null manager)
rather than being dropped:

```sql
SELECT DISTINCT
  o.business_region AS region,
  o.business_sub_region AS sub_region,
  CASE
    WHEN o.monday_owner_business_role = 'AE' THEN 'AE'
    WHEN o.monday_owner_business_role IN ('AM', 'Commercial AM', 'Scale AM', 'Territory AE') THEN 'AM'
    WHEN o.monday_owner_business_role IN ('Partner', 'CPM') THEN 'Partner'
    ELSE 'Other'
  END AS role,
  m.user_id AS manager_id,
  m.full_name AS manager_name
FROM bigbrain.l4.dim_opportunities o
JOIN bigbrain.l3.dim_salesforce_users u ON u.user_id = o.monday_owner_id
LEFT JOIN bigbrain.l3.dim_salesforce_users m ON m.user_id = u.manager_id
WHERE o.is_closed = 0
```

`'Other'` is a real bucket value in this result set — it is what makes cascading correct for
owners in an excluded role — but the Role picklist's own option list is hardcoded to offer only
`AE`, `AM` and `Partner`, never `Other`.

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

1. **Scope filters** — pick a few real filter combinations and confirm the matched owner set is
   exactly right: one region alone, one region + one role together (confirm AND across
   dimensions), two sub-regions in the same picklist (confirm OR within a dimension), and all
   four filters left empty (confirm it matches everyone, not no one — an empty-array bug here
   is the kind that silently sends nothing to anyone). Separately confirm the picklists
   cascade: selecting a region in the z2h page narrows the sub-region options to that region's
   real values, and a stale selection elsewhere clears rather than saving invisibly (§1, §10.11).
2. **Dedup** — replay a day of real opportunity history through the Big-deals job, using
   `bigbrain.l3.scd_opportunities` as the history source (§2). Assert that
   **every** opportunity reaching Closed Won produced exactly one notification and no
   accompanying stage, close-date or forecast-category message. This is the requester's headline
   requirement; test it explicitly rather than by inspection.
3. **Once-ever keys** — replay an opportunity that was won, reopened and re-won. Assert one
   BD_CLOSED_WON. Real examples exist and are not rare: over 6 months, 380 opportunities went
   `Closed Won → Buying process`, 210 went `Closed Lost → Buying process` and 171 went
   `Closed Lost → Closed Won`.
4. **Volume sanity** — partly done already. Measured over the 30 days to 2026-09-08 from
   `scd_opportunities`, at the default $10,000 floor, **globally across all owners**:

   | Big-deal notification | Events / 30 days | ≈ per day |
   | --- | --- | --- |
   | BD_CLOSED_WON | 546 | 18 |
   | BD_CLOSED_LOST | 606 | 20 |
   | BD_STAGE_CHANGE | 1,199 | 40 |
   | **BD_CLOSE_DATE_CHANGE** | **3,720** | **124** |

   §12's original guess was that Big deals would be the offender. It is more specific than that:
   **BD_CLOSE_DATE_CHANGE alone is roughly 3× every other Big-deal notification combined.** §7
   specifies "any change to close date, no minimum shift", and close dates are edited constantly —
   often in the same save as a stage advance, where §5.1 precedence will suppress it, but far from
   always.

   These are org-wide totals; a given recipient sees only whatever their own §1 scope filters
   select. Someone with broad or empty filters could plausibly receive 10–30 close-date DMs a
   day from this one notification; someone with a narrow region/role/manager combination sees
   far less.

   **Resolved as O13, 2026-09-08 — accepted as-is.** The requester confirmed this is fine because
   delivery is per-recipient scope, not the global total above; no minimum-shift threshold or
   digest change was requested. Ship BD_CLOSE_DATE_CHANGE per §7 unchanged. If real per-recipient
   volume turns out higher than expected once the scope resolver is live, revisit then rather
   than pre-emptively narrowing it now.

   Other measured populations, for scale: RN_UPCOMING_RENEWALS matches **9,280** renewals in a
   rolling 60-day window ($126.3M `arr_to_renew`) before the ARR threshold;
   EX_MEMBERS_EXCEEDING_SEATS matches **16,499** paying accounts (4,497 at ARR ≥ $10k), and §7
   applies no ARR slider to the Expansion category at all — worth revisiting.

   Still to measure per-recipient once the scope resolver exists: pick a rep, a front-line manager
   and a director and count actual daily sends for each.
5. **Threshold behaviour** — confirm an opportunity just under the slider value sends nothing and
   just over sends once.
6. **Timezone** — verify a user with no Salesforce timezone falls back to `Europe/London`, that
   a user with the literal value `GMT` also falls back rather than being treated as fixed UTC+0
   (§10.9 — this affects 55% of active users), and that no recipient double-sends or is skipped
   across a BST transition.
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
