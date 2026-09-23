# Meros Event Envelope spec

The product-agnostic event contract every Meros product emits to a collector. This
repo is the single source of truth for that contract:

- **[`SPEC.md`](SPEC.md)** — the human-readable specification.
- **[`event-envelope-1.json`](event-envelope-1.json)** — the machine authority (JSON
  Schema draft 2020-12). Where the two disagree, the schema wins.
- **[`examples/`](examples)** — conformance fixtures.

Published so any product — including open-source ones — can conform and self-check.
The reference emitter that implements it is `github.com/meros-co/meros-event`.

## Layout

```
SPEC.md                          the human specification
event-envelope-1.json            the schema (authority)
examples/valid/*.json            events that MUST validate (6)
examples/invalid/*.json          events that MUST NOT validate (4)
examples/invalid/manifest.json   filename -> the single rule each invalid case breaks
```

## What the schema enforces

- The fixed set of envelope keys (`additionalProperties: false`, including inside
  `source`/`subject`/`actor` — product data goes in `attrs`, never as new keys),
  required fields (top-level and `source.{product,version,instance}`), the `severity`
  enum (incl. `critical`), the ULID shape of `id`, the lowercase dotted
  `product.subject.verb` shape of `type`, UTC (`Z`) RFC3339 `occurred_at`, and the
  `subject`/`actor` sub-shapes when present.
- `seq`, `subject`, `actor`, `trace`, `source.site`, `source.edition` are optional.
- Unknown `type` and `source.product` **values** are accepted (forward-compatible);
  only their *shape* is fixed.

It does **not** enforce size limits (`attrs` ≤ 16 KiB, total ≤ 64 KiB) or dedupe —
those are collector responsibilities, not expressible in JSON Schema.

## Install (PHP)

Published as a Composer package so a PHP consumer gets the schema in `vendor/`:

```jsonc
"repositories": [{ "type": "vcs", "url": "https://github.com/meros-co/meros-event-spec.git" }],
"require": { "meros/event-spec": "^0.1" }
```

Then read `vendor/meros/event-spec/event-envelope-1.json`. (Public repo — no auth.)
For other languages, just fetch `event-envelope-1.json` directly.

## Conformance

Validate your emitted events against `event-envelope-1.json` with any JSON Schema
2020-12 validator. `examples/valid/` must all pass; `examples/invalid/` must all fail
(each pinned to one broken rule in `examples/invalid/manifest.json`) — a good check
that your validator is wired up correctly.

## License

MIT — see [LICENSE](LICENSE).
