# Ark guardrails

Local native PII, secret/DLP and prompt-injection detection, with optional
Wolf Small FP16. Construct `Ark::new(config)`, register its clone before
answering plugins, and register `ark.provider_guard()` after all provider
input modifiers. Both share one scanner and bounded admission capacity.

PII/DLP actions are `block` (default) or `redact` (`mask` is an alias); PI blocks. Model and scanner
errors refuse forwarding. Client replies are not scanned. Roles and text
field boundaries are preserved. Masking operates on merged original UTF-8
byte ranges and does not restore sensitive values in responses.

The `wolf` feature requires existing local FP16 assets and an explicit
probability threshold. An integrity manifest is optional; no asset migration
or new model revision is required. Pooling uses raw binary logits with training's
normalized logsumexp at temperature 0.1, followed by sigmoid.

Local development uses the sibling checkout `../patronus-ark-standalone/ark-core`
through a Cargo path dependency. No Core release is published by this work.
After an official Core release, replace the path dependency with that crates.io
version and regenerate the lockfile. Core is dual-licensed MIT OR Apache-2.0,
matching this gateway.

Optional `onnx-coreml`, `onnx-cuda`, `onnx-directml` and `onnx-tensorrt`
features forward to Core's ONNX Runtime execution providers. CPU is the
verified default; an explicitly selected provider must pass bundle warmup
and numerical checks. Binary feature names are `ark-coreml`, `ark-cuda`,
`ark-directml` and `ark-tensorrt`.

## Testing locally

`cargo test -p respan-gateway-plugin-ark --features wolf` runs native scanner,
HTTP boundary, masking, capacity and synthetic ONNX regression tests without
model downloads. The synthetic fixture checks integration mechanics; it does
not establish the detection quality of Wolf.

The separate real-model tests use the actual local FP16 bundle through Rust
ONNX Runtime, batch size 1, and `mockllm` as the HTTP provider:

```sh
ARK_WOLF_BUNDLE=/absolute/path/to/wolf-bundle \
  cargo test -p respan-gateway-plugin-ark --features wolf --test real_wolf \
  -- --ignored --nocapture --test-threads=1
```

They verify native PII/DLP findings and Wolf decisions independently, successful
OpenAI and Anthropic requests, original-byte masking, blocked requests never
reaching the provider, and a tool follow-up blocked at the provider edge.
PI-L1 is deliberately disabled in these tests so attack blocks prove that the
real Wolf path ran. The bundle must already exist; tests never download assets.

## Measuring locally

[Recorded CPU latency/RSS results and methodology](benchmarks/README.md) include
raw samples for baseline, native L1 and a real local Wolf session.


Run Core's release-mode `wolf_benchmark` example against the prepared local
bundle to measure startup, warm RSS and input latency on the target machine.
The synchronous runtime uses one warm scanner, one active scan and ONNX Runtime
batch size 1. Preparation uses the Rust `prepare_wolf_bundle` example; serving
and scanning never fetch model assets.

## Configuration and binary integration

The full setting table and a PII + DLP redaction example are in
[`docs/configuration.md`](../../../docs/configuration.md#local-ark-guardrails).
The crate follows the repository plugin guide: public core/kind APIs only,
workspace and optional binary dependencies, an optional `PluginsConfig` entry,
registration in `with_plugins`, and TOML composition tests. Observability runs
before the client guard; the provider guard follows input modifiers and precedes
pricing. No API key or secret is needed by this plugin.
