---
name: billdog-analyst
description: Answer questions about a BillDog project's numbers - revenue, MRR and ARR, churn, renewals, billing issues, trial conversion, LTV, paywall and placement conversion, revenue by country, product, offering, paywall or experiment arm, funnels, retention and event trends. Use when the user asks how their app, subscriptions or paywalls are doing, why revenue or conversion changed, or wants a weekly recap. Read-only.
---

# Analysing a BillDog project

Every number comes from `run_analytics_query` and a few list tools on the BillDog
MCP server. Nothing here changes the project.

## Before you answer

- **Pick the right (source, metric).** Use the table below. If a question does not
  fit it, call `discover_schema` once: it lists every valid pairing with a one-line
  hint, and an unsupported pairing fails.
- **Ranges** are `{ start, end }` as full UTC timestamps ending in `Z`, for example
  `2026-10-15T00:00:00Z`. Use the last 30 days when the user names no range, and say so.
  End at today's 00:00 UTC for whole days, or at the current time to include today
  so far, and say which.
- **Environment.** Queries read production unless you pass `sandbox: true`. Every
  result says which one it read in `environment`. If production comes back empty or
  all zero, retry with `sandbox: true` and tell the user the figures are test data.
  A project that is still being integrated often has only sandbox data.
- **Freshness.** `revenue` figures come from a daily rollup. `revenue` / `totals`
  carries `meta.freshness`: days after `final_through` are provisional or missing.
  Other reports do not carry it, so when the range ends in the last two days, run
  `revenue` / `totals` once for the same range and say which days are not final, so a
  short week is not read as a drop.
- **Every source and metric in the table below is valid**, including
  `revenue_segments`, `subscription_charts`, `subscription_cohorts`, `trial_cohorts`
  and `ltv_prediction`. If your copy of the tool's schema lacks one, it is out of
  date: `discover_schema` gives the current list.

## Which query answers which question

| Question | Query |
|---|---|
| Revenue, refunds, MRR, ARR, active subscriptions, churn rate, trial conversion, ARPU, new customers, all in one call | `revenue` / `totals` |
| This period against the last one | `revenue` / `compare` with `range.compare_to: "previous_period"` (or `"previous_year"`) |
| New, expansion, contraction, churned and net MRR | `subscriptions` / `mrr_movement` |
| Revenue by country, store, platform, product, offering, paywall or placement | `revenue_segments` / `breakdown` with `dims: ["<one of those>"]` |
| Revenue by experiment arm | `revenue_segments` / `breakdown`, `dims: ["experiment_variant"]`, `filters.experiment_id` |
| Top products | `products` / `top_n`, or `revenue_segments` / `top_n` with `dims: ["product"]` |
| Paywall impressions, purchases and conversion per paywall | `paywalls` / `breakdown` (grouped by paywall) |
| Presented, started, purchased | `paywalls` / `funnel` |
| Trial conversion by trial-start week or month | `revenue` / `trial_cohorts` with `range.granularity` |
| Renewal rate, billing issues and recovery, plan changes, ARPPU | `subscription_charts` / `totals` (or `timeseries`) with `filters.chart`: `renewal_rate`, `billing_issues`, `plan_changes` or `arppu` |
| Subscription retention, realized and predicted LTV | `subscription_cohorts` / `retention`, `breakdown` or `ltv_prediction` |
| A funnel across app events | a saved view first (`list_saved_views` → `get_saved_view_results`); else `events` / `funnel` with `filters.event_names` in order |
| Event volume, trends, top events | `events` / `timeseries`, `top_n` or `breakdown`; `list_event_names` for exact names |
| A paywall A/B test | `get_experiment_results` (see the billdog-experiments skill) |

Two easy mistakes: `revenue` / `breakdown` is revenue **per day**, and
`revenue` / `top_n` is the top **days**. Neither splits by product or country.
Use `revenue_segments` for that.

## Reading the result

- **Check `meta` before you report.**
  - `meta.dropped_dims` means a dimension was not valid for that source and the
    report was grouped by the default instead. For example, `paywalls` does not take
    `placement`; use `revenue_segments` with `dims: ["placement"]`.
  - `meta.note` with empty `rows` means the query needs more input (the
    experiment-arm split needs `filters.experiment_id`). Empty rows are not zero.
- **Name the measure you quote.** "Net" means different things in different
  sources: `revenue` / `totals` has `net_revenue_usd` and `proceeds_usd`, and
  `revenue_segments` has `revenue_net_of_refunds_usd`. Use the column's label and
  never add or compare figures from different sources as if they were one measure.
- **Turn ids into names.** Paywall, placement and offering segments come back as
  UUIDs. Resolve them with `list_paywalls`, `list_placements` and `list_offerings`
  before showing them. An id that is not in the list belongs to a deleted item: show
  the short id and say so.
- **Paywall money comes from `revenue_segments`.** The `revenue_usd` column of
  `paywalls` / `breakdown` is each customer's first verified purchase credited to the
  paywall, without the renewals that followed, so it can be far below
  `revenue_segments` by `paywall`.
  Quote `paywalls` for impressions, purchases and conversion rate only.
- **Not every wall is a paywall.** `paywalls` reports include sign-in and other walls.
  `list_paywalls` gives each one's `wall_type`; leave out walls that sell nothing when
  ranking conversion.
- **"Unknown" is a real share.** It holds transactions with no attribution for that
  dimension. Report its share rather than dropping it. In a period comparison, check
  "Unknown" in both periods first: if the earlier period's revenue sits there, every
  named segment shows false growth.
- **Rates on small numbers.** Put the counts beside every rate. A 100% churn rate
  on four subscriptions is four people.
- **Funnels count identified users only.** An `events` funnel can show far fewer
  users than the raw event counts from `list_event_names`.
- **Experiments.** `experiments` / `ab_results` is an uncorrected z-test. For a
  verdict, use `get_experiment_results`, and do not call a winner on
  `keep_running`.

## Recipes

When the client offers the BillDog MCP prompts, these match them:
`weekly_revenue_recap`, `funnel_drop_off_analysis`, `churn_root_cause` and
`error_session_review`.

**Weekly recap.** `revenue` / `totals` and `revenue` / `compare` for the last 7
days, `subscriptions` / `mrr_movement`, `revenue_segments` / `top_n` by product.
Write three short paragraphs: revenue and its change, MRR new versus churned versus
net, top products. End with one observation the user can act on.

**Why did revenue change?** Compare the two periods with `revenue` / `compare`, then
find where the change sits with `revenue_segments` / `compare` on `product`,
`country` and `platform`, one dimension per call. Check `subscription_charts`
`renewal_rate` and `billing_issues` before blaming the paywall.

**How are my paywalls doing?** `paywalls` / `breakdown` for conversion per paywall,
`paywalls` / `funnel` for where buyers stop, and `revenue_segments` / `breakdown`
by `paywall` and `placement` for the money. Resolve the ids to names.

**Why are customers churning?** `subscriptions` / `mrr_movement` for its size,
`subscription_charts` `renewal_rate` and `billing_issues` to separate voluntary
from involuntary churn, `subscription_cohorts` / `retention` to find the weakest
cohorts, and `revenue_segments` / `breakdown` by `product`.

## How to answer

Lead with the answer. Then give the range, the environment and any days not yet
final. Show only the figures the question needs, with their counts. Do not invent
industry benchmarks. Suggest a change to the project only when the user asks; making
one belongs to the other BillDog skills and needs the user's go-ahead.
