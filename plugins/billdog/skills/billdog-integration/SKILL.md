---
name: billdog-integration
description: Wire the BillDog SDK into an app, or diagnose one that is already wired. Use when the user asks to add BillDog, set up paywalls or subscriptions with BillDog, add BillDog analytics/surveys/messaging, fix a BillDog integration that shows nothing, or asks why a BillDog paywall is not appearing.
---

# Integrating BillDog

BillDog is paywalls, analytics, messaging, surveys, journeys and auth behind one
SDK and one dashboard.

The exhaustive per-platform reference — every install line, every docs URL — is
served at **https://billdog.io/SKILL.md**. Fetch it when you need a platform this
file does not cover. What follows is the procedure and the things that are easy
to get wrong.

## Diagnose before you change anything

A project is already wired if any of these exist: `.billdog.json`,
`@billdog.io/*` in `package.json`, `io.billdog:billdog-*` in a Gradle file, or
`billdog-ios-spm` in `Package.resolved`.

If it is wired, the problem is usually a missing placement or a missing identify
call — not a missing install. Diagnose first: call the MCP tool
`get_setup_status`, which returns exactly what is missing with dashboard links.

There is no published BillDog CLI on npm; `npx billdog` will fail. Without the
MCP server, verify by hand using the checklist at the end.

## Identifiers

Getting these wrong is the single most common failure, because neither fails
until install or compile time:

| | Correct | Does not exist |
|---|---|---|
| npm scope | `@billdog.io/…` | `@billdog/…` |
| Maven group | `io.billdog:…` | `com.billdog:…` |

- **Web** — `npm install @billdog.io/web`
- **React Native** — `npm install @billdog.io/react-native`
- **Expo** — `npm install @billdog.io/expo`
- **iOS** — SPM `https://github.com/billdogai/billdog-ios-spm`, products
  `BillDogFull` (everything) or `BillDogEng` (engagement only). Different
  modules, different public surfaces — pick one and import only that.
- **Android** — `io.billdog:billdog-core`, `io.billdog:billdog-analytics`,
  `io.billdog:billdog-auth`, and siblings. Entry point `io.billdog.core.BillDog`,
  a Kotlin `object`.

## Configure

The web SDK is a **default-exported singleton**. There is no `new BillDog(...)`
and no `init()`:

```ts
import BillDog from '@billdog.io/web';

BillDog.configure({
  apiKey: process.env.BILLDOG_API_KEY,
  environment: 'production',
});

// When your user signs in:
await BillDog.logIn('user_123');
```

## Placements

A placement is the named spot where a paywall can appear. Code references the
**placement identifier**, never a paywall id — that is what lets the paywall
change without a release. Create one under Paywalls → Placements.

A project with a valid key and an installed SDK but no placement is wired
correctly and still shows nothing. That is the most common "it doesn't work".

## Critical rules

- **Public keys in client code. Never a secret key.** If the user offers a
  secret key for a mobile or web app, say so and ask for the public one.
- **Never commit a key.** If there is no ignored env file, create one and add it
  to `.gitignore` in the same change.
- **Configure once, early** — before any other SDK call.
- **Identify before fetching a paywall**, or the user is anonymous and targeting
  will not match.
- **Do not invent identifiers.** If a package, class or method is not in this
  file or at https://billdog.io/SKILL.md, look it up. A plausible guess here
  costs a failed build in someone else's repo.

## Verify before reporting success

1. The SDK package is in the dependency manifest.
2. `configure` runs before any other BillDog call.
3. A customer id is set before any paywall fetch.
4. A placement identifier in code matches one in the dashboard.
5. Events appear under Analytics → Live within a minute.

## Reading and setting up the project over MCP

This plugin ships the BillDog MCP server (`https://api.billdog.io/mcp`). The
first call opens "Sign in with BillDog" in the browser; the user picks one
project and whether you may only read, or also create and publish.

- **Inspect instead of asking:** `get_setup_status`, `list_products`,
  `list_entitlements`, `list_offerings`, `list_paywalls`, `list_placements`,
  `resolve_placement_preview`, `get_recent_sdk_activity`.
- **Set up the catalog:** prefer one `apply_setup_plan` call over many single
  writes. New products, offerings and placements start inactive.
- **Paywalls:** `list_templates` → `create_paywall_from_template` or
  `design_paywall` → `update_paywall` → `preview_paywall` → `publish_paywall`.
  Publishing and anything that changes what live users see needs the publish
  permission; ask the user before you do it.

Customer contact details are never returned.

