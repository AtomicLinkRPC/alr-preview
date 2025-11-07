# AtomicLinkRPC (ALR) - User Guide

This guide provides practical steps and examples to help you get started with AtomicLinkRPC.

## 1. Concepts Cheat‑Sheet
| Term/Symbol | Meaning |
|------|---------|
| `Endpoint` | A live TCP connection + negotiated schema view between a client/service or between two peers. |
| `EndpointCtx` | Ambient binding placing an endpoint “on the stack” for the current thread. |
| `EndpointClass` | Base for user classes (static + instance methods become remotable). |
| `CommonEndpointClass` | Base for symmetrical user classes (static + instance methods become remotable in both directions). |
| `ClassRef<T>` | Distributed smart reference to a (local or remote) endpoint class instance. |
| `Async<T>` / `AsyncVal<T>` | Non‑blocking result placeholder (simple future analogue). |
| `AsyncRef<T>` | Shareable distributed async value enabling remote chaining. |
| `AsyncVoid` | Alias for void signaling an async fire‑and‑forget (still tracked). |
| `AsyncStatus` | Return type for async methods that can fail without returning a value. |
| `AsyncFrame` | Ambient variable for grouping and managing async operations with error handling. |
| `CallTimeout` | Ambient variable for setting timeouts on RPC calls. |
| `RemoteThread` / `RemoteThreadPool` | Scope remote execution affinity / parallelism. |
| `ByteBuffer` | Large block transport with reuse synchronization. |
| `Result<T,E>` | Explicit success/error return. |
| `Status` | Success/error state with optional error code and message. |
| `Evolve<T>` | Evolve policy applied to a parameter, return value or field. |
| `CallCtx` | Inspection for cancellation / timeout inside long operations. |
| Registry | Built‑in service discovery + load balancer + topology metadata. |

---

## 2. Defining a Service

Defining an RPC service in ALR is as simple as creating a C++ class.

1.  **Include the ALR header**: `#include "alr/endpoint.h"`
2.  **Inherit from `alr::EndpointClass` or `alr::CommonEndpointClass`**: This tells ALR codegen to expose the class via RPC.
3.  **Define your methods**: Use static methods for stateless services or instance methods for stateful services (can mix).

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
| Call timeout | `{ CallTimeout timeout(5000); ... }` | Sets 5s timeout for all RPC calls in scope. Stackable. |
| Async frame | `{ AsyncFrame frame; ... }` | Groups async operations; waits on scope exit. |
| Nested frames | `AsyncFrame outer; { AsyncFrame inner; ... }` | Each frame tracks its own operations independently. |

---

## 4. Creating a Service

A service listens for incoming connections and serves requests.

```cpp
// service_main.cpp
#include "alr/endpoint.h"
#include <iostream>

int main()
{
    // Create a connection info instance for a service. It will listen on the address/port specified.
    std::cout << "Service listening on port 55100..." << std::endl;

    return alr::ConnectionInfo()
        .setListenAddress("0.0.0.0:55100") // Default can be set in cfg file
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
    alr::Endpoint ep = alr::ConnectionInfo()
        .setConnectAddress("localhost:55100")
        .connect();

    // Make the async call. It returns immediately.
    alr::AsyncVal<std::string> result = Greeter::SayHelloAsync("async world");

    std::cout << "Async call sent." << std::endl;

    // Use .then() to provide a callback for when the reply arrives.
    result.then([result]() {
        if (result.isSuccess()) {
            std::cout << "Service replied: " << result.value() << std::endl;
        }
    });

    // Optionally wait for the async operation to complete (with optional timeout).
    result.wait();
    return 0;
}
```
> **Note**:  In user-declared interfaces you will write `alr::Async<T>`. In generated code you will see `alr::AsyncVal<T>`, and this is what you should use as the return type when calling into the method. They represent the same concept (an asynchronous value) but are distinct types by design.
> 
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

For peer-to-peer applications where both sides can call each other using the same interface, inherit from `alr::CommonEndpointClass`. ALR codegen generates a remote declaration under the namespace `remote` (can override).

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

ALR provides the `alr::Status` type to represent the result of an operation, conveying either success or failure with optional error codes and messages.

### The Status Type

The `alr::Status` type can represent:

- **Success**: Using `Status::Success` or a default-constructed `Status()`
- **Framework errors**: Using `EndpointState` error codes
- **Application errors**: Using custom enum error codes and/or string messages

