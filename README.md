# Extending Charming apps for headless interaction from the Pebble Index 01

Notes from wiring a handful of personal Charming apps up to the Pebble Index 01 AI voice ring, so they can be driven from the wrist as well as from a screen.

**[Visual overview →](https://nigelaclarke.github.io/charming-index01-guide/)**

## The stack

[Charming](https://usecharming.com/) lets you make real apps, quickly. You describe what you want and you get something you can load on your phone, open in a browser, and easily share with others. Charming handles the frontend and the backend, so you get an app that works the way normal apps work.

It also exposes your app's declared operations over MCP, and lets you handle raw HTTP requests directly. So the same app you built to look at is simultaneously a set of tools an agent can call and an endpoint a device can post to. Nothing about the UI has to change. You're adding a parallel entry point alongside it.

[Pebble Index 01](https://repebble.com) is a wearable that captures voice and transcribes it. No screen, no display of any kind. The Pebble app orchestrates an agent over that transcript, and that agent can carry MCP capabilities.

When paired together, you can build whatever custom app you want in Charming and pair it with a wearable agentic interface. It sits on your finger, it's there whether or not your phone is with you, it captures whether or not you have a connection, and it delivers once it can.

## Three core aspects

**Connect over MCP with a validated access token.** Charming exposes an MCP endpoint at `https://charm.ing/mcp`, and the client authenticates against it with a token scoped to the account ([client setup strings](https://charm.ing/docs/clients.txt), [auth contract](https://charm.ing/docs/technical-reference/authentication.md)). After that, every app's declared operations show up as callable tools, and the agent chooses between them on names and descriptions alone.

**Name operations and data so they explain themselves.** Costs nothing at build time, pays for itself immediately, and I got it wrong on the first two apps.

**Optionally, skip the agent when there's no judgement to exercise.** A ring firing a transcript into a capture app doesn't want a model in the loop. It wants the text stored verbatim and fast, so that path can be a plain HTTP POST with no MCP round trip. Keep the agent for the calls that genuinely need reasoning: which app, what did that fuzzy number mean, what should happen next.

---

# Quickstart

## 1. Configure the Pebble app

Both paths are configured in the Pebble phone app. The ring is the capture device; the transcription and the agent run on the phone or in the cloud depending on your settings.

**MCP server**

```
URL      https://charm.ing/mcp
Auth     Charming access token
Model    High capability
```

I'd recommend starting with the MCP sandbox model type set to "high capability". Routing across a dozen custom ops with overlapping names asks more of a model than the built-in actions do, so start there and dial back once your op names have settled. Use Local LLM keeps the agent on-device if you'd rather; speech recognition is a separate Cloud / Local / fallback setting ([reference](https://repebble.com/blog/how-i-use-my-index-01-production-update)).

**Webhook** (optional, for capture that needs no reasoning)

```
URL      https://charm.ing/app/<app-uuid>/api/ingest
Header   Authorization: Bearer chrm_...
Send     Transcript only
```

Use the UUID, not the `/<handle>/<app-name>` slug. The slug is the shareable human link; `/app/<uuid>` is the machine API base and the only one that routes to your handler.

## 2. Write the app

### Naming and discovery

Your op names land in a flat namespace next to every other app's ops. That's the whole problem.

- **Verb + domain noun.** `logWaterIntake`, `setEpisodeStatus`. Not `create`, `update`, `clear`. Bare nouns (`prefs`, `regions`) read as data and get skipped.
- **Rename in place.** Aliases double the surface the model reasons over.
- **Put policy in the app description.** It's matched before the agent sees any op. Include actions, synonyms (*jot, memo, voice note*), and any must/never rule: *always call `listEpisodes`; never answer from general knowledge*.
- **Say what the app can't do.** "There is no reset operation" stops an agent inventing one.

### Inputs

Voice transcription is fuzzy and agents extrapolate.

- **Normalise near-misses.** For an episode number, accept `S01E13`, `s1e13`, `1.13`, `1-13`; canonicalise to `1x13`.
- **Reject the rest loudly, with a message the agent can act on.** Never store an unrecognised value verbatim. Say what was wrong, what format you expect, and which op returns the valid set, so the next call is corrected rather than repeated.
- **No defaults for missing required values.** An absent identifier is a no-op, not a guess.
- **Enums over free numbers.** Named sizes on a 2× ladder (250 / 500 / 1000 ml) leave no gap for a fuzzy caller to land in. Keep a numeric escape hatch.
- **Units in the schema description**, not just in code: *glass (250 ml / 8.5 oz)*. The agent picks on that string alone.
- **Define "verbatim" explicitly where it matters.** *In the user's own words; do not summarise or add a heading.* Otherwise a model may summarise the transcript.
- **Accept an optional timestamp.** Declare an `at` field in the input schema and default to now when it's absent. Nothing fills it automatically: the agent supplies it when the user says "log that from this morning", and a webhook can pass the device's own recorded-at so a delayed delivery still lands on the right local day.
- **`additionalProperties: false` everywhere.** Turns a hallucinated field into an error instead of a silent drop.

### Outputs

- **Return full current state from every mutation.** Confirmation and new totals in one round trip. Biggest single improvement to how the ring feels.
- **Return the reasoning, not just values.** If your UI explains why an item is flagged, the op should too.
- **Declare an output schema.**
- **Set all four annotation hints explicitly** (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`). Charming doesn't infer them from method, and a route without `readOnlyHint` is invisible to view-only mode. External API calls are `openWorldHint: true`.
- **Read ops must not write.** Clients may cache or replay them. Push migrations onto the write path.

### Errors

A thrown error reaches the caller as bare `operation_failed`, which tells an agent nothing, so it retries the same bad call. Return them instead:

```js
{ status: "error", code: "bad_region", message: "...", ...unchangedState }
```

- **`code`** enumerated and machine-branchable: `bad_credentials`, `rate_limited`, `no_secret`.
- **`message`** instructive: what was wrong, what to send instead, which op to call to find out.
- **Unchanged state alongside**, so the absence of a change is itself a signal.
- **Separate your fault from upstream's.** "Credentials are fine, this is upstream" produces different agent behaviour than a credentials error.

### Webhook ingestion

Charming parses a declared route's body as JSON before your handler runs, so `multipart/form-data` gets rejected before arrival. Ingest lives in the default fetch export:

```js
export default {
  async fetch(request, env, ctx) {
    // raw Request; parse the form yourself
  }
};
```

- **Validate by hand, return real status codes.** 404 wrong path/method, 400 unparseable body, 400 missing field. The device's retry behaviour depends on it.
- **Add a read-only health op.** The catch-all is invisible to agents, so "why didn't my note arrive" has nothing to call. Return last-received timestamp, count, last rejection reason.
- **Don't assume auth means identity.** Charming requires a bearer on external `/api/*`, but that token is broad and doesn't tell you which holder is calling. Send and check a second header if you need to distinguish the device.
- **Add an idempotency key.** Webhooks retry; without one, a delivery timeout duplicates the record and counters double-count.
- **Stamp provenance at write time.** Fingerprint how much of the HTTP envelope survives (MCP dispatches in-process and carries almost nothing; a browser carries an origin). Fail safe: record unknown shapes as "unknown external", never as the agent.

### Identity and storage

Caller identity resolves only in a browser render session. **For MCP and API callers it's null**, so a per-user app dumps every agent write into the anonymous bucket and looks broken.

Fallback chain: platform user → payload `uid` → `client: "ui"` marker your frontend always sends → owner. Step three is what lets an agent write to a personal list without being handed an id.

- **One storage key per independently-mutable fact.** `mark:<uid>:<itemId>`, absent when unset. A single blob makes every write a read-modify-write, and a stale read resurrects values you cleared.
- **Pass caller params through generically.** Hand-listing silently drops what you forget.
- **Normalise legacy shapes on read**, rewrite on next write, gate migrations behind a per-user flag.
- **Drop unknown values** rather than storing them, so a bad write can't wedge a row into a state the UI can't cycle out of.
- **Cap and prune on write.** Unattended writers grow storage forever.

### Time and external APIs

- **Server owns "today".** Store an IANA timezone and derive the date in the backend. A device has no browser to ask.
- **Prefer stale over failed.** Cache the last good response; on soft failure return it marked `status: "stale"` with a reason and timestamp. A labelled stale answer beats an error when there's no screen.
- **Tokens:** cache with margin (25 min on a 30 min token), retry login on transport failure, and on 401/403 delete, re-login, retry once.
- **Secrets:** `{{secret:NAME}}` resolves in header and query-param values only. Write it literally; encoding helpers break it. Prefer headers, since query strings hit access logs. ([reference](https://charm.ing/docs/capabilities/secrets.md))
- **Detect a missing secret specifically** and return its own code, or it presents as a credentials failure.

### UI, when something else is writing

A render can fire while the user is mid-interaction. That's the failure mode a normal app never sees.

- **Hold in-progress input in state, not the DOM** — draft text *and* caret offsets, or a re-render throws the cursor to the end.
- **Guard refresh with a `saving` flag** so an inbound change can't clobber an in-flight write.
- **Send resolved values, not toggles.** `mark: "watched"` means a double tap can't drift local and stored state apart.
- **Surgical DOM updates over wiping the root.** A freshly inserted node has no previous state to transition from.
- **Backend is the single source of truth.** Diverge and the ring answers questions your UI can't corroborate.

## Three failures that cost the most

**Silent success.** An op that swallows a malformed value, stores it, and returns `ok` looks healthy for weeks. No screen reveals it, no next turn catches it. This is the worst outcome available to you.

**Unmodelled absence.** An agent can't reason about a gap, only about a record, so it fills gaps from training data. Storing 73 of 103 episodes made "not on your path" and "no record" identical, and the ring confidently reported a cut episode as essential. Store all 103, mark 30 `excluded`, give each a reason.

**Three writers, one blob.** A ring, an agent and a browser can all write inside the same second. Watch a cleared item reappear on the back of an unrelated toggle and you'll never use a single-blob layout again.

## Checklist

- [ ] Operation names are verb + domain noun, unambiguous in a flat namespace
- [ ] App description carries actions, synonyms, and any must/never policy
- [ ] Exclusions and unsupported cases stored as records, not omissions
- [ ] Near-miss inputs normalised, unrecognised inputs rejected with instructions
- [ ] `additionalProperties: false` on every schema
- [ ] Optional explicit timestamp and idempotency key on every write
- [ ] Writes return full current state
- [ ] Errors returned as `{ status, code, message }`, never thrown
- [ ] All four annotation hints set explicitly on every route
- [ ] Webhook path in the default fetch export: hand-validated, real status codes
- [ ] Provenance stamped at write time, failing safe on unknown callers
- [ ] Identity chain: platform user, payload id, UI marker, owner
- [ ] One storage key per independently-mutable fact
- [ ] Server owns date bucketing via a stored IANA timezone
- [ ] Cached fallback labelled stale rather than returned as an error
- [ ] Collection capped and pruned on write
- [ ] In-progress user input held in state, not only the DOM
- [ ] Backend is the single source of truth for anything the UI displays

## References

**Charming** — [llms-full.txt](https://charm.ing/docs/llms-full.txt) (full build manual) · [clients.txt](https://charm.ing/docs/clients.txt) (MCP setup per client) · [authentication](https://charm.ing/docs/technical-reference/authentication.md) · [secrets](https://charm.ing/docs/capabilities/secrets.md) · [privacy and sharing](https://charm.ing/docs/capabilities/privacy-and-sharing.md) · [data storage](https://charm.ing/docs/capabilities/data-storage.md) · [openapi.json](https://charm.ing/.well-known/openapi.json)

**Pebble Index 01** — [getting started](https://help.repebble.com/en/articles/15434751-index-01-getting-started-guide) · [FAQ and tips](https://help.repebble.com/en/articles/15948728-index-01-faq-and-tips) · [webhooks, MCP and local models](https://repebble.com/blog/how-i-use-my-index-01-production-update)