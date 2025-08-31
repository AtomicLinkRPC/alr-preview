# AtomicLinkRPC Comprehensive Guide

Consolidated overview, technical deep dive, examples, user guidance, API references, benchmarks, and promotional positioning for AtomicLinkRPC (ALR).

---
## 1. Overview
AtomicLinkRPC (ALR) is a native C++ RPC framework engineered for:

- Extreme throughput / low latency (tens to hundreds of millions RPC/s achievable in stress tests)
- Native ergonomics (headers are the interface; no IDL)
- Seamless evolution (schema negotiation eliminates fragile versioning)
- Integrated service discovery & adaptive load balancing (registry subsystem)
- Simple async patterns (`Async<T>`, `AsyncRef<T>`, `AsyncVoid`)

Key differentiators: tagless binary serialization, opportunistic batching, lock‑free hot paths, ambient context model, distributed object & async references.

---
## 2. Core Concepts
| Concept | Summary |
|---------|---------|
| Endpoint | Logical bidirectional TCP connection & negotiated schema view. |
| EndpointClass | Base for remotable classes (static & instance methods). |
| CommonEndpointClass | Symmetric base for classes creatable/callable on either/both peers. Codegen emits remote declaration under a root namespace (default `remote`). |
| Ambient Variables | Thread-scoped context objects (EndpointCtx, RemoteThread, RemoteThreadPool). |
| Async Primitives | `Async<T>` for simple futures; `AsyncRef<T>` for distributed chaining; `AsyncVoid` for tracked fire-and-forget. |
| ClassRef<T> | Distributed lifetime-managed references to endpoint class instances. |
| Registry | Built-in discovery + load aware routing + topology hints. |
| Schema Negotiation | Connection-time mapping + table reorder enabling tagless transport. |

---
## 3. Quick Start Snippets
```cpp
// service.h
class Greeter : public alr::EndpointClass {
public:
  static std::string hello(const std::string& name);
  static alr::Async<std::string> helloAsync(const std::string& name);
};

// service.cpp
std::string Greeter::hello(const std::string& n){ return "Hello " + n; }

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo().setConnectAddress("...").connect();
  auto s = Greeter::hello("World");
  auto a = Greeter::helloAsync("Async");
  a.then([a]{ printf("%s\n", a.value().c_str()); });
}
```

Distributed pipeline example:
```cpp
auto final = Img::encode( Img::sharpen( Img::load("cat.png"), 0.6f ), 90 );
final.wait(); const auto& img = final.value();
```

Symmetric class sketch (peer-to-peer):
```cpp
class Chat : public alr::CommonEndpointClass { public: void post(const std::string& m); };
Chat local;        // Local instance
remote::Chat peer; // Remote proxy (codegen under root namespace, default `remote`)
local.post("hi from me");
peer.post("hi from you");
```

---
## 4. Architecture Highlights
| Pillar | Mechanism |
|--------|-----------|
| Negotiated Schema | Exchange & merge class/method/type metadata; reorder visitor tables. |
| Tagless Encoding | Deterministic field ordering; no per-field tags → minimal bytes. |
| Batching | Per-thread staging + multi-thread merge reduces syscalls by 99%+. |
| Continuation Execution | Blocked sync caller services inbound dependent RPCs. |
| Flow Control | Per-thread quotas + dynamic recv flow units prevent overload. |
| Thread Affinity Tools | `RemoteThread` (single remote thread), `RemoteThreadPool` (pool). |
| Distributed References | Reference counts across endpoints; lazy materialization for `AsyncRef<T>`. |

---
## 5. Service Registry Essentials

- Backend sets service metadata (name, region, zone, tags) and registers via registry address.
- Clients specify desired service + optional locality; registry returns candidate set.
- Custom resolver callback chooses instance (least active threads, lowest memory usage, tag filtering, etc.).
- Heartbeats maintain liveness; stale instances are pruned.

---
## 6. Performance Snapshot
| Scenario | Throughput (ALR) | Notes |
|----------|------------------|-------|
| Tiny async (int32) | ~20M RPC/s | 8–9 byte framed messages, high batch density |
| Small struct flood | ~80–90M RPC/s | ~19 byte framed payloads |
| Multi ping-pong | ~10M RPC/s | 64 endpoints concurrency |
| Business logic mix | ~0.7–0.8M RPC/s | Larger response payloads |
| 2.5Gbps saturation | ~72M RPC/s | ~97% theoretical utilization |

Speedups vs representative gRPC runs: 25× to 900× depending on workload shape.

---
## 7. User Guide Essentials
| Task | Steps |
|------|-------|
| Define service | Inherit from `alr::EndpointClass` or `alr::CommonEndpointClass`, implement methods. |
| Build | Ensure ALR compiler runs; include generated `*_gen.h`, `*_gen.cpp`. |
| Connect | `alr::Endpoint ep = alr::ConnectionInfo().connect();` |
| Thread usage | Add `EndpointCtx` in new threads. |
| Async chain | Use `AsyncRef<T>` to keep intermediates remote. |
| Bulk transfer | Use `ByteBuffer` + `waitForBufferReady()` for reuse. |
| Error returns | Use `Result<T,E>` for domain errors. |

---
## 8. API Surface (Abbreviated)
| Category | Symbols |
|----------|---------|
| Endpoint Lifecycle | `Endpoint`, `EndpointCtx`, `EndpointWeakRef`, `EndpointConfig`, stats retrieval |
| Classes & Objects | `EndpointClass`, `CommonEndpointClass`, `ClassRef<T>` |
| Async | `Async<T>`, `AsyncVal<T>`, `AsyncRef<T>`, `AsyncVoid` |
| Thread Control | `RemoteThread`, `RemoteThreadPool` |
| Data Transport | `ByteBuffer` |
| Results & Context | `Result<T,E>`, `CallCtx` |
| Discovery | `ConnectionInfo`, `ServiceInstance`, `InstanceLoadInfo` |
| Perf | `PerfTimer`, `PerfResults` |