```cpp
#include "alr/endpoint.h"

// Define custom error codes
enum class MyErrorCode {
    InvalidInput,
    DatabaseError,
    NotFound
};

// Return Status from a method
alr::Status validateUser(const std::string& username) {
    if (username.empty()) {
        return alr::Status(MyErrorCode::InvalidInput, "Username cannot be empty");
    }
    if (!userExists(username)) {
        return alr::Status(MyErrorCode::NotFound, "User not found");
    }
    return alr::Status::Success;
}

// Check status on the caller side
alr::Status result = SomeService::validateUser("john");
if (result.isError()) {
    std::cerr << "Error: " << result.getFormattedMessage() << std::endl;
    auto code = result.getCode<MyErrorCode>();
    // Handle specific error codes
}
```

### `AsyncStatus` for Fire-and-Forget Methods

For asynchronous methods that don't return a value but may need to signal errors, use `alr::AsyncStatus`. From the caller's perspective, this appears as `AsyncVoid` (fire-and-forget), but if the method returns an error status and is executed within an `alr::AsyncFrame`, the error will be propagated back to the caller.

```cpp
// service.h
class DataProcessor : public alr::EndpointClass {
public:
    static alr::AsyncStatus processData(const std::string& data);
};

// service.cpp
alr::AsyncStatus DataProcessor::processData(const std::string& data) {
    if (data.empty()) {
        return alr::Status(MyErrorCode::InvalidInput, "Data cannot be empty");
    }
    
    // Process the data...
    
    return alr::Status::Success;
}

// client.cpp - using AsyncFrame to catch errors
alr::AsyncFrame frame;
frame.onError([](const alr::Status& err) {
    std::cerr << "Processing error: " << err.getFormattedMessage() << std::endl;
});

DataProcessor::processData("some data");  // Fire-and-forget, but errors are caught
```

---

## 9. Managing Async Operations with `AsyncFrame`

The `alr::AsyncFrame` ambient variable creates a scope for managing multiple asynchronous operations. It provides:

- Automatic waiting for all async operations on scope exit
- Centralized error handling for async operations
- Ability to cancel all operations in the frame

### Basic AsyncFrame Usage

```cpp
#include "alr/endpoint.h"
#include "alr/async.h"

void processMultipleItems() {
    alr::AsyncFrame frame;
    
    // Set up error handler
    frame.onError([](const alr::Status& err) {
        std::cerr << "Async operation failed: " << err.getFormattedMessage() << std::endl;
    });
    
    // Launch multiple async operations
    for (int i = 0; i < 10; i++) {
        MyService::processItemAsync(i);  // Returns AsyncVoid or AsyncStatus
    }
    
    // Frame destructor waits for all operations to complete
}
```

### AsyncFrame with Timeout

You can specify a timeout for waiting on frame completion:

```cpp
void processWithTimeout() {
    alr::AsyncFrame frame(5'000);  // Wait up to 5 seconds on scope exit
    
    frame.onError([](const alr::Status& err) {
        std::cerr << "Error: " << err.getFormattedMessage() << std::endl;
    });
    
    // Launch operations
    MyService::longRunningTask();
    MyService::anotherTask();
    
    // Explicitly wait before scope exit
    if (!frame.wait(3'000)) {
        std::cerr << "Operations timed out!" << std::endl;
        frame.cancel();  // Request cancellation on remote
    }
}
```

### Nested `AsyncFrame`

`alr::AsyncFrame` can be nested, with each tracking its own set of operations:

```cpp
void nestedFrames() {
    alr::AsyncFrame outerFrame;
    outerFrame.onError([](const alr::Status& err) {
        std::cerr << "Outer frame error: " << err.getMessage() << std::endl;
    });
    
    MyService::operation1();
    
    {
        alr::AsyncFrame innerFrame;
        innerFrame.onError([](const alr::Status& err) {
            std::cerr << "Inner frame error: " << err.getMessage() << std::endl;
        });
        
        MyService::operation2();
        MyService::operation3();
        
        // Inner frame waits here
    }
    
    MyService::operation4();
    
    // Outer frame waits here
}
```

### Service-Side Error Reporting

Methods executing within an async frame context can report errors back to the caller. There are two equivalent patterns for this:

#### Pattern 1: Using `AsyncVoid` with Explicit Error Calls

```cpp
// service.cpp
alr::AsyncVoid MyService::processItemAsync(int itemId) {
    // Check if the caller requested cancellation
    if (alr::CallCtx::isAsyncFrameCanceled()) {
        alr::CallCtx::sendAsyncFrameError(
            alr::Status(MyErrorCode::Canceled, "Operation was canceled"));
        return;
    }
    
    // Check for timeout
    if (alr::CallCtx::isTimedOut()) {
        alr::CallCtx::sendAsyncFrameError(
            alr::Status(MyErrorCode::Timeout, "Operation timed out"));
        return;
    }
    
    // Process the item...
    if (errorCondition) {
        alr::CallCtx::sendAsyncFrameError(
            alr::Status(MyErrorCode::ProcessingError, "Failed to process item"));
        return;
    }
    
    // Success - no action needed
}
```

