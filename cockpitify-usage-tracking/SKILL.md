---
name: cockpitify-usage-tracking
description: Add or extend Cockpitify usage analytics (screens, feature events, sessions) in an app whose backend is Supabase. Use when the user asks to add Cockpitify, track screens or features, "feature tracking", usage analytics, Ops.init or Ops.track, or when adding a new feature to an app that already calls Ops.track. Covers React Native / Expo, web, Flutter and plain HTTP.
---

# Cockpitify usage tracking

Cockpitify records which screens and features people use into an `ops` layer installed in the app's
**own** Supabase project. The developer then sees usage per feature, premium vs free, funnels and
who used what in the Cockpitify app. Use only the APIs below; don't invent others.

## Before you start

- The `ops` layer must already be installed in the app's Supabase project (the developer does this
  once in the Cockpitify app: Connect → one-tap install). Nothing to install on the database from code.
- No keys, URLs or new clients are needed: the SDK uses the app's existing Supabase client and the
  signed-in user's session.
- Find the platform first: `package.json` with `expo` or `react-native` → React Native / Expo;
  a web framework → web; `pubspec.yaml` → Flutter ([the Flutter reference](references/flutter.md));
  anything else → [the HTTP reference](references/http.md).

## React Native / Expo

```sh
npx expo install @cockpitify/react-native @react-native-async-storage/async-storage
```

`Ops.init` once, at module level of the root layout (not inside a component or effect), with the
app's existing Supabase client:

```tsx
// app/_layout.tsx (Expo Router)
import { Ops } from '@cockpitify/react-native';
import { OpsProvider } from '@cockpitify/react-native/expo-router';
import Constants from 'expo-constants';
import { supabase } from '@/lib/supabase'; // the app's own client

Ops.init(supabase, { appVersion: Constants.expoConfig?.version });

export default function RootLayout() {
  return <OpsProvider><Stack /></OpsProvider>;
}
```

- React Navigation instead of Expo Router: `const ops = useOpsNavigation()` from
  `@cockpitify/react-native/react-navigation`, then
  `<NavigationContainer ref={ops.ref} onReady={ops.onReady} onStateChange={ops.onStateChange}>`.
- Screens are then recorded automatically, by route pattern (`/quiz/[id]`), not the concrete path.
- AsyncStorage is a native module: if the app didn't have it, it needs a new build (an OTA update
  is not enough). Say so in your report.

## Web

```sh
npm install @cockpitify/js
```

```ts
import { Ops } from '@cockpitify/js';
import { supabase } from './supabase'; // the app's own client

Ops.init(supabase); // page views from the History API from here on
```

Options: `appVersion`, `pageViews` (default `true`), `screenName(pathname)` to name pages yourself.

## Not on Supabase Auth (React Native and web)

If the app signs users in with something else (Firebase Auth, its own server), send batches to the
app's own endpoint instead of the Supabase client:

```ts
import { Ops, httpTransport } from '@cockpitify/react-native'; // or '@cockpitify/js'

Ops.init(httpTransport({ endpoint: 'https://api.example.com/ops', getToken: () => auth.currentUser?.getIdToken() }));
```

That endpoint verifies the token and calls `ops.ingest_batch(user_id, batch, country)` in the
Supabase project (see [the HTTP reference](references/http.md)). Add the endpoint only if the
developer asks; otherwise report that it is needed.

## Events: what to track

`Ops.track(name, props?)` marks the moment a feature was **really used**, not every tap.

1. Read the code and list the app's main features first (what a user comes to the app to do, what
   leads to a purchase).
2. For each feature, track its start and, if it has one, its completion:
   `memorization_started` / `memorization_completed`, `report_exported`, `purchase_started`,
   `chat_message_sent`.
3. Rules:
   - `name`: snake_case, object then past-tense verb, at most 100 characters. Never put ids or user
     input in the name (`quiz_started`, not `quiz_42_started`).
   - `props`: flat, at most 20 keys; values string, number or boolean (`level: 3`, `mode: 'review'`,
     `duration_s: 42`). No objects or arrays.
   - **No personal data**: no e-mail, name, phone or free text the user typed. (The SDK redacts
     strings that look like e-mails or phone numbers, but don't rely on it.)
   - Recording never throws and calls before `Ops.init` are queued: don't wrap calls in try/catch
     or guard them.
4. Keep event names stable: Cockpitify features, funnels and alerts refer to them by name.

## Paywalls

Wherever the app shows a paywall (RevenueCat Paywalls, a custom paywall screen, an upgrade sheet),
call `Ops.paywallView({ offering, placement })` once when it appears (Flutter:
`Ops.paywallView(offering: …, placement: …)`):

- `offering`: the RevenueCat offering id the paywall shows (`offering.identifier`); RevenueCat puts
  the same id on the purchase, so Cockpitify works out each paywall's conversion.
- `placement`: where in the app it was opened (`onboarding`, `settings`, `feature_locked`), short
  snake_case.

Don't also send a separate `paywall_viewed` event: `paywallView` is never sampled out and is the one
Cockpitify reads.

## Don't

- Don't call `Ops.init` twice or create a second Supabase client for it.
- Don't add `Ops.screen` calls where screens are already recorded automatically.
- Don't call `Ops.optOut()` unless the app already has a tracking-consent setting to connect it to
  (then `Ops.optOut()` / `Ops.optIn()` follow that setting).
- Don't change navigation, app config or native files for Cockpitify.

## Verify

Ask the developer to open the app, visit a few screens and use a tracked feature, then check
Cockpitify → the app → Summary → Feature usage → **Setup and settings**: the **Integration status**
list ticks "Events are arriving", "Screens are recorded", "Custom events" and "Events link to users",
and **Live events** shows each screen and event within seconds. If "Events link to users" stays
open, `Ops.init` is not getting the app's signed-in Supabase client.

## Report

When done, list:

- the files you changed (and whether a new native build is needed);
- a table of features for the developer to define in Cockpitify (Feature usage → Define a feature):

  | Feature (name) | Screens (route patterns, `/prefix/*` allowed) | Events |
  |---|---|---|
  | Review | `/review/*` | `review_started`, `review_completed` |

  A feature's key is made from its name (`Recitation check` → `recitation_check`); use the same key
  as `feature` when recording AI cost (see the cockpitify-ai-cost skill), so cost and usage line up.
