# AtomicLinkRPC Technical Documentation

## 1. Design Objectives
| Objective | Approach |
|-----------|----------|
| Max throughput & low latency | Direct TCP, opportunistic batching, lock-free hot paths |
| Deterministic threading | Per-thread queue slots + continuation handling |
| Transparent distributed objects | ID-based lifetime managed references (ClassRef) |
| Simple async semantics | Return type based design (`Async<T>`, `AsyncRef<T>`, `AsyncVoid`) |
| Zero-friction API evolution | Handshake schema negotiation + visitor table remap |
| Built-in service discovery & load balancing | Native registry + client resolver callback |

---
## 2. Compilation & Code Generation
The ALR compiler (libclang powered) scans user headers searching for classes inheriting `alr::EndpointClass` or `alr::CommonEndpointClass`. For each:

1. Collect class metadata: method names, static vs instance, parameter types, return types, struct & enum dependencies.
2. Emit generated files (`*_gen.h / *_gen.cpp`) implementing:

   - Remote invocation stubs (caller side wrappers invoking serializer).
   - Dispatch tables mapping remote IDs to user method entry points.
   - Serialization visitor implementations for encountered user types.
3. Preserve stable *semantic IDs* (derived from qualified names / signatures) allowing cross-version mapping.
4. Regenerate only when structural changes detected (first build or existing content mismatch) -> fast incremental builds.

---
## 3. Handshake Phase
Upon connection establishment:

1. Each endpoint sends a compact schema descriptor (classes, methods, param & return type shapes, struct field lists, type kinds, ordering).
2. Each side computes intersection sets (removes unknown artifacts) and produces mapping arrays from local indices to remote indices.
3. One endpoint reorders its visitation/serialization tables to line up exactly with remote ordering (ensuring symmetrical numerically indexed layout).
4. Result: runtime serialization can omit per-field tags—only raw values emitted in deterministic order. Non-overlapping parameters / fields are skipped.

After receiving the remote schema, the remainder of the steps typically take less than 1ms.

---
## 4. Serialization Strategy
| Feature | Implementation |
|---------|---------------|
| Tag elimination | Pre-aligned joint schema eliminates runtime field identifiers. |
| Compact primitives | Variable length integer packing (different compared to gRPC). |
| Nested types | Flattened to a contiguous sequence of leaf values. |
| Default values | Default values (of any type, including nested) represented by a `default` control byte, with 3 or more consecutive default values (of any type) converted to a single `default run`. |
| Containers | `default` if empty, length prefix + contiguous element serialization (vector), keys (set), key-value pairs (map). |
| Fixed arrays | Flattened contiguous element serialization with `default` representation. |
| Optional / shared / ref types | Non-`default` control byte only when required. |
| `AsyncRef<T>` references | Transmit integer ID; value optionally transferred from remote side if `value()` called. |
| `ByteBuffer` | Frame referencing contiguous block; send-side marks pending; reuse gated by async completion. |
| Non-default values | All non-default values are type safe, e.g. can't write an int, read a string, else `InvalidMessageError` exception + faulted endpoint. |
| Message consistency |  Must read up to the very last byte, not more, not less, else `InvalidMessageError` exception + faulted endpoint. |

---
## 5. Message Framing & Batching
Every logical RPC message contains at minimum: (a) total size, (b) message kind, (c) queue/thread slot, (d) target method ID (and optionally object ID). Invocations with no parameters can be as small as 4 bytes. Batching pipeline:

1. Thread starts writing messages into per-thread staging buffer until thresholds (`maxBatchMsgs` / `maxMsgBatchSize`) reached. Notifies the aggregator thread after the first write with timestamp.
2. Aggregator thread checks yield time reached for writing threads, attempts to acquire its staging buffer (will re-schedule on failed acquire). 
3. If staging buffer successfully acquired, aggregator merges it into network batch (`mergeBatchBufferSize`) minimizing syscalls.
4. Receiver demultiplexes directly into per-thread queues (service threads vs client threads separated by even/odd slot assignment).

Flow control counters (send/recv flow units) throttle producers when remote consumption lags.

---
## 6. Threading & Continuations

- Each endpoint allocates queue slots; threads claim slot indexes (odd = service, even = client orientation—ensuring cross symmetry).
- Synchronous call blocking: while waiting for reply, the thread can *temporarily service inbound* framed calls targeting its slot (continuation). This reduces deadlock potential and median latency for dependent RPC chains.
- `RemoteThread`: reserves single remote processing thread; all subsequent RPCs pinned, enabling reuse of remote thread-local state.
- `RemoteThreadPool`: binds a logical remote queue allowing up to N threads to pull; retains ordering at dequeue but executes concurrently.

---
## 7. Async Model
| Type | Purpose | Behavior |
|------|---------|----------|
| `Async<T>` / `AsyncVal<T>` | Simple one-shot result | Callback (`then`), wait, cancel, timeout state. Value materialized locally. |
| `AsyncRef<T>` | Distributed reference | Value may remain remote across chained RPCs; only serialized back if/when accessed. Reference counting spans endpoints. |
| `AsyncVoid` | Fire-and-forget (still tracked) | Endpoint faulted if error occurs (no silent failure). |

