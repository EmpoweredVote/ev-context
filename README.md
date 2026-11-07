# ev-context

Cross-subdomain shared client state for Empowered Vote apps — compass answers, address, read-rank verdicts, anything the user owns that should follow them across subdomains.

## What it is

A single static HTML page that owns the canonical guest state in `localStorage` on its own origin (`https://context.empowered.vote/`). Every EV app (`compass.`, `essentials.`, `readrank.`, `treasurytracker.`, …) embeds it as a hidden iframe and reads/writes via `postMessage`. Browsers isolate `localStorage` per full origin, so without this broker each subdomain has its own siloed copy. With it, all subdomains see the same data — guests get a unified context that follows them across apps without ever signing in.

Live updates fan out across tabs via a `BroadcastChannel` inside the broker, so saving in one tab updates the radar / address / verdicts in every other open tab instantly.

## Wire format

The broker is intentionally schemaless — store anything serializable. By convention EV apps namespace their data under top-level keys:

```json
{
  "compass": {
    "a": { "abortion": 5, "guns": 3 },
    "s": ["uuid", "uuid"],
    "i": { "abortion": true },
    "w": { "abortion": "free-form write-in" }
  },
  "address": {
    "formatted": "100 W Kirkwood Ave, Bloomington, IN 47404",
    "lat": 39.1653,
    "lng": -86.5264
  },
  "verdicts": { "quote-id": "agreed" }
}
```

Each app reads/writes only its own slice and merges back the whole object. The broker doesn't enforce or interpret any schema.

## postMessage protocol

| Inbound message                                    | Outbound reply                                       |
|----------------------------------------------------|------------------------------------------------------|
| `{ type: 'ev-context:get', requestId }`            | `{ type: 'ev-context:value', value, requestId }`     |
| `{ type: 'ev-context:set', value, requestId }`     | `{ type: 'ev-context:ack', ok, requestId }`          |
| `{ type: 'ev-context:subscribe', requestId }`      | `{ type: 'ev-context:value', value, requestId }` then `update`s |
| `{ type: 'ev-context:clear', requestId }`          | `{ type: 'ev-context:ack', ok, requestId }`          |

Live push (no requestId): `{ type: 'ev-context:update', value }` — fires whenever any tab on any allowed origin writes a new value.

On iframe load the broker also posts `{ type: 'ev-context:ready' }` to its parent so embedders know they can start sending requests.

## Origin allowlist

- `https://*.empowered.vote` (any subdomain, including the apex)
- `http://localhost:*` and `http://127.0.0.1:*` (any port for local dev)

Anything else is silently ignored.

## Deployment

Deploy as a Render Static Site:

1. Create a new static site at `context.empowered.vote` (or whichever subdomain you choose — the client lib defaults to `https://context.empowered.vote/`).
2. Build command: empty
3. Publish directory: `.`
4. Response headers: none are set on the live site, so any page can iframe it. To limit embedding to EV subdomains, add a `Content-Security-Policy: frame-ancestors https://*.empowered.vote` header in `EmpoweredVote/ev-infra` (see below).

That's it — there's no build step. Once deployed, every EV app importing `evContext` from `@empoweredvote/ev-ui` automatically picks it up.

## Render settings live in ev-infra

This repo's Render services (`ev-context`) are managed by the Render Blueprint in
[`EmpoweredVote/ev-infra`](https://github.com/EmpoweredVote/ev-infra) (`render.yaml`).

- Build and start commands, rewrites, headers, custom domains, plans, cron
  schedules and build filters change **only through a PR to ev-infra**. Render
  syncs its `main` branch automatically.
- A setting changed in the Render dashboard is **undone on the next sync**.
- Env var **values** are the exception: set them in the Render dashboard. Add a
  new variable's name to ev-infra (with `sync: false`) so the record stays complete.
- Do not add a `render.yaml` to this repo. Render does not read it.
