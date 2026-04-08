# Exile API v3 Protocol

## Overview

v3 is a redesign of the Exile gate protocol that replaces the v2 architecture (three separate gRPC connections with independent lifecycles) with a single bidirectional stream and a set of focused domain services.

### v2 architecture (current)

```
Client                                          Server
  |                                                |
  |====== JobQueueStream (bidir gRPC) =============|  Stream 1: jobs + ACK
  |                                                |
  |====== EventStream (bidir gRPC) ================|  Stream 2: events + ACK
  |                                                |
  |------ SubmitJobResults (unary RPC) ----------->|  Connection 3: results
  |                                                |
  |------ 43 other RPCs on GateService ----------->|  All on one service
```

Problems: three connection state machines, no backpressure, no lease/deadline on jobs, fire-and-forget result submission, monolithic service with 43 RPCs, data model issues (parallel arrays, string enums, inconsistent types).

### v3 architecture (new)

```
Client                                          Server
  |                                                |
  |====== WorkStream (single bidir gRPC) ==========|  Everything: jobs, events,
  |  Register, Pull, Result, Ack, Nack,            |  results, acks, heartbeats
  |  ExtendLease, Heartbeat                        |
  |                                                |
  |------ AgentService (unary RPCs) -------------->|  Agent CRUD, skills, status
  |------ CallService (unary RPCs) --------------->|  Dial, transfer, hold, recording
  |------ RecordingService (unary RPCs) ---------->|  Search, download, labels
  |------ ScrubListService (unary RPCs) ---------->|  List, add, update, remove
  |------ ConfigService (unary RPCs) ------------->|  Config, certs, logging
```

## Package structure

```
tcnapi/exile/
  v3/
    types.proto         Shared entity types (Pool, Record, Agent, events, enums)
    worker.proto        WorkerService — the unified work stream
    agent.proto         AgentService — agent management + skills
    call.proto          CallService — dialing, transfers, hold, recording
    recording.proto     RecordingService — voice recording search/retrieval
    scrublist.proto     ScrubListService — content blocking lists
    config.proto        ConfigService — configuration, certs, logging
```

All v3 protos share a single package: `tcnapi.exile.gate.v3`. Types defined in `types.proto` are referenced directly (no cross-package imports).

The v2 protos (`core/v2/`, `gate/v2/`) remain unchanged for backward compatibility during migration.

---

## WorkStream protocol

`WorkerService.WorkStream` is the core of v3. It is a single bidirectional gRPC stream that replaces `JobQueueStream`, `EventStream`, and `SubmitJobResults`.

### Message types

**Client to server** (`WorkRequest`):

| Message | Purpose |
|---------|---------|
| `Register` | First message. Identifies the client, declares capabilities. |
| `Pull` | Request work. `max_items` controls how many items the server may send. |
| `Result` | Submit the output of a completed job. Supports chunked delivery. |
| `Ack` | Confirm processing of one or more events (batch). |
| `Nack` | Reject a work item for immediate redelivery to another client. |
| `ExtendLease` | Request more time on a work item before deadline. |
| `Heartbeat` | Liveness signal. |

**Server to client** (`WorkResponse`):

| Message | Purpose |
|---------|---------|
| `Registered` | Response to Register. Returns client_id, heartbeat interval, default lease, max inflight. |
| `WorkItem` | A unit of work (job or event) with a lease deadline. |
| `ResultAccepted` | Confirms the server persisted a result. Delivery guarantee. |
| `LeaseExtended` | Confirms a lease extension with new deadline. |
| `LeaseExpiring` | Warning that a lease is about to expire. |
| `NackAccepted` | Confirms the work item was returned to the queue. |
| `Heartbeat` | Server liveness signal. |
| `StreamError` | Non-fatal error (e.g., invalid work_id). |

### Connection lifecycle

#### 1. Registration

The client opens the stream and sends `Register` as the first message:

