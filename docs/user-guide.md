# AtomicLinkRPC (ALR) - User Guide

This guide provides practical steps and examples to help you get started with AtomicLinkRPC.

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [AtomicLinkRPC (ALR) - User Guide](#atomiclinkrpc-alr---user-guide)
  - [1. Concepts Cheat‑Sheet](#1-concepts-cheatsheet)
  - [2. Defining a Service](#2-defining-a-service)
    - [Example: A Simple Greeter Service](#example-a-simple-greeter-service)
  - [3. Ambient Context Patterns](#3-ambient-context-patterns)
  - [4. Creating a Service](#4-creating-a-service)
  - [5. Creating a Client and Making Calls](#5-creating-a-client-and-making-calls)
    - [Synchronous Call](#synchronous-call)
    - [Asynchronous Call](#asynchronous-call)
  - [6. Stateful Services](#6-stateful-services)
  - [7. Symmetric Services with `CommonEndpointClass`](#7-symmetric-services-with-commonendpointclass)

<!-- /code_chunk_output -->


---

## 1. Concepts Cheat‑Sheet
| Term | Meaning |
|------|---------|
| Endpoint | A live TCP connection + negotiated schema view between a client/service or between two peers. |
| EndpointCtx | Ambient binding placing an endpoint “on the stack” for the current thread. |
| EndpointClass | Base for user classes (static + instance methods become remotable). |
| CommonEndpointClass | Base for symmetrical user classes (static + instance methods become remotable in both directions). |
| ClassRef<T> | Distributed smart reference to a (local or remote) endpoint class instance. |
| Async<T> / AsyncVal<T> | Non‑blocking result placeholder (simple future analogue). |
| AsyncRef<T> | Shareable distributed async value enabling remote chaining. |
| AsyncVoid | Alias for void signaling an async fire‑and‑forget (still tracked). |
| RemoteThread / RemoteThreadPool | Scope remote execution affinity / parallelism. |
| ByteBuffer | Large block transport with reuse synchronization. |
| Result<T,E> | Explicit success/error return. |
| CallCtx | Inspection for cancellation / timeout inside long operations. |
| Registry | Built‑in service discovery + load balancer + topology metadata. |

---

## 2. Defining a Service

Defining an RPC service in ALR is as simple as creating a C++ class.

1.  **Include the ALR header**: `#include "alr/endpoint.h"`
2.  **Inherit from `alr::EndpointClass`**: This tells the ALR compiler to expose the class via RPC.
3.  **Define your methods**: Use static methods for stateless services or instance methods for stateful services.

### Example: A Simple Greeter Service

```cpp
// greeter.h
#include "alr/endpoint.h"
#include <string>

// A simple, stateless service with one static method.
class Greeter : public alr::EndpointClass
{
public:
    static std::string SayHello(const std::string& name);
};

// greeter.cpp (Your implementation)
#include "greeter.h"

std::string Greeter::SayHello(const std::string& name)
{
    return "Hello, " + name;
}
```

---

## 3. Ambient Context Patterns
| Pattern | Example | Notes |
|---------|---------|-------|
| Simple connect | `Endpoint ep = ConnectionInfo.connect();` | Puts context on current thread automatically. |
| Worker threads | `EndpointCtx ctx(ep);` inside thread entry | Threads inherit ability to issue RPCs. |
| Scoped remote pool | `{ RemoteThreadPool pool(8); ... }` | Fan‑out parallelism; `pool.wait()` to join. |
| Dedicated remote thread | `{ RemoteThread rt; ... }` | All calls handled by single remote thread. |

---

## 4. Creating a Service

A service listens for incoming connections and serves requests.

```cpp
// service_main.cpp
#include "alr/endpoint.h"
#include <iostream>
#include <promise>

int main()
{
    // Create a connection info instance for a service. It will listen on the address/port specified.
    std::cout << "Service listening on port 55100..." << std::endl;

    return alr::ConnectionInfo()
        .setListenAddress("0.0.0.0:55100")
        .listen();
}
```

---

## 5. Creating a Client and Making Calls

The client connects to the service and invokes remote methods.

### Synchronous Call

A synchronous call blocks until the reply is received.

```cpp
// client_main.cpp
#include "alr/endpoint.h"
#include "greeter.h" 
#include "greeter_gen.h" // Include the generated client-side stub
#include <iostream>

int main()
{
    // Connect to the service. The current thread is now implicitly 
    // associated with this endpoint.
    alr::Endpoint ep = alr::ConnectionInfo()
        .setConnectAddress("localhost:55100")
        .connect();

    // Call the remote method just like a local static function.
    std::string reply = Greeter::SayHello("world");
    std::cout << "Service replied: " << reply << std::endl;
    return 0;
}
```

### Asynchronous Call

For non-blocking calls, define your method to return `alr::Async<T>` (or `alr::AsyncVoid`).

```cpp
// greeter.h
#include "alr/endpoint.h"
#include "alr/async.h" // Include for alr::Async
#include <string>

class Greeter : public alr::EndpointClass
{
public:
    static alr::Async<std::string> SayHelloAsync(const std::string& name);
};

// client_main.cpp
#include "alr/endpoint.h"
#include "alr/async.h"
#include "greeter_gen.h"
#include <iostream>

int main()
{
    alr::Endpoint ep = alr::ConnectionInfo().setConnectAddress("localhost:55100").connect();

    // Make the async call. It returns immediately.
    alr::Async<std::string> result = Greeter::SayHelloAsync("async world");

    std::cout << "Async call sent." << std::endl;

    // Use .then() to provide a callback for when the reply arrives.
    result.then([result]() {
        if (res.isSuccess()) {
            std::cout << "Service replied: " << result.value() << std::endl;
        }
    });

    // Optionally wait for the async operation to complete (with optional timeout).
    result.wait();
    return 0;
}
```

---

## 6. Stateful Services

To create stateful services, use instance methods and create objects of your service class on the client side. ALR will create a corresponding remote instance.

```cpp
// service.h
#include "alr/endpoint.h"

class Counter : public alr::EndpointClass
{
public:
    Counter(int initialValue = 0) : _count(initialValue) {}

    void increment() { _count++; }
    int getCount() const { return _count; }

private:
    int _count;
};

// client.cpp
#include "alr/endpoint.h"
#include "counter_gen.h"
#include <iostream>

int main()
{
    alr::Endpoint ep = alr::ConnectionInfo()
        .setConnectAddress("localhost:55100")
        .connect();

    // This creates a *remote* instance of the Counter class on the service.
    Counter myCounter(10);

    myCounter.increment();
    myCounter.increment();

    int finalCount = myCounter.getCount(); // RPC call to get the value

    std::cout << "Final count on service: " << finalCount << std::endl; // Outputs 12

    // The remote instance is automatically destroyed when myCounter goes out of scope.
    return 0;
}
```
ALR's `alr::ClassRef<T>` system ensures that the remote object's lifetime is managed automatically.

---

## 7. Symmetric Services with `CommonEndpointClass`

For peer-to-peer applications where both sides can call each other, inherit from `alr::CommonEndpointClass`. The ALR compiler generates a remote declaration under a `remote` namespace.

```cpp
// chat.h
#include "alr/endpoint.h"
#include <string>
#include <iostream>

class Chat : public alr::CommonEndpointClass 
{
public:
  void post(const std::string& msg) { 
      std::cout << "Received: " << msg << std::endl; 
  }
};

// p2p_main.cpp
#include "alr/endpoint.h"
#include "chat_gen.h"

int main() {
  // Simplified connection logic
  alr::Endpoint ep = alr::ConnectionInfo().setConnectAddress("...").connect();
  
  Chat local;        // Lives here; callable from the peer
  remote::Chat peer; // Proxy to Chat living on the remote endpoint

  local.post("Hi from me"); // Prints locally
  peer.post("Hi from you"); // Prints on the remote peer
}
```
