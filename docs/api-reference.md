# AtomicLinkRPC (ALR) - API Reference

This document provides a reference for the key classes and types in the ALR framework.

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [AtomicLinkRPC (ALR) - API Reference](#atomiclinkrpc-alr---api-reference)
  - [Supported Native C++ Types](#supported-native-c-types)
  - [Core Classes & Lifecycle](#core-classes--lifecycle)
    - [`alr::ConnectionInfo`](#alrconnectioninfo)
    - [`alr::Endpoint`](#alrendpoint)
    - [`alr::EndpointClass`](#alrendpointclass)
    - [`alr::CommonEndpointClass`](#alrcommonendpointclass)
  - [Ambient Context Variables](#ambient-context-variables)
    - [`alr::EndpointCtx`](#alrendpointctx)
    - [`alr::RemoteThreadPool`](#alrremotethreadpool)
    - [`alr::RemoteThread`](#alrremotethread)
    - [`alr::CallCtx`](#alrcallctx)
  - [Asynchronous and Data Types](#asynchronous-and-data-types)
    - [`alr::Async<T>`](#alrasynct)
    - [`alr::AsyncRef<T>`](#alrasyncreft)
    - [`alr::Result<T, E>`](#alrresultt-e)
    - [`alr::ClassRef<T>`](#alrclassreft)
    - [`alr::ByteBuffer`](#alrbytebuffer)
  - [Exception Handling](#exception-handling)

<!-- /code_chunk_output -->

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
-   **Arrays**: C-style arrays of any other supported type (e.g., `int[10]`).
-   **Structs**: User-defined `struct`s containing any combination of supported types.

---

## Core Classes & Lifecycle

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
    .setRegistryAddress("registry.internal:5500")
    .setPublicAddress("external.host:55100")
    .setListenAddress("0.0.0.0:12345")
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
The base class for any user-defined class that you want to expose via RPC. Inheriting from this class makes its public static and instance methods and any nested public base classes available to the ALR compiler.

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
A base class for symmetric services where both peers have the same interface and can call each other. In this case the ALR compiler generates a proxy class in the `remote` namespace to interact with the peer's instance.

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
Provides information about the currently executing RPC from within a service method implementation. This is useful for implementing long-running tasks that can be canceled.

**Usage:**
```cpp
void MyService::longRunningTask() {
    while (work_to_do) {
        if (alr::CallCtx::isCanceled()) {
            // Clean up and return
            break;
        }
        // ... do work ...
    }
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

## Exception Handling

ALR uses exceptions to report connection-level and configuration errors.

| Exception                     | When It's Thrown                                                              |
| ----------------------------- | ----------------------------------------------------------------------------- |
| `NoEndpointOnStackError`      | An RPC is made from a thread without an active `alr::EndpointCtx`.            |
| `UndefinedRemoteMethodError`  | A call is made to a method that doesn't exist on the remote (version skew).   |
| `NoBackendServiceError`       | The registry could not provide a backend for the requested service.           |
| `ConnectionClosedError`       | The underlying connection was closed unexpectedly.                            |
| `InvalidConfigurationError`   | The `ConnectionInfo` settings are invalid.                                    |
