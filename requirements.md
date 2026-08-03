# Requirements

**Status:** Draft · **Version:** 0.6

## 1. Scope

A device samples environmental sensor data, buffers it locally against connectivity
loss, and publishes it to a backend that stores the data and exposes it through a query
interface and a dashboard.

In scope: device application, ingest service, storage, query interface, dashboard,
local deployment.

Out of scope: see section 7.

Solution design, component structure and technology selection are documented separately
in `architecture.md`.

## 2. Conventions

Requirements are stated using the following keywords:

| Keyword | Meaning |
|---|---|
| **shall** | Mandatory. Subject to verification. |
| **should** | Recommended. Deviation requires justification. |
| **may** | Optional. Implementation is free to choose. |
| **shall not** | Prohibited. |

Each requirement contains exactly one **shall**. Text marked *Rationale* is explanatory
and not binding.

## 3. System overview

```
┌─────────────┐           ┌───────────┐         ┌─────────────┐
│   Device    │──────────▶│  Message  │────────▶│   Ingest    │
│             │           │  broker   │         │   service   │
│ Sensor      │           └───────────┘         └──────┬──────┘
│ Ring buffer │                                        │
│ Transport   │                                        ▼
└─────────────┘                                 ┌─────────────┐
                                                │  Time-series│
                                                │   storage   │
                                                └──────┬──────┘
                                                       │
                                         ┌─────────────┴────────────┐
                                         ▼                          ▼
                                   ┌──────────┐              ┌───────────┐
                                   │  Query   │              │ Dashboard │
                                   │interface │              │           │
                                   └──────────┘              └───────────┘
```

| Component | Responsibility |
|---|---|
| Device | Sampling, local buffering, publishing |
| Message broker | Decoupling of device and backend |
| Ingest service | Validation, deduplication, persistence |
| Time-series storage | Retention of sensor records |
| Query interface | Retrieval by device and time range |
| Dashboard | Visualization of readings and operational metrics |

The component structure above describes responsibilities, not an implementation.
Technology selection is documented in `architecture.md`.

## 4. Device requirements

### D-1 Sampling

The device **shall** sample temperature and relative humidity at a configurable rate.

### D-2 Default sample rate

The default sample rate **shall** be 1 Hz.

### D-3 Record content

Each record **shall** contain the following fields:

| Field | Type | Description |
|---|---|---|
| `seq` | uint32 | Sequence number within the session, monotonically increasing |
| `t_mono_ms` | uint64 | Monotonic time of acquisition, measured since device start |
| `temperature_c` | float | Temperature reading |
| `humidity_pct` | float | Relative humidity reading |

*Rationale: The sequence number enables gap detection and backend deduplication.
`t_mono_ms` is only interpretable within a single session and relative to the send
time reported per D-4.*

### D-4 Message content

Each published message **shall** contain the following fields and one or more records
as defined in D-3:

| Field | Type | Description |
|---|---|---|
| `device_id` | string | Unique device identifier |
| `session_id` | uint32 | Identifier of the current device session, generated at start |
| `t_sent_mono_ms` | uint64 | Monotonic time at which the message was handed to the transport |
| `dropped_total` | uint32 | Cumulative count of records discarded per D-9 |

*Rationale: These fields are constant across all records of a message. Carrying them
once per message rather than per record avoids redundancy during replay, where many
buffered records are transmitted together.*

*`t_sent_mono_ms` allows the backend to determine the age of each record as
`t_sent_mono_ms - t_mono_ms`, which is what makes correct timestamps possible for
records replayed after an outage.*

### D-5 Session identifier

The device **shall** generate a new session identifier on every start, with a
negligible probability of collision with its own preceding sessions.

*Rationale: The sequence number restarts on device restart. Without a session
identifier, the backend would discard records of a new session as duplicates of the
preceding one. Persisting the sequence number instead would require flash storage,
which is out of scope. The trade-off is that duplicates spanning a restart are no
longer detectable; buffered records are lost on restart in any case.*

