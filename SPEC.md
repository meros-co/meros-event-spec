# Meros Event Envelope — Specification (v1)

The product-agnostic contract every Meros product emits to *a collector* (a local
Imperio, the Meros cloud, both, or neither). This document is the human-readable
spec; the machine authority is [`event-envelope-1.json`](event-envelope-1.json)
(JSON Schema draft 2020-12). Where they disagree, the schema wins.

**A product configured with zero collectors behaves exactly as it does with none —
no network attempts, no degradation.** Observability is opt-in and never
load-bearing.

## The envelope

One event is one flat JSON object. The local log is newline-delimited JSON (one
event per line, UTF-8).

```json
{
  "envelope": 1,
  "id": "01J9Z3K7QPX2ABCDEFGHJKMNPQ",
  "occurred_at": "2026-09-06T10:47:03.142Z",
  "seq": 10427,
  "source": { "product": "sluice", "version": "0.6.0", "instance": "e7c1a9f2", "site": "auditorium", "edition": "desktop" },
  "type": "sluice.route.confirmed",
  "severity": "info",
  "subject": { "kind": "route", "id": "snd-3f2a>rcv-91c0", "name": "Pulpit Mic → Lobby Amp" },
  "actor": { "kind": "user", "id": "u_4412", "name": "David" },
  "attrs": { "state": "confirmed", "transport": "nmos-is05" },
  "trace": "recall-88"
}
```

| Field | Req | Meaning |
|-------|-----|---------|
| `envelope` | ✔ | Format version. `1` for this spec. Additive fields never bump it; only a breaking change does. |
| `id` | ✔ | ULID, emitter-generated. Idempotency key — a collector dedupes on `(source.instance, id)`. |
| `occurred_at` | ✔ | RFC3339 UTC, millisecond precision, emitter clock. When it happened. Ends in `Z`. |
| `source.product` | ✔ | Lowercase product slug; the namespace owner for `type`. |
| `source.version` | ✔ | Emitter software version — correlates behaviour with a release. |
| `source.instance` | ✔ | Stable per-install id, generated locally on first run. Never a MAC address. |
| `source.site` |  | Optional local operator label. |
| `source.edition` |  | Optional `desktop` / `server` / `appliance`. |
| `type` | ✔ | `product.subject.verb`, lowercase, past tense. The product owns its namespace. |
| `severity` | ✔ | `debug` \| `info` \| `notice` \| `warning` \| `error` \| `critical`. |
| `seq` |  | Optional monotonic per `(product, instance)`, from 1, persisted — lets a collector detect gaps. Omit if you can't persist a counter. |
| `subject` |  | Optional `{ kind, id, name? }` — what the event is about. |
| `actor` |  | Optional `{ kind, id?, name? }` — who/what caused it. `kind` ∈ `user` \| `system` \| `device` \| `external`. Absent ≠ `system`. |
| `attrs` |  | Product-specific object, opaque to the collector. ≤ 16 KiB. |
| `trace` |  | Optional correlation id tying a caused chain together. |

Total serialized event ≤ 64 KiB. Product data lives **only** inside `attrs`, never as
new top-level keys (`additionalProperties: false`). A collector adds its own receipt
metadata (e.g. `received_at`, `clock_suspect`) in its own store — never inside the
stored envelope, which is kept verbatim.

## `type` namespace

`product.subject.verb`, lowercase, dot-separated, past tense. Each product owns its
namespace (`source.product` is the owner), so there is no central authority and no
cross-product collisions. Unknown `type` **values** and unknown `source.product`
**values** are accepted (forward-compatible); only their *shape* is fixed.

## Transports

An emitter implements **at least one**:

- **HTTP** — `POST {collector}/v1/events`, body one event or a JSON array (batch
  ≤ 500). Bearer-token auth. Response `202 {"accepted":n,"duplicates":m,…}`.
  Idempotent on `(source.instance, id)`.
- **MQTT** — publish to `meros/events/{product}/{instance}`, QoS 1, one event per
  message. The collector subscribes `meros/events/#`.

## Clock discipline

Emitters stamp `occurred_at` from their own clock. A collector records receipt time
and, if `occurred_at` skews beyond a configured bound (default 30 s), flags the event
in its own store but still stores it. Correlation across sources is guaranteed only
to ±(NTP jitter + transport latency); the target is human-scale (~10 ms), not
sample-accurate.

## Conformance

Validate emitted events against [`event-envelope-1.json`](event-envelope-1.json).
[`examples/valid/`](examples/valid) must all validate; [`examples/invalid/`](examples/invalid)
must all be rejected (each pinned to the one rule it breaks in
[`examples/invalid/manifest.json`](examples/invalid/manifest.json)).

## Reference emitters

- **Go** — `github.com/meros-co/meros-event` (public, MIT).
- **PHP** — `meros/event` (private).

## Versioning

`envelope` is `1`. Additive changes (new optional fields) do not bump it; a breaking
change ships a new schema (`event-envelope-2.json`) and bumps `envelope`.