#### Pattern 2: Using `AsyncStatus` with Return Values

Conceptually equivalent to Pattern 1, but uses return values instead of explicit error calls:

```cpp
// service.cpp
alr::AsyncStatus MyService::processItemAsync(int itemId) {
    // Check if the caller requested cancellation
    if (alr::CallCtx::isAsyncFrameCanceled()) {
        return alr::Status(MyErrorCode::Canceled, "Operation was canceled");
    }
    
    // Check for timeout
    if (alr::CallCtx::isTimedOut()) {
        return alr::Status(MyErrorCode::Timeout, "Operation timed out");
    }
    
    // Process the item...
    if (errorCondition) {
        return alr::Status(MyErrorCode::ProcessingError, "Failed to process item");
    }
    
    // Success - return Status::Success (or default-constructed Status())
    return alr::Status::Success;
}
```

**Key Points:**

- Both patterns are functionally equivalent from the caller's perspective
- The caller sees both methods as returning `AsyncVoid` (fire-and-forget)
- When `AsyncStatus` returns `Status::Success`, no error is propagated (behaves like `AsyncVoid`)
- When `AsyncStatus` returns an error status, it automatically propagates to the caller's `AsyncFrame` (equivalent to calling `CallCtx::sendAsyncFrameError()`)
- Choose Pattern 1 for methods that logically don't have a return value concept
- Choose Pattern 2 for cleaner code when you have multiple early-return error conditions


---

## 10. Setting Call Timeouts with `CallTimeout`

The `alr::CallTimeout` ambient variable sets a timeout for RPC calls made within its scope. This applies to both synchronous and asynchronous calls.

### Basic `CallTimeout` Usage

```cpp
#include "alr/endpoint.h"

void makeCallWithTimeout() {
    alr::CallTimeout timeout(5'000);  // 5 second timeout
    
    // This call will timeout if it takes longer than 5 seconds
    std::string result = MyService::longOperation();
    
    if (timeout.isSyncRequestTimedOut()) {
        std::cerr << "The operation timed out!" << std::endl;
        // result is in an unspecified state
    }
}
```

### `CallTimeout` with Async Operations

When used with async operations and `alr::AsyncFrame`, timeouts can be detected on the service side:

```cpp
// client.cpp
void asyncWithTimeout() {
    alr::AsyncFrame frame;
    alr::CallTimeout timeout(3'000);  // 3 second timeout
    
    frame.onError([](const alr::Status& err) {
        std::cerr << "Operation failed: " << err.getFormattedMessage() << std::endl;
    });
    
    MyService::longAsyncOperation();

    // Do some other work
    std::this_thread::sleep_for(std::chrono::milliseconds(5'000));
}

// service.cpp
alr::AsyncVoid MyService::longAsyncOperation() {
    auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(10'000);
    
    while (std::chrono::steady_clock::now() < deadline) {
        // Periodically check for timeout
        if (alr::CallCtx::isTimedOut()) {
            alr::CallCtx::sendAsyncFrameError(
                alr::Status("Operation exceeded timeout"));
            return;
        }
        
        // Do some work
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}
```

### Nested `CallTimeout` (Stackable)

`alr::CallTimeout` is stackable, allowing nested scopes with different timeouts:

```cpp
void nestedTimeouts() {
    alr::CallTimeout outerTimeout(10'000);  // 10 seconds
    
    MyService::operation1();  // Has 10 second timeout
    
    {
        alr::CallTimeout innerTimeout(2'000);  // 2 seconds (overrides outer)
        MyService::quickOperation();  // Has 2 second timeout
    }
    
    MyService::operation2();  // Back to 10 second timeout
}
```

---

## 11. API Evolution and Versioning

ALR supports evolving your API over time without breaking changes to existing clients or services. This is crucial in distributed systems where you can't easily coordinate updates.

### API Versioning Basics

- **Major.Minor.Patch** versioning scheme (SemVer)
- Incompatible changes increase the **major** version
- Backward-compatible changes increase the **minor** version
- Patches are for backward-compatible bug fixes

### Evolving Methods

- **Additive Changes**: Add new methods or parameters. Old clients ignore unknown methods.
- **Parameter Changes**: Safe to add new parameters. Old clients using the old method signature continue to work.
- **Return Type Changes**: The method can still be invoked, however the return value will not be transmitted, and the remote will read their local default. Care should be taken since in most cases, this would be fatal unless the return value is not really required.
- **Evolve Policies**: Use `alr::Evolve<T>` to add optional access policies to parameters, returns or fields.

### Using API Evolution Policies