### D-6 Local buffering

The device **shall** hold sampled records in a ring buffer until they are successfully
published.

### D-7 Buffer capacity

The buffer capacity **shall** be configurable, with a default of 1024 records.

*Rationale: 1024 records at 1 Hz covers approximately 17 minutes of outage, which spans
typical transient network failures such as reconnect, gateway restart or a brief site
outage. The memory footprint of approximately 16 KB is negligible on a microcontroller
with hundreds of kilobytes of RAM but not on one with single-digit kilobytes, hence
configurability. Outages beyond this window require flash-backed persistence, which is
out of scope.*

### D-8 Buffer capacity constraint

The buffer capacity **shall** be a power of two, enforced at compile time.

*Rationale: Allows bitmask indexing instead of modulo on wrap-around.*

### D-9 Overflow behaviour

When the buffer is full, the device **shall** discard the oldest record.

*Rationale: For telemetry, recent samples carry more value than historical ones.*

### D-10 Loss accounting

The device **shall** report the cumulative number of discarded records in the
`dropped_total` field of every published message.

*Rationale: Makes data loss observable rather than silent. A cumulative counter
tolerates lost records, whereas a per-record delta would not.*

### D-11 Buffer release

The device **shall** remove a record from the buffer only after it has been
successfully published.

### D-12 Behaviour during connectivity loss

The device **shall** continue sampling and buffering while no connection to the broker
is available.

### D-13 Replay after reconnect

On reconnect, the device **shall** publish buffered records in sampling order.

### D-14 Delivery guarantee

The device **shall** provide at-least-once delivery.

*Rationale: Exactly-once would require device-side persistence and a transactional
protocol; see section 7.*

### D-15 Timestamps

The device **shall** report monotonic time since start rather than wall-clock time,
both for record acquisition and for message transmission.

*Rationale: The device has no real-time clock and no reliable wall-clock reference at
startup. Reporting both timestamps on the same monotonic scale allows the backend to
derive record age without any assumption about clock synchronisation.*

### D-16 Memory allocation

The device application **shall not** perform dynamic memory allocation after
initialization.

## 5. Backend requirements

### B-1 Record intake

The ingest service **shall** consume records published by devices.

### B-2 Validation

The ingest service **shall** reject records that are missing required fields or contain
values outside their defined range.

### B-3 Rejection logging

The ingest service **shall** log rejected records with the reason for rejection.

### B-4 Deduplication

The ingest service **shall** discard records whose `(device_id, session_id, seq)`
combination has already been persisted.

### B-5 Timestamp derivation

The ingest service **shall** derive an absolute timestamp for each record as the time
of receipt minus the difference between the message send time and the record
acquisition time.

*Rationale: `t_sent_mono_ms - t_mono_ms` yields the age of the record at the moment of
transmission. The residual error is the network transit time, independent of how long
the record was buffered.*

### B-6 Persistence

The ingest service **shall** write valid records to time-series storage.

### B-7 Batched writes

The ingest service **should** write records in batches rather than individually.

*Rationale: Limits write load on the storage backend.*

### B-8 Query interface

The system **shall** expose an interface that returns records filtered by device
identifier and time range.

### B-9 Operational metrics

The ingest service **shall** expose the following metrics: records received, records
rejected, records deduplicated, write latency, count of active devices, and count of
active alerts.

### B-10 Dashboard

The system **shall** provide a dashboard displaying sensor readings per device and the
operational metrics defined in B-9.

### B-11 Threshold evaluation

The ingest service **shall** evaluate each valid record against configurable upper and
lower thresholds for temperature and humidity.

### B-12 Alert activation

The ingest service **shall** raise an alert for a device when a reading crosses a
configured threshold.

### B-13 Alert deactivation

The ingest service **shall** clear an active alert only after the reading has returned
past the threshold by a configurable hysteresis margin.

*Rationale: Clearing at the threshold itself causes repeated activation and
deactivation when readings fluctuate around the limit.*