For full details refer to `api-reference.md`.

---
## 9. Technical Deep Dive (Condensed)

- **Handshake:** Builds intersection schema; one side reorders tables -> tagless encoding.
- **Serialization:** Visitors serialize directly into contiguous buffers; struct fields omitted if not shared by both sides.
- **Batching:** `maxBatchMsgs`, `maxMsgBatchSize`, time budget combine to shape flush rhythm; aggregator coalesces multi-thread buffers.
- **Flow Control:** Byte + flow unit budgets enforce fairness (throttle producers early under pressure).
- **Continuations:** Synchronous call stack can process inbound work, collapsing multi-RPC dependency latency.
- **AsyncRef:** Reference holds remote state; accessing value triggers transfer; chaining reduces wire traffic.

---
## 10. Evolution Model
| Change | Compatibility |
|--------|---------------|
| Add method | Safe, ignored by old side |
| Remove method | Guard callers with `remoteCaps::hasMethod::<ns1>::<ns2>::<Class>::<method>()`; unguarded legacy call triggers `UndefinedRemoteMethodError` (diagnostic) |
| Add struct field | Safe; silently omitted to older peer |
| Remove struct field | Safe; field absent in new view |
| Reorder params/fields | Safe; mapping by identity |
| Change struct field name or type | Treated as a different field (identity = type + name). After handshake each side transmits only the intersection set: the old-named field is NOT sent to the newer peer (it has no matching identity there) and the new field is NOT sent to the older peer. Each side sees defaults for fields the other lacks. Use `remoteCaps::hasStructField::<...>()`  to branch on presence. Remove deprecated field after migration window. |
| Change method parameter name or type | Parameter identity = type + name. Non-overlapping changed parameter is simply omitted from transmission; the receiving side supplies/observes a default value for that parameter. The method itself still maps and is invocable both ways. Use `remoteCaps::hasMethodParam::<...>()` (and `hasMethodReturn` if needed) to branch on presence. Stage removals by optionally moving deprecated params to the tail before final deletion. |

---
## 11. Performance Tuning Quick Table
| Goal | Action |
|------|-------|
| Higher throughput | Increase `maxBatchMsgs` / `maxMsgBatchSize` cautiously |
| Lower latency | Decrease `maxBatchYieldUs` |
| Limit memory | Adjust `maxAllocMemory`, `asyncValueThrottle` |
| More parallel remote work | Use `RemoteThreadPool(N)` |
| Deterministic remote ordering | Use `RemoteThread` |

---
## 12. Diagnostics & Observability
| Tool | Usage |
|------|-------|
| `EndpointStats` | Batch histograms, send/recv counts |
| `EndpointQueueStats` | Per-queue load insight |
| `PerfTimer` | Wrap benchmarks for structured logs |
| Debug Modes | `setLocalDebugMode`, `setRemoteDebugMode` for message tracing |

---
## 13. Registry Policy Patterns
| Policy | Resolver Logic Sketch |
|--------|-----------------------|
| Least active threads | Pick instance with min `numActiveThreads` |
| Memory efficiency | Min ratio `allocatedMemMB / maxServiceMemMB` |
| Locality first | Prefer exact region/zone else fallback |
| Tag weighted | Filter by tag/value; weighted random selection |

---
## 14. Promotional Angles (Recap)

- Replace multi-layer RPC stack with lean native path → fewer cores, reduced latency budgets.
- Accelerate refactors & API iteration (no brittle IDL migrations).
- Built-in control plane (registry) simplifies platform architecture.

---
## 15. Migration Outline

1. Select existing service boundary.
2. Derive classes from `EndpointClass`; remove intermediate DTO wrappers if present.
3. Build, run generation.
4. Deploy ALR variant in parallel (dark traffic or mirrored requests).
5. Compare metrics; incrementally shift production traffic.

---
## 16. Benchmark Interpretation Aids
If high throughput but rising tail latency: check batch yield param; consider multi-connection distribution. If low batching histogram: calls too sparse—aggregate at call sites or widen concurrency.

---
## 17. Exception Summary
| Exception | Trigger |
|-----------|--------|
| NoEndpointOnStackError | Thread lacks context |
| WrongEndpointOnStackError | Using object bound to different endpoint |
| UndefinedRemoteMethodError | Version skew: method invoked without prior capability guard |
| InvalidMessageError | Schema mismatch or corruption |
| InvalidConfigurationError | Bad endpoint / config values |
| NoBackendServiceError | Registry cannot resolve service |
| ConnectionClosedError | Unexpected disconnect |
| GenericError | Other internal reported issue |

---
## 18. Security
Enable TLS via `ConnectionInfo::setOpenSsl(...)`. Minimal overhead added relative to overall performance improvements; maintain separate policies per service tier if needed.

---
## 19. Future Directions

- Additional language bindings.
- Extended runtime introspection & tracing export.
- Richer generated capability metadata dumping (debug tooling).

---
## 20. Closing Summary
ALR collapses the traditional trade-off triangle (Performance, Simplicity, Evolvability) by pushing complexity to compile-time analysis and connection-time schema alignment. Whether optimizing cost, latency budgets, or iteration speed, ALR provides a compelling foundation for modern high-scale C++ distributed systems.

For deeper details: see [technical.md](./technical.md), [api-reference.md](./api-reference.md), and [benchmarks.md](./benchmarks.md).
