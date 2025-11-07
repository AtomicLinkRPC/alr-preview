# Migration Guide: From gRPC to AtomicLinkRPC

## Introduction

This guide is designed for developers familiar with gRPC who are considering migrating to AtomicLinkRPC (ALR). Rather than providing a mechanical translation of gRPC patterns to ALR equivalents, this guide focuses on a more fundamental shift: **rethinking your RPC design with the freedom of native C++**.

The key question this guide addresses is:

> *"How would I have written this to begin with if I didn't have gRPC's constraints, but could instead follow a more natural 'feels like local C++' model using ALR?"*

ALR removes many of the artificial constraints imposed by traditional RPC frameworks, allowing you to design distributed systems that feel like writing regular C++ code.

---

## Core Philosophy Shift

### gRPC Mindset: IDL-First, Message-Oriented
With gRPC, you start by defining your API in Protocol Buffers (`.proto` files), which imposes several constraints:

- All communication is message-based (every RPC requires a message type, even for simple calls)
- Call patterns must be chosen upfront (unary, server streaming, client streaming, bidirectional streaming)
- Data must be serialized into protobuf message types
- The server cannot initiate calls to the client without client-driven streams
- Complex objects require translation to/from protobuf messages

### ALR Mindset: C++-First, Method-Oriented
With ALR, you write C++ interfaces directly:

- Methods are called with native C++ types (primitives, STL containers, custom structs, object references)
- Use direct parameters for simple calls, structs only when it makes sense (fewer type definitions needed)
- Parameters and struct fields support default values and evolve gracefully (add, remove, reorder)
- Call patterns emerge naturally from return types (`T` for sync, `Async<T>` for async, `AsyncRef<T>` for chained operations)
- The server can call back to the client naturally (they're symmetric endpoints)
- Your business logic structs *are* your wire format (no intermediate translation layer)
- Object references can be passed around and methods invoked on instances from both endpoints

---

## Key Conceptual Differences

### 1. No More Predefined Call Types

**gRPC Constraint:**
You must choose your RPC pattern at IDL definition time:
```protobuf
service MyService {
  rpc UnaryCall(Request) returns (Response);
  rpc ServerStream(Request) returns (stream Response);
  rpc ClientStream(stream Request) returns (Response);
  rpc BidiStream(stream Request) returns (stream Response);
}
```

Once defined, you're locked into that pattern. A single RPC call can't freely mix request/response interactions.

**ALR Freedom:**
Call patterns emerge from your method signatures and implementation:
```cpp
class MyService : public alr::EndpointClass
{
public:
    // Synchronous call - blocks and returns with result
    CityInfo getCityInfo(const std::string& cityName);
    
    // Asynchronous call - returns immediately, result arrives later
    alr::Async<CityInfo> getCityInfoAsync(const std::string& cityName);
    
    // Streaming via callbacks - service pushes data to client
    void streamWeatherUpdates(WeatherClient& client, const Location& loc);
    
    // Chained async operations - results stay remote, but optionally accessed from either endpoint
    alr::AsyncRef<ProcessedData> processData(alr::AsyncRef<RawData> input);
    
    // Fire-and-forget with error tracking
    AsyncVoid logEvent(const Event& evt);
};
```

You can mix and match patterns freely, and even change them later without breaking the API (thanks to schema negotiation).

**Migration Strategy:**

1. Identify what you're *really* trying to accomplish with each gRPC call
2. Express it naturally in C++ without thinking about RPC patterns
3. Let the return type determine the call semantics

### 2. True Bidirectional Communication

**gRPC Constraint:**
The server cannot call client functions directly. To achieve server-to-client communication, you must:

- Set up a client-initiated stream
- Have the server send responses on that stream
- Manage the stream lifecycle explicitly

```protobuf
// Client must initiate even for server-originated data
service Notifications {
  rpc Subscribe(stream Ack) returns (stream Notification);
}
```

**ALR Freedom:**
Both endpoints can define classes with methods that the other can call. Communication is truly symmetric:

```cpp
// Client-side interface
class ChatClient : public alr::EndpointClass
{
public:
    void onMessage(const ChatMessage& msg);
    void onUserJoined(const std::string& username);
};

// Service-side interface
class ChatService : public alr::EndpointClass
{
public:
    void sendMessage(ChatClient& client, const ChatMessage& msg);
    void subscribe(ChatClient& client);
    
private:
    void broadcastToAll(const ChatMessage& msg) {
        for (auto& client : _clients) {
            client->onMessage(msg);  // Server calling client method!
        }
    }
    std::vector<alr::ClassRef<ChatClient>> _clients;
};
```

The server can call `client.onMessage()` just as naturally as the client calls `service.sendMessage()`.

**Migration Strategy:**

1. Identify server-to-client notifications in your gRPC design
2. Instead of managing bidirectional streams, define methods on a client class
3. Pass client object references to the service
4. Let the service invoke client methods directly

### 3. Native Types vs. Message Translation

**gRPC Constraint:**
All data must be defined in protobuf and translated to/from your business logic types:

```protobuf
message Location {
  double latitude = 1;
  double longitude = 2;
}

message PointOfInterest {
  string name = 1;
  string type = 2;
  Location location = 3;
}
```

```cpp
// Your business logic type
struct InternalPOI {
    std::string name;
    std::string category;
    GeoLocation geoLoc;
    // ... many other fields
};

// Conversion boilerplate everywhere
PointOfInterest ToProto(const InternalPOI& poi) {
    PointOfInterest proto;
    proto.set_name(poi.name);
    proto.set_type(poi.category);
    proto.mutable_location()->set_latitude(poi.geoLoc.lat);
    proto.mutable_location()->set_longitude(poi.geoLoc.lon);
    return proto;
}
```

**ALR Freedom:**
Your C++ structs *are* the wire format:

```cpp
struct Location {
    float latitude;
    float longitude;
};

struct PointOfInterest {
    std::string name;
    std::string type;
    Location location;
    // Add any fields you want - they evolve gracefully
};

class CityGuide : public alr::EndpointClass {
public:
    PointOfInterest getPointOfInterest(const Location& loc);
};
```

No conversion code. No impedance mismatch. Your business logic types are your RPC types.

**Migration Strategy:**

1. Use your existing business logic structs directly in ALR method signatures
2. Remove all protobuf message definitions and conversion code
3. Add new fields to structs as needed (old clients/services handle them gracefully)
4. Use `Evolve<T, WritePolicy, ReadPolicy>` on fields where you need strict versioning guarantees

### 4. Direct Parameters vs. Wrapper Messages

**gRPC Constraint:**
Every RPC call requires a protobuf message type, even for simple operations with just a few parameters:

```protobuf
// Must define a message even for simple calls
message GetUserRequest {
  int32 user_id = 1;
  bool include_details = 2;
}

message UpdateScoreRequest {
  string player_name = 1;
  int32 score = 2;
  string level = 3;
}

service GameService {
  rpc GetUser(GetUserRequest) returns (UserResponse);
  rpc UpdateScore(UpdateScoreRequest) returns (ScoreResponse);
}
```

This leads to:

- Message explosion: many simple wrapper messages cluttering your `.proto` files
- Extra boilerplate: constructing message objects for every call
- Reduced readability: harder to see what parameters a call actually takes

**ALR Freedom:**
Use direct parameters for simple calls, structs only when it makes sense:

```cpp
class GameService : public alr::EndpointClass
{
public:
    // Simple calls: use direct parameters (feels like local C++)
    static UserInfo getUser(int userId, bool includeDetails = false);
    static void updateScore(const std::string& playerName, int score, 
                            const std::string& level);
    
    // Complex calls: use structs when you have many related fields
    static GameState startGame(const GameConfig& config);
    static MatchResult playMatch(const MatchSettings& settings);
};

// Client usage - natural C++ function calls
auto user = GameService::getUser(42, true);
GameService::updateScore("player1", 1000, "hard");

GameConfig config{.maxPlayers = 4, .difficulty = "medium", /* ... */};
auto state = GameService::startGame(config);
```

**Key Benefits:**

1. **Fewer Type Definitions**: No need to create wrapper messages for simple calls
2. **Default Values**: Parameters can have defaults that work naturally:
   ```cpp
   static Result search(const std::string& query, 
                        int maxResults = 10,
                        bool caseSensitive = false,
                        const std::optional<std::string>& language = std::nullopt);
   ```

3. **Natural Evolution**: Parameters evolve just like struct fields:

    - Add new parameters with optional defaults (schema handshake prevents sending to old endpoints)
    - Remove parameters (schema handshake prevents old remote from sending, old remote receives their own default)
    - Reorder parameters (matched by name/type, not position)
    - Apply evolve policies to individual parameters

4. **Clean API**: The method signature *is* the documentation:
   ```cpp
   // Crystal clear what this needs
   static void logEvent(int severity,
                        const std::string& eventType, 
                        const std::string& message);
   
   // vs gRPC's opaque message
   // rpc LogEvent(LogEventRequest) returns (Empty);
   // (have to look up LogEventRequest definition to see what's needed)
   ```

**Rule of Thumb:**

- **Use direct parameters** when you have ~7 or fewer simple, independent values
- **Use a struct** when you have:

    - Many parameters (8+)
    - Logically grouped data (Location with lat/lon)
    - Data that's reused across multiple methods
    - Complex nested structures

**Evolution Example:**

```cpp
// Version 1
static CityInfo getCityInfo(const std::string& cityName);

// Version 2 - add optional parameter with default
static CityInfo getCityInfo(const std::string& cityName,
                            bool includeWeather = false);

// Version 3 - add another parameter, reorder for clarity
static CityInfo getCityInfo(const std::string& cityName,
                            const std::string& language = "en",
                            bool includeWeather = false);

// Old clients calling with just cityName still work!
// New clients can use additional parameters
```

**Evolve Policies on Parameters:**

Just like struct fields, parameters can have evolve policies:

```cpp
class DataService : public alr::EndpointClass
{
public:
    // Require that new service always read the version parameter if the client knows it
    static Data getData(int id, 
                        alr::EvolveIfKnown<int> version);
    
    // Warn if client sends enum values unknown to the service
    static void updateData(const Data& data,
                           alr::EvolveStrict<UpdateType> updateType =
                           UpdateType::BaseData);
};
```

**Migration Strategy:**

1. Review your gRPC message definitions
2. For messages with only a few fields, convert to direct parameters
3. Keep as structs when you have many fields or logical grouping
4. Optionally add default values to parameters when you want specific defaults (otherwise type defaults like `""` for strings are used automatically)
5. Reduce the total number of type definitions needed
6. Enjoy cleaner, more readable API signatures

### 5. Rich Call Patterns Without Boilerplate

**gRPC Constraint:**
Complex interactions require managing streams and callbacks:

```cpp
// Client-side streaming with gRPC
ClientContext context;
RecordRouteSummary summary;
unique_ptr<ClientWriter<Location>> writer(stub_->RecordRoute(&context, &summary));

for (const auto& location : locations) {
    writer->Write(location);
}
writer->WritesDone();
Status status = writer->Finish();
```

**ALR Freedom:**
The same pattern is just a regular method with state:

```cpp
class RouteRecorder : public alr::EndpointClass
{
public:
    void recordLocation(const Location& loc) {
        _locations.push_back(loc);
        _totalDistance += calculateDistance(_lastLocation, loc);
        _lastLocation = loc;
    }
    
    RouteSummary getSummary() const {
        return {_locations.size(), _totalDistance, ...};
    }
    
private:
    std::vector<Location> _locations;
    Location _lastLocation;
    float _totalDistance = 0;
};

// Client usage - feels like local object
RouteRecorder recorder;
for (const auto& loc : myRoute) {
    recorder.recordLocation(loc);  // Just call methods!
}
auto summary = recorder.getSummary();
```

For fire-and-forget patterns, use `AsyncVoid`:

```cpp
class RouteRecorder : public alr::EndpointClass
{
public:
    AsyncVoid recordLocation(const Location& loc);  // Non-blocking, still tracked
};
```

**Migration Strategy:**

1. Replace streaming RPCs with stateful remote objects
2. Use instance methods instead of streaming writes
3. Use `AsyncVoid` for fire-and-forget calls (errors still tracked)
4. Use `Async<T>` when you need the result later
5. Use `AsyncRef<T>` for chained operations where intermediate results can be accessed by either endpoint

### 6. Remote Object Semantics

**gRPC Constraint:**
gRPC doesn't have native support for remote objects. Each RPC is stateless unless you manually implement sessions:

```protobuf
service Calculator {
  rpc CreateSession(Empty) returns (SessionId);
  rpc Add(AddRequest) returns (AddResponse);  // Must include SessionId
  rpc GetResult(SessionId) returns (Result);
  rpc CloseSession(SessionId) returns (Empty);
}

message AddRequest {
  string session_id = 1;
  int32 value = 2;
}
```

You must manually manage session IDs, route calls to the right instance, and handle cleanup.

**ALR Freedom:**
Remote objects are first-class citizens with automatic lifetime management:

```cpp
// Service
class Calculator : public alr::EndpointClass
{
public:
    Calculator(int initialValue = 0) : _memory(initialValue) {}
    
    void add(int value) { _memory += value; }
    void subtract(int value) { _memory -= value; }
    int getMemory() const { return _memory; }
    
private:
    int _memory;
};

// Client usage
Calculator calc1(10);      // Remote instance created
Calculator calc2(20);      // Another independent remote instance

calc1.add(5);              // Call methods on specific instance
calc2.subtract(3);

int result1 = calc1.getMemory();  // Each maintains its own state
int result2 = calc2.getMemory();

// Instances automatically destroyed when references are released
```

You can also pass object references between calls:

```cpp
// Service
class Session : public alr::EndpointClass {
public:
    int getUserId() const { return _userId; }
private:
    int _userId;
};

class UserService : public alr::EndpointClass {
public:
    static string getUserInfo(Session& session) {
        return "User " + to_string(session.getUserId());
    }
};

// Client
Session session;
string info = UserService::getUserInfo(session);  // Pass object reference!
```

**Migration Strategy:**

1. Identify session-based gRPC services
2. Convert them to ALR classes with instance state
3. Remove manual session ID management
4. Let ALR handle object lifetime and routing automatically
5. Use `ClassRef<T>` or `T&` to pass object references

### 7. Async Composition and Chaining

**gRPC Constraint:**
Asynchronous operations require callback management and don't compose well:

```cpp
// Chaining async operations in gRPC is verbose
stub_->async()->Scale(&context1, &request1, &queue1, tag1);
// Wait for completion, extract result
stub_->async()->Sharpen(&context2, &request2, &queue2, tag2);
// Wait again, extract result
stub_->async()->Encode(&context3, &request3, &queue3, tag3);
// Final wait and extraction
```

**ALR Freedom:**
Async operations compose naturally with `Async<T>` and `AsyncRef<T>`:

```cpp
// Service
class ImageService : public alr::EndpointClass
{
public:
    static alr::AsyncRef<Image> scale(alr::AsyncRef<Image> in, float factor);
    static alr::AsyncRef<Image> sharpen(alr::AsyncRef<Image> in, float amount);
    static alr::AsyncRef<Image> encode(alr::AsyncRef<Image> in, int quality);
};

// Client - chain operations, only final result transferred
Image source = loadImage("photo.jpg");
auto scaled = ImageService::scale(source, 0.5f);
auto sharpened = ImageService::sharpen(scaled, 0.7f);
auto encoded = ImageService::encode(sharpened, 90);

// Nothing transferred back until we explicitly access the value
const Image& final = encoded.value();  // Only the final image sent back
```

`AsyncRef<T>` is more advanced than `Async<T>`: it maintains bookkeeping on both endpoints, allowing either side to access the value. By default, intermediate results remain on the callee side, saving bandwidth. However, either endpoint can access the actual value when needed.

**Migration Strategy:**

1. Replace callback chains with `Async<T>` return types
2. Use `AsyncRef<T>` for pipelines where intermediate results can stay remote but may be accessed from either endpoint
3. Use `.then()` callbacks when you need to react to completion
4. Use `.wait()` when you need to synchronize
5. Let the framework optimize data transfer automatically

### 8. Error Handling

**gRPC Constraint:**
Errors are communicated through status codes and must be checked for every call:

```cpp
Status status = stub_->GetData(&context, request, &response);
if (!status.ok()) {
    cerr << "RPC failed: " << status.error_message() << endl;
    // Handle error
}
```

For streaming, errors can occur at multiple points and must be handled separately.

**ALR Freedom:**
Multiple error handling strategies:

**Using `Result<T, E>`:**
```cpp
// Service
class DataService : public alr::EndpointClass {
public:
    static alr::Result<Data, std::string> getData(int id);
};

// Client
auto result = DataService::getData(42);
if (result.isOk()) {
    process(result.value());
} else {
    logError(result.error());
}
```

**Using `Status` type:**
```cpp
class DataService : public alr::EndpointClass {
public:
    static alr::Status performOperation();
};

auto status = DataService::performOperation();
if (!status.isSuccess()) {
    if (status.isApplicationError()) {
        auto code = status.getCode<MyErrorCodes>();
        // Handle specific error
    }
}
```

**Using `AsyncFrame` for grouped async operations:**
```cpp
AsyncFrame frame;
frame.onError([](const Status& error) {
    logError("Operation failed: {}", error.message());
});

// All async operations in this scope tracked by frame
DataService::operation1();
DataService::operation2();
DataService::operation3();

// Frame waits for all operations on scope exit
// Error callback invoked if any operation fails
```

**Migration Strategy:**

1. Replace status code checking with `Result<T, E>` for operations that can fail
2. Use `Status` type for operations that need rich error information
3. Use `AsyncFrame` to track groups of async operations
4. Service methods can call `CallCtx::sendAsyncFrameError()` to propagate errors
5. Use `AsyncStatus` for fire-and-forget operations that need error reporting

### 9. Timeouts and Cancellation

**gRPC Constraint:**
Timeouts are set per-call with contexts:

```cpp
ClientContext context;
context.set_deadline(chrono::system_clock::now() + chrono::seconds(5));
Status status = stub_->LongOperation(&context, request, &response);
```

Cancellation requires managing context lifecycle.

**ALR Freedom:**
Timeouts use ambient variables (RAII-based):

```cpp
// Service
class WorkService : public alr::EndpointClass {
public:
    static alr::Async<OpResult> longOperation(int param);
};

// Client
{
    CallTimeout timeout(5000);  // 5 second timeout for this scope
    auto result = WorkService::longOperation(42);
    // Do other work...
    result.wait();
    
    if (result.isTimedOut()) {
        // Handle timeout
    }
}
```

Cancellation is explicit on async operations:

```cpp
auto operation = WorkService::longOperation(42);

// Cancel after some condition
if (shouldCancel()) {
    operation.cancel();
}

operation.wait();
if (operation.isCanceled()) {
    // Handle cancellation
}
```

Service-side checks:

```cpp
Async<OpResult> WorkService::longOperation(int param)
{
    for (int i = 0; i < 1000; i++) {
        if (CallCtx::isTimedOut()) {
            return {};
        }
        if (CallCtx::isAsyncFrameCanceled()) {
            return {};
        }
        // Do work
    }
    return OpResult { ... }
}
```

**Migration Strategy:**

1. Replace per-call deadline contexts with `CallTimeout` ambient variable
2. Use `AsyncVal<T>::cancel()` for explicit cancellation
3. Add periodic `CallCtx::isTimedOut()` checks in long-running service methods
4. Use `CallCtx::isAsyncFrameCanceled()` to respect frame cancellation
5. Use `AsyncFrame::cancel()` to cancel groups of operations

---

## Practical Migration Steps

### Step 1: Understand Your Current gRPC Architecture

Before migrating, map out:

1. **Service definitions**: What services do you have?
2. **Call patterns**: Which are unary, streaming, bidirectional?
3. **Message types**: What data structures are being transmitted?
4. **State management**: How are sessions/contexts managed?
5. **Client-server interaction**: Who initiates what?

### Step 2: Rethink Without Constraints

For each gRPC service, ask:

- If I didn't have to use protobuf, what would my data structures look like?
- If I could call methods directly, what would the API be?
- If the server could call the client, how would I design it differently?
- If I had stateful remote objects, what would simplify?

### Step 3: Define ALR Interfaces

Create your ALR service classes:

```cpp
// Instead of .proto file, write C++ headers
class MyService : public alr::EndpointClass
{
public:
    // Direct translation of your rethought API
    alr::Result<Data, alr::Status> getData(int id);
    AsyncVoid updateData(const Data& data);
    void streamUpdates(MyClient& client, const Filter& filter);
};
```

### Step 4: Implement Service Logic

Implement your methods using native C++:

```cpp
Result<Data, Status> MyService::getData(int id)
{
    Data* data = _repository.find(id);
    if (!data) {
        return Status("Not found");
    }
    return *data;
}
```

### Step 5: Update Client Code

Replace gRPC client code:

```cpp
// Before (gRPC)
MyServiceClient client(channel);
ClientContext context;
Request request;
Response response;
Status status = client.GetData(&context, request, &response);

// After (ALR)
Endpoint ep = ConnectionInfo().setConnectAddress("host:port").connect();
auto result = MyService::getData(42);
```

### Step 6: Handle API Evolution

Use ALR's evolution features to maintain compatibility:

```cpp
struct WeatherConditions {
    // V1 field - deprecated, optional to read/write
    alr::EvolveDeprecated<std::string> conditions;

    // V2 field - stable, must write and read if remote knows
    alr::EvolveIfKnown<float> temperature;

    // V3 field - new, must write if remote knows, read is optional
    alr::EvolveOptional<int> hour;
};
```

### Step 7: Leverage Advanced Features

Take advantage of ALR-specific capabilities:

**Service Registry:**
```cpp
// Service registration
ConnectionInfo()
    .setRegistryAddress("registry:port")
    .setServiceName("my-service")
    .setRegion("us-west")
    .setZone("zone-a")
    .connect();

// Client discovery
ConnectionInfo()
    .setRegistryAddress("registry:port")
    .setServiceName("my-service")
    .setRegion("us-west")
    .setResolverCallback([](const auto& instances) {
        // Custom load balancing logic
        return selectLowestLatency(instances);
    })
    .connect();
```

**Async Frames:**
```cpp
AsyncFrame frame;
frame.onError([](const Status& error) {
    logError("Batch operation failed: {}", error.message());
});

for (const auto& item : batch) {
    MyService::processItem(item);  // All tracked by frame
}
// Frame waits for all operations on scope exit
```

---

## Common Migration Patterns

### Pattern 1: Unary RPC → Synchronous Method

**gRPC:**
```protobuf
service Greeter {
  rpc SayHello(HelloRequest) returns (HelloReply);
}
```

**ALR:**
```cpp
class Greeter : public alr::EndpointClass {
public:
    static std::string sayHello(const std::string& name);
};
```

### Pattern 2: Server Streaming → Callback Method

**gRPC:**
```protobuf
service DataStream {
  rpc StreamData(Request) returns (stream Data);
}
```

**ALR:**
```cpp
class DataClient : public alr::EndpointClass {
public:
    void onData(const Data& data);
};

class DataStream : public alr::EndpointClass {
public:
    void subscribe(DataClient& client, const Request& request);
};
```

### Pattern 3: Client Streaming → Stateful Object

**gRPC:**
```protobuf
service Collector {
  rpc Collect(stream Sample) returns (Summary);
}
```

**ALR:**
```cpp
class Collector : public alr::EndpointClass {
public:
    AsyncVoid addSample(const Sample& sample);
    Summary getSummary() const;
    
private:
    std::vector<Sample> _samples;
};
```

### Pattern 4: Bidirectional Streaming → Symmetric Endpoints

**gRPC:**
```protobuf
service Chat {
  rpc ChatStream(stream Message) returns (stream Message);
}
```

**ALR:**
```cpp
class ChatParticipant : public alr::EndpointClass {
public:
    void sendMessage(const Message& msg);
    void onMessage(const Message& msg);
};

// Both client and server can be ChatParticipants
```

---

## Performance Considerations

### Async for Throughput
Use `AsyncVoid` for fire-and-forget high-throughput scenarios:

```cpp
for (const auto& event : events) {
    EventLogger::logEvent(event);  // AsyncVoid - non-blocking
}
```

### AsyncRef for Large Data Pipelines
Keep intermediate results remote, accessible from either endpoint:

```cpp
auto processed = Pipeline::stage1(rawData);
auto filtered = Pipeline::stage2(processed);
auto final = Pipeline::stage3(filtered);

// Only final result transferred back to caller on explicit access
const Result& output = final.value();
```

---

## Troubleshooting

### Checking Remote Capabilities
Use generated capability checks to handle version differences:

```cpp
if (remoteCaps::hasMethod::MyService::newMethod()) {
    MyService::newMethod();
} else {
    MyService::oldMethod();
}

if (remoteCaps::hasStructField::MyStruct::newField()) {
    data.newField = value;
}
```

### Debugging Connection Issues
```cpp
Endpoint ep = ConnectionInfo()
    .setConnectAddress("host:port")
    .connect();

if (!ep.isConnected()) {
    cerr << "Connection failed: " << ep.getError() << endl;
}
```

### Handling Endpoint Faults
```cpp
try {
    MyService::operation();
} catch (const EndpointException& e) {
    cerr << "Endpoint error: " << e.what() << endl;
    // Reconnect or handle error
}
```

---

## Summary

Migrating from gRPC to ALR is not just about translating protobuf to C++ structs and RPC calls to method invocations. It's about rethinking your distributed system design with the freedom that comes from working with native C++ throughout.

**Key Mindset Shifts:**

1. **From messages to objects**: Think in terms of object references and method calls
2. **From wrapper messages to direct parameters**: Use natural function signatures with default values
3. **From streams to callbacks**: Let the service call back to the client naturally
4. **From manual sessions to automatic lifetime**: Let ALR manage remote object lifecycles
5. **From rigid patterns to fluid composition**: Mix sync, async, fire-and-forget as needed
6. **From defensive coding to graceful evolution**: Add fields and parameters freely, check capabilities when needed
7. **From external tools to integrated features**: Use built-in registry, batching, and load balancing

The result is code that's more maintainable, more performant, and feels like writing regular C++, because it is.

