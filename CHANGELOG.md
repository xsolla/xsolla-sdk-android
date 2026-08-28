# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [3.0.53] - 28-08-2026

### Fixed

- Requesting a payment or login flow while another one is still open no longer risks the second flow being lost or opening on top of the first; flows are served one at a time, so the next one opens as soon as the current one closes
- A payment or login flow the system refuses to start (for example when the activity it was requested from is already finishing) now completes with a result instead of hanging, and no longer leaves every later flow unable to open for the rest of the app session
- Changing a system setting while a payment is open (dark mode, font size, display size, language) or resizing the app in split screen no longer interrupts the flow by recreating the screen that hosts it
- A malformed payment redirect link sent to the app by another installed app no longer crashes it while a payment is in progress; such a link is now ignored
- A purchase now completes with a result when the order reports no invoice, or reports only invoices other than the one being awaited, instead of leaving the purchase listener silent for the rest of the session
- An unexpected internal failure while an asynchronous operation is in flight is now reported instead of being discarded, so a call can no longer be left without a result
- A payment that succeeded only after an earlier attempt failed (a card declined, then paid with another method) is now reported as the successful purchase, instead of the failed attempt sometimes being reported in its place
- A `RetryProfile.exponentialBackout(...)` profile with no `maxIntervalMillis` no longer retries with no delay at all, sending its whole attempt budget to the backend as one burst; the delay now grows from `baseIntervalMillis` as documented
- `maxRandomExtraDelayMillis` on a `RetryProfile.exponentialBackout(...)` profile is now added to the delay between attempts; it previously had no effect on any profile that set it
- Cancelling a flow through `BillingFlowCanceller` now always reports `USER_CANCELED`; depending on the exact moment the cancellation landed, it could previously report a generic error instead

### Changed

- A `RetryProfile.exponentialBackout(...)` profile now grows its delay as `baseIntervalMillis * 2^n`, matching its documented formula; it previously grew by `e^n`, which is steeper
- A `RetryProfile.exponentialBackout(...)` profile now makes exactly `maxNumAttempts` attempts, the same as `RetryProfile.uniform(...)` with the same value; it previously made one more

## [3.0.52] - 19-08-2026

### Fixed

- Server API calls now report unexpected failures during request building or response handling as an error result instead of letting the exception escape on the calling thread or leave the returned future uncompleted

## [3.0.51] - 14-08-2026

### Fixed

- `queryPurchasesAsync(ProductType.INAPP)` no longer reports an owned virtual currency listed in the inventory as an in-app purchase; virtual currency is read with `ProductType.VIRTUAL_CURRENCY`, which returns the owned balance
- Catalog and event item types are now matched case-insensitively (item `type`, `bundle_type`, and event `notification_type`), so an unexpectedly cased value from the backend no longer resolves to `UNKNOWN` or an unrecognized event

### Changed

- `setExternalTransactionToken()` in `BillingFlowParams` is public now

## [3.0.50] - 23-07-2026

### Added

- `BillingFlowParams.ProductDetailsParams.Builder.setQuantity(int)` (default `1`): the number of units of the product to include in the order. Applies to every product type — a quantity of N on a bundle or virtual currency package multiplies its delivered contents. Read back via `getQuantity()`

### Fixed

- Validating a purchase that delivers multiple content items (such as a bundle) no longer risks intermittent failures from being rate-limited by the store API

## [3.0.49] - 30-06-2026

### Changed

- `launchBillingFlow` now rejects a standalone virtual currency product (a SKU resolving to `ProductType.VIRTUAL_CURRENCY`) with `BillingResponseCode.ITEM_UNAVAILABLE` before starting any payment, since such an order completes without crediting the currency balance. Virtual currency is sold through a virtual currency package (a `ProductType.BUNDLE` whose contents credit the balance on purchase); `ProductType.VIRTUAL_CURRENCY` remains usable with `queryPurchasesAsync` to read the owned balance. The check keys on the product's true catalog type, so it also applies to a virtual currency fetched as `inapp` via `allowVirtualCurrencyAsInApp`