```
Client -> Register {
  client_name: "sati-finvi-prod-1"
  client_version: "2.32.1"
  capabilities: [WORK_TYPE_LIST_POOLS, WORK_TYPE_GET_POOL_RECORDS, ...]
}

Server -> Registered {
  client_id: "c-abc-123"
  heartbeat_interval: "30s"
  default_lease: "300s"
  max_inflight: 20
}
```

- `capabilities` filters which work types the server will send to this client. An empty list means "all types". This replaces the v2 pattern where every client received every job type regardless of whether it could handle it.
- `max_inflight` is the server-imposed upper bound on concurrent work items. The client's `Pull.max_items` is capped at this value.
- `heartbeat_interval` tells the client how often to send heartbeats. If the server doesn't receive a heartbeat or any other message within 2x this interval, it considers the client dead.

#### 2. Pulling work

After registration, the client requests work by sending `Pull`:

```
Client -> Pull { max_items: 5 }
```

The server tracks **capacity** per client:

```
capacity += pull.max_items     (on each Pull received)
capacity -= 1                  (on each WorkItem sent)
```

The server will never send more `WorkItem` messages than the client's remaining capacity. This is **credit-based flow control** — the client controls its own concurrency.

The server holds the Pull if no work is available (long-poll semantics). When work arrives, it sends `WorkItem` messages up to the client's capacity.

#### 3. Processing work items

Each `WorkItem` has a `category` that determines what the client must do:

```
Server -> WorkItem {
  work_id: "W-12345"
  deadline: "2026-04-07T20:05:00Z"    // 5 minutes from now
  category: WORK_CATEGORY_JOB
  attempt: 1
  task: { list_pools: {} }
}
```

**Jobs** (`WORK_CATEGORY_JOB`) require a `Result` response:

```
Client -> Result {
  work_id: "W-12345"
  final: true
  payload: { list_pools: { pools: [...] } }
}

Server -> ResultAccepted { work_id: "W-12345" }
```

**Events** (`WORK_CATEGORY_EVENT`) require an `Ack`:

```
Server -> WorkItem {
  work_id: "W-67890"
  deadline: "2026-04-07T20:01:00Z"
  category: WORK_CATEGORY_EVENT
  attempt: 1
  task: { agent_call: { call_sid: 42, call_type: CALL_TYPE_INBOUND, ... } }
}

Client -> Ack { work_ids: ["W-67890"] }
```

Acks can be batched — multiple event IDs in a single message.

#### 4. Replenishing capacity

Each `Result` or `Ack` implicitly frees capacity. The client sends additional `Pull` messages when it's ready for more work:

```
Client -> Pull { max_items: 2 }     // "I finished 2 items, send me 2 more"
```

A typical client loop:

```
1. Send Pull(max_items=N)         // N = number of worker slots
2. Receive WorkItem               // process it
3. Send Result or Ack             // complete it
4. Send Pull(max_items=1)         // replenish one slot
5. goto 2
```

The client is free to batch — process several items, then send a single `Pull(max_items=3)` to replenish three slots at once.

#### 5. Heartbeats

Both sides send `Heartbeat` messages at the interval specified in `Registered.heartbeat_interval`:

```
Server -> Heartbeat { client_time: "..." }
Client -> Heartbeat { client_time: "..." }
```

If either side stops receiving messages (heartbeats or otherwise) for 2x the heartbeat interval, the connection is considered dead.

### Lease semantics

Every `WorkItem` carries a `deadline` — an absolute timestamp. This is the **lease**.

#### Normal completion

Client sends `Result` or `Ack` before the deadline. Server sends `ResultAccepted`. The work item is done.

#### Lease expiry

If the client doesn't send `Result`, `Ack`, or `ExtendLease` before the deadline, the server **reclaims** the work item and makes it available for redelivery to any connected client.

The server sends a warning before expiry:

```
Server -> LeaseExpiring {
  work_id: "W-12345"
  deadline: "2026-04-07T20:05:00Z"
  remaining: "30s"
}
```