ALR provides the `alr::Evolve<T, WritePolicy, ReadPolicy>` template wrapper to enforce runtime access patterns on struct fields, method parameters, and return values. This helps manage API evolution by verifying correct usage based on remote endpoint capabilities.

#### The Evolve Template Wrapper

The core building block is:
```cpp
template<class T, EvolvePolicy WritePolicy = EvolvePolicy::Required,
                  EvolvePolicy ReadPolicy = EvolvePolicy::Optional>
class Evolve { /* ... */ };
```

- Wraps a value of type `T`
- Behaves like `T` for most operations (implicit conversions)
- Tracks whether the value has been written or read
- Enforces policies at runtime based on remote capabilities
- Can be applied to fields, parameters, and return values

See [Evolve Policies](./evolve-policies.md) for more details.

#### Basic Usage on Struct Fields

```cpp
struct Product
{
    std::string name;                              // No policy
    alr::EvolveWriteRequired<double> price;        // Must write when remote knows
    alr::EvolveOptional<std::string> description;  // Conditional on remote
    alr::EvolveIfKnown<int> stockCount;            // Require both read/write if known
};
```

#### Usage on Method Parameters and Return Values

```cpp
class MyService : public alr::EndpointClass {
public:
    // Policy on parameter
    void processData(alr::EvolveStrict<int> importantValue);
    
    // Policy on return value
    alr::EvolveStrict<StatusFlags> getStatus();
};
```

#### Available Policy Modes

| Policy Mode | Write Behavior | Read Behavior |
|-------------|----------------|---------------|
| `Optional` | Optional to write | Optional to read |
| `RequiredIfKnown` | Must write if remote knows | Must read if remote knows |
| `ProhibitedIfUnknown` | Prohibited if remote doesn't know | Prohibited if remote doesn't know |
| `FollowsKnown` | Must write if known, must be unset if unknown | Must read if known, prohibited if unknown |
| `Required` | Always required | Always required |

#### Enum Policy Modifiers

For enum types, add validation:

| Enum Policy Mode | Validation |
|-------------|----------------|
| `_EnumValue` | Remote must know the exact enum value |
| `_EnumStrict` | Remote must know the value and matching name |
| `_EnumFlags` | Remote must know all set flag bits |
| `_EnumFlagsStrict` | Remote must know all flags with matching names |

#### Accessing Values

**Implicit read (marks as read):**
```cpp
Product product;
double price = product.price;  // Implicit conversion, marks as read
```

**Explicit write (marks as written):**
```cpp
product.price = 29.99;  // Direct assignment, marks as written

// Or cast to reference:
auto& priceRef = static_cast<double&>(product.price);
priceRef = 29.99;
```

**Using helper functions:**
```cpp
#include <alr/evolve.h>

// Works for both plain T and Evolve<T>
const auto& price = evolve::read(product.price);
auto& priceRef = evolve::write(product.price);
priceRef = 29.99;
```

#### Remote Capability Checks

Use generated `remoteCaps` functions to check what the remote endpoint supports:

```cpp
// Check if remote has a field
if (remoteCaps::hasStructField::Product::stockCount()) {
    product.stockCount = 100;  // Safe to set
}

// Check if remote has an enum value
if (remoteCaps::hasEnum::StatusFlags(EnumCaps::Value, StatusFlags::Active)) {
    status = StatusFlags::Active;
}

// Check if remote has all flag bits
StatusFlags flags = StatusFlags::Active | StatusFlags::Premium;
if (remoteCaps::hasEnum::StatusFlags(EnumCaps::Flags, flags)) {
    status = flags;  // Remote knows all flags
}
```

#### Policy Violation Callbacks

Override callbacks to handle violations:

```cpp
class MyCallbacks : public alr::Callbacks {
public:
    void evolvePolicyViolated(
        EndpointWeakRef ep,
        const EvolvePolicyViolation& violation) override {
        
        // Log, throw, or handle gracefully
        std::cerr << "Policy violation: " << violation.detail << std::endl;
    }
};

// Register callbacks
ep.setCallbacks(std::make_shared<MyCallbacks>());
```

The `EvolvePolicyViolation` structure contains:

- `detail` - Detailed description with fix suggestions
- `methodName`, `structName`, `valueName`, `valueType`
- `capsCheck` - Suggested capability check code
- `writePolicy`, `readPolicy` - Policies in effect
- `remoteHas`, `isWritten`, `isRead` - State information

#### Best Practices

1. **Use policies selectively** - Not every field needs a policy
2. **Apply to leaf fields** - Prefer policies on leaf fields over nested structs
3. **Use type aliases** - Define version-based aliases for maintainability
4. **Check remote capabilities** - Use `remoteCaps` when conditionally setting values
5. **Handle violations** - Implement custom callbacks for production monitoring