### Fixed

- A completed bundle purchase confirmed through the inventory strategy is now reported as its individual content items (both on the live purchase update and the order-status recovery path), matching the events strategy

### Added

- `ProductType.VIRTUAL_CURRENCY`: virtual currency SKUs now resolve through `queryProductDetailsAsync` as this type
- `allowVirtualCurrencyAsInApp` on `ConfigWithoutIntegration.Common` (default `false`): when enabled, a SKU requested as `inapp` that resolves to a virtual currency is returned instead of rejected as a type mismatch (the `ProductDetails` still reports its true `virtual_currency` type) — mirrors `allowBundlesAsInApp`
- `BundleDetails.Type` now distinguishes `VIRTUAL_CURRENCY_PACKAGE` and `PARTNER_SIDE_CONTENT` (`UNKNOWN` reserved for unrecognized types)
- `queryPurchasesAsync(ProductType.VIRTUAL_CURRENCY)` returns the user's owned virtual currency balances (inventory strategy)
- Nested bundles are now expanded recursively into their leaf content items with quantities multiplied through each level (e.g. a bundle containing a virtual currency package); `ProductDetails.getBundleDetails()` content items expose their own nested contents via `getContents()`

## [3.0.48] - 29-06-2026

### Added

- Added `allowBundlesAsInApp` to `ConfigWithoutIntegration.Common` (default: `false`): when enabled, a SKU requested as `ProductType.INAPP` that resolves to a bundle is returned instead of being rejected as a type mismatch, letting a caller that does not distinguish in-app products from bundles fetch both in a single `inapp` query (the returned `ProductDetails` still reports its true `bundle` type)

## [3.0.47] - 26-06-2026

### Added

- Added support for **bundle products**: bundle SKUs resolve through `queryProductDetailsAsync` as products of type `ProductType.BUNDLE` and are purchasable like any other product, `ProductDetails.getBundleDetails()` exposes the bundle's catalog composition (bundle type, total content price, and content items with their SKU, type, name, quantity, and price), and a completed bundle purchase is reported as its individual content items across both live purchase updates and restored purchases (standard bundles only for now)

### Changed

- The product cache now expires entries after 1 hour by default (previously cached indefinitely), and `queryProductDetailsAsync` now dispatches at most 4 concurrent chunk requests by default (previously unbounded). Pass `null` for `productCacheTtlMillis` or `maxParallelProductRequests` on `ConfigWithoutIntegration.Common` to restore the never-expire / unbounded behavior

### Fixed

- `queryProductDetailsAsync` now drains backend pagination per chunk: if a chunk response reports that more results remain for the requested SKUs, the still-missing SKUs are re-requested until the backend reports no more remain, rather than being treated as unavailable

## [3.0.46] - 17-06-2026

### Performance

- `queryProductDetailsAsync` now fetches and caches product details per SKU: only uncached SKUs trigger a backend call, results are reused across subsequent queries until the cache TTL expires, and large SKU sets are split into parallel chunk requests to stay within backend page limits and avoid rate limiting

### Changed

- Removed transitive dependencies on `com.xsolla.android:store` and `com.xsolla.android:inventory`; the required functionality is now built into the SDK directly

### Added