The client can respond with `ExtendLease`:

```
Client -> ExtendLease {
  work_id: "W-12345"
  extension: "300s"
}

Server -> LeaseExtended {
  work_id: "W-12345"
  new_deadline: "2026-04-07T20:10:00Z"
}
```

#### Explicit rejection

If the client can't process an item (e.g., database is down), it sends `Nack`:

```
Client -> Nack {
  work_id: "W-12345"
  reason: "database unavailable"
}

Server -> NackAccepted { work_id: "W-12345" }
```

The work item is immediately returned to the queue. Unlike lease expiry (which waits for the deadline), nack is instant.

#### Redelivery tracking

Each `WorkItem` has an `attempt` field. First delivery is `attempt: 1`. After a lease expiry or nack, the next delivery is `attempt: 2`, and so on. Clients can use this to:

- Detect **poison-pill** items that repeatedly fail (e.g., skip after attempt 3).
- Implement **deduplication** if the same item is delivered to multiple clients during a lease race.

### Chunked results

For large results (e.g., thousands of pool records), the client sends multiple `Result` messages with `final: false`, then a final one with `final: true`:

```
Client -> Result { work_id: "W-1", final: false, payload: { get_pool_records: { records: [...first 100...] } } }
Client -> Result { work_id: "W-1", final: false, payload: { get_pool_records: { records: [...next 100...] } } }
Client -> Result { work_id: "W-1", final: true,  payload: { get_pool_records: { records: [...last 50...], next_page_token: "" } } }

Server -> ResultAccepted { work_id: "W-1" }
```

The lease remains active during chunked submission. The server accumulates chunks and only considers the job complete when `final: true` is received.

### Error handling

**Job execution errors** — submit an `ErrorResult`:

```
Client -> Result {
  work_id: "W-12345"
  final: true
  payload: { error: { message: "stored procedure timeout", code: "DB_TIMEOUT" } }
}
```

**Stream errors** — the server sends `StreamError` for non-fatal issues:

```
Server -> StreamError {
  work_id: "W-99999"
  code: "INVALID_WORK_ID"
  message: "work item W-99999 not found or already completed"
}
```

Fatal errors (authentication failure, server shutdown) close the stream with a gRPC status code. The client reconnects with exponential backoff and re-registers.

### Reconnection

On stream disconnect:

1. All work items with unexpired leases remain "in progress" on the server until their deadlines pass, then get reclaimed.
2. The client reconnects with exponential backoff (recommended: 2s base, 30s max, 20% jitter).
3. The client sends a new `Register` message.
4. The client sends `Pull` to request work.
5. Any work items that expired during the disconnect will be redelivered (with incremented `attempt`).

No state needs to survive reconnection on the client side — the lease mechanism handles everything server-side.

---

## Work item types

### Jobs (require Result)

| WorkType enum | Task message | Result message | Description |
|---------------|-------------|---------------|-------------|
| `LIST_POOLS` | `ListPoolsTask` | `ListPoolsResult` | List all available pools/campaigns |
| `GET_POOL_STATUS` | `GetPoolStatusTask` | `GetPoolStatusResult` | Get pool status and record count |
| `GET_POOL_RECORDS` | `GetPoolRecordsTask` | `GetPoolRecordsResult` | Fetch records from a pool (paginated) |
| `SEARCH_RECORDS` | `SearchRecordsTask` | `SearchRecordsResult` | Search records by filters (paginated) |
| `GET_RECORD_FIELDS` | `GetRecordFieldsTask` | `GetRecordFieldsResult` | Read fields from a specific record |
| `SET_RECORD_FIELDS` | `SetRecordFieldsTask` | `SetRecordFieldsResult` | Write fields to a record |
| `CREATE_PAYMENT` | `CreatePaymentTask` | `CreatePaymentResult` | Create a payment record |
| `POP_ACCOUNT` | `PopAccountTask` | `PopAccountResult` | Pop/retrieve account for agent |
| `EXECUTE_LOGIC` | `ExecuteLogicTask` | `ExecuteLogicResult` | Run custom business logic |
| `INFO` | `InfoTask` | `InfoResult` | Return client app info and metadata |
| `SHUTDOWN` | `ShutdownTask` | `ShutdownResult` | Graceful shutdown request |
| `LOGGING` | `LoggingTask` | `LoggingResult` | Process a logging request |
| `DIAGNOSTICS` | `DiagnosticsTask` | `DiagnosticsResult` | Collect and return system diagnostics |
| `LIST_TENANT_LOGS` | `ListTenantLogsTask` | `ListTenantLogsResult` | Retrieve log entries (paginated) |
| `SET_LOG_LEVEL` | `SetLogLevelTask` | `SetLogLevelResult` | Change runtime log level |

