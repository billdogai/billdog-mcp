---
name: billdog-experiments
description: Run BillDog paywall A/B experiments and feature flags. Use when the user asks to test two paywalls against each other, start, pause, end or ship the winner of a BillDog experiment, read experiment results, or create, roll out, turn off or read a BillDog feature flag.
---

# BillDog experiments and feature flags

Both are managed over the BillDog MCP server. Anything that changes what live
users see needs the user's go-ahead first; ask before each such call.

## Paywall A/B experiments

An experiment shows a different paywall per variant at the same place in the
app. The app code does not change: the SDK resolves the placement and BillDog
assigns the variant.

1. **Look first.** `list_placements`, `list_paywalls` and `list_experiments`.
   Only one experiment can run on a paywall at a time.
2. **Every variant's paywall must be published** before the experiment can
   start. Build one with `list_templates` → `create_paywall_from_template` or
   `design_paywall`, then `preview_paywall`, then `publish_paywall` (ask first).
3. **Create** with `create_experiment`. It is saved as a draft and serves
   nothing. The first variant is the control unless one says `is_control`.
   Traffic splits evenly unless every variant gives `traffic_percent`, which must
   add up to 100.
4. **Start** with `start_experiment` (ask first). On a live project the start
   waits for an owner or admin to approve it: when the result says
   `awaiting_approval`, give the user its `approve_url`.
5. **Read results** with `get_experiment_results`: users, conversions, revenue
   per user, lift against the control, p-value and a verdict of `winner`,
   `loser`, `keep_running` or `no_data`. A result is significant only at
   p < 0.05 with 100+ users per arm. Do not call a winner on `keep_running`.
6. **Finish** with `stop_experiment` (ask first):
   - `pause` sends everyone back to the control and can be started again.
   - `end` does the same and cannot be restarted.
   - `ship_winner` with `winner_variant_id` serves the winner to everyone.

Results read production unless you pass `sandbox: true`. An experiment tested
only on a development build will show `no_data` in production.

## Feature flags

1. **Look first.** `list_feature_flags`. Reuse an existing key rather than
   creating a near-duplicate.
2. **Create** with `create_feature_flag`. It is created off: every user gets
   false. For a multivariate flag give two or more `variants` whose
   `rollout_percentage` adds up to 100, each with an optional JSON payload.
3. **Wire it into the app** by key. Look up the flag getter for the platform in
   https://billdog.io/docs/feature-flags or with `get_sdk_integration_guide`;
   do not guess a method name. Handle the "off" value as the default path.
4. **Roll out** with `update_feature_flag` (ask first). Turning it on needs a
   `rollout_percentage` above 0; 100 means everyone. Start small when the user
   has not said otherwise. On a live project the change may wait for approval.
5. **Check exposure** with `get_feature_flag_results`: users and evaluations per
   value over the last 1–90 days, counted from the `$feature_flag_called` events
   the SDKs send. Zero evaluations after a release usually means the app does
   not read the flag yet, or reads a different key.

Audience rules (release conditions) are edited in the BillDog dashboard, not over
MCP. If the user asks for targeting, say so and link
https://billdog.io/docs/feature-flags.