- Added `productCacheTtlMillis` to `ConfigWithoutIntegration.Common` to control how long cached product details remain valid (default: indefinite)
- Added `maxItemsPerProductRequest` to `ConfigWithoutIntegration.Common` to cap the number of SKUs per backend request when fetching uncached products (default: backend page cap of 50)
- Added `maxParallelProductRequests` to `ConfigWithoutIntegration.Common` to bound the number of concurrent chunk requests dispatched during a single `queryProductDetailsAsync` call (default: unbounded, relying on OkHttp's per-host concurrency cap)

## [3.0.45] - 01-06-2026

### Fixed

- Fixed a bug where a failed product query blocked subsequent retry attempts from re-fetching

### Changed

- Retries are now restricted to IO (transport) errors only
- Improved inventory and store retry error handling and reporting
- Improved request/response debug logging

## [3.0.44] - 08-05-2026

### Added

- Added additional parameters to the webshop-based billing flow launch method (`WebShop.launchBillingFlow()`)
- Introduced a builder-based `WebShop.launchBillingFlow()`

## [3.0.43] - 27-04-2026

### Changed

- Improved token refresh for social based authentication

## [3.0.42] - 23-04-2026

### Changed

- Updated `Login`: 6.0.18 -> 6.0.19 (ported internals to OkHttp3 to resolve transient dependency conflicts)

## [3.0.41] - 20-04-2026

### Added

- Added optional multi-unit SKU collapsing (`ConfigWithoutIntegration::withCollapseRestoredMultiUnitPurchases`) to merge restored multi-unit purchases into a single batch instead of splitting them

## [3.0.40] - 09-04-2026

### Fixed

- `AsyncRetryScheduler` now correctly handles throwables too

## [3.0.39] - 01-04-2026

### Changed

- Updated `Payments`: 1.4.20 -> 1.4.21

### Fixed

- Fixed PayStation WebView not recovering from main-frame load errors and connectivity changes
- Fixed stale payment redirect pulling the app to the foreground when another payment flow is active
- Fixed redirect nonce not appended to custom redirect URLs (only the default deep-link URL needs it)
- Fixed `paymentFlowActive` flag shared across `BillingClient` instances causing incorrect redirect routing when multiple instances coexist; active flows are now tracked per-nonce in `PaymentFlowRegistry`

## [3.0.38] - 24-03-2026

### Added

- Added opt-in app relaunch from PayStation redirect on cold start (`ConfigWithoutIntegration.Payments::withRedirectAppRelaunch`). When the app is killed during a payment flow in an external browser, tapping "Back to Game" in PayStation now relaunches the app instead of silently failing.

## [3.0.37] - 17-03-2026

### Added

- Added optional cancellation reason querying (`ConfigWithoutIntegration.Payments::withQueryCancellationReasonEnabled`) to distinguish between clean cancels and failed payments

### Fixed

- Fixed token-based billing flow not checking invoice status after activity result

## [3.0.36] - 12-03-2026

### Changed

- Made `ActivityUtils` public

### Fixed

- Fixed an infinite loop in `forJWT` Config method
- Fixed a bug where the payment flow could get stuck when the proxy activity is destroyed externally

## [3.0.35] - 03-02-2026

### Changed

- Updated `Payments`: 1.4.19 -> 1.4.20 (update blacklisted browser list, etc).
- Updated `Login`: 6.0.17 -> 6.0.18

## [3.0.34] - 22-01-2026

### Fixed

- A bug that would result in string formatter sometimes incorrectly interpolating the arguments

### Changed

- Improved retryable asynchronous method error handling

## [3.0.33] - 21-01-2026

### Added

- Added support for payment flow cancellation

## [3.0.32] - 14-01-2026

### Added

- Added support for domain overriding (payments, see `ConfigWithoutIntegration.Payments.DomainOverrideConfig`)

### Changed

- Updated `Payments`: 1.4.18 -> 1.4.19

## [3.0.31] - 13-01-2026

### Changed

- Updated `Payments`: 1.4.17 -> 1.4.18 (support for navigation events in custom tabs based payment activity)

## [3.0.30] - 30-12-2025

### Added

- Added support for manual JWT token updating (`Config.Integration.Xsolla.ForJWT` + `JWTRefresher`)

## [3.0.29] - 30-12-2025

### Added

- Added support for navigation events for payment activity (`ConfigWithoutIntegration.Payments.EventListeners`)

### Changed

- Updated `Payments`: 1.4.16 -> 1.4.17

## [3.0.28] - 29-12-2025

### Added

- Added support for logo visibility setting in `Config.Payments`

## [3.0.27] - 10-12-2025

### Added

- Added `ProviderUtils` helper (browser related utilities)
- Added e-mail consent opt-in setting (`Config.Payments::withEmailCollectionConsentOptInEnabled`)
- Added config log dumping on billing client creation in debug mode
- Implemented provider blacklisting, prioritization, etc settings for `Config.Payments.CustomTabs` and `Config.Payments.TrustedWebActivity`
- Implemented overridable retry policy settings (see `Config.Common::withRetryPolicies` and `RetryPolicies`) for all asynchronous server requests

### Fixed

- Improved compatibility with some of the browsers that wouldn't correctly redirect back into app (e.g. `Amazon Silk`)
- Blacklisted `Microsoft Edge` from being a compatible provider
- Blacklisted `Samsung Internet` from being a compatible TWA provider (has a partial, broken implementation)
- Fixed custom tabs provider initialization order (controlled now)

### Changed

- Updated `Store`: 2.5.14 -> 2.5.15
- Updated `Payments`: 1.4.15 -> 1.4.16
- Moved out `RetryProfile` from `Common.Payments`
- Moved `PendingOrderStatusQueryRetryProfileOverride` setting from `Common.Payments` to `RetryPolicies`
- Removed `-keepattributes *Annotation*` from the user-facing `consumer-rules.pro`

## [3.0.26] - 02-12-2025

### Added

- Support for `browser_type` field inside payment tokens

## [3.0.25] - 28-11-2025

### Fixed

- Redirect URL fixes

## [3.0.24] - 24-11-2025

### Changed

- Improved logging

## [3.0.23] - 21-11-2025

### Added

- Support for `install_source` parameter in payment tokens

### Changed

- Updated `Store` library: 2.5.13 -> 2.5.14

## [3.0.22] - 20-11-2025

### Changed

- Improved payment flow cancellation detection algorithm in certain situations
- Reduced the default number of attempts allowed for querying stuck payment flow status (20 -> 12)

## [3.0.21] - 17-11-2025

### Added

- Added `invokePurchasesUpdatedForMissingOrderId` enables the `PurchasesUpdatedListener.onPurchasesUpdated()` invocation on purchases with a missing order ID (based on customized payment tokens)

## [3.0.20] - 14-11-2025

### Changed

- Improved invoice status querying for both standard and token based flows

### Fixed

- Fixed logger not respecting log levels in certain scenarios

## [3.0.19] - 12-11-2025

### Added

- Support for `country` field when creating a payment token based on the locale override

## [3.0.18] - 07-11-2025

### Fixed

- WebShop URL parameter name fix

## [3.0.17] - 06-11-2025

### Fixed

- CustomTabs and TrustedWebActivity related fixes

### Changed

- Updated `Payments` library: 1.4.13 -> 1.4.15

## [3.0.16] - 21-10-2025

### Fixed

- Redirect button text can now be customized through PA settings

## [3.0.15] - 20-10-2025

### Added

- Added support for customized payment tokens (the result is handled on the backend)

## [3.0.14] - 15-10-2025

### Added

- Ability to control via a flag whether only personalized products are fetched from the catalog

## [3.0.13] - 07-10-2025

### Added

- Ability to enable Buy Button solution for billing flows

### Changed

- Updated `Store` library: 2.5.12 -> 2.5.13

## [3.0.12] - 30-09-2025

### Fixed

- Improved order status querying retry profiles

### Added

- Ability to override the order status querying retry profile using the payments config

## [3.0.11] - 23-09-2025

### Fixed

- Order status querying fix for billing flows launched using an externally generated purchase token

## [3.0.10] - 22-09-2025

### Added

- Advanced events manager scheduling

### Changed

- Updated `Payments` library: 1.4.12 -> 1.4.13
- Updated `Login` library: 6.0.16 -> 6.0.17

## [3.0.9] - 10-09-2025

### Fixed

- Ability to open TWA without a splash screen

## [3.0.8] - 09-09-2025

### Added

- Added support for 'free' purchases, i.e. purchases that have no cost

## [3.0.7] - 01-09-2025

### Added

- Extended `IntegrationUtils` with additional utility methods

## [3.0.6] - 29-08-2025

### Changed

- Updated `Payments` library: 1.4.11 -> 1.4.12

## [3.0.5] - 28-08-2025

### Added

- Google Play store's country code can now be queried using `IntegrationUtils.queryGooglePlayCountryCodeAsync()` utility method

## [3.0.4] - 25-08-2025

### Fixed

- Focus monitor would sometimes crash under certain circumstances
- Fixed events API payload parsing

## [3.0.3] - 21-08-2025

### Added

- Added support for a custom project ID in Xsolla Events API

## [3.0.2] - 18-08-2025

### Fixed

- Fixed pending order status querying during the billing flow

## [3.0.1] - 15-08-2025

### Added

- Added support for the Webshop oriented billing flows
- Added the utility helper for opening URLs in an external browser

## [3.0.0] - 12-08-2025

### Changed

- Promoted the SDK to a **major release**. This release marks the end of the alpha series
- Updated `Payments` library 1.4.9 -> 1.4.11

### Added

- Added utilities for acquiring authentication tokens through social network access tokens 
- Added support for social access token authentication method
- Logging for anything that relies on retry logic

## [2.1.26-alpha] - 05-08-2025

### Changed

- Xsolla Events API improvements
- Purchase token generation improvements (compatibility with older tokens is not affected)
- Updated the `Payments` library 1.4.8 -> 1.4.9

### Fixed

- Additional checks for non-compatible browser apps

## [2.1.25-alpha] - 24-07-2025

### Added

- Added support for Xsolla Events API when dealing with deferred purchases

## [2.1.24-alpha] - 22-07-2025

### Added

- Added support for payment method ID when launching a billing flow

## [2.1.23-alpha] - 22-07-2025

### Added

- Added support for launching the billing flow using a pre-generated payment token

## [2.1.22-alpha] - 16-07-2025

### Added

- Added support for `tracking ID` when launching a billing flow

## [2.1.21-alpha] - 15-07-2025

### Changed

- Order status is now forcefully checked whenever the payment activity closes with an unexpected result

## [2.1.20-alpha] - 20-06-2025

### Changed

- Updated `Login` library 6.0.14 -> 6.0.16

## [2.1.19-alpha] - 13-06-2025

### Added

- `queryPurchasesAsync` now also returns unconsumed product instances

## [2.1.18-alpha] - 12-06-2025

### Fixed

- Proper payment cancellation tracking for non-ChromeTab-based activities

## [2.1.17-alpha] - 11-06-2025

### Added

- Added support for `AccountIdentifiers` and exposed the related functionality

### Changed

- Moved from `BillingFlowParams.ProductDetailsParams` to `BillingFlowParams`:
	- `setCustomPayload`
	- `setExternalId`
- Renamed in `BillingFlowParams`:
	- `setCustomPayload` -> `setDeveloperPayload`
	- `setExternalId` -> `setExternalTransactionId`
- Removed support in `BillingFlowParams.ProductDetailsParams.Builder` for:
	- `setOfferToken`
- Removed support in `BillingFlowParams.Builder` for:
	- `setIsOfferPersonalized`
- Removed support in `ProductDetails` for:
	- `RecurrenceMode`
	- `PricingPhase`
	- `PricingPhases`
	- `SubscriptionOfferDetails`
	- `getSubscriptionOfferDetails`
	
### Fixed

- Eliminated redundant characters in purchase tokens

## [2.1.16-alpha] - 09-06-2025

### Changed

- Added support for `external ID` for launched billing flows
- Improved error reporting on billing flow initialization

## [2.1.15-alpha] - 03-06-2025

### Changed

- Improved error message reporting across the SDK

## [2.1.14-alpha] - 10-04-2025

### Changed

- Updated `Payments` library 1.4.7 -> 1.4.8

## [2.1.13-alpha] - 12-03-2025

### Added

- Added more modes for the experimental Xsolla Login Widget

## [2.1.12-alpha] - 27-12-2025

### Added

- Added ability to refresh the login token directly

### Changed

- Updated `Payments` library 1.4.6 -> 1.4.7

## [2.1.11-alpha] - 26-12-2025

### Fixed

- A fix for a potential crash related to dereferencing uninitialized objects

### Changed

- Updated `Payments` library 1.4.5 -> 1.4.6

## [2.1.10-alpha] - 18-02-2025

### Fixed

- An issue that would lead to a broken state due to an abrupt connection loss during the authentication stage

### Changed

- Adjusted the retry parameters for backend-oriented methods:
	- Reduced the number of attempts before an operation is viewed as failed (5 -> 3)
	- Reduced the maximum amount of time between the subsequent attempts with the `exponential` retry method

## [2.1.9-alpha] - 12-02-2025

### Fixed

- An issue that would prevent the payment activity to broadcast the correct billing result

## [2.1.8-alpha] - 10-02-2025

### Fixed

- An issue that would prevent the product prices to be parsed correctly

## [2.1.7-alpha] - 05-02-2025

### Added

- ProGuard rules

### Changed

- Updated `Login` library 6.0.11 -> 6.0.14
- Updated `Store` library 2.5.10 -> 2.5.11
- Updated `Payments` library 1.4.4 -> 1.4.5

## [2.1.6-alpha] - 02-12-2024

### Changed

- Added support for opening payments view in an external system browser (`ConfigWithoutIntegration.Payments.Activity.System`)
- Updated `Payments` library 1.4.3 -> 1.4.4

## [2.1.5-alpha] - 22-11-2024

### Changed

- Updated `Payments` library 1.4.2 -> 1.4.3

## [2.1.4-alpha] - 14-10-2024

### Added

- Added an internal support for external transaction ID in `BillingFlowParams.ProductDetailsParams`

## [2.1.3-alpha] - 08-10-2024

### Added

- Added support for a developer payload using `BillingFlowParams.ProductDetailsParams.setDeveloperPayload()`

## [2.1.2-alpha] - 01-10-2024

### Changed

- Updated `Login` library 6.0.10 -> 6.0.11
- Updated `Payments` library 1.4.0 -> 1.4.2

## [2.1.1-alpha] - 23-09-2024

### Added

- Implemented `getAuthenticationToken`
- Ability to set the custom user ID property via the payments config
- Updated `Login` library 6.0.9 -> 6.0.10

## [2.1.0-alpha] - 06-09-2024

### Added

- Added experimental support for Xsolla Login Widget.

## [2.0.2] - 04-09-2024

### Changed

- Updated `Store` library 2.5.7 -> 2.5.10

## [2.0.1] - 27-08-2024

### Added

- Re-try logic for asynchronous methods (back-off approach).

## [2.0.0] - 14-08-2024

### Changed

- Synced version across the platforms.

## [0.0.11] - 23-07-2024

### Changed

- Changed visibility of some classes.

## [0.0.10] - 19-07-2024

### Added

- Authentication using a `Xsolla Login` widget.

## [0.0.9] - 18-07-2024

### Changed

- Removed dependency on a customized payments library.
- Updated external dependencies.

## [0.0.8] - 18-07-2024

### Fixed

- Fixed `ProxyActivity` visibility.

## [0.0.7] - 18-07-2024

### Fixed

- Fixed a potential crash on purchasing flow launch.

## [0.0.6] - 08-07-2024

### Fixed

- Revised visibility of some classes.

## [0.0.5] - 05-07-2024

### Fixed

- Fixed game engine version not being passed to analytics.

## [0.0.4] - 20-06-2024

### Added

- Introduced `Config.Analytics` for analytics related settings.
- Library version tag and name are now available via `BuildConfig`.

## [0.0.3] - 07-06-2024

### Changed

- Migrated SDK to `com.xsolla.android.mobile` package.
- Made Google Play Billing dependency `compile-only`.

## [0.0.2] - 05-06-2024

### Added

- Introduced a changelog.

## [0.0.1] - 05-06-2024

### Added

- Initial implementation.
