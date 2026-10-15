---
name: billdog-migration
description: Migrate an app from RevenueCat, Superwall, Adapty, Mixpanel, Amplitude, OneSignal, Braze, Typeform or SurveyMonkey to BillDog. Use when the user asks to switch away from one of those, compare it to BillDog, or move paywall/analytics/messaging/survey code across.
---

# Migrating to BillDog

Two halves that are done separately, and confusing them is the usual mistake.

## 1. Code — rewrite the call sites

Find every call into the competitor SDK first (imports, configure/init, user
identify, purchase, restore, paywall presentation, event tracking) and list
them before changing anything. Then replace each with the BillDog equivalent
from the `billdog-integration` skill and https://billdog.io/SKILL.md.

BillDog's migration CLI, which automates this as a `git apply`-able patch, is
not published yet — do not tell the user to `npx` it.

Supported sources: RevenueCat, Superwall, Adapty, Mixpanel, Amplitude,
OneSignal, Braze, Typeform, SurveyMonkey.

To recreate the catalog, read the old products/entitlements/offerings from the
codebase or the user, then create them with the MCP tool `apply_setup_plan`.

## 2. Data — the dashboard

Customers, subscriptions and entitlements move through the importer at
**https://billdog.io/dashboard/import**. Neither the code rewrite nor the MCP
server moves customer data; do not offer to.

## Review the change before you finish

In particular:

- Imports must resolve to `@billdog.io/…` (npm) or `io.billdog:…` (Maven).
  Neither `@billdog/…` nor `com.billdog:…` exists.
- Competitor concepts do not always map one-to-one. List anything you could not
  map for the user instead of guessing — those are theirs to decide.
- A change that looks complete is not a change that compiles. Build it.

## Then verify

Follow the verification checklist in the `billdog-integration` skill. A
migration that builds can still be missing a placement or an
identify call.
