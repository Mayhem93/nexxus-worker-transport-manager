# nexxus-worker-transport-manager

> Executable Transport Manager worker for a Nexxus deployment — the process that figures out **who** should receive a change event and routes it to the right transport-specific queue.

[![License: MPL 2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)](https://opensource.org/licenses/MPL-2.0)
[![Node.js](https://img.shields.io/badge/node-%3E%3D24.0.0-brightgreen.svg)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-6.0.0-blue.svg)](https://www.typescriptlang.org/)

---

## What this is

`nexxus-worker-transport-manager` is the **runnable Transport Manager worker process** for a Nexxus deployment. It's a thin bootstrap around [`@mayhem93/nexxus-worker-lib`](https://www.npmjs.com/package/@mayhem93/nexxus-worker-lib): reads a config file, resolves the pluggable services (logger, DB adapter, MQ adapter, Redis), and starts consuming the `transport-manager` queue.

Same config shape, same pluggable-service pattern, same lifecycle as the API and the Writer — this is just the process at a different point in the pipeline.

---

## What it does

The Transport Manager sits between the Writer and the transport workers. Its job is subscription resolution and per-transport routing:

1. **Consume** the `transport-manager` queue — every event the Writer emits (`model_created` / `model_updated` / `model_deleted`) lands here.
2. **Resolve subscriptions** — query Redis to find every device subscribed to a channel that matches this change. Both unfiltered subscriptions (all `task` updates) and filtered ones (only `priority=high` tasks) are considered; filtered subs get their `FilterQuery` evaluated against the actual changed data.
3. **Group by transport** — subscribers are keyed as `deviceId|transport`, so this step buckets them into per-transport groups (WebSocket today; MQTT/SSE/etc. in the future).
4. **Publish per-transport payloads** — one message per transport pool, onto that transport's queue (e.g. `websockets-transport`), carrying the device IDs and channel keys the downstream worker needs to actually deliver the notification.

End-to-end:

```
nexxus-worker-writer  →  DB
         ↓
[transport-manager queue]  →  nexxus-worker-transport-manager
                                        ↓
                                        Redis lookup + filter evaluation
                                        ↓ (grouped per transport)
                              [websockets-transport queue]  →  ws transport worker  →  connected clients
```

---

## Why a Transport Manager instead of the Writer routing directly

The Writer could theoretically look up subscribers and publish per-transport payloads itself. Two reasons that's a bad idea:

- **Different scaling profile.** Writes are one thing; subscription fan-out is another. A single write can produce thousands of notifications; that fan-out step should scale independently of write throughput.
- **CPU cost of filter evaluation.** Filtered subscriptions require running the FilterQuery DSL against the actual changed data for every candidate filter on the affected model. Mixing that workload with DB writes means slow filter evaluations back up write persistence.
- **Isolation.** A bug or slowdown in subscription resolution shouldn't stop writes from landing. Separate workers = separate failure domains.
- **Horizontal scale.** Multiple `nexxus-worker-transport-manager` instances consume from the same queue; you scale routing independently of write throughput or transport throughput.

---

## What goes through the Transport Manager

**Every event the Writer emits.** Concretely: `model_created`, `model_updated`, `model_deleted` payloads for app models, each carrying whatever data downstream steps need for filter evaluation and delivery.

- `model_created` — Writer already persisted. Transport Manager finds subscribers (unfiltered subs, plus any filter that matches the new document) and enqueues per-transport notifications.
- `model_updated` — Writer already applied the patch and returned a partial post-update state. Transport Manager evaluates every registered filter against that partial state, and notifies the matching subscribers.
- `model_deleted` — Writer already deleted. Transport Manager finds who was subscribed to that model/id and notifies them. No filter evaluation (nothing left to filter on).

## What does NOT go through the Transport Manager

- **Anything the Writer doesn't emit an event for** — built-in model writes (Application, User, admin), authentication, reads. None of it touches this worker.
- **Subscription registration / unregistration** — devices subscribing to channels write directly to Redis via the API. This worker only *reads* subscriptions to find recipients.
- **The actual push to a client** — that's the transport worker's job (WebSocket today; MQTT / SSE / others when they land). The Transport Manager just says "these devices, via this transport, need to hear about this change".
- **Any DB writes** — the Writer already persisted before this worker sees the event. Transport Manager reads Redis and reads app schemas from the DB at startup; it never writes.

Rule of thumb: **if it's a change event that needs to reach subscribers, it comes through here. Delivery itself does not.**

---

## Configuration

Same shape as the Writer's config:

```json
{
  "database": { "host": "localhost", "port": 9200 },
  "message_queue": { "host": "localhost", "port": 5672, "user": "guest", "password": "guest" },
  "redis": { "host": "localhost", "port": 6379, "cluster": false, "password": "1234test" },
  "app": {
    "name": "my-transport-manager",
    "logger": "WinstonNexxusLogger",
    "database": "NexxusElasticsearchDb",
    "message_queue": "NexxusRabbitMq",
    "management": { "port": 5002, "token": "replace-me" },
    "hub": { "endpoint": "http://hub:8080", "token": "replace-me" }
  },
  "logger": { "level": "info", "logType": "json", "transports": [ { "type": "stdout" } ] }
}
```

The DB adapter is configured because the Transport Manager needs each app's model schema at startup (to evaluate FilterQueries against the correct field types). It does not write to the DB during normal operation.

Config file lookup order (first hit wins):

1. Explicit path passed to `NexxusConfigManager`
2. `NXX_CONF_PATH` environment variable
3. `/etc/nexxus/nexxus.conf.json` (the default)

---

## Running

**Prerequisites:**

- Node.js ≥ 24
- Same infrastructure the Writer talks to: DB (for schema loading), MQ, Redis
- The `transport-manager` queue plus each configured downstream transport queue (e.g. `websockets-transport`) existing on the broker

**Build and start:**

```bash
npm install
npm run build
npm start
```

`npm start` runs `node --enable-source-maps dist/index.js`. Multiple instances against the same `transport-manager` queue are safe — the broker load-balances between consumers.

---

## Status

🚧 **Pre-alpha.** Queue payloads and downstream transport names may still shift alongside the underlying library.

---

## Related

- [`nexxus-lib`](https://github.com/Mayhem93/nexxus-lib) — the umbrella framework: config manager, base service, pluggable-service resolvers, worker framework
- [`@mayhem93/nexxus-worker-lib`](https://www.npmjs.com/package/@mayhem93/nexxus-worker-lib) — the actual worker framework this repo bootstraps
- [`@mayhem93/nexxus-core-lib`](https://www.npmjs.com/package/@mayhem93/nexxus-core-lib) — shared types, config manager, logger, Hub client

---

## License

MPL-2.0
