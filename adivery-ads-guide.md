# Adivery Ads Integration Guide

A reusable guide based on how ads are implemented in **Arzham**. Use this when adding [Adivery](https://adivery.com) monetization to other Flutter projects.

---

## Overview

| Piece | Role |
|-------|------|
| `adivery` package | Official Flutter plugin (Android only) |
| `AdConstants` | App ID + placement IDs (with build-time overrides) |
| `AdService` | One-time SDK init + platform / entitlement gating |
| `AdBanner` | UI widget that shows a banner only to free users |
| `isProProvider` | Subscription state — Pro users never see ads |

**Ad types supported by the plugin:** banner, interstitial, rewarded, app-open, native.

**What Arzham ships today:** banner ads on the home screen for free Android users only.

---

## 1. Adivery publisher setup

Before writing code:

1. Create an account at [adivery.com](https://adivery.com).
2. Register your Android app and copy the **App ID** (UUID).
3. Create ad units in the dashboard and copy each **Placement ID** (UUID).
4. Match placement type to usage:
   - **Banner** → `BannerAd` widget
   - **Interstitial** → `prepareInterstitialAd` + `show`
   - **Rewarded** → `prepareRewardedAd` + `show`
   - **App open** → `prepareAppOpenAd` + `showAppOpenPlacement`
   - **Native** → `NativeAd` class

Keep IDs out of scattered widgets — centralize them (see step 3).

---

## 2. Add the dependency

In `pubspec.yaml`:

```yaml
dependencies:
  adivery: ^4.8.8
```

Run:

```bash
flutter pub get
```

No extra Gradle or manifest entries were required in Arzham beyond `INTERNET` (which most apps already have). The plugin’s own `AndroidManifest.xml` is empty; it piggybacks on your app manifest.

---

## 3. Centralize IDs — `AdConstants`

**File:** `lib/core/constants/ad_constants.dart`

```dart
class AdConstants {
  AdConstants._();

  static const appId = 'YOUR-APP-ID-HERE';

  /// Override at build time:
  /// flutter run --dart-define=ADIVERY_BANNER_PLACEMENT_ID=your-placement-id
  static const homeBannerPlacementId = String.fromEnvironment(
    'ADIVERY_BANNER_PLACEMENT_ID',
    defaultValue: 'YOUR-BANNER-PLACEMENT-ID',
  );
}
```

**Why `String.fromEnvironment`?**

- Different placement IDs per flavor (dev/staging/prod) without code changes.
- CI can inject IDs via `--dart-define` without committing secrets to git (optional).

**For a new project**, add more constants as you add ad types:

```dart
static const interstitialPlacementId = String.fromEnvironment(
  'ADIVERY_INTERSTITIAL_PLACEMENT_ID',
  defaultValue: '',
);

static const rewardedPlacementId = String.fromEnvironment(
  'ADIVERY_REWARDED_PLACEMENT_ID',
  defaultValue: '',
);
```

---

## 4. Initialize once — `AdService`

**File:** `lib/services/ad_service.dart`

```dart
import 'package:adivery/adivery.dart';
import 'package:flutter/foundation.dart';

import '../core/constants/ad_constants.dart';

class AdService {
  const AdService._();

  static bool _initialized = false;

  /// Adivery is Android-only in the current plugin.
  static bool get isSupported =>
      !kIsWeb &&
      defaultTargetPlatform == TargetPlatform.android &&
      !const bool.fromEnvironment('FLUTTER_TEST');

  static Future<void> initialize() async {
    if (_initialized || !isSupported) return;

    AdiveryPlugin.initialize(AdConstants.appId);

    if (kDebugMode) {
      AdiveryPlugin.setLoggingEnabled(true);
    }

    _initialized = true;
  }

  /// Hide ads for paying users and unsupported platforms.
  static bool shouldShowAds({required bool isPro}) => !isPro && isSupported;
}
```

**Design decisions:**

| Check | Reason |
|-------|--------|
| `!kIsWeb` | Plugin has no web implementation |
| `TargetPlatform.android` | `BannerAd` renders `AndroidView`; other platforms get a fallback text widget |
| `FLUTTER_TEST` | Skips native SDK during `flutter test` |
| `!isPro` | Ads are a free-tier feature; subscribers should not see them |
| `kDebugMode` logging | Easier to debug load failures without shipping verbose logs |

Call `initialize()` **before** `runApp`, as early as other SDK setup:

**File:** `lib/main.dart`

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await AdService.initialize();
  // ... other init (prefs, notifications, etc.)

  runApp(const MyApp());
}
```

---

## 5. Gate ads on subscription — `isProProvider`

Arzham uses Riverpod. The pattern is: **one source of truth for “user paid”**, widgets read it, ads respect it.

```dart
// Simplified pattern
final isProProvider = NotifierProvider<ProEntitlement, bool>(ProEntitlement.new);

class ProEntitlement extends Notifier<bool> {
  @override
  bool build() => prefs.getBool('is_pro') ?? false;

  Future<void> setPro(bool value) async {
    state = value;
    await prefs.setBool('is_pro', value);
  }
}
```

When the user purchases or restores a subscription, call `setPro(true)`. When it expires, `setPro(false)`.

**Debug tip (Arzham):** `kForceProForTesting` in `dev_config.dart` forces Pro in debug builds so you can develop without ads. Set it to `false` when testing ad placement.

---

## 6. Banner widget — `AdBanner`

**File:** `lib/shared/widgets/ad_banner.dart`

```dart
import 'package:adivery/adivery_ads.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../core/constants/ad_constants.dart';
import '../../providers/subscription_providers.dart';
import '../../services/ad_service.dart';

class AdBanner extends ConsumerWidget {
  const AdBanner({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isPro = ref.watch(isProProvider);

    if (!AdService.shouldShowAds(isPro: isPro)) {
      return const SizedBox.shrink();
    }

    final placementId = AdConstants.homeBannerPlacementId;
    if (placementId.isEmpty) {
      return const SizedBox.shrink();
    }

    return Center(
      child: BannerAd(
        placementId,
        BannerAdSize.BANNER,
        onAdLoaded: (_) {
          if (kDebugMode) {
            debugPrint('Adivery banner loaded: $placementId');
          }
        },
        onError: (_, reason) {
          if (kDebugMode) {
            debugPrint('Adivery banner error ($placementId): $reason');
          }
        },
      ),
    );
  }
}
```

**Banner sizes** (`BannerAdSize`):

| Enum | Size (dp) |
|------|-----------|
| `BANNER` | 320 × 50 |
| `LARGE_BANNER` | 320 × 100 |
| `MEDIUM_RECTANGLE` | 300 × 250 |

**Placement in UI (Arzham home screen):**

```dart
Column(
  children: [
    Expanded(child: /* main content */),
    Padding(
      padding: EdgeInsetsDirectional.only(
        bottom: navigationBarClearance(context) - spacing,
      ),
      child: const AdBanner(),
    ),
  ],
)
```

Pad above the bottom nav so the banner does not sit under the tab bar. Adjust padding per screen layout.

**`AdListItem` stub:** Arzham also has an `AdListItem` widget reserved for inline list ads. It currently returns `SizedBox.shrink()` — a placeholder for future native or medium-rectangle slots between list rows.

---

## 7. Android configuration

**`AndroidManifest.xml`** — only requirement in Arzham:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

No Adivery-specific `<meta-data>`, activities, or receivers were added.

**`build.gradle.kts`** — no Adivery-specific `minSdk`, ProGuard rules, or dependencies. The plugin handles native wiring via Flutter’s plugin registration.

---

## 8. Extending to other ad formats

Arzham only uses banners today. The `adivery` package supports more. Pattern from the official example:

### Interstitial / Rewarded / App open (full-screen)

Initialize listeners and preload after `AdiveryPlugin.initialize`:

```dart
static Future<void> initialize() async {
  // ... existing init ...

  AdiveryPlugin.prepareInterstitialAd(AdConstants.interstitialPlacementId);
  AdiveryPlugin.prepareRewardedAd(AdConstants.rewardedPlacementId);
  AdiveryPlugin.prepareAppOpenAd(AdConstants.appOpenPlacementId);

  AdiveryPlugin.addListener(
    onInterstitialLoaded: (placement) { /* preload ready */ },
    onInterstitialClosed: (placement) {
      // Preload again for next show
      AdiveryPlugin.prepareInterstitialAd(placement);
    },
    onRewardedClosed: (placement, isRewarded) {
      if (isRewarded) { /* grant reward */ }
      AdiveryPlugin.prepareRewardedAd(placement);
    },
    onError: (placement, reason) {
      debugPrint('Ad error ($placement): $reason');
    },
  );
}
```

Show when appropriate (e.g. after an action, on app resume):

```dart
Future<void> showInterstitialIfReady() async {
  if (!AdService.shouldShowAds(isPro: isPro)) return;

  final id = AdConstants.interstitialPlacementId;
  final loaded = await AdiveryPlugin.isLoaded(id);
  if (loaded == true) {
    AdiveryPlugin.show(id);
  }
}

Future<void> showAppOpenIfReady() async {
  final id = AdConstants.appOpenPlacementId;
  final loaded = await AdiveryPlugin.isLoaded(id);
  if (loaded == true) {
    AdiveryPlugin.showAppOpenPlacement(id);
  }
}
```

**Best practices:**

- Preload after init and again after each close.
- Do not show interstitials on every navigation — use natural breakpoints.
- App-open ads: show on cold start / return from background, not mid-task.
- Always gate with `shouldShowAds(isPro: ...)`.

### Native ads (custom layout in a list)

```dart
class _MyScreenState extends State<MyScreen> {
  NativeAd? _nativeAd;

  @override
  void initState() {
    super.initState();
    if (AdService.isSupported) {
      _nativeAd = NativeAd(
        AdConstants.nativePlacementId,
        onAdLoaded: () => setState(() {}),
        onError: (reason) => debugPrint('Native ad error: $reason'),
      );
      _nativeAd!.loadAd();
    }
  }

  @override
  void dispose() {
    _nativeAd?.destroy();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_nativeAd?.isLoaded != true) return const SizedBox.shrink();

    return Column(
      children: [
        if (_nativeAd!.headline != null) Text(_nativeAd!.headline!),
        if (_nativeAd!.image != null) _nativeAd!.image!,
        if (_nativeAd!.callToAction != null)
          ElevatedButton(
            onPressed: _nativeAd!.recordClick,
            child: Text(_nativeAd!.callToAction!),
          ),
      ],
    );
  }
}
```

Call `nativeAd.destroy()` when the widget is removed to avoid leaks.

---

## 9. File map (copy to a new project)

```
lib/
├── core/constants/ad_constants.dart   # App ID + placement IDs
├── services/ad_service.dart           # Init + shouldShowAds()
├── shared/widgets/ad_banner.dart      # Banner UI + optional AdListItem
├── providers/subscription_providers.dart  # isProProvider (or your billing state)
└── main.dart                          # await AdService.initialize()
```

Wire `AdBanner` (or other ad widgets) only on screens where ads belong. Keep gating logic in the service/widgets, not duplicated in every screen.

---

## 10. Build & run

**Default (uses IDs from `AdConstants`):**

```bash
flutter run
```

**Override placement ID at build time:**

```bash
flutter run --dart-define=ADIVERY_BANNER_PLACEMENT_ID=your-staging-placement-id
```

**Release build:**

```bash
flutter build apk --release \
  --dart-define=ADIVERY_BANNER_PLACEMENT_ID=your-prod-placement-id
```

Test on a **real Android device** or emulator with Google Play services. Ads may not fill in debug or outside Iran depending on Adivery inventory.

---

## 11. Testing checklist

- [ ] `AdService.initialize()` runs before first `BannerAd` is built
- [ ] Banner appears for free users on Android
- [ ] Banner hidden when `isPro == true`
- [ ] Banner hidden on iOS / web (no crash, just empty space)
- [ ] `flutter test` passes (`FLUTTER_TEST` skips SDK init)
- [ ] Bottom padding clears nav bar / FAB / system inset
- [ ] Debug logs show load success or error reason
- [ ] Release APK tested on device with real placement IDs

---

## 12. Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| Empty space, no ad | Pro user, wrong platform, empty placement ID, or no fill |
| `onError` in logs | Wrong placement ID, app not approved, or network issue |
| Ads in tests fail | Ensure `FLUTTER_TEST` guard in `isSupported` |
| Banner under nav bar | Add bottom padding (see home screen pattern) |
| iOS shows error text | Expected — plugin is Android-only today |

Enable logging in debug:

```dart
AdiveryPlugin.setLoggingEnabled(true);
```

---

## 13. Minimal copy-paste starter (new project)

1. Add `adivery: ^4.8.8` to `pubspec.yaml`.
2. Copy `ad_constants.dart`, `ad_service.dart`, `ad_banner.dart`.
3. Replace App ID and placement ID with your Adivery dashboard values.
4. Call `await AdService.initialize()` in `main()`.
5. Add `const AdBanner()` where you want the banner.
6. Implement `isPro` (or any entitlement flag) and pass it to `shouldShowAds`.

If you do not have subscriptions yet, simplify gating:

```dart
static bool shouldShowAds() => isSupported;
```

Add `isPro` back when you introduce a paid tier.

---

## Reference links

- [adivery on pub.dev](https://pub.dev/packages/adivery)
- [Adivery publisher dashboard](https://adivery.com)

---

*Generated from the Arzham codebase implementation. Last updated: June 2026.*
