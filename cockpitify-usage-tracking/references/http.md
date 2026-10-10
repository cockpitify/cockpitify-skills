# Plain HTTP (Ingest Protocol v1)

For platforms without an SDK. Send batches to the app's Supabase project:

```
POST {SUPABASE_URL}/rest/v1/rpc/ops_ingest
apikey: <anon / publishable key>
Authorization: Bearer <the signed-in user's access token, or the anon key>
Content-Type: application/json

{ "batch": {
  "v": 1,
  "sdk": { "name": "my-app-http", "version": "1.0.0" },
  "sentAt": "2026-10-03T14:02:11.000Z",
  "device": { "id": "<uuid kept on the device>", "platform": "android", "os": "14", "app": "2.3.1", "locale": "tr-TR" },
  "events": [
    { "id": "<uuid>", "t": "2026-10-03T14:01:02.000Z", "type": "session_start", "name": "$start", "session": "<uuid>" },
    { "id": "<uuid>", "t": "2026-10-03T14:01:50.000Z", "type": "screen", "name": "/quiz/[id]", "ms": 48200, "session": "<uuid>" },
    { "id": "<uuid>", "t": "2026-10-03T14:01:50.000Z", "type": "event", "name": "quiz_started", "props": { "level": 3 }, "session": "<uuid>" }
  ]
} }
```

Rules:

- `device.id`: a random UUID made on first launch and stored. No advertising id, no fingerprinting.
- `event.id`: a fresh UUID per event; the server keeps the first copy, so retrying a batch is safe.
- `type`: `screen` is sent **when the screen is left**, with `ms` = time on it and `name` = the route
  **pattern** (`/quiz/[id]`, never `/quiz/42`); `event` = a feature was used; `session_start` /
  `session_end` (`ms` = length). A session ends after 30 minutes in the background.
- 1–100 events per batch, body at most 64 KB. Props: flat, ≤ 20 keys, string (≤ 200 chars), number
  or boolean, no personal data.
- The user comes only from the bearer token, never from the body. Events sent before sign-in are
  attributed to the user once the same device sends with a token.
- Queue on the device (≤ 1000, drop the oldest), send every 30 s, at 50 events and when the app goes
  to the background; back off on failure. Never block the UI or throw into app code.
- Response: `{ "accepted": n, "config": { "enabled", "sampleRate", "flushSec" } }`. `enabled: false`
  means drop the queue and stop; honour `sampleRate` per device (from the device id) and
  `retryAfterSec` when present.

Not on Supabase Auth: post the batch to your own endpoint, verify your own token there, and call
`ops.ingest_batch(user_id, batch, country)` in Postgres (`country`: ISO code if known, else null).
