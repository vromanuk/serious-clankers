# Skeptic Reliability

## Intent

Review whether a change contains failure and recovers from it: failure modes and redundancy, calls to dependencies (timeouts, retries, idempotency, circuit breakers), protection from load (shedding, quotas, backlogs, constant work), isolation (control vs data plane, static stability, sharding, cells), caches and consumers, rollback and rollout safety, health checks, and recovery. Runs standalone, or inside skeptic stage 2 (architecture) as a `### Reliability` subsection when the diff touches failure-relevant code.

## Triggers

- **SHOULD** apply when the user asks for a reliability or resilience review, or when skeptic stage 2 runs and the diff touches remote calls, retries, consumers or producers, queues, caches, configuration, deployment or health checks, startup dependencies, shared tenant resources, or state that must survive a crash.
- **SHOULD NOT** apply to diffs with none of these (report `reliability: none` with a reason).

## Behaviors

### Behavior: Failure walk

The agent SHALL state, for each new or changed component, dependency, and remote call, what happens when it is slow, down, or returns garbage, and SHALL flag single points of failure and unstated failure behavior, weighted by likelihood and impact.

#### Scenario: New hard dependency on the request path

- **GIVEN** a handler that now calls a metadata service synchronously on every request, with no cached copy
- **WHEN** reviewing reliability
- **THEN** the agent flags the new hard dependency, states that the handler fails whenever the metadata service does, and offers serving from a locally held last-known-good copy

### Behavior: Unknown outcomes and idempotency

The agent SHALL flag code that treats a timeout as "not applied" for a non-idempotent operation, and side-effecting calls retried or redelivered without an idempotency key recorded atomically with the effect.

#### Scenario: Random object name inside a retry loop

- **GIVEN** an upload retried on timeout that generates a new random object name on each attempt
- **WHEN** reviewing reliability
- **THEN** the agent flags that a retry after an unknown outcome can store the data twice, and suggests choosing the name once per unit of work so a retry overwrites its own object

### Behavior: Retry amplification

The agent SHALL flag retries at more than one layer of a call chain, and hot-path retries bounded only per request with no shared retry budget.

#### Scenario: Application retries around a retrying SDK

- **GIVEN** an application retry loop of three attempts around an SDK client that also retries three times
- **WHEN** reviewing reliability
- **THEN** the agent flags multiplied retries (up to nine attempts per call), and suggests retrying at one layer, with a success-funded retry budget if the call is on a hot path

### Behavior: Load protection

The agent SHALL flag services with no admission limit, reject paths that do expensive work, shedding of health checks, global-only limits in multi-tenant services, and consumers with no backlog plan.

#### Scenario: Consumer sized exactly to steady state

- **GIVEN** a consumer scaled to the average arrival rate, calling a downstream store with no rate limit
- **WHEN** reviewing reliability
- **THEN** the agent flags that recovery after an outage needs extra capacity for catch-up and that catch-up will hit the store with a burst, and asks for a time-to-drain estimate, a message-age metric, and a cap on the store's request rate

### Behavior: Static stability and isolation

The agent SHALL flag data planes that stop, refuse work, or cannot start when the control plane is unavailable, and designs where one tenant or one poison input can reach every worker.

#### Scenario: Config refresh failure crashes the service

- **GIVEN** a service that exits when its periodic config fetch fails
- **WHEN** reviewing reliability
- **THEN** the agent flags that a control-plane outage becomes a data-plane outage, and suggests keeping the last valid config (also on disk) and recording its age

### Behavior: Consumer correctness

The agent SHALL flag offsets committed or messages acknowledged before side effects are durable, messages dropped after failures instead of dead-lettered, unbounded redelivery of one message, and processing that can outlast its lease.

#### Scenario: Log and skip on a decode failure

- **GIVEN** a consumer that logs a decode error and commits past the record
- **WHEN** reviewing reliability
- **THEN** the agent flags data loss, and suggests capping attempts and moving the record to a dead-letter channel with an alarm and a way to reprocess it

### Behavior: Change safety

The agent SHALL flag data-format changes shipped in one step, readers that reject unknown fields, config changes outside the reviewed pipeline, all-at-once rollouts, and health checks that call shared dependencies.

#### Scenario: Writer and reader switch formats in one deploy

- **GIVEN** a change that starts compressing records written to a topic and adds decompression to the consumer in the same release
- **WHEN** reviewing reliability
- **THEN** the agent flags that old consumers and a rolled-back producer cannot read the new records, and suggests prepare (read both) then activate (write new), verifying every consumer first

#### Scenario: Readiness probe calls the database

- **GIVEN** a readiness probe that queries the shared database
- **WHEN** reviewing reliability
- **THEN** the agent flags that a database blip removes every pod at once, and suggests a local-only readiness check with dependency health reported separately

### Behavior: Proportionate findings

The agent SHALL NOT demand redundancy, cells, or multi-region for a low-likelihood, low-impact failure, and SHALL say why the risk is acceptable instead.

#### Scenario: Internal batch tool

- **GIVEN** a one-off internal batch tool that reads a file and writes a report, with no other dependents
- **WHEN** reviewing reliability
- **THEN** the agent does not ask for redundancy or cells and reports the failure behavior in one line

## Constraints

### Constraint: Stage boundary

The agent MUST NOT repeat hard-rule floors (`HR-io-timeout`, `HR-retry-backoff`), in-process async concerns (skeptic-async), alert and SLO design (observability), or security controls (skeptic-security); it cross-references them instead.

<!-- skillet-version: 1.7.0 -->