### B-14 Alert exposure

The system **shall** expose active alerts through the query interface and the
dashboard.

### B-15 Alert state on restart

The ingest service **shall** reconstruct alert state from stored records after a
restart.

*Rationale: Alert state held only in memory would silently reset, leaving an ongoing
threshold violation unreported.*

## 6. Non-functional requirements

### NF-1 Host testability

Device logic **shall** be testable on a development host without sensor hardware and
without a running broker.

*Rationale: Sensor, transport and clock are accessed through interfaces, allowing
substitution in tests.*

### NF-2 Deterministic timing in tests

Time-dependent tests **shall** run against a simulated clock.

*Rationale: Tests must not depend on wall-clock delays.*

### NF-3 Ring buffer independence

The ring buffer **shall** be usable independently of the remaining device code.

### NF-4 Ring buffer test coverage

Ring buffer tests **shall** cover empty and full states, wrap-around, and overflow
behaviour.

### NF-5 Continuous integration

CI **shall** build and test both device and backend code on every push to the
repository.

### NF-6 Runtime analysis

CI **shall** include a build instrumented with address and undefined-behaviour
sanitizers.

### NF-7 Local deployment

The complete system **shall** be startable on a development host with a single command.

### NF-8 Device instance count

The local deployment **shall** support a configurable number of simulated device
instances.

### NF-9 Failure injection

The simulated device **shall** support reproducible injection of connection loss,
elevated latency, burst traffic, and clock jumps.

### NF-10 End-to-end latency

Under normal operating conditions, a sampled record **should** be visible in the
dashboard within two sampling intervals of its acquisition.

*Rationale: Expressed relative to the sample rate rather than as a fixed duration, so
the target scales with configuration.*

## 7. Out of scope

| Item | Rationale |
|---|---|
| Authentication and transport encryption | Production deployment would require per-device mutual authentication. Deferred to keep focus on the data path. |
| Flash-backed buffering | Extends outage tolerance beyond minutes but requires wear levelling and post-reset consistency handling. |
| Over-the-air updates | Separate subsystem with rollback and signature requirements. |
| Exactly-once delivery | Requires device-side persistence and a transactional protocol. At-least-once with deduplication is the conventional trade-off for telemetry. |
| Actuation and device control | The system is telemetry-only. A downlink path would require command acknowledgement, idempotency handling and per-device authentication, none of which are in scope. Threshold monitoring per B-11 to B-15 reports conditions but does not act on them. |
| Multi-tenancy | No functional requirement within the current scope. |
| Custom frontend | The dashboard requirement is met by an off-the-shelf tool. |
| Multi-producer ring buffer | The buffer is single-producer, single-consumer by design. Multiple producers would require compare-and-swap or locking. |
| Physical sensor hardware in phases 1 to 4 | A simulated device allows reproducible failure injection and parallel instances, neither of which is practical with a single board. |

## 8. Milestones

| Phase | Milestone | Acceptance |
|---|---|---|
| 1 | Ring buffer complete | NF-4 satisfied, CI green, sanitizer build clean |
| 2 | Device logic complete | Test demonstrates D-12 and D-13 without a running broker |
| 3 | End-to-end path closed | A simulated record reaches time-series storage |
| 4 | Observability | B-9 and B-10 satisfied; multiple device instances running per NF-8 |
| 5 | Physical hardware | Sensor readings from a physical device, with application logic unchanged |

## 9. Open questions

- Retention period for stored records
- Clock drift over long sessions: the monotonic clock accumulates oscillator error,
  which becomes relevant for records buffered over extended outages
- Source of entropy for the session identifier on a device without persistent state
- Default threshold values and hysteresis margin for B-11 and B-13
- Whether an alert requires a single reading or sustained violation over several
  samples to activate
- Expected value ranges for B-2 validation
- Behaviour when a device is offline for longer than the buffer window: is silent loss
  acceptable beyond the D-10 counter?