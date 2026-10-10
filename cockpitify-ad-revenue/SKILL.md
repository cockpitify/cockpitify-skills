---
name: cockpitify-ad-revenue
description: Record AdMob ad revenue per user for Cockpitify in a React Native / Expo app that shows ads with react-native-google-mobile-ads (banners, interstitial, rewarded, rewarded interstitial, app open, native). Use when the user asks to connect ad revenue to users, add AdMob revenue to Cockpitify's margin or profit, or adds a new ad to an app that uses Cockpitify.
---

# Cockpitify ad revenue (AdMob)

AdMob reports what each impression earned to the app (impression-level ad revenue). The
`@cockpitify/react-native/admob` helpers send it to the app's own Supabase project, next to the user,
and Cockpitify adds it to the user's margin and the app's profit. Use only the APIs below.

## Before you start

- Usage tracking must be set up: `@cockpitify/react-native` (0.1.2 or later) installed and
  `Ops.init(supabase)` called once at app start. If it isn't, set it up first (see the
  cockpitify-usage-tracking skill); if it is, leave it alone.
- Find every ad: `BannerAd`, `GAMBannerAd`, native ads, and `InterstitialAd`, `RewardedAd`,
  `RewardedInterstitialAd`, `AppOpenAd` (created with `createForAdRequest` or through the hooks
  `useInterstitialAd`, `useRewardedAd`, `useRewardedInterstitialAd`, `useAppOpenAd`).

Import from the subpath, not the package root:

```tsx
import { onAdPaid, recordAdRevenue, trackAdRevenue } from '@cockpitify/react-native/admob';
```

## Connect each ad

| Ad | What to add |
|---|---|
| `BannerAd`, `GAMBannerAd` | prop `onPaid={onAdPaid('banner', unitId)}` |
| Native ad | in its paid handler: `onAdPaid('native', unitId)` |
| `InterstitialAd` | `trackAdRevenue(ad, 'interstitial')` |
| `RewardedAd` | `trackAdRevenue(ad, 'rewarded')` |
| `RewardedInterstitialAd` | `trackAdRevenue(ad, 'rewarded_interstitial')` |
| `AppOpenAd` | `trackAdRevenue(ad, 'app_open')` |

- `trackAdRevenue` once per ad object, right after `createForAdRequest`, before `load()`. It returns
  a function that stops listening: call it when the ad object is dropped (an effect cleanup). If the
  app creates a new ad object after each show, track each new one and stop the old.
- With the hooks, record each new paid event the hook returns:

  ```tsx
  const { revenue } = useRewardedAd(unitId);
  useEffect(() => { if (revenue) recordAdRevenue(revenue, 'rewarded', unitId); }, [revenue]);
  ```

- Format names are exactly: `banner`, `interstitial`, `rewarded`, `rewarded_interstitial`,
  `app_open`, `native`. Pick by the ad's type, not by where it is shown.

## Rules

- Pass AdMob's paid event as it is. Never compute, convert or round the value, currency or precision
  (other currencies are converted to USD by Cockpitify at the day's ECB rate).
- Keep existing `paid` listeners (other analytics or mediation tools); add alongside them.
- Don't change how ads are loaded, shown or how often.
- No new packages besides `@cockpitify/react-native`, no native config or `app.json` changes.

## The developer must also

Turn on **Impression-level ad revenue** in AdMob (Settings → Account → Account controls). Code can't
do this, and without it AdMob sends no paid events. Remind them in your report.

## Verify

Show an ad in the app (test ads may report a value of 0), then open Cockpitify → the app → Margin tab
→ **Connect AdMob ad revenue**: it switches from "Waiting for the first impression…" to the count of
impressions within seconds. The setup checklist under
Feature usage → Setup and settings also ticks "Ad revenue".

## Report

List the files you changed and which ad got which format:

| File | Ad (unit id or name) | Format | How (onAdPaid / trackAdRevenue / recordAdRevenue) |
|---|---|---|---|
