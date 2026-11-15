---
name: billdog-headless
description: Build a custom (bring-your-own-UI, headless) paywall on BillDog - fetch the offering a placement selects for this user, draw it with the app's own UI (SwiftUI, Jetpack Compose, React Native, Expo, Flutter, web, or a game engine over REST), buy a package, and handle holdout, withheld, gated and unknown placements and impressions correctly. Use when the user wants their own paywall screen instead of a BillDog-designed one, mentions headless or BYO-UI paywalls, or integrates BillDog into Unity, Unreal, Godot, a TV or desktop app without an SDK.
---

# Headless (bring-your-own-UI) paywalls

In headless mode the server decides **what** to sell to this user (the offering,
from the placement, its campaigns and experiments) and the app decides **how** to
show it. The app never receives a designed paywall: `paywall` is always null.

The full guide with complete samples for every platform is
https://billdog.io/docs/headless-paywalls (Markdown:
https://billdog.io/docs/headless-paywalls.md). Use it for anything this file does
not cover; do not guess an SDK method.

## The rules, on every platform

1. **Use the placement identifier the project has.** Read it with `list_placements`
   over the BillDog MCP server, or ask. An unknown placement is 404
   `PLACEMENT_NOT_FOUND`, surfaced as a typed error. It never falls back to another
   offering, so show nothing and report the typo.
2. **`offering` is the selection, and it can be null.** Null means nothing was
   selected for this user: show nothing. Do not take an entry from `offerings`
   instead, and do not fall back to the current offering.
3. **Read `presentation` before drawing.** All of these arrive as a normal success:
   - `holdout: true`: this user is in a holdout. Show nothing.
   - `withheld: true` (sent only when true), with `withheld_reason` `holdout` or
     `audience_excluded`: `offering` is null on purpose.
   - `gated: true` with `required_entitlement`: reaching this screen needs access,
     and in headless mode the app owns that gate. Check the entitlement and let a
     user who has it through.
4. **Track only what you showed.** Call `trackPaywallPresented(offering, placement)`
   when the screen appears and `trackPaywallDismissed(offering, placement)` with the
   same offering when it goes away. Never track for a holdout, a withheld result or
   a null offering: a holdout is measured by the absence of an impression. No
   `paywall_id` is sent.
5. **Fetch when you are about to show it.** A read that resolves a running
   experiment or campaign records the exposure and counts the campaign impression
   (and its frequency cap) whether or not the screen appears. Don't prefetch screens
   the user may never reach.
6. **Unlock from entitlements, not from the purchase call alone.** After a purchased
   or restored result, read customer info and check the entitlement the screen
   sells.
7. **Prices come from the store product on the package**, not from the offering's
   metadata. Never write a price into copy.

## Platform API

| Platform | Fetch | Result | Buy / restore | Unknown placement |
|---|---|---|---|---|
| iOS | `try await BillDog.shared.getOffering(placement:)` | `PlacementOffering`: `offering`, `presentation`, `shouldPresent` | `BillDog.shared.purchase(package:)` → `PurchaseResult`; `BillDog.shared.restore()` | throws `BillDogError.placementNotFound` |
| Android | `BillDog.getOffering(placement)` (suspend; `BillDog` is a Kotlin `object`) | `PlacementOffering` with `shouldPresent` | `BillDog.purchase(activity, pkg)` → `PurchaseResult`; `BillDog.restorePurchases()` → `RestoreResult` | throws `BillDogException`, `error` is `BillDogError.PlacementNotFound` |
| React Native, Expo | `BillDog.getOffering(placement)` | `OfferingResult`: `offering`, `presentation`, `placement` | `BillDog.purchasePackage(pkg)` and `BillDog.restore()` never throw; switch on `result.type` | rejects with `PlacementNotFoundError` |
| Flutter | `BillDog.shared.getOffering(placement)` (never throws) | sealed `OfferingResult`: `OfferingAvailable`, `OfferingHoldout`, `OfferingWithheld`, `OfferingNotConfigured`, `OfferingFailed` | `BillDog.shared.purchasePackage(pkg)`; `BillDog.shared.restorePurchases()` | `OfferingFailed` whose `error` is a `PlacementNotFoundError` |
| Web | `BillDog.getOffering(placement)` on the default export of `@billdog.io/web` | `OfferingResult`: `offering`, `presentation`, `placement` | `BillDog.purchaseProduct(productId)` hands the visitor to Stripe checkout; the purchase is recorded when they return | throws `PlacementNotFoundError` from `@billdog.io/core` |

