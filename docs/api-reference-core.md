# API Reference - Core

This document covers the essential ALR API components for building RPC applications. For advanced features like registry and discovery, see [API Reference - Advanced](./api-reference-advanced.md). For information on evolve policies, see [Evolve Policies](./evolve-policies.md).

## Table of Contents

- [Supported Native C++ Types](#supported-native-c-types)
- [Core Classes and Lifecycle](#core-classes-and-lifecycle)
- [Ambient Context Variables](#ambient-context-variables)
- [Asynchronous and Data Types](#asynchronous-and-data-types)
- [Return Modes](#return-modes)
- [Exception Handling](#exception-handling)

---

## Supported Native C++ Types

ALR natively supports a wide range of C++ types for use in your remote methods and structs:

-   **Integer Types**: All variations, such as `int`, `unsigned long`, `int8_t`, `uint64_t`, etc.
-   **Floating-Point Types**: `float`, `double`.
-   **Boolean**: `bool`.
-   **Enums**: Standard C++ `enum` and `enum class`.
-   **Strings**: `std::string`.
-   **Containers**:
 
    -   `std::vector<T>`
    -   `std::map<K, V>`
    -   `std::set<T>`
    -   `std::shared_ptr<T>`
    -   `std::optional<T>`

-   **Arrays**:

    - C-style arrays of any other supported type (e.g., `MyStruct[10]`).
    - `std::array<T>` of any other supported type (e.g., `std::array<MyStruct, 10>`).

-   **Structs**: User-defined `struct`s containing any combination of supported types.

---

## Core Classes and Lifecycle

### `alr::ConnectionInfo`
A fluent builder class used to configure and create client, service or registry endpoints.

**Usage:**
```cpp
// Create a service that listens on a specific address
// service.cpp
alr::ConnectionInfo()
    .setListenAddress("0.0.0.0:55100")
    .setOpenSsl(true, "cert.pem", "key.pem") // Enable TLS
    .listen();

// Create a client that connects to a service
// client.cpp
alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("myservice.com:55100")
    .connect();

// Register a backend service with a registry
// service.cpp
alr::ConnectionInfo()
    .setServiceName("my-service")
    .setRegistryAddress("registry.host:5500")
    .setPublicAddress("external.host:55101")
    .setListenAddress("0.0.0.0:55101")
    .listen();
```

### `alr::Endpoint`
Represents a network connection to a service/peer. It is the main handle for interacting with a connection.

**Key Methods:**

-   `bool isConnected()`: Checks if the connection is live.
-   `void disconnect()`: Closes the connection.
-   `bool waitForIdle(uint32_t timeoutMs)`: Waits for all outstanding operations to complete.
-   `EndpointStats getLocalStats(bool reset)`: Retrieves local performance statistics.
-   `EndpointState getState()`: Returns a non-`Ok` state enum value if the endpoint is in a faulted state.
-   `std::string getError()`: Returns the last error message if the endpoint is in a faulted state.

### `alr::EndpointClass`
The base class for any user-defined class that you want to expose via RPC. Inheriting from this class makes its public static and instance methods and any nested public base classes available to ALR codegen.

**Usage:**
```cpp
// service.h
#include "alr/endpoint.h"

class MyService : public alr::EndpointClass
{
    // ... methods ...
};
```

### `alr::CommonEndpointClass`
A base class for symmetric services where both peers have the same interface and can call each other. In this case ALR codegen generates a proxy class in the `remote` namespace to interact with the peer's instance.

**Usage:**
```cpp
// common.h
class ChatPeer : public alr::CommonEndpointClass {
public:
    void ReceiveMessage(const std::string& msg);
};

// In application code:
// common.cpp
ChatPeer localPeer;      // Instance on this endpoint
remote::ChatPeer remote; // Proxy to the instance on the other endpoint

localPeer.ReceiveMessage("Printed locally");
remote.ReceiveMessage("Sent to the remote peer");
```

---

## Ambient Context Variables

These RAII types manage the RPC context for a thread.

### `alr::EndpointCtx`
A stack-based object that sets the active `alr::Endpoint` for the current scope. This is essential for multi-threaded applications where different threads need to use different connections.

**Usage:**
```cpp
// client.cpp
alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("myservice.com:55100")
    .connect();
// The connecting thread implicitly has an EndpointCtx.

// For a new thread:
std::thread worker([ep]() {
    alr::EndpointCtx ctx(ep); // Set the context for this thread
    MyService::doSomething(); // This call will use 'ep'
});
```

### `alr::RemoteThreadPool`
Distributes RPCs from the current scope across a pool of remote threads, enabling parallel execution on the service.

**Usage:**
```cpp
// client.cpp
alr::RemoteThreadPool pool(8); // Use up to 8 threads on the remote side.
MyService::doAsyncTask1();
MyService::doAsyncTask2();
// ...
pool.wait(); // Block until all tasks in the pool are done.
```
> Note: The remote endpoint will not create 8 threads up-front. Instead, *up to* 8 threads from the remote endpoint's global thread pool can work on tasks initiated from this local thread.

### `alr::RemoteThread`
Guarantees that all RPCs within its scope are processed by a single, dedicated thread on the remote endpoint, ensuring sequential execution.

**Usage:**
```cpp
// client.cpp
alr::RemoteThread remoteThread; // Subsequent calls are serialized on one dedicated remote thread.
MyService::doTaskA(); // Guaranteed to execute before doTaskB
MyService::doTaskB();
```

### `alr::CallCtx`
Provides information about the currently executing RPC from within a service method implementation. This is useful for implementing long-running tasks that can be canceled or timed out.

**Key Methods:**

-   `static bool isCanceled()`: Returns true if the client has requested cancellation.
-   `static bool isTimedOut()`: Returns true if the call has exceeded its timeout (set via `CallTimeout`).
-   `static bool isAsyncFrameCanceled()`: Returns true if the async frame has been canceled.
-   `static bool isAsyncFrameOk()`: Returns true if the async frame is still active and not canceled.
-   `static void sendAsyncFrameError(const Status& error)`: Sends an error back to the caller's async frame.

**Usage:**
```cpp
void MyService::longRunningTask() {
    while (work_to_do) {
        if (alr::CallCtx::isCanceled()) {
            // Clean up and return
            break;
        }
        if (alr::CallCtx::isTimedOut()) {
            // Timeout exceeded
            break;
        }
        /* ... do work ... */
    }
}
```

### `alr::CallTimeout`
An ambient variable that sets a timeout for RPC calls made within its scope. Applies to both synchronous and asynchronous calls. Stackable - nested instances restore the previous timeout on scope exit.

**Constructor:**
```cpp
CallTimeout(uint32_t timeoutMs);  // 0 means no timeout
```

**Key Methods:**

-   `bool isSyncRequestTimedOut() const`: Check if the last synchronous request timed out.
-   `static uint32_t getTimeoutMs()`: Get the timeout value for the current thread.

**Usage:**
```cpp
// client.cpp
alr::CallTimeout timeout(5000);  // 5 second timeout

std::string result = MyService::longOperation();

if (timeout.isSyncRequestTimedOut()) {
    std::cerr << "Operation timed out!" << std::endl;
    // result is in an unspecified state
}
```

**Service-side timeout checking:**
```cpp
// service.cpp
alr::AsyncVoid MyService::longOperation() {
    auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(10);
    
    while (std::chrono::steady_clock::now() < deadline) {
        if (alr::CallCtx::isTimedOut()) {
            // Client's timeout was exceeded
            alr::CallCtx::sendAsyncFrameError(
                alr::Status("Operation timed out"));
            return;
        }
        // Do work...
    }
}
```

### `alr::AsyncFrame`
An ambient variable that creates a scope for managing multiple asynchronous operations. Automatically waits for all async operations to complete on scope exit. Provides centralized error handling and cancellation support.

**Constructor:**
```cpp
AsyncFrame(uint32_t scopeExitTimeoutMs = 0);  // Optional timeout on scope exit
```

**Key Methods:**

-   `bool wait(uint32_t timeoutMs = 0)`: Wait for all operations in the frame to complete.
-   `bool isSuccess() const`: Returns true if all operations completed successfully.
-   `bool isError() const`: Returns true if any operation resulted in an error.
-   `StatusType getStatusType() const`: Get the type of status (Success, FrameworkError, ApplicationError).
-   `const std::string& getErrorMessage() const`: Get the error message if any.
-   `const Status& getStatus() const`: Get the complete status object.
-   `void cancel()`: Request cancellation of all operations in the frame.
-   `void detach()`: Detach the frame so it won't wait on scope exit.
-   `void onError(std::function<void(const Status&)>&& cb)`: Set a callback for error notifications.

**Usage:**
```cpp
// client.cpp
void processMultipleItems() {
    alr::AsyncFrame frame;
    
    frame.onError([](const alr::Status& err) {
        std::cerr << "Error: " << err.getFormattedMessage() << std::endl;
    });
    
    // Launch multiple async operations
    for (int i = 0; i < 10; i++) {
        MyService::processItemAsync(i);
    }
    
    // Frame destructor waits for all operations
}
```

---

## Asynchronous and Data Types

### `alr::Async<T>`
A future-like return type for asynchronous methods. The client receives an `alr::AsyncVal<T>` object immediately after an async call.

-   **`T`**: The type of the value that will be returned. Use `alr::AsyncVoid` for methods that don't return a value.
-   **`then(callback)`**: Attaches a callback to be executed upon completion.
-   **`value()`**: Blocks and waits for the result, error or timeout, with optional timeout.
-   **`wait()`**: Blocks until the operation completes, errors or times out, with optional timeout.
-   **`isSuccess()` / `isCanceled()` / `isTimedOut()`**: Check the status of the operation.
-   **`cancel()`**: Attempts to cancel the remote operation.

### `alr::AsyncRef<T>`
An advanced version of `alr::Async<T>` that represents a remote future. It can be passed in subsequent RPC calls, enabling remote function chaining without sending intermediate results back to the originator. The value is only transferred over the network if/when `.value()` is called.

### `alr::Status`
Represents the success or failure state of an operation. Can hold framework errors (using `EndpointState` codes) or application-defined errors (using custom enum codes).

**Constructor Patterns:**
```cpp
// Success
alr::Status status = alr::Status::Success;

// Framework error
alr::Status status(alr::EndpointState::Timeout, "Operation timed out");

// Application error with enum code and message
enum class MyError { InvalidInput, NotFound };
alr::Status status(MyError::InvalidInput, "Username cannot be empty");

// Application error with just enum code
alr::Status status(MyError::NotFound);  // explicit constructor

// Application error with just message
alr::Status status("Generic error message");  // explicit constructor
```

**Key Methods:**

-   `bool isSuccess() const`: Returns true if the status represents success.
-   `bool isError() const`: Returns true if the status represents an error.
-   `StatusType getType() const`: Returns the type of status (Success, FrameworkError, ApplicationError).
-   `const std::string& getMessage() const`: Returns the error message.
-   `std::string getFormattedMessage() const`: Returns a formatted error message with type and code.
-   `template<typename T> T getCode() const`: Returns the error code cast to the specified enum type.

### `alr::AsyncStatus`
An alias for `alr::Status` when used as a return type for asynchronous methods. From the caller's perspective, methods returning `AsyncStatus` behave like `AsyncVoid` (fire-and-forget), but if the method returns an error status within an `AsyncFrame` context, the error is propagated back to the caller's error handler.

**Usage:**
```cpp
// service.h
class DataService : public alr::EndpointClass {
public:
    static alr::AsyncStatus processData(const std::string& data);
};

// service.cpp
alr::AsyncStatus DataService::processData(const std::string& data) {
    if (data.empty()) {
        return alr::Status(MyError::InvalidInput, "Data cannot be empty");
    }
    // Process data...
    return alr::Status::Success;
}

// client.cpp - errors caught by AsyncFrame
alr::AsyncFrame frame;
frame.onError([](const alr::Status& err) {
    std::cerr << "Processing failed: " << err.getFormattedMessage() << std::endl;
});

DataService::processData("some data");  // Fire-and-forget, but errors are caught
```

### `alr::Result<T, E>`
A return type for functions that can return either a success value or an error. It is a sum type that holds either a `T` or an `E`. Can be combined with `alr::Async<T>` or `alr::AsyncRef<T>`.

-   **`T`**: The success value type.
-   **`E`**: The error value type.
-   **`isSuccess()` / `isError()`**: Check the result state.
-   **`value()`**: Returns the success value.
-   **`error()`**: Returns the error value.

### `alr::ClassRef<T>`
A smart reference that manages the lifetime of local and remote objects. It can be passed as a parameter or a struct field, and either side can call methods on the referenced object, regardless of where it resides.

-   **`T`**: The type of the remote object, which must derive from `alr::EndpointClass` or `alr::CommonEndpointClass`.

### `alr::ByteBuffer`
An optimized container for sending and receiving large blocks of binary data. It is more efficient than `std::vector<uint8_t>` for large payloads and is synchronized with the async framework, allowing for safe reuse after an async operation completes. For use with async calls, it can be waited on with an optional timeout.

---

## Return Modes

ALR supports several return modes for RPC methods, each optimized for different use cases. Understanding these modes is essential for designing efficient APIs and managing API evolution.

### Overview of Return Modes

**Synchronous Modes:**

- **`void`**: Blocks until completion, no return value
- **`T`**: Blocks until completion, returns value of type `T`
- **`Result<T, E>`**: Blocks until completion, returns either success (`T`) or error (`E`
- **`Status`**: Blocks until completion, returns success/failure status with optional error details

**Asynchronous Modes:**

- **`AsyncVoid`**: Fire-and-forget, no return value (tracked by framework)
- **`AsyncStatus`**: Fire-and-forget with error reporting via `AsyncFrame` on failure only
- **`Async<T>`**: Returns immediately with `AsyncVal<T>`, allows callbacks and waiting
- **`Async<Result<T, E>>`**: Combines async execution with explicit error handling
- **`AsyncRef<T>`**: Advanced mode for remote future chaining without intermediate data transfer

### Basic Usage Examples

```cpp
// service.h
class DataService : public alr::EndpointClass {
public:
    // Synchronous modes
    static void logMessage(const std::string& msg);                    // void
    static int getCount();                                             // T
    static Result<User, ErrorCode> findUser(const std::string& id);   // Result<T, E>
    static Status updateRecord(const Record& rec);                     // Status
    
    // Asynchronous modes
    static AsyncVoid notifyAsync(const std::string& event);            // AsyncVoid
    static AsyncStatus processAsync(const Data& data);                 // AsyncStatus
    static Async<std::string> fetchDataAsync();                        // Async<T>
    static Async<Result<User, ErrorCode>> getUserAsync(int id);       // Async<Result<T, E>>
};
```

### Return Mode Categories

ALR categorizes return modes into five main groups. **Types can evolve within a category but not between categories:**

1. **Void Mode**: `void`
2. **Value Mode**: `T`, `Result<T, E>`, `ClassRef<T>`
3. **Async Void Mode**: `AsyncVoid`, `AsyncStatus`
4. **Async Value Mode**: `Async<T>`, `Async<Result<T, E>>`, `Async<ClassRef<T>>`
5. **Async Ref Mode**: `AsyncRef<T>`

> **Important:** Changing a method's return mode category (e.g., from `T` to `Async<T>`) breaks compatibility with older clients/services. The caps check for that method will return `false` on both endpoints.

For detailed information on return mode evolution and best practices, see [Return Modes and API Evolution](./api-reference-advanced.md#return-modes-and-api-evolution).

---

## Exception Handling

ALR uses exceptions sparingly and as a last resort. **Exceptions are not used for normal error propagation** - instead, use `Result<T, E>`, `Status`, or `AsyncStatus` for explicit, structured error handling. Exceptions in ALR typically indicate programming errors, configuration issues, or catastrophic failures rather than expected error conditions.

**When Exceptions Are Thrown:**

Exceptions are thrown in situations where:

- An API is used incorrectly (e.g., calling RPC without an `EndpointCtx`)
- Configuration is invalid or inconsistent
- A capability check was missed and a method is called that the remote doesn't support
- The connection has failed in an unrecoverable way
- Internal framework invariants are violated

**Exception Reference:**

| Exception                     | When It's Thrown                                                              |
| ----------------------------- | ----------------------------------------------------------------------------- |
| `NoEndpointOnStackError`      | An RPC is made from a thread without an active `alr::EndpointCtx`.            |
| `WrongEndpointOnStackError`   | An operation is performed on an endpoint different from the one on the stack. |
| `UndefinedRemoteMethodError`  | A call is made to a method that doesn't exist on the remote (version skew).   |
| `InvalidMessageError`         | A received message is malformed or inconsistent (should not occur normally).  |
| `InvalidConfigurationError`   | The `ConnectionInfo` settings are invalid or inconsistent.                    |
| `NoBackendServiceError`       | The registry could not provide a backend for the requested service.           |
| `ConnectionClosedError`       | The underlying connection was closed unexpectedly.                            |
| `GenericError`                | A general framework error occurred.                                           |

**Best Practices:**

✅ **DO:**

- Use `Result<T, E>` for operations with expected error cases
- Use `Status` or `AsyncStatus` for operations that can fail
- Check capabilities with `remoteCaps::hasMethod::MethodName()` when version skew is possible (e.g., rolling deployments)
- Handle connection state changes via callbacks rather than catching exceptions

❌ **DON'T:**

- Use exceptions for normal control flow
- Rely on catching exceptions for expected error conditions
- Call methods without checking capabilities when deploying mixed versions

**Example - Capability Checks for Mixed Versions:**

```cpp
// Only needed when version skew is possible (e.g., during rolling deployments)
if (remoteCaps::hasMethod::newFeature()) {
    Service::newFeature();
} else {
    // Fallback for older services
    Service::legacyApproach();
}

// For homogeneous deployments, just call directly - feels like local C++
Service::standardMethod();  // No capability check needed
```

**Example - Proper Error Handling:**

```cpp
// Good: Use Result for expected errors
enum class UserError { NotFound, InvalidId };
Result<User, UserError> result = UserService::getUser(42);
if (result.isError()) {
    // Handle error gracefully
}

// Bad: Don't use exceptions for normal errors
try {
    User user = UserService::getUser(42);  // Throws on not found - bad design
} catch (...) {
    // Exception-based error handling - avoid this pattern
}
```