### Events (require Ack only)

| WorkType enum | Entity message (from types.v3) | Description |
|---------------|-------------------------------|-------------|
| `AGENT_CALL` | `AgentCall` | Agent-to-customer call interaction data |
| `TELEPHONY_RESULT` | `TelephonyResult` | Call outcome and telemetry |
| `AGENT_RESPONSE` | `AgentResponse` | Agent's response to a call (key/value) |
| `TRANSFER_INSTANCE` | `TransferInstance` | Call transfer tracking |
| `CALL_RECORDING` | `CallRecording` | Recording metadata |
| `TASK` | `Task` | Background task state change |

---

## Domain services

The 43 unary RPCs from `gate.v2.GateService` are split into focused services. Each service handles one domain and can be evolved independently.

### AgentService (`agent/v3`)

Agent lifecycle, state management, and skills.

| RPC | Method | Description |
|-----|--------|-------------|
| `GetAgent` | `GET /agents/{id}` | Get agent by partner_agent_id or user_id (merged from v2's two separate RPCs) |
| `ListAgents` | `GET /agents` | List with filters and pagination (v2 was streaming-only) |
| `UpsertAgent` | `POST /agents/{id}` | Create or update agent profile (no password — see below) |
| `SetAgentCredentials` | `POST /agents/{id}/credentials` | Set password separately from profile (security fix) |
| `GetAgentStatus` | `GET /agents/{id}/status` | Current state, session, connected party |
| `UpdateAgentStatus` | `POST /agents/{id}/status` | Change agent state |
| `MuteAgent` | `POST /agents/{id}/mute` | Mute agent audio |
| `UnmuteAgent` | `POST /agents/{id}/unmute` | Unmute agent audio |
| `AddAgentCallResponse` | `POST /agents/{id}/call_responses` | Record agent's call response |
| `ListHuntGroupPauseCodes` | `GET /agents/{id}/pause_codes` | Available pause codes |
| `ListSkills` | `GET /skills` | All skills in the system |
| `ListAgentSkills` | `GET /agents/{id}/skills` | Skills assigned to an agent |
| `AssignAgentSkill` | `POST /agents/{id}/skills` | Assign a skill with proficiency |
| `UnassignAgentSkill` | `DELETE /agents/{id}/skills/{skill_id}` | Remove a skill |

### CallService (`call/v3`)

Dialing, transfers, hold, recording control, and compliance.

| RPC | Method | Description |
|-----|--------|-------------|
| `Dial` | `POST /agents/{id}/dial` | Initiate outbound call |
| `Transfer` | `POST /agents/{id}/transfer` | Transfer call (was GET-with-body in v2) |
| `SetHoldState` | `POST /agents/{id}/hold` | Hold or unhold (replaces 6 RPCs in v2) |
| `StartCallRecording` | `POST /agents/{id}/recording/start` | Start recording |
| `StopCallRecording` | `POST /agents/{id}/recording/stop` | Stop recording |
| `GetRecordingStatus` | `GET /agents/{id}/recording/status` | Check if recording |
| `ListComplianceRulesets` | `GET /compliance/rulesets` | Available NCL rulesets |

### RecordingService (`recording/v3`)

Voice recording search and retrieval.

| RPC | Method | Description |
|-----|--------|-------------|
| `SearchVoiceRecordings` | `POST /search` | Search with filters and pagination (v2 was streaming-only) |
| `GetDownloadLink` | `POST /download_link` | Get download/playback URLs (was GET-with-body in v2) |
| `ListSearchableFields` | `GET /searchable_fields` | Available search fields |
| `CreateLabel` | `POST /labels` | Add metadata label to recording |

### ScrubListService (`scrublist/v3`)

Content blocking list management.

| RPC | Method | Description |
|-----|--------|-------------|
| `ListScrubLists` | `GET /scrublists` | List all scrub lists |
| `AddEntries` | `POST /scrublists/{id}/entries` | Add entries to a list |
| `UpdateEntry` | `PATCH /scrublists/{id}/entries/{content}` | Update an entry |
| `RemoveEntries` | `POST /scrublists/{id}/entries:remove` | Remove entries |

### ConfigService (`config/v3`)

Client configuration, certificate rotation, and platform logging.

| RPC | Method | Description |
|-----|--------|-------------|
| `GetClientConfiguration` | `GET /client_configuration` | Get org config (payload now `Struct` instead of opaque string) |
| `GetOrganizationInfo` | `GET /organization_info` | Get org details |
| `RotateCertificate` | `POST /rotate_certificate` | Rotate mTLS certificate |
| `Log` | `POST /log` | Send log messages to platform |
| `AddRecordToJourneyBuffer` | `POST /journey_buffer` | Push record to customer context |

---

## Entity type changes (types/v3)

### Parallel arrays replaced

v2 used two synchronized arrays for task metadata:

```protobuf
// v2 — broken by design
repeated google.protobuf.Value task_data_keys = 20;
repeated google.protobuf.Value task_data_values = 21;
```

v3 uses a proper repeated message:

```protobuf
// v3
message TaskData {
  string key = 1;
  google.protobuf.Value value = 2;
}

// Used in AgentCall, TelephonyResult
repeated TaskData task_data = 40;
```

### call_type: string to enum

v2 used a raw `string` for call type despite having a `CallType` enum in the same package. Every consumer had to write string-to-enum conversion. v3 uses `CallType` everywhere:

```protobuf
// v2
string call_type = 3;

// v3
CallType call_type = 3;
```

### TelephonyOutcome: flat enum to categorized

v2 had a single `Result` enum with 60+ values ranging from 0 to 9500. v3 splits this into category + detail:

```protobuf
message TelephonyOutcome {
  Category category = 1;   // ANSWERED, NO_ANSWER, BUSY, MACHINE, INVALID, FAILED, CANCELED, PENDING
  Detail detail = 2;       // GENERIC, LINKCALL, DNC_CHECK_PERSONAL, CARRIER_REJECT, etc.
}
```

Consumers switch on `category` first, then `detail` if needed. Most business logic only cares about the category.

### Durations: int64 to google.protobuf.Duration

v2 used raw `int64` fields for durations (with ambiguous units — some microseconds, some milliseconds). v3 uses `google.protobuf.Duration`:

```protobuf
// v2
int64 talk_duration = 4;

// v3
google.protobuf.Duration talk_duration = 10;
```

### Record payload: string to Struct

```protobuf
// v2
string json_record_payload = 3;

// v3
google.protobuf.Struct payload = 3;
```

### Filter operators unified

v2 had two incompatible filter types:
- `core.v2.Filter.Operator` — only `EQUAL`
- `gate.v2.SearchOption.Operator` — `EQUAL`, `CONTAINS`, `NOT_EQUAL`

v3 has a single `Filter` with a full operator set: `EQUAL`, `NOT_EQUAL`, `CONTAINS`, `GREATER_THAN`, `LESS_THAN`, `IN`, `EXISTS`.

### TransferInstance destination fixed

v2 had a `map<string, bool> skills` field sitting outside the `oneof entity` in `Destination`, making it unclear which destination types it applied to. v3 moves skills inside a dedicated `QueueTarget` message within the oneof:

```protobuf
message Destination {
  oneof target {
    AgentTarget agent = 1;
    OutboundTarget outbound = 2;
    QueueTarget queue = 3;    // skills live here
  }
}
```

### Transfer initiation: booleans to enum

v2 used two ambiguous boolean flags (`start_as_pending`, `started_as_conference`). v3 uses a single enum:

```protobuf
enum TransferInitiation {
  TRANSFER_INITIATION_DIRECT = 1;
  TRANSFER_INITIATION_PENDING = 2;
  TRANSFER_INITIATION_CONFERENCE = 3;
}
```

### Skill proficiency: optional

v2 used a non-optional `int64 proficiency` — impossible to distinguish "not assigned" from "proficiency = 0". v3 marks it `optional`.

### AgentEvent/CallerEvent removed

`AgentEvent` and `CallerEvent` were defined in `gate/v2/entities.proto` but never referenced in any v2 RPC, message, or client code. They were orphaned definitions. v3 drops them entirely.

### Config payload typed

v2 returned `string config_payload` — an opaque JSON blob. v3 returns `google.protobuf.Struct config_payload` for type-safe access.

### Diagnostics portable

v2's `DiagnosticsResult` had 50+ fields specific to the Java runtime (heap stats, GC info, Hikari pool metrics). v3 uses generic `Struct` sections that any runtime can populate:

```protobuf
message DiagnosticsResult {
  google.protobuf.Struct system_info = 1;    // OS, hardware, container
  google.protobuf.Struct runtime_info = 2;   // JVM, Go runtime, Python, etc.
  google.protobuf.Struct database_info = 3;  // Connection pool, query stats
  google.protobuf.Struct custom = 4;         // Plugin-specific
}
```

---

## Complete sequence examples

### Example 1: normal job lifecycle

```
Client                                          Server
  |                                                |
  | Register(name, version, capabilities)          |
  |----------------------------------------------->|
  |                                                |
  |          Registered(client_id, heartbeat, lease)|
  |<-----------------------------------------------|
  |                                                |
  | Pull(max_items=3)                              |
  |----------------------------------------------->|
  |                                                |
  |          WorkItem(W-1, deadline, JOB,           |
  |            list_pools: {})                      |
  |<-----------------------------------------------|
  |                                                |
  |          WorkItem(W-2, deadline, JOB,           |
  |            get_pool_status: {pool_id: "P-1"})   |
  |<-----------------------------------------------|
  |                                                |
  |  (processing W-1...)                           |
  |                                                |
  | Result(W-1, final=true, list_pools: {pools:..})|
  |----------------------------------------------->|
  |                                                |
  |          ResultAccepted(W-1)                    |
  |<-----------------------------------------------|
  |                                                |
  | Pull(max_items=1)          // replenish 1 slot |
  |----------------------------------------------->|
  |                                                |
  |  (processing W-2...)                           |
  |                                                |
  |          WorkItem(W-3, deadline, EVENT,          |
  |            agent_call: {call_sid: 42, ...})     |
  |<-----------------------------------------------|
  |                                                |
  | Result(W-2, final=true, get_pool_status: {...}) |
  |----------------------------------------------->|
  |          ResultAccepted(W-2)                    |
  |<-----------------------------------------------|
  |                                                |
  | Ack(work_ids: ["W-3"])                         |
  |----------------------------------------------->|
  |                                                |
  | Pull(max_items=2)         // replenish 2 slots |
  |----------------------------------------------->|
```

### Example 2: lease expiry and redelivery

```
Client A                Server               Client B
  |                        |                      |
  | Pull(max_items=1)      |                      |
  |----------------------->|                      |
  |                        |                      |
  |  WorkItem(W-5,         |                      |
  |   deadline=T+5m, JOB)  |                      |
  |<-----------------------|                      |
  |                        |                      |
  | (client A crashes)     |                      |
  |    X                   |                      |
  |                        |                      |
  |                        | (time passes...)     |
  |                        |                      |
  |                        | LeaseExpiring(W-5)   |  (no one listening)
  |                        |                      |
  |                        | (deadline passes)    |
  |                        | W-5 reclaimed        |
  |                        |                      |
  |                        |   Pull(max_items=2)  |
  |                        |<---------------------|
  |                        |                      |
  |                        |  WorkItem(W-5,       |
  |                        |   deadline=T+10m,    |
  |                        |   attempt=2, JOB)    |
  |                        |--------------------->|
  |                        |                      |
  |                        |  Result(W-5, ...)    |
  |                        |<---------------------|
  |                        |                      |
  |                        |  ResultAccepted(W-5) |
  |                        |--------------------->|
```

### Example 3: nack and immediate redelivery

```
Client                                          Server
  |                                                |
  |          WorkItem(W-7, deadline, JOB,           |
  |            create_payment: {...})               |
  |<-----------------------------------------------|
  |                                                |
  |  (database is down, can't process)             |
  |                                                |
  | Nack(W-7, reason="database unavailable")       |
  |----------------------------------------------->|
  |                                                |
  |          NackAccepted(W-7)                      |
  |<-----------------------------------------------|
  |                                                |
  |  (W-7 immediately available for another client) |
```

### Example 4: chunked result

```
Client                                          Server
  |                                                |
  |          WorkItem(W-10, deadline=T+5m, JOB,     |
  |            get_pool_records: {pool: "P-1"})    |
  |<-----------------------------------------------|
  |                                                |
  |  (query returns 1000 records)                  |
  |                                                |
  | Result(W-10, final=false,                      |
  |   get_pool_records: {records: [1..500]})       |
  |----------------------------------------------->|
  |                                                |
  | Result(W-10, final=true,                       |
  |   get_pool_records: {records: [501..1000],     |
  |     next_page_token: ""})                      |
  |----------------------------------------------->|
  |                                                |
  |          ResultAccepted(W-10)                   |
  |<-----------------------------------------------|
```

### Example 5: lease extension

```
Client                                          Server
  |                                                |
  |          WorkItem(W-15, deadline=T+5m, JOB,     |
  |            execute_logic: {name: "heavy_calc"}) |
  |<-----------------------------------------------|
  |                                                |
  |  (still processing after 4 minutes...)         |
  |                                                |
  |          LeaseExpiring(W-15,                    |
  |            deadline=T+5m, remaining="60s")      |
  |<-----------------------------------------------|
  |                                                |
  | ExtendLease(W-15, extension="300s")            |
  |----------------------------------------------->|
  |                                                |
  |          LeaseExtended(W-15,                    |
  |            new_deadline=T+10m)                  |
  |<-----------------------------------------------|
  |                                                |
  |  (finishes processing)                         |
  |                                                |
  | Result(W-15, final=true, execute_logic: {...})  |
  |----------------------------------------------->|
  |                                                |
  |          ResultAccepted(W-15)                   |
  |<-----------------------------------------------|
```

---

## Migration from v2

v2 and v3 are independent packages. The server can implement both simultaneously, allowing clients to migrate incrementally.

### Mapping v2 to v3

| v2 | v3 |
|----|-----|
| `gate.v2.GateService.JobQueueStream` | `worker.v3.WorkerService.WorkStream` (jobs come as `WorkItem` with `WORK_CATEGORY_JOB`) |
| `gate.v2.GateService.EventStream` | `worker.v3.WorkerService.WorkStream` (events come as `WorkItem` with `WORK_CATEGORY_EVENT`) |
| `gate.v2.GateService.SubmitJobResults` | `worker.v3.Result` message on the WorkStream |
| `gate.v2.GateService.GetAgentStatus` | `agent.v3.AgentService.GetAgentStatus` |
| `gate.v2.GateService.ListAgents` | `agent.v3.AgentService.ListAgents` |
| `gate.v2.GateService.Dial` | `call.v3.CallService.Dial` |
| `gate.v2.GateService.Transfer` | `call.v3.CallService.Transfer` |
| `gate.v2.GateService.PutCallOnSimpleHold` | `call.v3.CallService.SetHoldState(target=CALL, action=HOLD)` |
| `gate.v2.GateService.TakeCallOffSimpleHold` | `call.v3.CallService.SetHoldState(target=CALL, action=UNHOLD)` |
| `gate.v2.GateService.HoldTransferMemberCaller` | `call.v3.CallService.SetHoldState(target=TRANSFER_CALLER, action=HOLD)` |
| `gate.v2.GateService.UnholdTransferMemberCaller` | `call.v3.CallService.SetHoldState(target=TRANSFER_CALLER, action=UNHOLD)` |
| `gate.v2.GateService.HoldTransferMemberAgent` | `call.v3.CallService.SetHoldState(target=TRANSFER_AGENT, action=HOLD)` |
| `gate.v2.GateService.UnholdTransferMemberAgent` | `call.v3.CallService.SetHoldState(target=TRANSFER_AGENT, action=UNHOLD)` |
| `gate.v2.GateService.SearchVoiceRecordings` | `recording.v3.RecordingService.SearchVoiceRecordings` |
| `gate.v2.GateService.GetVoiceRecordingDownloadLink` | `recording.v3.RecordingService.GetDownloadLink` |
| `gate.v2.GateService.GetClientConfiguration` | `config.v3.ConfigService.GetClientConfiguration` |
| `gate.v2.GateService.RotateCertificate` | `config.v3.ConfigService.RotateCertificate` |
| `gate.v2.GateService.ListScrubLists` | `scrublist.v3.ScrubListService.ListScrubLists` |
| `gate.v2.GateService.ListSkills` | `agent.v3.AgentService.ListSkills` |
| `gate.v2.GateService.GetAgentById` | `agent.v3.AgentService.GetAgent(user_id=...)` |
| `gate.v2.GateService.GetAgentByPartnerId` | `agent.v3.AgentService.GetAgent(partner_agent_id=...)` |
| `gate.v2.GateService.UpsertAgent` (with password) | `agent.v3.AgentService.UpsertAgent` + `SetAgentCredentials` |

### Migration steps for a sati client

1. **Add v3 dependency** alongside v2 in `buf.yaml`.
2. **Implement WorkStream client** using the new protocol. Replace `GateClientJobQueue`, `GateClientEventStream`, and `GateClient.submitJobResults()` with a single `WorkStreamClient`.
3. **Replace unary RPC stubs** from `GateServiceGrpc` with individual service stubs (`AgentServiceGrpc`, `CallServiceGrpc`, etc.).
4. **Update entity mappings** in plugin implementations to use `types.v3` messages (e.g., `TaskData` instead of parallel arrays, `CallType` enum instead of string).
5. **Remove v2 dependency** once all clients have migrated.

### What the sati client no longer needs

| v2 component | Why it's gone in v3 |
|-------------|-------------------|
| `GateClientAbstract` (3 subclasses) | Single `WorkStreamClient` with one reconnect loop |
| `GateClientEventStream` | Events come through WorkStream |
| `GateClientPollEvents` | Deprecated in v2, deleted in v3 |
| `GateClientJobStream` | Deprecated in v2, deleted in v3 |
| `QueueExecutor` / semaphore | `Pull(max_items)` IS the concurrency control |
| ACK buffer with synchronized list | Lease handles retransmission server-side |
| 45-second hung connection detection | Heartbeat + lease handles this |
| 10-second empty batch sleep | Server holds Pull until work is available |
| `TaskDataHelper` (zip parallel arrays) | `repeated TaskData` — no helper needed |
| `CallTypeUtils` (450-line reflection) | `CallType` is an enum — direct access |
