# Flutter

The Flutter package (`ops_flutter`) is not on pub.dev yet. Until it is, send events over HTTP
([http.md](http.md)) from a small Dart class, or ask the developer whether they have the package
from Cockpitify directly.

When the package is available:

```dart
import 'package:ops_flutter/ops_flutter.dart';

await Supabase.initialize(url: …, anonKey: …);              // the app's existing setup
await Ops.init(Supabase.instance.client, appVersion: '2.3.1');

MaterialApp(navigatorObservers: [Ops.observer]);              // GoRouter: GoRouter(observers: [Ops.observer], …)

Ops.track('quiz_started', {'level': 3});                      // when a feature is used
```

- Screens come from route names (`RouteSettings.name`); id-like segments become `[id]`
  (`/quiz/42` → `/quiz/[id]`). Unnamed routes and dialogs are not screens, so give the app's main
  routes names if they have none.
- Without Supabase: `Ops.init(CallbackTransport((batch) => postToYourServer(batch)))`; the server
  verifies its own token and calls `ops.ingest_batch(user_id, batch, country)`.
- The event rules in SKILL.md apply unchanged.
