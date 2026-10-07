# Extending Charming apps for headless interaction from the Pebble Index 01

Notes from wiring personal Charming apps up to the Pebble Index 01 voice ring, so they can be driven from the wrist as well as from a screen.

By [Nigel Clarke](https://www.nigelclarke.ca) · **[Visual overview →](https://nigelaclarke.github.io/charming-index01-guide/)**

## The stack

**Charming turns an idea into an app. Pebble Index 01 puts that app in a ring.**

[Charming](https://usecharming.com/) handles the frontend, backend and storage. Every declared operation is also reachable over MCP and, through a webhook, over plain HTTP. The app you built to look at is also a set of tools an agent can call and an endpoint a device can post to. The UI doesn't change.

[Pebble Index 01](https://repebble.com) is a screenless ring that captures and transcribes voice. The Pebble phone app runs an agent over the transcript, and that agent can call MCP tools. It captures offline and delivers once it has a connection.

## Two paths in

| Path | Use it for | Endpoint | Credential |
| --- | --- | --- | --- |
| **MCP, per app** | Anything that needs judgement within one app: what a fuzzy number meant, what to do next | `https://mcp.charm.ing/<owner>/<app-name>` | `chrm_user_*` personal access token |
| **MCP, shared** | Reaching every app you own from one connection: which app, then what to do | `https://charm.ing/mcp` | `chrm_user_*` personal access token |
| **Webhook** | Verbatim capture with no model in the loop | `POST https://charm.ing/api/v1/hooks/webhook_<uuid>` | `chrm_hook_*` secret for one op |

---

# Quickstart

## 1. Connect the ring over MCP

In the Pebble app, add an MCP server. There are two URLs, and you can use either one or both:

```
# Direct: one app, its ops as tools
URL      https://mcp.charm.ing/<owner>/<app-name>

# General: every app on your account
URL      https://charm.ing/mcp

Auth     Bearer chrm_user_...
Model    High capability
```

**Direct, per-app URL (faster, less ambiguity).** It exposes one app, with each declared route as a tool named after its op (`logWaterIntake({ size: "glass" })`). There's no app id to resolve and no op selector, so each request is one call. Use it for the apps you talk to most, and whenever you have many apps or several similar ones (two trackers, two lists) that the agent could confuse. Copy the URL from the app's settings (MCP section), or use **Add as MCP server** in the Charming widget.

**General, shared URL (the ring can see all your apps).** `https://charm.ing/mcp` is one connection that reaches every app you own, including ones you build later, with no new server to add. It doesn't expose ops as tools. Instead the agent calls `list_apps` to pick the app, `get_app` to see its ops, then `query_app` (reads) or `mutate_app` (writes). That's more hops per request, and the agent has to choose the right app from your spoken words, so app names and descriptions matter more. Use it for occasional apps, or as a catch-all next to your direct connections. The same server also exposes building tools (`create_app`, `update_app`, `delete_app`). The ring's agent can reach them, though `delete_app` needs confirmation.

- **Combining them:** connect the per-app URL for your daily-driver apps and the shared URL for everything else. If you do, the same app is reachable two ways. Say in that app's description that its direct tools should be used first.
- **Keep op names distinct across apps** if you connect several per-app servers. They still share one tool list in the Pebble agent.
- **Only declared routes become tools.** Routes marked `public: false` and anything handled only in `default.fetch` are invisible. Route annotations become the tool's MCP hints.
- **Token:** create one in Account settings > Connections. Pick 7, 30 or 90 days, or a custom date up to a year. It carries your whole account, even at a per-app URL. When it expires the ring gets a `401` sign-in challenge it can't follow, so set a reminder to replace it by hand.
- **Model:** start with "high capability". Routing across many custom ops asks more of a model than the built-in actions. Dial back once your op names settle. Speech recognition (Cloud / Local / fallback) is a separate setting ([reference](https://repebble.com/blog/how-i-use-my-index-01-production-update)).

## 2. Optional: add a webhook for capture

Declare the capture op as a route, then create a webhook on it (`create_webhook({ app_id, op: "captureNote" })` over MCP, or App settings > Webhooks). You get a URL, a secret shown once, and the op's schemas. In the Pebble app:

```
URL      https://charm.ing/api/v1/hooks/webhook_<uuid>
Header   Authorization: Bearer chrm_hook_...
Send     Transcript only
```

- **Form bodies work as-is.** `multipart/form-data` and url-encoded text fields are converted to the input schema's types. File parts (audio) are dropped and the op still runs.
- **Match the input schema to the payload.** The Charming docs example posts `transcription`, `recordedAt` (epoch ms) and `audio`. Confirm Pebble's real field names from a delivery log in App settings > Webhooks, then map `recordedAt` to your `at` timestamp.
- **The platform validates.** A missing or wrong secret returns `401`, a disabled webhook `403`, a body that fails `inputSchema` `400`, rate limits `429` with `Retry-After`.
- **The secret reaches only that one op.** Rotate or disable it from settings. Rotation is immediate with no grace window, so update the ring in the same sitting.
- **Limits:** 3 webhooks per app on Free (25 per owner). 120 requests/min per webhook, 300/min per sender address. The handler gets parsed fields, not raw bytes, so HMAC body signatures can't be verified.

---

# Writing the app

## Naming and discovery

Op names are all the agent has to choose with.

- **Verb + domain noun.** `logWaterIntake`, `setEpisodeStatus`. Not `create`, `update`, `clear`. Bare nouns (`prefs`, `regions`) read as data and get skipped.
- **Rename in place.** Aliases double the surface the model reasons over.
- **Put policy in the app description.** Include actions, synonyms (*jot, memo, voice note*) and any must/never rule: *always call `listEpisodes`; never answer from general knowledge*. The description is length-capped (`description_too_long`), so keep it tight.
- **Say what the app can't do.** "There is no reset operation" stops an agent inventing one.

## Inputs

Voice transcription is fuzzy and agents extrapolate.

- **Normalise near-misses.** For an episode number, accept `S01E13`, `s1e13`, `1.13`, `1-13` and canonicalise to `1x13`.
- **Reject the rest with instructions.** Never store an unrecognised value. Say what was wrong, the expected format, and which op returns the valid set.
- **No defaults for missing required values.** An absent identifier is a no-op, not a guess.
- **Enums over free numbers.** Named sizes on a 2× ladder (250 / 500 / 1000 ml) leave no gap for a fuzzy caller. Keep a numeric escape hatch.
- **Units in the schema description:** *glass (250 ml / 8.5 oz)*. The agent picks on that string alone.
- **Define "verbatim" where it matters:** *in the user's own words; do not summarise or add a heading*.
- **Accept an optional `at` timestamp**, defaulting to now. The agent fills it for "log that from this morning", and the webhook fills it from the device's recorded time so a late delivery lands on the right day.
- **Accept an optional idempotency key on every write.** Senders retry on `429`, and an `executor_error` can report `execution: "may_have_run"`.
- **`additionalProperties: false` everywhere.** A hallucinated field becomes an error instead of a silent drop.

## Outputs

- **Return full current state from every mutation.** Confirmation and new totals in one round trip. Biggest single improvement to how the ring feels, and it lets the UI render straight from `onStateChange` events.
- **Return the reasoning, not just values.** If the UI explains why an item is flagged, the op should too.
- **Declare an output schema, and make it allow your error shape.** Output is enforced: a result that doesn't match fails with `invalid_output`.
- **Set all four annotation hints** (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`). Charming doesn't infer them from the method. Without `readOnlyHint`, `query_app` returns `not_read_only` and viewers are denied. External API calls are `openWorldHint: true`.
- **Read ops must not write.** This is enforced: a read-only route that writes fails with `forbidden_write`. Push migrations onto the write path.

## Errors

A thrown error reaches the caller as a bare `operation_failed`, so the agent retries the same bad call. Return errors instead:

```js
{ status: "error", code: "bad_region", message: "...", ...unchangedState }
```

- **`code`** is enumerated and machine-branchable: `bad_credentials`, `rate_limited`, `no_secret`.
- **`message`** says what was wrong, what to send instead, and which op to call.
- **Unchanged state alongside**, so the absence of a change is itself a signal.
- **Separate your fault from upstream's.** "Credentials are fine, this is upstream" produces different agent behaviour than a credentials error.

## Identity and provenance

`ctx.env.user` carries the signed-in caller's public identity (`{ id, handle?, name?, image? }`) for MCP and browser callers. It is `null` for anonymous callers, app API keys, unbound render tokens, and for webhooks and Routines, which run as the app.

- **Fallback chain:** `env.user` > payload `uid` > owner. The last step is what makes a webhook write land in your personal list.
- **Identity is for personalisation, not authorisation.** Access is enforced by sharing roles. `end-user` (writes data, can't edit source) suits a shared household app. Gate UI with `window.charming.viewer.can(op)`.
- **Stamp provenance at write time.** Webhook deliveries carry `x-charming-trigger: webhook`. Record anything you can't identify as "unknown external", never as the agent.

## Storage

- **One key per independently-mutable fact.** `mark:<uid>:<itemId>`, absent when unset. A single blob makes every write a read-modify-write, and a stale read resurrects values you cleared.
- **Pass caller params through generically.** Hand-listing silently drops what you forget.
- **Normalise legacy shapes on read**, rewrite on next write, gate migrations behind a per-user flag.
- **Drop unknown values** so a bad write can't wedge a row into a state the UI can't cycle out of.
- **Cap and prune on write.** Unattended writers grow storage forever.

## Time, external APIs and secrets

- **Server owns "today".** Store an IANA timezone and derive the date in the backend. A ring has no browser to ask.
- **Prefer stale over failed.** Cache the last good response; on soft failure return it as `status: "stale"` with a reason and timestamp.
- **Prefetch with a Routine.** An hourly, daily or weekly Routine on a no-input op keeps that cache fresh. Throw on hard failures so 5 consecutive failures auto-disable it and email you.
- **Tokens:** cache with margin (25 min on a 30 min token), retry login on transport failure, and on 401/403 delete, re-login, retry once.
- **Secrets:** declare `charming:secrets/fetch@1.0` and list each origin in `manifest.permissions.server.fetch` (an empty allowlist denies everything). Use `{{secret:NAME}}` or `env.secrets.NAME` in header or query-param values only. Write it literally, since `URLSearchParams` and `encodeURIComponent` break it. Prefer headers; query strings land in access logs. ([reference](https://charm.ing/docs/capabilities/secrets.md))
- **Catch a missing secret.** `env.fetch` rejects the whole request when a secret isn't set. Catch that and return `no_secret`, or it reads as a credentials failure.

## UI, when something else is writing

A write can arrive while the user is mid-interaction. A normal app never sees this.

- **Subscribe to `window.charming.onStateChange`.** It fires for every write (ring, agent, Routine, webhook, another browser) with the op name and its result (up to 32 KiB). On `reconnect-resync`, refetch everything. Don't poll.
- **Update the DOM surgically** instead of wiping the root, or you lose typed text, focus and selection.
- **Hold in-progress input in state**, including draft text and caret offsets.
- **Guard refresh with a `saving` flag** so an inbound change can't clobber an in-flight write.
- **Send resolved values, not toggles.** `mark: "watched"` means a double tap can't drift local and stored state apart.
- **Pass UI state back** with `updateContext()` and `recordAction()` so a connected agent knows what's on screen.
- **Backend is the single source of truth.** Otherwise the ring answers questions the UI can't corroborate.

## Diagnostics

- **`get_app`** includes `recentIssues` when the current revision has runtime failures. `GET /app/<id>/activity` gives the full log.
- **Webhook deliveries** are logged with request and response in App settings > Webhooks, and `list_webhooks` reports `last_outcome` and `last_error`.
- **Add a read-only health op anyway** (last received, count, last rejection reason) so the ring's agent can answer "why didn't my note arrive?".

## Three failures that cost the most

**Silent success.** An op that swallows a malformed value, stores it, and returns `ok` looks healthy for weeks. No screen reveals it and no next turn catches it.

**Unmodelled absence.** An agent can't reason about a gap, only a record, so it fills gaps from training data. Storing 73 of 103 episodes made "not on your path" and "no record" identical, and the ring confidently called a cut episode essential. Store all 103, mark 30 `excluded`, give each a reason.

**Three writers, one blob.** A ring, an agent and a browser can all write in the same second. Watch a cleared item reappear after an unrelated toggle and you'll never use a single-blob layout again.

## Checklist

- [ ] Ring connected at the per-app MCP URL for daily apps, the shared `charm.ing/mcp` URL for the rest; token expiry noted
- [ ] Op names are verb + domain noun, distinct across connected apps
- [ ] App description carries actions, synonyms and must/never policy
- [ ] Exclusions and unsupported cases stored as records, not omissions
- [ ] Near-miss inputs normalised; unrecognised inputs rejected with instructions
- [ ] `additionalProperties: false` on every schema
- [ ] Optional `at` timestamp and idempotency key on every write
- [ ] Writes return full current state
- [ ] Errors returned as `{ status, code, message }`, never thrown; output schema allows that shape
- [ ] All four annotation hints set on every route
- [ ] Capture webhook created with `create_webhook` on a declared op; field names checked against a real delivery
- [ ] Identity chain: `env.user`, payload id, owner
- [ ] Provenance stamped at write time, failing safe on unknown callers
- [ ] One storage key per independently-mutable fact; collections capped and pruned
- [ ] Server owns date bucketing via a stored IANA timezone
- [ ] Cached fallback labelled stale; Routine keeps it fresh
- [ ] Secret origins allowlisted; missing secret returns `no_secret`
- [ ] UI subscribed to `onStateChange`, updates surgically, holds input in state

## References

**Charming:** [llms-full.txt](https://charm.ing/docs/llms-full.txt) (full build manual) · [Connect one app over MCP](https://charm.ing/docs/guides/connect-an-app-over-mcp.md) · [Every app is accessible over MCP](https://charm.ing/docs/concepts/every-app-is-accessible-over-mcp.md) · [Connect to any AI agent](https://charm.ing/docs/capabilities/connect-any-agent.md) · [Webhooks](https://charm.ing/docs/capabilities/webhooks.md) · [Routines](https://charm.ing/docs/capabilities/routines.md) · [Apps that work with their agent](https://charm.ing/docs/guides/agent-connected-apps.md) · [Browser runtime](https://charm.ing/docs/technical-reference/browser-runtime.md) · [Authentication](https://charm.ing/docs/technical-reference/authentication.md) · [Secrets](https://charm.ing/docs/capabilities/secrets.md) · [Privacy and sharing](https://charm.ing/docs/capabilities/privacy-and-sharing.md) · [Data storage](https://charm.ing/docs/capabilities/data-storage.md) · [openapi.json](https://charm.ing/.well-known/openapi.json)

**Pebble Index 01:** [Getting started](https://help.repebble.com/en/articles/15434751-index-01-getting-started-guide) · [FAQ and tips](https://help.repebble.com/en/articles/15948728-index-01-faq-and-tips) · [Webhooks, MCP and local models](https://repebble.com/blog/how-i-use-my-index-01-production-update)

---

Written by Nigel Clarke ([www.nigelclarke.ca](https://www.nigelclarke.ca)). Corrections welcome.