Platform notes:

- **iOS and Android:** `shouldPresent` already folds rule 3 in: it is true only for a
  non-null offering that is neither in a holdout nor withheld.
- **React Native and Expo:** a package's price and name are `pkg.product?.priceString`
  and `pkg.product?.displayName` (there is no `storeProduct`, and no `status` field
  on the result). Expo is the React Native SDK re-exported and needs a development
  build; the native module is not in Expo Go. In a `useEffect`, keep the offering you
  showed in a local variable for the cleanup's `trackPaywallDismissed`, because the
  cleanup closes over the first render's state.
- **Flutter:** switch on the sealed result; `OfferingAvailable` is the only case that
  draws.
- **Web:** pass the product's `id` to `purchaseProduct`.

## Without an SDK: REST

For Unity, Unreal, Godot, TV or desktop apps. Official Unity, Roku, Capacitor,
Cordova and Kotlin Multiplatform SDKs are not released yet, and Roku Pay purchases
are not supported.

1. **Device proof, once per install.** Generate and store with the app's secrets:
   - device id: `$BillDogAnonymousID:` followed by a random UUID;
   - device key: `bdk_` followed by 32 random bytes in base64url without padding
     (43 characters).
   
   Send them as `X-BillDog-Device` and `X-BillDog-Device-Key` on every request.
   BillDog binds the key the first time it sees the id. Never generate a new key for
   an id you already sent. A 401 `PROOF_INVALID` means the pair is malformed,
   unknown or revoked: make a new pair (a new anonymous customer, who gets purchases
   back by restoring).
2. **Send the public key as `X-BillDog-API-Key` on every call.** A JSON body
   `api_key` is accepted as a fallback, and the header wins when both are sent.
   Never put the key in `Authorization: Bearer` or a `?api_key=` query parameter:
   neither is read.
3. **Read the placement:**

   ```
   POST https://api.billdog.io/v1/get-wall-for-placement
   X-BillDog-API-Key: bd_public_…
   X-BillDog-Device: $BillDogAnonymousID:…
   X-BillDog-Device-Key: bdk_…

   { "render_mode": "headless", "placement_identifier": "onboarding",
     "platform": "unity" }
   ```

   `platform` is required: `ios`, `android`, `web`, `amazon`, `roku`, or `unity` for
   a player with no mobile store. The answer is `data.offering` (snake_case, packages
   with `product.app_store_product_id` / `play_store_product_id` / `stripe_price_id`),
   `data.presentation` and `data.paywall: null`.
4. **Customer endpoints** (`process-purchase` for a store receipt from your own store
   code, `check-entitlement`, `get-customer-info`, `create-web-checkout`) need the
   device proof; without it they answer 403 `PROOF_REQUIRED`. Where a body asks for
   `app_user_id`, send the device id.
5. **Impressions:** send `paywall_presented` and `paywall_dismissed` to
   `ingest-events` with `offering_id`, `offering_identifier` and
   `placement_identifier`.

Request and response bodies: https://billdog.io/docs/api-reference.

## Check it works

- `resolve_placement_preview` (MCP) shows what a client gets for a placement: the
  campaign, the offering and its packages.
- After a test run, `get_recent_sdk_activity` should show `paywall_presented`. The
  billdog-analyst skill reads conversion once purchases arrive.