Cancellation: Caller sets cancellation flag; on remote side, user logic polls via `CallCtx`. Timeouts appear as `isTimedOut()` state.

---
## 8. Distributed Object Lifetime Management
Each user-created instance of an `EndpointClass` obtains an object ID. Reference flows:

1. Construction on local side -> local registry entry.
2. Passing as parameter (ClassRef or reference) transmits object ID; remote side creates *proxy* (if necessary) linking to local lifetime.
3. Reference counts incorporate: local references, remote references, in-flight RPC references.
4. Destruction blocks until refcount zero → ensures deterministic cleanup without GC pauses.

---
## 9. Service Registry & Discovery
### 9.1 Roles
| Role | Description |
|------|-------------|
| Registry Node | Accepts backend service registrations, heartbeats, exposes instance list to clients. Supports chaining for scaling + locality overlays. |
| Backend Service | Ordinary service process publishing: name, region, zone, tags, versions, load metrics, memory usage, connections, active threads. |
| Client Resolver | Queries registry, optionally filters by service name, locality (region/zone), custom tags, then uses callback to pick candidate address; performs adaptive load distribution. |

### 9.2 Registration Lifecycle

1. Backend connects to registry using `ConnectionInfo` with `setRegistryAddress()` + service metadata.
2. Periodic re-registration / heartbeat updates load metrics (`InstanceLoadInfo`).
3. On missed heartbeats (timeout window) registry prunes instance.
4. Chaining: a registry may itself register upstream to a parent registry (fan-out tree / multi-region mesh).

### 9.3 Load Resolution
Client resolution callback receives vector<ServiceInstance>. Example policies:

- Least active threads
- Lowest memory utilization ratio (allocated / max)
- Geographically closest (exact region > same region different zone > cross region)
- Tag expression match (e.g., `tier=prod && version>=2025.08`)

### 9.4 Locality Awareness
Clients can pre-filter by specifying desired region/zone in `ConnectionInfo`. Registry supplies subset accordingly (implementation merges backend metadata vs client hints).

---
## 10. Error & Fault Model
| Condition | Handling |
|-----------|----------|
| Serialization mismatch | `InvalidMessageError` (should be prevented by handshake) |
| Unknown method (version skew) | Should be proactively avoided: guard optional calls with generated `remoteCaps::hasMethod::<...>()`. An unexpected invocation throws `UndefinedRemoteMethodError` (diagnostic – not a control‑flow primitive). |
| Async operation internal failure (`AsyncVoid`) | Endpoint enters faulted state; subsequent operations fail fast |
| Connection drop | `ConnectionClosedError`; user may reconnect (new handshake) |
| No backend available | `NoBackendServiceError`; retry with backoff / alternate region |
| Wrong endpoint usage on thread | `WrongEndpointOnStackError` |
| Missing ambient context | `NoEndpointOnStackError` |

`Endpoint::getError()` surfaces last fatal error string for diagnostics.

---
## 11. Performance Mechanics
### 11.1 Batching Parameters (Excerpt)
| Config Field | Purpose |
|--------------|---------|
| `maxBatchMsgs` | Upper bound number of logical messages coalesced per thread batch. |
| `maxMsgBatchSize` | Byte size cap for a single thread batch. |
| `threadBatchBufferSize` | Preallocated staging size per thread. |
| `mergeBatchBufferSize` | Aggregator merge buffer size (multi-thread -> socket). |
| `maxBatchYieldUs` | Microsecond window to opportunistically extend batch. |

### 11.2 Flow Control Fields
| Field | Purpose |
|-------|---------|
| `maxThreadSendBytes` | Per-thread outbound byte quota before throttling |
| `maxTotalSendBytes` | Global endpoint outbound budget |
| `maxThreadSendQueues` | Limit concurrent send queue structures per endpoint |
| `maxThreadRecvFlowUnits*` | Adaptive thresholds for inbound consumption pacing |

### 11.3 Async Throttle
`asyncValueThrottle` caps simultaneous outstanding async values to limit memory pressure.

### 11.4 Lock-Free Structures
Critical components likely include:

- Multi-producer single-consumer ring buffers or queues for message staging.
- Atomic refcount & state transitions for async values / distributed refs.
- Wait-free fast-path for message enqueuing (fallback to blocking only under resource saturation).

---
## 12. Latency Sampling & Instrumentation
`alr::PerfTimer` and `PerfResults`:

- Wrap code region to capture send/recv deltas, batch histogram, queue stats, latency samples.
- Latency capture: allocate vector, record microsecond durations, produce min/avg/max/percentiles (90/95/99). Useful to validate batching still within SLA.

Example:
```cpp
alr::PerfTimer t("PingSweep","Measure roundtrips",1000000, alr::PerfLogMode::BatchDetailed);
for(int i=0;i<1000000;i++) Service::ping();
```

