# Example brief: a Kafka producer in kafka-common

A brief filled in from `brief-template.md`, for a Kafka producer in a Rust library crate. The design and decisions
are specific to this task; the rules come from the skeptic stages it points to.

````text
You are building the Kafka producer in kafka-common with the user. Read this whole brief before writing code.

## How the user works
- Top-down, by wishful thinking: the behavior they wish they had comes first, as calls to pieces that do not exist
  yet; each piece's inside comes later, one at a time. Never start from the bodies.
- The user must understand every piece as it is built: the high-level picture first, why each decision and not
  the other options, simple diagrams where they help.
- The user makes every decision: defaults, concurrency primitives, simple or fuller, skip or build. Propose
  options and wait; never pick silently.

## Context
- Repo and branch: the workspace's repo, on a feature branch for the producer.
  Crate: kafka-common, feature `producer`.
- Requirements: kafka-common/spec/PRODUCER_SPEC.md. Build against it; never change it to match the code.
- Code to imitate: the crate's consumer module (module doc, job struct), its consumer settings (settings docs), its
  error type (error docs), and an existing config type built with constructors plus `with_settings(.. ::default())`.
- Do not commit or push unless the user asks. No README or docs unless asked.

## Agreed design

Public (data types in src/types/, re-exported from lib.rs under `#[cfg(feature = "producer")]`):
- `BootstrapServers::parse(&str) -> Result<_, InvalidBootstrapServers>`: one or more host:port, no scheme.
- `ProducerConfig` (private fields): `plaintext(BootstrapServers)`, `tls(BootstrapServers, TLS)` (existing
  crate::config::TLS), `with_settings(ProducerSettings)`.
- `ProducerSettings` (pub fields, `Default`): delivery_semantics, max_batch_wait, max_batch_bytes, compression,
  buffer_memory_bytes, close_deadline, statistics_interval, metrics_namespace.
- `DeliverySemantics { AtMostOnce { acks }, AtLeastOnce { acks, delivery_timeout } }`,
  `DeliveryTimeout { Unlimited, After(Duration) }`, `Acks { Leader, AllReplicas }`, `Compression { None, Lz4, Zstd }`.
- `ProducerRecord<'a> { topic: &str, key: Option<&[u8]>, value: &[u8], partition: Option<i32>,
  headers: &[RecordHeader<'a>], timestamp_ms: Option<i64> }` with `get_size_bytes()`;
  `RecordHeader<'a> { key: &str, value: Option<&[u8]> }`; `RecordMetadata { partition: i32, offset: i64 }`;
  `type SendResult = Result<Vec<RecordMetadata>, SendError>`.
- `SendError { Rejected(KafkaError), NotDelivered(KafkaError) }` with `pub(crate) from_delivery_error`;
  `ProducerError { Kafka(KafkaError), CloseDeadlineExceeded { undelivered_records: usize } }`.
- `KafkaProducer`: `start(ProducerConfig) -> Result<Self, ProducerError>`,
  `async send(&self, &[ProducerRecord<'_>]) -> SendFuture`, `async close(self) -> Result<(), ProducerError>`.
  `SendFuture: Future<Output = SendResult>`.

Private (src/producer/):
- unacknowledged_sends.rs: `UnacknowledgedSends<C>` over `slab::Slab`; `UnacknowledgedSend { unreported_records,
  sent_at, state: SendState<C> }`; `SendState { AwaitingAcknowledgments { completion, record_metadata:
  Vec<Option<RecordMetadata>> }, Failed }`; `CompletedSend<C>`; `SendKey`; `SentRecordId(usize)` (high 32 bits send
  key, low 32 record index). Methods: insert, record_acknowledged, record_failed, find_oldest_sent_at,
  count_unacknowledged_records. A slot is freed only after every record of its send reported. A refused enqueue at
  record i calls record_failed for i onward. `SharedUnacknowledgedSends` holds it behind a std Mutex.
- buffer_memory.rs: `BufferMemory { reserve(bytes).await -> Duration, release(bytes), count_used_bytes() }` over a
  tokio Semaphore (acquire_many + forget, add_permits). A send larger than the limit is Rejected before reserve.
- client_config.rs: `build_client_config(&ProducerConfig) -> ClientConfig`.
- callbacks.rs: `ProducerCallbacks` (ProducerContext with DeliveryOpaque = usize, and ClientContext::stats).
- metrics.rs: `ProducerMetrics`: kafka_producer_records{outcome}, kafka_producer_send_duration_ms{outcome},
  kafka_producer_buffer_wait_seconds, and from statistics: buffered_records, buffered_bytes, oldest_send_seconds,
  batch_records, request_latency_ms. No topic or partition labels.

Concurrency:
- One `KafkaProducer` per process, shared as `Arc`; `send(&self)` from any task.
- rdkafka `ThreadedProducer`. librdkafka's own threads do batching, compression, and network I/O. No thread pool of
  our own.
- Per record, the poll thread only gives back bytes, updates UnacknowledgedSends, and completes a oneshot:
  no allocation, no blocking, no log except on Rejected.
