# Phew

Dependency bootstrap for a Raspberry Pi agent runtime, using Rust 1.98.1 and
edition 2024. The executable starts a multithreaded Tokio runtime, initializes
tracing, logs a startup message, and exits. Agent execution, hibernation, HTTP
routes, Gemini calls, metrics recording, and transport adapters are future work.

`crates/phew-core` is a local crate reserved for our Agent, Model, Tool, Store,
and Transport contracts. It deliberately has no adapter dependencies or trait
interfaces yet.

| Purpose | Dependency |
| --- | --- |
| Multithreaded scheduler | Tokio 1.54 |
| Hot state | Moka 0.12, async cache support |
| Warm state | zstd 0.14 |
| Cold state | redb 4.4 |
| State encoding | postcard 1.1 and Serde |
| Optional dataflow | hydro_lang 0.17.0-alpha.6 |
| Optional Kafka adapter dependency | rskafka 0.6 |
| HTTP API and Gemini REST | Axum 0.8, reqwest 0.13 with JSON and Rustls |
| Observability | tracing, tracing-subscriber, metrics, metrics-exporter-prometheus |
| Property tests and benchmarks | proptest, Criterion |

Requested release lines are constrained in `Cargo.toml`; `Cargo.lock` records the
resolved versions. Hydro is pinned exactly because it is a prerelease. Its
[toolchain](https://github.com/hydro-project/hydro/blob/main/rust-toolchain.toml)
also pins Rust 1.98.1.

```sh
cargo fetch --locked
cargo check --workspace --all-targets --locked
cargo run --locked
```

Kafka support is enabled by default. `--no-default-features` omits rskafka.
`--features hydro` enables Hydro's build and embedded runtime support;
`--features hydro-sim` additionally enables deterministic simulator tooling.
These features install the tooling; dataflows and simulator tests remain to be
written. Validate optional dependencies with:

```sh
cargo check --workspace --all-targets --all-features --locked
```

The Prometheus exporter uses its recorder API so a future Axum `/metrics` route
can expose it without a separate exporter server or push gateway. rskafka's
optional compression codecs are disabled initially. zstd still compiles native C
code, and reqwest's Rustls provider may compile native crypto code; this stack
does not require librdkafka or its C++ toolchain.

Local Kafka uses the official
[Apache Kafka 4.4.0 image](https://kafka.apache.org/44/getting-started/docker/)
as a single KRaft broker/controller with a persistent Docker volume:

```sh
docker compose up -d --wait kafka
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
docker compose down
```

Host applications connect to `localhost:9092`; containers on the Compose network
connect to `kafka:19092`. Kafka is bound to the host's loopback interface for local
development. `docker compose down` preserves broker data.

For a 64-bit Raspberry Pi OS, the Rust target is `aarch64-unknown-linux-gnu`.
Building on the Pi uses its native toolchain; cross-compiling from macOS also
requires a Linux ARM64 linker and C compiler for native dependencies.
