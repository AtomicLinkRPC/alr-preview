# AtomicLinkRPC (ALR) - Overview

## What is ALR?

**AtomicLinkRPC (ALR)** is a next-generation, high-performance C++ framework for building asynchronous and scalable Remote Procedure Call (RPC) applications. It is engineered from the ground up to provide a seamless, native C++ development experience, eliminating the boilerplate and complexity associated with traditional RPC frameworks like gRPC.

ALR's core philosophy is to let developers work with their own C++ types directly, without the need for Interface Definition Languages (IDLs) or generated cumbersome message classes with their getters and setters. This results in a framework that is not only incredibly fast but also intuitive and easy to integrate into existing projects.

## Core Value Proposition

- **Native C++ First**: Your C++ headers *are* the interface. Use your existing classes, native types, and STL containers directly. Keep your normal idioms like constructors, default arguments, references, smart pointers, and `std::optional`.
- **Extreme Performance**: ALR utilizes direct TCP communication, a compact zero-tag binary layout, opportunistic batching, and lock-free/wait-free hot paths. This results in performance of millions to hundreds of millions of RPCs per second.
- **Async Made Trivial**: Return `alr::Async<T>` (or `AsyncVoid`) for non-blocking calls. You can chain callbacks or wait for results, simplifying asynchronous programming.
- **Distributed Object References**: Pass `alr::ClassRef<T>` or `T&` representing local or remote instances. ALR transparently routes calls and manages object lifetimes across the network.
- **Seamless Evolution**: Add, remove, or reorder parameters or struct fields without breaking older clients or services. Optionally apply API evolve policies to individual parameters, fields and return values to enforce access patterns. A handshake-time schema negotiation and layout mapping allows different versions to interoperate automatically.
- **Built-in Service Registry**: ALR includes a sophisticated service registry for dynamic service discovery, registration, health monitoring, and client-side adaptive load balancing. No need for external tools like Consul or etcd.
- **Minimal Dependencies**: ALR has minimal dependencies: libclang for the build-time codegen and optional OpenSSL for TLS.

## When to Use ALR
ALR is ideal for performance-critical and latency-sensitive systems where developer velocity is important:

- High-throughput microservices and backend fleets
- Real-time analytics, trading, games, control systems, ETL, and machine learning inference
- Edge and region/zone-aware deployments
- Large-scale fan-out/fan-in RPC graphs needing adaptive routing
- Evolutionary platforms where API churn is expected

## Key Features

### 1. No IDL, Just C++
With ALR, your C++ header files are your API definition. Simply derive your service classes from `alr::EndpointClass` or `alr::CommonEndpointClass`, and ALR codegen handles the rest, generating the necessary RPC stubs behind the scenes.

### 2. Ambient Context
ALR uses a powerful "Ambient Variable" system rooted in thread-local storage. This allows you to set up an endpoint connection once per thread, and all subsequent RPCs automatically use that context without needing to be explicitly passed. This keeps your business logic clean and free of RPC-specific clutter.

```cpp
// Connect once
alr::Endpoint ep = alr::ConnectionInfo().setConnectAddress("...").connect();

// All subsequent calls on this thread use the connection implicitly
auto result = MyRemoteService::doWork(); 
```

### 3. Seamless Asynchrony
ALR makes asynchronous programming trivial with `alr::Async<T>`. A method returning this type will be non-blocking. The returned object can be used with callbacks, awaited, or checked for completion.

```cpp
// Define an async method
alr::Async<std::string> Greeter::SayHello(const std::string& name);

// Call it without blocking
alr::AsyncVal<std::string> result = Greeter::SayHello("world");

result.then([result]() {
    // This callback will be invoked once the result becomes available, is canceled, or times out
    std::cout << "Received: " << result.value() << std::endl;
});
```

### 4. Automatic Lifetime Management
ALR provides `alr::ClassRef<T>` to manage the lifecycle of remote object instances automatically. References can be passed between endpoints, and the framework ensures an object is only destroyed when all local, remote and in-flight references are gone.

### 5. Unmatched Performance
By using direct TCP sockets and a highly optimized, lock-free architecture, ALR achieves groundbreaking performance:

- **Over 200 million RPC/s** on localhost.
- **72 million RPC/s** over a 2.5Gbps network.
- Opportunistic batching packs messages tightly, maximizing network throughput and dramatically reducing system calls.

### 6. Transparent API Evolution
ALR endpoints exchange schemas during the initial handshake. This allows them to dynamically map parameters and fields, meaning:

- You can add, remove, or reorder method parameters and struct fields.
- Old clients can talk to new services (and vice-versa) without breaking.
- Business logic can query what the remote supports and adapt for multiple versions.
- The framework intelligently sends only the data that both endpoints understand.
  
### 7. Built-In Registry & Service Discovery
ALR includes a sophisticated service registry and discovery system that enables automatic load balancing and high availability:

- One or more services can be set up as a registry.
- Backend services register themselves with the registry and send periodic heartbeats.
- Registry services can be chained together to allow for a large fan-out.
- Clients can filter backend services by service name, region, and zone for locality-aware routing.
- A user-provided callback on the client-side allows for custom service resolution logic (e.g., lowest latency, custom tags).
- The client just needs to be pointed to the resolver, and the rest is automatic.

### 8. API Evolution Policies

ALR provides fine-grained control over API evolution through the `alr::Evolve<T, WritePolicy, ReadPolicy>` template wrapper. This wrapper can be applied to individual struct fields, method parameters, and return values to enforce runtime access patterns and validate correct API usage.

Key capabilities:

- **Write and Read Policies** - Separate control for sending vs receiving data
- **Remote Capability Awareness** - Policies adjust based on what the remote endpoint knows
- **Enum Validation** - Ensure both endpoints understand enum values and flags
- **Runtime Verification** - Catch version mismatches early during the development phase
- **Detailed Diagnostics** - Violation messages include actionable fixes

## Why ALR over gRPC?

| Aspect                 | gRPC                                   | AtomicLinkRPC (ALR)                               |
| ---------------------- | -------------------------------------- | ------------------------------------------------- |
| **Development Model**  | IDL-first (`.proto` files)             | **C++-first (native types)**                      |
| **Performance**        | High, but limited by HTTP/2 & Protobuf | **Orders of magnitude higher**                    |
| **Ease of Use**        | Steep learning curve, complex APIs     | **Intuitive, feels like local C++**               |
| **API Changes**        | Often leads to breaking changes        | **Resilient and flexible**                        |
| **Dependencies**       | Numerous external dependencies         | **Minimal (Libclang for build, optional OpenSSL)**|
| **Service Discovery**  | Requires external systems (e.g. etcd)  | **Built-in and integrated**                       |

ALR merges the ergonomics of ordinary C++ with a transport and execution engine engineered for scale: schema negotiation for evolution, ambient scoping for simplicity, integrated registry for operability, and raw performance that redefines RPC ceilings.
