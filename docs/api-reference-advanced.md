# API Reference - Advanced

This document covers advanced ALR features for specialized use cases including service discovery, dynamic thread management, and performance monitoring. For core API information, see [API Reference - Core](./api-reference-core.md).

## Table of Contents

- [Return Modes and API Evolution](#return-modes-and-api-evolution)
- [Registry and Discovery](#registry-and-discovery)
- [Thread Control and Queue Management](#thread-control-and-queue-management)
- [Performance Monitoring](#performance-monitoring)
- [Service Instances and Load Information](#service-instances-and-load-information)

---

## Return Modes and API Evolution

ALR supports multiple return modes for RPC methods, each with different characteristics and evolution constraints. Understanding these modes is critical for designing robust, evolvable APIs.

### The Five Return Mode Categories

Return modes are grouped into five categories. **You can evolve types within a category, but changing between categories breaks compatibility:**

1. **Void Mode**: `void`
2. **Value Mode**: `T`, `Result<T, E>`, `ClassRef<T>`
3. **Async Void Mode**: `AsyncVoid`, `AsyncStatus`
4. **Async Value Mode**: `Async<T>`, `Async<Result<T, E>>`, `Async<ClassRef<T>>`
5. **Async Ref Mode**: `AsyncRef<T>`

### Detailed Return Mode Reference

#### 1. Void Mode: `void`

**Purpose:** Synchronous call that blocks until completion but returns no value.

**Usage:**
```cpp
// service.h
static void logMessage(const std::string& msg);

// client.cpp
DataService::logMessage("Operation started");  // Blocks until complete
```

**Evolution:** Cannot evolve to return a value without breaking compatibility. To add return values, create a new method.

---

#### 2. Value Mode

##### `T` - Synchronous Value Return

**Purpose:** Blocks until a response is received and returns the value.

**Usage:**
```cpp
// service.h
static std::string getUserName(int userId);

// client.cpp
std::string name = UserService::getUserName(42);  // Blocks until response
```

##### `Result<T, E>` - Explicit Error Handling

**Purpose:** Returns either a success value or an error without using exceptions.

**Usage:**
```cpp
// service.h
enum class UserError { NotFound, InvalidId, DatabaseError };
static Result<User, UserError> getUser(int id);

// service.cpp
Result<User, UserError> UserService::getUser(int id) {
    if (id <= 0) {
        return UserError::InvalidId;  // Implicitly converted to Result
    }
    User user = database.find(id);
    if (!user.isValid()) {
        return UserError::NotFound;
    }
    return user;  // Implicitly converted to Result
}

// client.cpp
auto result = UserService::getUser(42);
if (result.isSuccess()) {
    User user = result.value();
    // Use user...
} else {
    UserError err = result.error();
    // Handle error...
}
```

##### `ClassRef<T>` - Remote Object References

**Purpose:** Returns a reference to a remote object whose methods can be called from either side.

**Usage:**
```cpp
// service.h
static ClassRef<Session> createSession(const std::string& token);

// client.cpp
auto session = AuthService::createSession("token123");
session->performAction();  // Call method on remote object
```

**Evolution within Value Mode:**

- ✅ **Safe:** `T` → `Result<T, E>` (with caution, see below)
- ✅ **Safe:** Evolving struct `T` by adding/removing fields
- ❌ **Breaks compatibility:** `T` → `Async<T>`

**Caution when changing `T` to `Result<T, E>`:**
While `remoteCaps::hasMethod::MyService::Foo()` returns `true` (method can be invoked), `remoteCaps::hasMethodReturn::MyService::Foo()` returns `false` because the return type schema changed. The caller will receive a default-constructed value instead of the actual result. **Best practice:** Use struct return types and evolve fields within the struct rather than changing the return type itself.

---

#### 3. Async Void Mode

##### `AsyncVoid` - Fire-and-Forget with Tracking

**Purpose:** Asynchronous call that doesn't return a value. The framework tracks the call and enters a faulted state on low-level failures.

**Usage:**
```cpp
// service.h
static AsyncVoid logEventAsync(const std::string& event);

// client.cpp
EventService::logEventAsync("user_login");  // Returns immediately
// Framework tracks completion
```

##### `AsyncStatus` - Efficient Error-Only Reporting

**Purpose:** Fire-and-forget call that only sends a response on failure, reducing network traffic.

**Usage:**
```cpp
// service.h
static AsyncStatus processDataAsync(const Data& data);

// service.cpp
AsyncStatus DataService::processDataAsync(const Data& data) {
    if (!validate(data)) {
        return Status(ErrorCode::InvalidData, "Validation failed");
    }
    // Process data...
    return Status::Success;  // No response sent on success
}

// client.cpp - with AsyncFrame for error handling
{
    alr::AsyncFrame frame;
    frame.onError([](const alr::Status& err) {
        std::cerr << "Error: " << err.getFormattedMessage() << std::endl;
    });
    
    // Launch multiple fire-and-forget operations
    for (const auto& item : items) {
        DataService::processDataAsync(item);  // Only errors are reported back
    }
}  // Frame waits for all operations, catches errors
```

**Evolution within Async Void Mode:**

- ✅ **Safe:** `AsyncVoid` ↔ `AsyncStatus`
- ❌ **Breaks compatibility:** `AsyncVoid` → `Async<T>`

---

#### 4. Async Value Mode

##### `Async<T>` - Asynchronous Value Return

**Purpose:** Returns immediately with a future/promise that can be waited on or have callbacks attached.

**Usage:**
```cpp
// service.h
static Async<std::string> fetchDataAsync(int id);

// client.cpp - callback style
auto future = DataService::fetchDataAsync(42);
future.then([](const std::string& data) {
    std::cout << "Received: " << data << std::endl;
});

// client.cpp - blocking style
auto future = DataService::fetchDataAsync(42);
std::string data = future.value();  // Blocks until ready
```

##### `Async<Result<T, E>>` - Async with Explicit Errors

**Purpose:** Combines asynchronous execution with structured error handling.

**Usage:**
```cpp
// service.h
static Async<Result<User, UserError>> fetchUserAsync(int id);

// client.cpp
auto future = UserService::fetchUserAsync(42);
future.then([](const Result<User, UserError>& result) {
    if (result.isSuccess()) {
        User user = result.value();
        // Use user...
    } else {
        // Handle error...
    }
});
```

**Evolution within Async Value Mode:**

- ✅ **Safe:** `Async<T>` → `Async<Result<T, E>>` (with same caution as Value Mode)
- ✅ **Safe:** Evolving struct `T` in `Async<T>`
- ❌ **Breaks compatibility:** `Async<T>` → `T`

---

#### 5. Async Ref Mode: `AsyncRef<T>`

**Purpose:** Advanced mode for remote futures that can be passed between endpoints in RPC calls and stored in structs without transferring the underlying data. The value is only transmitted to the non-owning endpoint when that endpoint explicitly accesses it (e.g., via `.value()`), enabling efficient remote function chaining and deferred data transfer.

**Usage:**
```cpp
// service.h
static AsyncRef<Image> fetchImageAsync(const std::string& url);
static Async<bool> processImage(AsyncRef<Image> img);

// client.cpp - chain operations without transferring image data
auto imgFuture = ImageService::fetchImageAsync("http://example.com/img.jpg");
auto processFuture = ImageService::processImage(imgFuture);  // Image stays remote
bool success = processFuture.value();  // Only boolean result transferred
```

**Evolution within Async Ref Mode:**

- ✅ **Safe:** Evolving struct `T` in `AsyncRef<T>` by adding/removing fields
- ❌ **Breaks compatibility:** `AsyncRef<T>` to any other return mode category

> **Note:** `AsyncRef<T>` is its own category. While you can safely evolve the type `T` itself (e.g., add fields to a struct), you cannot change from `AsyncRef<T>` to other modes like `Async<T>` or `T` without breaking compatibility.

---

### Best Practices for Return Mode Evolution

#### ✅ DO:

1. **Use struct return types** for methods that may need to return additional data in the future:
   ```cpp
   struct UserInfo {
       std::string name;
       int age;
       // Can add fields later without breaking compatibility
   };
   static UserInfo getUser(int id);
   ```

2. **Stay within the same return mode category** when evolving APIs.

3. **Use `Result<T, E>` for operations with well-defined error cases:**
   ```cpp
   enum class FileError { NotFound, PermissionDenied, IoError };
   static Result<FileData, FileError> readFile(const std::string& path);
   ```

4. **Use `AsyncStatus` for bulk fire-and-forget operations** where only errors need reporting.

5. **Check capabilities** before calling methods on mixed-version deployments:
   ```cpp
   if (remoteCaps::hasMethod::newMethod()) {
       Service::newMethod();
   } else {
       // Fallback for older services
   }
   ```

#### ❌ DON'T:

1. **Change return mode categories** (e.g., `T` → `Async<T>`) on existing methods. Create a new method instead:
   ```cpp
   // Instead of changing getUser():
   static User getUser(int id);                    // Keep existing
   static Async<User> getUserAsync(int id);        // Add new async version
   ```

2. **Change the return type directly** (e.g., `int` → `Result<int, Error>`). While the method can still be invoked, `remoteCaps::hasMethodReturn::SomeService::SomeFunc()` will return `false`, and the caller receives a default-constructed value.

3. **Rely on exceptions** for common error cases. Use `Result<T, E>` for explicit, structured error handling.

4. **Mix return modes** for conceptually similar operations without a clear reason.

### Return Mode Compatibility Matrix

| From ↓ / To → | void | T | Result&lt;T,E&gt; | Status | AsyncVoid | AsyncStatus | Async&lt;T&gt; | AsyncRef&lt;T&gt; |
|---------------|------|---|-------------|--------|-----------|-------------|----------|-------------|
| **void** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **T** | ❌ | ✅* | ⚠️ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Result&lt;T,E&gt;** | ❌ | ⚠️ | ✅* | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Status** | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **AsyncVoid** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| **AsyncStatus** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| **Async&lt;T&gt;** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅* | ❌ |
| **AsyncRef&lt;T&gt;** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅* |

- ✅ = Safe evolution within category
- ⚠️ = Technically possible but `hasMethodReturn` returns false; caller gets default value
- ❌ = Breaks compatibility (caps check fails)
- \* = Can evolve the type `T` itself (e.g., add fields to struct)

### Summary

Return modes are a fundamental part of ALR's design for building evolvable APIs. By understanding the five return mode categories and following the best practices outlined above, you can design APIs that are both efficient and maintainable across version changes. Remember: evolve types within a category, but avoid changing categories for existing methods.

---

## Registry and Discovery

### Service Registry Setup

ALR supports a distributed registry pattern where backend services register themselves and clients discover them dynamically.

#### Starting a Registry Node

```cpp
// registry.cpp
#include "alr/connection_info.h"

alr::ConnectionInfo()
    .setListenAddress("0.0.0.0:5500")
    .setAsRegistry(true)
    .listen();
```

#### Registering a Backend Service

```cpp
// service.cpp
alr::ConnectionInfo()
    .setServiceName("profile")           // Logical service name
    .setRegion("us-east")                // Region hint
    .setZone("use1a")                    // Zone/availability hint
    .setRegistryAddress("registry-host:5500")
    .setPublicAddress("backend-host-1:5501")
    .setListenAddress("0.0.0.0:5501")
    .listen();
```

#### Client-Side Service Discovery

```cpp
// client.cpp
alr::Endpoint ep = alr::ConnectionInfo()
    .setServiceName("profile")
    .setConnectAddress("registry-host:5500")
    .setResolveServiceCB([](const std::vector<alr::ServiceInstance>& list) {
        // Choose service instance based on load
        const alr::ServiceInstance* best = nullptr;
        for (const auto& instance : list) {
            if (best == nullptr || 
                instance.loadInfo.numActiveThreads < best->loadInfo.numActiveThreads) {
                best = &instance;
            }
        }
        return best ? best->address : std::string();
    })
    .connect();
```

### `alr::ServiceInstance`

Represents a registered backend service instance discovered via the registry.

**Key Fields:**

- `AboutInfo aboutInfo`: Metadata about the service including name, version, region, zone, tags
- `std::string address`: Network address (host:port)
- `InstanceLoadInfo loadInfo`: Current load information
- `std::chrono::system_clock::time_point updateTime`: Last update timestamp

### `alr::AboutInfo`

Contains metadata about a service.

**Key Fields:**

- `std::string frameworkVersion`: ALR framework version
- `std::string productName`: Service product name
- `std::string productVersion`: Service version
- `std::string buildId`: Build identifier
- `std::string serviceName`: Logical service name
- `std::string region`: Geographic region
- `std::string zone`: Availability zone
- `std::vector<std::string> tags`: Custom tags for filtering
- `bool isBackendService`: Whether this is a backend service

### `alr::InstanceLoadInfo`

Provides runtime load metrics for a service instance.

**Key Fields:**

- `uint32_t maxServiceMemMB`: Maximum service memory in MB
- `uint32_t allocatedMemMB`: Currently allocated memory
- `uint32_t numConnections`: Active connection count
- `uint32_t numHardwareThreads`: CPU thread count on host
- `uint32_t numActiveThreads`: Currently active service threads

---

## Thread Control and Queue Management

### `alr::RemoteThreadPool`

Controls thread scheduling on the remote endpoint for parallel execution.

**Constructor:**
```cpp
// Use up to 8 remote threads for tasks from this scope
alr::RemoteThreadPool pool(8);
```

**Key Methods:**

- `bool wait(uint32_t timeoutMs = 0)`: Block until all tasks in the pool complete or timeout
- `static void detachActiveQueue()`: Detach the current thread's queue
- `static bool waitForActiveQueueIdle(uint32_t timeoutMs = 0)`: Wait for all tasks on active queue to complete

**Usage Example:**
```cpp
// client.cpp
alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service:55100")
    .connect();

{
    alr::EndpointCtx ctx(ep);
    alr::RemoteThreadPool pool(8);
    
    // Launch up to 8 parallel tasks on remote
    MyService::asyncTask1();
    MyService::asyncTask2();
    MyService::asyncTask3();
    
    pool.wait();  // Wait for all to complete
}
```

> **Note:** The remote endpoint's thread pool is managed globally. Specifying 8 threads means *up to* 8 threads from the remote's global pool can work on tasks initiated from this scope. Threads are reused across multiple client pools.

### `alr::RemoteThread`

Guarantees sequential execution on a single remote thread.

**Constructor:**
```cpp
// All subsequent RPC calls in this scope execute on one dedicated remote thread
alr::RemoteThread remoteThread;
```

**Usage Example:**
```cpp
{
    alr::EndpointCtx ctx(ep);
    alr::RemoteThread singleThread;
    
    MyService::doTaskA();  // Guaranteed order
    MyService::doTaskB();  // Executed after doTaskA
    MyService::doTaskC();  // Executed after doTaskB
}  // Thread released when scope exits
```

### Message Queue Management

ALR uses thread-local message queues to coordinate RPC execution. The framework handles queue assignment transparently, but advanced users can control queue behavior:

- **Initiator Queue**: Client initiates RPC, waits for response
- **Response Queue**: Server sends response
- **Async Response Queue**: Asynchronous callback handling

The ambient context (via `EndpointCtx`) implicitly manages queue assignment per thread.

---

## Performance Monitoring

### `alr::EndpointStats`

Retrieved via `Endpoint::getLocalStats()`, provides performance metrics.

**Usage:**
```cpp
alr::EndpointStats stats = ep.getLocalStats(true);  // true = reset after reading
// stats contain message counts, timing info, etc.
```

### `alr::QueueStats`

Per-queue performance statistics.

**Usage:**
```cpp
alr::EndpointQueueStats queueStats;
// Available through endpoint inspection
```

### Performance Monitoring with `alr::CallCtx`

Access call-level information for diagnostics:

```cpp
void MyService::taskWithMonitoring() {
    // Check if call was canceled
    if (alr::CallCtx::isCanceled()) {
        return;  // Early exit
    }
    
    // Check timeout state
    if (alr::CallCtx::isTimedOut()) {
        // Handle timeout
        return;
    }
    
    // Perform monitoring/diagnostics
    if (!alr::CallCtx::isOk()) {
        // Call is in error state
        return;
    }
}
```

**CallCtx Methods:**

- `bool isOk()`: Returns true if call is active and not canceled/timed out
- `bool isCanceled()`: Check for cancellation from client
- `bool isTimedOut()`: Check if timeout has occurred

---

## Service Instances and Load Information

### Querying Service Instances

```cpp
#include "alr/registry.h"

alr::AboutInfo query;
query.serviceName = "profile";
query.region = "us-east";

std::vector<alr::ServiceInstance> instances = 
    alr::Registry::getServiceInstances(query);

for (const auto& instance : instances) {
    std::cout << "Instance: " << instance.address << std::endl;
    std::cout << "Active threads: " << 
        instance.loadInfo.numActiveThreads << std::endl;
}
```

### Custom Instance Selection Logic

When connecting, provide a callback to select the best instance:

```cpp
alr::Endpoint ep = alr::ConnectionInfo()
    .setServiceName("compute")
    .setConnectAddress("registry-host:5500")
    .setResolveServiceCB([](const std::vector<alr::ServiceInstance>& list) {
        // Select instance with lowest active threads
        const alr::ServiceInstance* best = nullptr;
        for (const auto& instance : list) {
            if (best == nullptr || 
                instance.loadInfo.numActiveThreads < 
                best->loadInfo.numActiveThreads) {
                best = &instance;
            }
        }
        return best ? best->address : std::string();
    })
    .connect();
```

The resolution callback is invoked to:

- **Distribute load**: Choose least-loaded instance
- **Zone affinity**: Prefer instances in specific zones
- **Fallback logic**: Handle registry unavailability
- **Custom filtering**: Filter by tags or metadata
