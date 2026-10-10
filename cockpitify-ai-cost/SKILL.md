---
name: cockpitify-ai-cost
description: Record each user's AI cost for Cockpitify by wrapping server-side AI calls with @cockpitify/node (OpenAI, Anthropic, DeepSeek, Groq, Gemini, OpenRouter, Vercel AI SDK, raw fetch, speech/image calls). Use when the user asks to connect AI cost to users, track AI spend per user or per feature, or adds a new AI call to a backend (Supabase Edge Functions or Node) of an app that uses Cockpitify.
---

# Cockpitify AI cost

`@cockpitify/node` writes one row per AI call (user, provider, model, tokens, cost) to `ops.usage`
in the app's **own** Supabase project. Cockpitify then shows AI cost per user, per model and per
feature, and profit per user (revenue minus AI cost). Use only the APIs below.

## Before you start

- The `ops` layer must already be installed in the app's Supabase project (Cockpitify app: Connect →
  one-tap install).
- AI calls must happen on a server: Supabase Edge Functions (Deno) or Node. Never wrap calls in the
  mobile or web client, and never move a key into the client.
- Find every AI call first: SDK clients (`new OpenAI(`, `new Anthropic(`), the Vercel AI SDK
  (`generateText`, `streamText`), and raw `fetch` to AI endpoints.

## Install

- Supabase Edge Functions (Deno): import `npm:@cockpitify/node` (or add it to the function's
  `deno.json` import map if the project uses one).
- Node: `npm install @cockpitify/node` (use the project's package manager).

## Wrap the client, per request

```ts
import OpenAI from 'npm:openai';
import { Ops } from 'npm:@cockpitify/node';

const ai = Ops.wrap(new OpenAI({ apiKey }), {
  userId: user.id,          // the Supabase auth id of the end user who caused the call
  supabase: adminClient,    // a service-role supabase-js client of the same project
  feature: 'tafsir_chat',   // optional: which feature this call belongs to
  appVersion: req.headers.get('x-app-version') ?? undefined, // optional (ops 0.32.0+): AI cost per release
});
```

- `userId`: the app's end user (from the verified JWT, `auth.getUser()`), never a client-sent value
  and never the developer. Wrap per request, because the user changes.
- `supabase`: a service-role client (the function's `SUPABASE_SERVICE_ROLE_KEY` / secret key). Reuse
  the one the function already has; don't add new secrets.
- `feature`: snake_case key. Use the same key as the Cockpitify feature that covers this part of the
  app (a feature's key is its name in snake_case: `Tafsir chat` → `tafsir_chat`), so AI cost per
  feature lines up with usage. One key per product feature, not per prompt.

Supported clients, with the wrapped client behaving exactly like the original:

| Client | Calls recorded |
|---|---|
| OpenAI SDK and compatibles via `baseURL` (DeepSeek, Groq, OpenRouter, Gemini's `generativelanguage.googleapis.com/v1beta/openai`) | `chat.completions.create` (streaming or not), `chat.completions.stream()`, `responses.create` |
| Anthropic SDK | `messages.create` (streaming or not), `messages.stream()`, the same under `beta.messages` |
| Raw `fetch` | `const f = Ops.wrapFetch(fetch, opts)`, then use `f` for the AI request (OpenAI-compatible chat, Responses and Anthropic; JSON and SSE) |
| Vercel AI SDK | `wrapLanguageModel({ model, middleware: Ops.middleware(opts) })` |

The provider is inferred from the client or URL; pass `provider: '…'` only for an unusual endpoint.

## Calls not measured in tokens

Speech-to-text, text-to-speech and images are recorded by hand after the call succeeds:

```ts
Ops.record({ userId, supabase, provider: 'groq', model: 'whisper-large-v3', units: 742, unit: 'seconds', feature: 'recitation_check' });
Ops.record({ userId, supabase, provider: 'elevenlabs', model: 'eleven_multilingual_v2', units: 48000, unit: 'characters', costUsd: 0.36 });
```

`unit`: lower-case (`seconds`, `characters`, `images`, `requests`). Pass `costUsd` only when the
provider returns the real cost; without it the row stays unpriced rather than guessed.

## Behaviour you can rely on (don't add code for it)

- Streaming is untouched: chunks reach the caller unchanged and without delay. OpenAI-compatible
  streams get `stream_options.include_usage` added so usage is reported; the extra usage-only chunk
  is hidden. A stream cut short is recorded with estimated tokens.
- Fail-open: a recording failure never throws into the caller or slows the AI call; it goes to
  `onError` (default: one console warning). Don't wrap calls in extra try/catch for Cockpitify.
- On Supabase Edge Functions, writes outlive the response (`EdgeRuntime.waitUntil`). In Node scripts,
  queues or serverless outside Supabase, `await Ops.flush()` before the process ends.
- Setup mistakes throw at wrap time (missing `userId`, neither `supabase` nor `sink`): fix them,
  don't catch them.

## Don't

- Don't change prompts, models, parameters, streaming or error handling.
- Don't compute prices or pass `prices` unless the developer asks; ops prices rows itself.
- Don't record calls made for the developer (admin tools, batch jobs with no end user) against a user.

## Verify

Ask the developer to make an AI request in the app, then open Cockpitify → the app → Margin tab →
**Attribute AI cost to users**. The **Connect AI cost to users** screen
switches from "Waiting for the first call…" to "Connected" with the
call's tokens and cost within seconds, and the user's detail shows it. The Feature usage setup
checklist also ticks "AI cost". Calls listed as "unpriced" use a model ops has no price for yet.

## Report

List the files you changed and a table of the AI calls:

| File | Provider / model | How (wrap / wrapFetch / middleware / record) | feature |
|---|---|---|---|
