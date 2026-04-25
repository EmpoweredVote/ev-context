# compass-store-broker

Cross-subdomain shared compass state for Empowered Vote apps.

## What it is

A single static HTML page that owns the canonical guest-compass state in `localStorage` on its own origin. Every EV app (`compass.empowered.vote`, `essentials.empowered.vote`, `readrank.empowered.vote`, etc.) embeds it as a hidden iframe and reads/writes via `postMessage`. Browsers isolate `localStorage` per full origin, so without this broker each subdomain has its own copy. With it, all subdomains see the same state — guests get a unified compass that follows them across apps without ever signing in.

Live updates fan out across tabs via a `BroadcastChannel` inside the broker, so calibrating in one tab updates the radar in every other open tab instantly.

## Wire format

The broker is intentionally schemaless — it stores whatever JSON you `set` and returns it on `get`. By convention EV apps store:

```json
{
  "a": { "abortion": 5, "guns": 3 },          // answers keyed by short_title
  "s": ["uuid", "uuid"],                       // selectedTopics (UUIDs)
  "i": { "abortion": true },                   // invertedSpokes
  "w": { "abortion": "free-form write-in" },   // optional write-ins
  "v": { "quote-id": "agreed" }                // optional read-rank verdicts
}
```

## postMessage protocol

| Inbound message                              | Outbound reply                                   |
|----------------------------------------------|--------------------------------------------------|
| `{ type: 'compass-store:get', requestId }`    | `{ type: 'compass-store:value', value, requestId }` |
| `{ type: 'compass-store:set', value, requestId }` | `{ type: 'compass-store:ack', ok, requestId }` |
| `{ type: 'compass-store:subscribe', requestId }` | `{ type: 'compass-store:value', value, requestId }` then `update`s |
| `{ type: 'compass-store:clear', requestId }`  | `{ type: 'compass-store:ack', ok, requestId }`   |

Live push (no requestId): `{ type: 'compass-store:update', value }` — fires whenever any tab on any allowed origin writes a new value.

On iframe load the broker also posts `{ type: 'compass-store:ready' }` to its parent so embedders know they can start sending requests.

## Origin allowlist

- `https://*.empowered.vote` (any subdomain, including the apex)
- `http://localhost:*` and `http://127.0.0.1:*` (any port for local dev)

Anything else is silently ignored.

## Deployment

Deploy as a Render Static Site:

1. Create a new static site at `compass-store.empowered.vote` (or whichever subdomain you choose — the client lib defaults to `https://compass-store.empowered.vote/`).
2. Build command: empty
3. Publish directory: `.`
4. The `render.yaml` in this repo sets the `frame-ancestors` CSP and `X-Frame-Options: ALLOWALL` so other EV subdomains can iframe it.

That's it — there's no build step. Once deployed, every EV app importing `compassStore` from `@empoweredvote/ev-ui` automatically picks it up.