---
## 13. Memory Management

- Pre-sized batch buffers reduce fragmentation.
- ByteBuffer reuses large contiguous allocations; `waitForBufferReady()` coordinates reuse after async send completes.
- AsyncRef values optionally materialize lazily to avoid duplicate large payload copies.
- EndpointConfig `maxAlloc*` fields enforce upper bounds to protect process stability.

---
## 14. Security Layer (TLS)
`ConnectionInfo::setOpenSsl()` toggles TLS (OpenSSL). TLS adds per-connection handshake + encryption overhead; overall batching & compact messages counterbalance typical cost. Mixed secure/unsecure endpoints can interoperate (handshake enforces mode per connection).

---
## 15. Evolution & Compatibility Rules
| Change | Behavior |
|--------|----------|
| Add method | New method invocable only by updated peers; older peers ignore; no break. |
| Remove method | Guard client code with `remoteCaps::hasMethod::<Class>::<method>()` before invoking. Unguarded old callers will hit `UndefinedRemoteMethodError` (indicates missed capability check). |
| Reorder parameters | Ignored (mapped by name/signature not position). |
| Add struct field | Not sent to older side; default-initialized there. |
| Remove struct field | Omitted from transmission; receiving side's field absent. |
| Type or name change | Treated as distinct; identity based on type + name. |

---
## 16. Failure Recovery
| Scenario | Strategy |
|----------|----------|
| Endpoint faulted due to async failure | Log `getError()`, rebuild / discard endpoint, optionally reconnect. |
| Registry node loss | Clients failover to chained/secondary registry; backends re-register on reconnect. |
| Sudden load spike | Flow control throttles; consider scaling backend instances (registry picks them up). |
| Memory pressure | Tune `EndpointConfig` / reduce outstanding async operations. |

---
## 17. Capability Lookup (Version Awareness)
During code generation, ALR emits a hierarchical namespace `remoteCaps` (one per service domain) containing constexpr-ish inline functions that answer whether the *remote peer* supports a given:

* Class: `remoteCaps::hasClass::myservice::MyClass()`
* Method: `remoteCaps::hasMethod::myservice::MyClass::newFeature()`
* Method parameter / return value: `remoteCaps::hasMethodParam::...`, `remoteCaps::hasMethodReturn::...`
* Struct: `remoteCaps::hasStruct::myservice::Payload()`
* Struct field: `remoteCaps::hasStructField::myservice::Payload::newField()`

These functions perform an O(1) boolean table lookup populated during the handshake (after schema intersection & visitor table reorder). Use them to write forward/backward compatible logic without try/catch branching. Example (adapted from CityGuide sample):

```cpp
WeatherInfo info = CityGuideService::getWeatherInfo(location);
if (remoteCaps::hasStructField::cityguide::WeatherInfo::conditionsV2()) {
   // Use richer v2 struct fields
   logV2(info.conditionsV2);
} else {
   // Fallback to legacy single string field
   logLegacy(info.conditions);
}

if (remoteCaps::hasMethod::cityguide::CityGuideService::getWeatherForecast()) {
   auto forecast = service.getWeatherForecast(location);
   consume(forecast);
}
```

Guidelines:

1. Never rely on catching `UndefinedRemoteMethodError` for normal version branching – treat it as a signal you *forgot* a capability guard.
2. Remove legacy fields/methods only after telemetry shows negligible calls from versions lacking the replacement.
3. Prefer additive evolution (add new field, keep old until sunset window ends; move deprecated members to struct tail with a comment so it is easy to track and maintain).

## 18. Future Extension Points (Indicative)

- Cross-language bindings via same negotiated schema (Rust, Python modern re-implementation, etc.).
- Observability hooks (structured tracing, OpenTelemetry exporters) leveraging existing queue stats & latency capture.

---
## 19. Key Public Interfaces Snapshot
(See `api-reference.md` for per-symbol detail.)

- Endpoint lifecycle: `Endpoint`, `EndpointCtx`, `EndpointConfig`, stats & queue introspection.
- Async primitives: `Async<T>`, `AsyncVal<T>`, `AsyncRef<T>`, `AsyncVoid`.
- Remotable classes: `EndpointClass`, `CommonEndpointClass`, `ClassRef<T>`.
- Thread control: `RemoteThread`, `RemoteThreadPool`.
- Flow & results: `Result<T,E>`, `CallCtx`.
- Data transport: `ByteBuffer`.
- Discovery: `ConnectionInfo`, `ServiceInstance`, `InstanceLoadInfo` (registry aware set).

---
## 20. Summary
ALR achieves its performance and flexibility goals by shifting complexity from per-call runtime work (tag parsing, dynamic reflection, multi-layer protocol stacks) to a one-time connection handshake and compile-time code generation. The result: a lean hot path (serialize raw native layouts directly) and a developer experience indistinguishable from local C++ method invocation—while enabling distributed, evolvable systems with built-in discovery.
