# Cafe Bazaar API Implementation Guide

This document describes how **Badanam** integrates with [Cafe Bazaar](https://cafebazaar.ir) (Iran's Android app store) for in-app subscriptions, purchase restoration, and app rating. It is written so you can replicate the same pattern in other Flutter apps.

---

## Table of contents

1. [Overview](#overview)
2. [Architecture in Badanam](#architecture-in-badanam)
3. [Prerequisites (Cafe Bazaar panel)](#prerequisites-cafe-bazaar-panel)
4. [Implementation guide for other apps](#implementation-guide-for-other-apps)
5. [Configuration reference](#configuration-reference)
6. [Android native setup](#android-native-setup)
7. [Gradle and dependency setup](#gradle-and-dependency-setup)
8. [Dart / Flutter billing layer](#dart--flutter-billing-layer)
9. [Entitlements and feature gating](#entitlements-and-feature-gating)
10. [UI integration](#ui-integration)
11. [App rating via Bazaar deep link](#app-rating-via-bazaar-deep-link)
12. [Error handling and edge cases](#error-handling-and-edge-cases)
13. [Testing and sandbox](#testing-and-sandbox)
14. [Troubleshooting](#troubleshooting)
15. [File checklist](#file-checklist)
16. [Security notes](#security-notes)

---

## Overview

### What Cafe Bazaar provides

Cafe Bazaar is the primary Android app distribution channel in Iran. For monetization it offers:

- **In-app billing (IAB)** for one-time purchases and subscriptions
- **Purchase verification** using an RSA public key you configure in the developer panel
- **Deep links** to open your app's store page inside the Bazaar client

Badanam uses Bazaar exclusively on Android. iOS and web are not supported for payments (the billing layer no-ops on those platforms).

### What we use on the Flutter side

| Layer | Package / API | Purpose |
|-------|---------------|---------|
| Billing SDK | [`flutter_poolakey`](https://pub.dev/packages/flutter_poolakey) | Flutter wrapper around Bazaar's official [Poolakey](https://github.com/cafebazaar/Poolakey) Android library |
| State management | Riverpod `NotifierProvider` | Connection state, purchase flow, SKU prices |
| Entitlements | `isProProvider` | Single boolean gating premium features |
| Rating | Custom `MethodChannel` + `url_launcher` | Open Bazaar rate page natively, fallback to web |

Poolakey talks to the **Bazaar app** installed on the device (`com.farsitel.bazaar`). If Bazaar is missing, outdated, or the user is not signed in, connection and purchases fail gracefully with localized error messages.

---

## Architecture in Badanam

```
┌─────────────────────────────────────────────────────────────────┐
│                         Flutter UI                              │
│  SplashScreen ──initialize()──► BazaarBilling                   │
│  PaywallSheet / SettingsScreen ──subscribe()──►                 │
│  ProgramTile / ProgramDetail ──isProProvider──► feature gates   │
│  RatingPromptService ──openRatePage()──► BazaarRatingService    │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     flutter_poolakey                 MethodChannel
     (connect, subscribe,              (com.badanam.app/bazaar)
      getAllSubscribedProducts,        openRatePage
      getSubscriptionSkuDetails)
              │                             │
              ▼                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Android (MainActivity.kt)                    │
│  Poolakey native SDK ◄──► Bazaar app (com.farsitel.bazaar)      │
│  Intent: bazaar://details?id=<packageId>                        │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
              Cafe Bazaar servers (subscription validation)
```

### Lifecycle summary

1. **App launch (SplashScreen)** — `bazaarBillingProvider.notifier.initialize()` runs in the background. This connects to Bazaar and syncs existing subscriptions into `isProProvider` before the user reaches home.
2. **Paywall / Settings** — Calls `initialize()` again if needed, loads live SKU prices, and exposes purchase + restore actions.
3. **Purchase** — `FlutterPoolakey.subscribe(skuId)` opens Bazaar's payment UI. On success, `isProProvider` is set to `true`.
4. **Restore** — `FlutterPoolakey.getAllSubscribedProducts()` queries active subscriptions and updates `isProProvider`.
5. **Feature gates** — UI reads `isProProvider` and `canAccessProgram()` to lock/unlock premium content.

---

## Prerequisites (Cafe Bazaar panel)

Before writing code, complete these steps in the [Cafe Bazaar developer panel (Pishkhan / پیشخوان)](https://pishkhan.cafebazaar.ir):

### 1. Register the app

- Upload at least one APK/AAB to Bazaar (or use internal testing track).
- Note your **application ID** (e.g. `com.badanam.app`). This must match `applicationId` in `android/app/build.gradle.kts`.

### 2. Enable in-app billing

- In Pishkhan, open your app → **In-app billing / پرداخت درون‌برنامه‌ای**.
- Copy the **RSA public key**. Paste it into `lib/core/billing/bazaar_config.dart` as `bazaarRsaPublicKey`.

### 3. Create subscription products

Badanam defines three SKUs in Pishkhan:

| SKU ID | Plan | Notes |
|--------|------|-------|
| `monthly` | Monthly subscription | Production plan |
| `yearly` | Yearly subscription | Production plan |
| `test` | 5-minute test subscription | Sandbox only — expires quickly for QA |

Product IDs in code **must exactly match** Pishkhan. See `BazaarProductIds` in `bazaar_config.dart`.

### 4. Test accounts

- Add sandbox tester accounts in Pishkhan.
- Install the **Bazaar app** on a physical Android device (emulators often lack a working Bazaar client).
- Sign into Bazaar with a test account before purchasing.

### 5. Permissions

Bazaar billing requires the `PAY_THROUGH_BAZAAR` permission (added in `AndroidManifest.xml`).

---

## Implementation guide for other apps

Follow these steps to add the same Bazaar integration to a new Flutter project.

### Step 1 — Add dependencies

In `pubspec.yaml`:

```yaml
dependencies:
  flutter_poolakey: ^2.2.0+1.0.0
  flutter_riverpod: ^2.6.1   # or your state solution
  url_launcher: ^6.3.2       # rating web fallback
```

Run `flutter pub get`.

### Step 2 — Configure Android Gradle

**`android/build.gradle.kts`** — Poolakey is hosted on JitPack:

```kotlin
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://jitpack.io") }
    }
}
```

**`android/settings.gradle.kts`** — Add the Poolakey Gradle 8 patch (see [Gradle and dependency setup](#gradle-and-dependency-setup)). Without this, builds fail because older `flutter_poolakey` releases reference deprecated `jcenter()`.

### Step 3 — Android manifest

Add the billing permission and declare the Bazaar package so Android 11+ can resolve intents:

```xml
<uses-permission android:name="com.farsitel.bazaar.permission.PAY_THROUGH_BAZAAR" />

<queries>
    <package android:name="com.farsitel.bazaar" />
</queries>
```

Full reference: `android/app/src/main/AndroidManifest.xml`.

### Step 4 — Create billing config

Create `lib/core/billing/bazaar_config.dart`:

```dart
const bazaarPackageName = 'com.your.app';  // match applicationId

const bazaarRsaPublicKey = '...';  // from Pishkhan panel

abstract final class BazaarProductIds {
  static const monthly = 'monthly';
  static const yearly = 'yearly';
  static const test5Min = 'test';

  static const subscriptionSkus = [monthly, yearly, test5Min];

  static String forPlan(SubscriptionPlan plan) => switch (plan) {
        SubscriptionPlan.monthly => monthly,
        SubscriptionPlan.yearly => yearly,
        SubscriptionPlan.test5Min => test5Min,
      };
}
```

Replace package name, RSA key, and SKU IDs with your own values.

### Step 5 — Implement the billing provider

Copy and adapt `lib/core/billing/bazaar_billing_provider.dart`. The core responsibilities are:

| Method | What it does |
|--------|--------------|
| `initialize()` | Connect to Bazaar via Poolakey; dedupe concurrent calls |
| `_connect()` | `FlutterPoolakey.connect(rsaKey, callbacks)` with 15s timeout |
| `subscribe(plan)` | `FlutterPoolakey.subscribe(skuId, payload: plan.name)` |
| `refreshEntitlements()` | Re-connect if needed, then sync subscriptions |
| `_syncEntitlements()` | `getAllSubscribedProducts()` → update pro flag |
| `_loadSkuPrices()` | `getSubscriptionSkuDetails(skus)` → display live prices |

**Important implementation details:**

- **Platform guard** — Only run on Android (`!kIsWeb && defaultTargetPlatform == TargetPlatform.android`).
- **Activity timing** — Delay connection ~500 ms after first frame; Poolakey needs a live `Activity`.
- **Connection deduplication** — Use `_connectInFlight` so splash + paywall don't double-connect.
- **Disconnect handler** — Set connection state to `failed` when Bazaar disconnects mid-session.

### Step 6 — Entitlements provider

Create a simple pro flag:

```dart
final isProProvider = NotifierProvider<ProEntitlement, bool>(ProEntitlement.new);

bool canAccessProgram(Program program, bool isPro) =>
    !program.isPremium || isPro;
```

Billing writes to this provider; UI reads from it. Keep entitlements separate from billing state so feature gates stay simple.

### Step 7 — Initialize on app start

In your splash or root widget:

```dart
WidgetsBinding.instance.addPostFrameCallback((_) {
  ref.read(bazaarBillingProvider.notifier).initialize();
});
```

Badanam does this in `SplashScreen` so Pro status is restored before navigation to home.

### Step 8 — Build paywall UI

Minimum paywall requirements:

1. Call `initialize()` in `initState` (post-frame callback).
2. Show plan options with `billing.skuPrices[plan]` (fallback to static copy if prices fail to load).
3. Disable purchase button when `!billing.isAvailable` or connection is in progress.
4. Wire **Purchase** → `billing.subscribe(selectedPlan)`.
5. Wire **Restore** → `billing.refreshEntitlements()` and check `isProProvider`.

See `lib/features/subscription/paywall_sheet.dart` and `restore_purchases.dart`.

### Step 9 — Gate premium features

Anywhere you need Pro:

```dart
final isPro = ref.watch(isProProvider);
if (program.isPremium && !canAccessProgram(program, isPro)) {
  showPaywallSheet(context);
}
```

### Step 10 — Optional: Bazaar rating deep link

If you want in-app "Rate us" that opens Bazaar:

1. Add `MainActivity.kt` method channel handler (see [App rating](#app-rating-via-bazaar-deep-link)).
2. Add `BazaarRatingService` in Dart.
3. Optionally prompt after N sessions (`RatingPromptService`).

### Step 11 — Test on device

1. Build a release or signed debug APK.
2. Install Bazaar + your app on a real Android phone.
3. Sign into Bazaar.
4. Purchase the `test` SKU (5-minute subscription) in sandbox.
5. Verify Pro unlocks, restore works after reinstall, and prices load.

---

## Configuration reference

### `bazaar_config.dart`

```dart
/// Android application ID published on Cafe Bazaar.
const bazaarPackageName = 'com.badanam.app';

/// Cafe Bazaar in-app billing RSA public key from Pishkhan panel.
const bazaarRsaPublicKey = 'MIHNMA0GCSqGSIb3...';

/// Product IDs configured in Cafe Bazaar Pishkhan panel.
abstract final class BazaarProductIds {
  static const monthly = 'monthly';
  static const yearly = 'yearly';
  static const test5Min = 'test';  // 5-minute sandbox SKU

  static const subscriptionSkus = [monthly, yearly, test5Min];

  static String forPlan(SubscriptionPlan plan) => switch (plan) {
        SubscriptionPlan.monthly => monthly,
        SubscriptionPlan.yearly => yearly,
        SubscriptionPlan.test5Min => test5Min,
      };
}
```

### Subscription plans enum

Defined in `lib/core/entitlements/entitlements_provider.dart`:

```dart
enum SubscriptionPlan { monthly, yearly, test5Min }
```

Production UI (`PaywallSheet`, Settings) only shows `monthly` and `yearly`. `test5Min` exists for QA.

---

## Android native setup

### Manifest permissions and queries

```xml
<uses-permission android:name="com.farsitel.bazaar.permission.PAY_THROUGH_BAZAAR" />

<queries>
    <package android:name="com.farsitel.bazaar" />
    <!-- other queries... -->
</queries>
```

The `<queries>` entry is required on Android 11+ (API 30+) so the app can detect and interact with the Bazaar package.

### MainActivity — rating method channel

Poolakey handles billing through its own plugin registration. **Rating** uses a custom channel because Poolakey does not expose a rate-app API.

`android/app/src/main/kotlin/com/badanam/app/MainActivity.kt`:

```kotlin
class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "openRatePage" -> {
                        val packageId = call.argument<String>("packageId") ?: packageName
                        result.success(openBazaarRatePage(packageId))
                    }
                    else -> result.notImplemented()
                }
            }
    }

    private fun openBazaarRatePage(packageId: String): Boolean {
        val bazaarPackage = "com.farsitel.bazaar"
        return try {
            packageManager.getPackageInfo(bazaarPackage, 0)
            val intent = Intent(Intent.ACTION_EDIT).apply {
                data = Uri.parse("bazaar://details?id=$packageId")
                setPackage(bazaarPackage)
            }
            startActivity(intent)
            true
        } catch (_: Exception) {
            false
        }
    }

    companion object {
        private const val CHANNEL = "com.badanam.app/bazaar"
    }
}
```

**For other apps:** change `CHANNEL` to `com.your.app/bazaar` and use the same string in Dart's `MethodChannel`.

---

## Gradle and dependency setup

### JitPack repository

Poolakey artifacts are pulled from GitHub via JitPack. Add to `android/build.gradle.kts`:

```kotlin
maven { url = uri("https://jitpack.io") }
```

### Poolakey / Gradle 8 compatibility patch

`flutter_poolakey` versions may still reference `jcenter()`, which Gradle 8+ removed. Badanam patches the cached plugin at settings evaluation time:

```kotlin
// android/settings.gradle.kts
fun patchFlutterPoolakeyForModernGradle() {
    val pubCache = System.getenv("PUB_CACHE")
        ?: "${System.getProperty("user.home")}/.pub-cache"
    // ... replaces jcenter() with mavenCentral()
    // ... fixes dependencies block structure
}

patchFlutterPoolakeyForModernGradle()
```

Run this **before** `pluginManagement { }`. After `flutter pub get`, the patch modifies `~/.pub-cache/hosted/pub.dev/flutter_poolakey-*/android/build.gradle`.

If you upgrade `flutter_poolakey`, verify the patch still applies or check whether upstream fixed Gradle 8 support.

### Application ID

`android/app/build.gradle.kts`:

```kotlin
defaultConfig {
    applicationId = "com.badanam.app"
    // ...
}
```

Must match `bazaarPackageName` and the app registered in Pishkhan.

---

## Dart / Flutter billing layer

### Provider structure

`BazaarBilling` extends Riverpod's `Notifier<BazaarBillingStatus>`:

```dart
class BazaarBillingStatus {
  final BazaarConnectionState connection;  // idle | connecting | connected | failed
  final bool isPurchasing;
  final String? lastError;
  final Map<SubscriptionPlan, String> skuPrices;
  bool get isAvailable => !kIsWeb && defaultTargetPlatform == TargetPlatform.android;
}

final bazaarBillingProvider =
    NotifierProvider<BazaarBilling, BazaarBillingStatus>(BazaarBilling.new);
```

### Connection flow

```
initialize()
    │
    ├─ not Android? → return
    ├─ already connected? → return
    └─ _connect()
           │
           ├─ delay 500ms (Activity ready)
           ├─ FlutterPoolakey.connect(rsaKey, onSucceed, onFailed, onDisconnected)
           ├─ timeout 15s
           │
           ├─ success → _syncEntitlements() + _loadSkuPrices()
           └─ failure → connection=failed, lastError localized
```

### Purchase flow

```
subscribe(plan)
    │
    ├─ not available → error snackbar, return false
    ├─ not connected → initialize(), bail if still disconnected
    ├─ isPurchasing = true
    ├─ FlutterPoolakey.subscribe(BazaarProductIds.forPlan(plan), payload: plan.name)
    │
    ├─ PURCHASED → isProProvider.setPro(true), return true
    ├─ user cancelled → clear error, return false
    └─ other error → lastError = formatted message, return false
```

### Entitlement sync

```dart
final subscriptions = await FlutterPoolakey.getAllSubscribedProducts();
final hasActiveSubscription = subscriptions.any(
  (purchase) =>
      purchase.purchaseState == PurchaseState.PURCHASED &&
      BazaarProductIds.subscriptionSkus.contains(purchase.productId),
);
ref.read(isProProvider.notifier).setPro(hasActiveSubscription);
```

Only SKUs listed in `subscriptionSkus` grant Pro. Unknown product IDs are ignored.

### Live pricing

```dart
final details = await FlutterPoolakey.getSubscriptionSkuDetails(
  BazaarProductIds.subscriptionSkus,
);
// Map sku → SubscriptionPlan → display price string (e.g. "۱۰۰ هزار تومان")
```

Prices are optional UI sugar. If the call fails, the paywall falls back to static strings in `FaStrings.subscriptionPlanPrice()`.

---

## Entitlements and feature gating

### Pro provider

```dart
class ProEntitlement extends Notifier<bool> {
  @override
  bool build() => false;
  void setPro(bool value) => state = value;
}
```

Pro state is **in-memory only** in Badanam. It is re-derived from Bazaar on every app launch via `_syncEntitlements()`. You do not need local persistence unless you want offline grace periods (not implemented here).

### Where Pro is checked

| Location | Behavior |
|----------|----------|
| `program_tile.dart` | Shows lock icon on premium programs |
| `program_detail_screen.dart` | Blocks start workout → opens paywall |
| `workout_summary_screen.dart` | PR timeline visible only for Pro |
| `settings_screen.dart` | Shows subscription status + purchase UI |

Helper:

```dart
bool canAccessProgram(Program program, bool isPro) =>
    !program.isPremium || isPro;
```

---

## UI integration

### Splash — background init

`lib/features/splash/splash_screen.dart`:

```dart
WidgetsBinding.instance.addPostFrameCallback((_) async {
  ref.read(bazaarBillingProvider.notifier).initialize();
  // ... other startup tasks
});
```

### Paywall bottom sheet

`showPaywallSheet(context)` → `PaywallSheet`:

- Initializes billing on open
- Plan selector (`SubscriptionPlanOption`) with live or fallback prices
- Purchase button disabled when not Android, connecting, or purchasing
- Restore purchases link
- Error text when connection fails

### Settings — inline subscription management

Settings duplicates purchase/restore UX for users who prefer managing subscription outside the paywall modal. Same provider, same `subscribe()` / `restorePurchases()` helpers.

### Restore helper

`lib/features/subscription/restore_purchases.dart`:

```dart
Future<bool> restorePurchases(BuildContext context, WidgetRef ref, {
  bool popNavigatorOnSuccess = false,
}) async {
  await ref.read(bazaarBillingProvider.notifier).refreshEntitlements();
  final isPro = ref.read(isProProvider);
  // Show success / not-found snackbar
}
```

---

## App rating via Bazaar deep link

### Dart service

`lib/core/engagement/bazaar_rating_service.dart`:

1. Try native `MethodChannel('com.badanam.app/bazaar').invokeMethod('openRatePage', {'packageId': bazaarPackageName})`.
2. If Bazaar app missing or channel fails → open `https://cafebazaar.ir/app/$bazaarPackageName` via `url_launcher`.

### Session-based prompt

`RatingPromptService` shows a dialog after exactly **5 sessions** (Android only, once ever). Uses `SharedPreferences` key `rating_prompt_shown`.

---

## Error handling and edge cases

### Platform availability

| Platform | Billing | Rating |
|----------|---------|--------|
| Android + Bazaar installed | Full support | Native deep link |
| Android, no Bazaar | `isAvailable=true` but connect fails | Web fallback |
| iOS / Web | `isAvailable=false`, buttons disabled | No-op |

### User cancellation

Purchase errors containing `cancel` or `user` in code/message are treated as silent cancellation (no error snackbar).

### Connection errors

Mapped to friendly Persian strings:

| Condition | Message key |
|-----------|-------------|
| Not connected / connect failed | `FaStrings.bazaarConnectionFailed` |
| Not signed into Bazaar | `FaStrings.bazaarNotSignedIn` |
| Other `PlatformException` | Raw message or generic code-based message |

### Disconnect during session

`onDisconnected` callback sets `connection = failed`. Next purchase calls `initialize()` again.

### Concurrent initialize()

`_connectInFlight` ensures multiple callers await the same connection future.

### Release builds

ProGuard is enabled (`isMinifyEnabled = true`). Poolakey/Bazaar classes are not explicitly kept in `proguard-rules.pro`; if billing breaks only in release, add keep rules for `ir.cafebazaar.poolakey.**`.

---

## Testing and sandbox

### Recommended test flow

1. Create `test` SKU in Pishkhan (short-duration subscription).
2. Add your Google/Bazaar test account as sandbox user.
3. Install release/signed build (some billing APIs behave differently on unsigned debug builds — test both).
4. Purchase `test` plan → confirm Pro unlocks immediately.
5. Wait for expiry (~5 min) → relaunch app → confirm Pro is revoked after `_syncEntitlements()`.
6. Re-purchase → use **Restore** on a fresh install before purchasing again.

### What to verify

- [ ] Connection succeeds within 15s on cold start
- [ ] SKU prices appear in paywall (or fallback text shows)
- [ ] Monthly/yearly purchase completes
- [ ] User cancel does not show error
- [ ] Restore finds active subscription after reinstall
- [ ] Premium content locks/unlocks with `isProProvider`
- [ ] Rating opens Bazaar app page
- [ ] App behaves safely when Bazaar is uninstalled

### test5Min plan

Defined in code but hidden from production paywall UI. Enable manually in debug builds if you need quick subscription lifecycle testing:

```dart
// Temporarily add to _plans in PaywallSheet:
SubscriptionPlan.test5Min,
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Gradle build fails on `jcenter()` | Old flutter_poolakey | Apply `patchFlutterPoolakeyForModernGradle()` in settings.gradle.kts |
| `connect` always times out | Bazaar not installed, outdated, or no network | Install/update Bazaar, sign in, use real device |
| Purchase succeeds but Pro false | SKU ID mismatch | Ensure Pishkhan product IDs match `BazaarProductIds` exactly |
| `SecurityException` / permission denied | Missing manifest permission | Add `PAY_THROUGH_BAZAAR` permission |
| Cannot detect Bazaar app (API 30+) | Missing `<queries>` | Add `<package android:name="com.farsitel.bazaar" />` |
| Works in debug, fails in release | ProGuard stripping | Add keep rules for Poolakey |
| Prices empty | Not connected yet or SKU not published | Wait for connect; verify products active in Pishkhan |
| "Not signed in" errors | No Bazaar account on device | Sign into Bazaar app first |
| Rating channel not implemented | Wrong channel name | Match Kotlin `CHANNEL` and Dart `MethodChannel` strings |

---

## File checklist

Copy or adapt these files when porting to another app:

| File | Role |
|------|------|
| `lib/core/billing/bazaar_config.dart` | Package name, RSA key, SKU IDs |
| `lib/core/billing/bazaar_billing_provider.dart` | Core billing logic |
| `lib/core/entitlements/entitlements_provider.dart` | `SubscriptionPlan`, `isProProvider` |
| `lib/features/subscription/paywall_sheet.dart` | Paywall UI |
| `lib/features/subscription/restore_purchases.dart` | Restore helper |
| `lib/features/subscription/subscription_plan_option.dart` | Plan card widget |
| `lib/core/engagement/bazaar_rating_service.dart` | Rating deep link |
| `lib/core/engagement/rating_prompt_service.dart` | Optional rating prompt |
| `android/app/src/main/AndroidManifest.xml` | Permission + queries |
| `android/app/src/main/kotlin/.../MainActivity.kt` | Rating method channel |
| `android/build.gradle.kts` | JitPack repository |
| `android/settings.gradle.kts` | Poolakey Gradle patch |
| `pubspec.yaml` | `flutter_poolakey`, `url_launcher` |

### Strings to localize

See `FaStrings` keys: `paywall*`, `bazaarConnectionFailed`, `bazaarNotSignedIn`, `subscriptionPlan*`, `ratingPrompt*`.

---

## Security notes

1. **RSA public key** — Safe to embed in the app. It verifies purchase signatures from Bazaar; it cannot authorize fraudulent charges.
2. **Server-side validation** — Badanam trusts client-side Poolakey responses only. For high-value subscriptions, consider a backend that validates purchase tokens with Bazaar's server API.
3. **No secrets in repo** — The RSA key is public by design. Do not commit Bazaar panel passwords or signing keys.
4. **Payload field** — `subscribe(..., payload: plan.name)` attaches opaque metadata to the purchase. Useful for analytics; not used for authorization in Badanam.

---

## Quick start checklist for a new app

```
[ ] Register app in Cafe Bazaar Pishkhan
[ ] Create subscription SKUs (monthly, yearly, + test SKU)
[ ] Copy RSA public key → bazaar_config.dart
[ ] Set applicationId == bazaarPackageName
[ ] Add flutter_poolakey to pubspec.yaml
[ ] Add JitPack + Poolakey Gradle patch
[ ] Add PAY_THROUGH_BAZAAR permission + Bazaar queries to manifest
[ ] Copy bazaar_billing_provider.dart + entitlements_provider.dart
[ ] Call initialize() on splash
[ ] Build paywall + restore flow
[ ] Gate features with isProProvider
[ ] (Optional) Add MainActivity rating channel
[ ] Test on physical Android device with Bazaar installed
```

---

## Related links

- [Cafe Bazaar developer documentation](https://developers.cafebazaar.ir)
- [Poolakey (official Android SDK)](https://github.com/cafebazaar/Poolakey)
- [flutter_poolakey on pub.dev](https://pub.dev/packages/flutter_poolakey)
- [Badanam marketing notes — Bazaar positioning](MARKETING_PROPOSITIONS.md)

---

*Last updated from Badanam codebase v1.0.1+2 — Flutter 3.12+, flutter_poolakey ^2.2.0+1.0.0.*