- `close` runs flush(close_deadline), counts what is left, and drops the producer, all in spawn_blocking.

Defaults: AtLeastOnce { acks: Leader, delivery_timeout: Unlimited }; max_batch_wait 10 ms; max_batch_bytes
1,000,000; Lz4; buffer_memory_bytes 64 MiB; close_deadline 20 s; statistics_interval 10 s;
metrics_namespace "kafka-producer". RECORD_OVERHEAD_BYTES: measure, then decide with the user.

Allocations: none per record from our code; per send, one oneshot and the metadata Vec.

Tests decided: proptest for UnacknowledgedSends and the SentRecordId round trip; tables for error mapping, client
config, record size, BootstrapServers::parse; service tests on the existing broker and toxiproxy (send resolves in
input order; unknown topic Rejected; broker cut, send waits, recovery delivers all; close delivers buffered records;
close past the deadline gives CloseDeadlineExceeded; two tasks share one producer).

## Decision log
D1. Results through `SendFuture` only, not callbacks; keeps caller code off librdkafka's single poll thread.
D2. No `flush`; no caller needs it, it stalls every task sharing the producer, and it can wait forever.
D3. No idempotence; avoids librdkafka's limit of about 5 partitions per broker and the IDEMPOTENT_WRITE ACL.
D4. Acks default Leader, as originally agreed.
D5. max_batch_wait 10 ms (franz-go's default), not 50 ms.
D6. Borrowed `ProducerRecord`; librdkafka copies anyway, so owned records only add an allocation.
D7. `RecordMetadata` without the topic; the caller has it, results are in input order.
D8. `slab` for UnacknowledgedSends; out-of-order completion, memory bounded by open sends, small reusable keys.
D9. `ThreadedProducer`, not FutureProducer; no allocation or 100 ms retry sleep per record.
D10. One shared producer per process, as the spec requires.
D11. Names: SendFuture, UnacknowledgedSends, SentRecordId, BufferMemory, ProducerCallbacks, DeliverySemantics,
     max_batch_wait, max_batch_bytes, buffer_memory_bytes; ProducerConfig/ProducerSettings split.

## Stay in the loop
Stop at each stop point below. Never put two in one message, never continue because a step looks routine.
Stop points:
1. The public face: types and signatures, no bodies.
2. Every default and constant, including RECORD_OVERHEAD_BYTES.
3. UnacknowledgedSends and SendState: transitions, slot reuse, late reports.
4. SentRecordId: the packing and its limits.
5. Error mapping: every librdkafka error code to Rejected or NotDelivered.
6. Client config: every librdkafka key and value.
7. BufferMemory: reserve/forget/release, cancel safety, the oversize case.
8. ProducerCallbacks: exactly what runs on the poll thread per record.
9. KafkaProducer::send and close: the full flow and the shutdown order.
10. Metrics: each instrument and when it updates.
11. Any change outside src/producer and src/types (Cargo.toml, BUCK, lib.rs).
12. Anything outside the agreed design, any new dependency, any performance trade-off, any `unsafe`: ask first.

Each stop contains, in order: where we are (the design diagram with this piece marked); what it does; a diagram
when there is a flow, states, or ownership (ASCII, no Mermaid); why this shape and why not the other options
(point to the decision log); the code for this piece only (about 150 lines at most); how it works; risks and edge
cases; tests and what each proves; checks (format, lint, tests, and the skeptic stages run); open decisions as
lettered options with the recommendation first. End with "Continue?" and wait.

Decisions not in the decision log always stop: defaults, concurrency primitives, simple or fuller, skip or build,
public API, dependencies, `unsafe`, performance trade-offs, test strategy, files outside the change.

## Build order
types -> UnacknowledgedSends and SentRecordId -> error mapping and client config -> BufferMemory -> callbacks and
KafkaProducer -> metrics -> service tests.

## Writing with skeptic
Skeptic is the source of the rules. Before writing each piece, load the stages for it, follow them while writing,
then run their questions on the piece and fix what fails before showing it.
Stages live in ~/.agents/skills/<stage>/SKILL.md, with their references.
- Every piece: skeptic-naming, skeptic-comments, skeptic-hard-rules.
- Public face and types: skeptic-architecture (components.md, type-driven.md, rust-apis.md), design-components,
  skeptic-complexity, skeptic-solid.
- Settings and defaults: skeptic-comments (comments.md § Settings and numbers), skeptic-reliability.
- UnacknowledgedSends, SentRecordId, error mapping, client config: skeptic-testability (pure-core.md),
  skeptic-complexity, skeptic-conventions.
- BufferMemory, ProducerCallbacks, KafkaProducer: skeptic-architecture, skeptic-reliability, skeptic-async.
- Metrics: skeptic-observability.
- Tests: skeptic-unit-tests, skeptic-testability (testing.md).
Read the Scars list in skeptic/SKILL.md before the first piece.

## Checks after each piece
cargo fmt -p kafka-common; cargo clippy -p kafka-common --all-features -- -D warnings;
cargo test -p kafka-common --features producer. Report the results as they came out.
````
