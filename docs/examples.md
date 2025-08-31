# AtomicLinkRPC Examples

Practical, self‑contained examples showcasing core ALR concepts. Each code block omits generated headers (`*_gen.h`) for brevity—these are produced automatically by the ALR compiler once you derive from `alr::EndpointClass` or `alr::CommonEndpointClass`.

---
## 1. Minimal Synchronous Call
```cpp
// service.h
class Greeter : public alr::EndpointClass
{
public:
  static std::string hello(const std::string& name);
};

// service.cpp
std::string Greeter::hello(const std::string& name) { return "Hello " + name; }

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100") // Default can be set in cfg file
    .connect();

  auto r = Greeter::hello("World");
  printf("%s\n", r.c_str());
}
```
Highlights:
- No IDL file.
- Ambient endpoint context: after `connect()` the thread can call remote methods directly.

---
## 2. Asynchronous Result (`alr::Async<T>`)
```cpp
// service.h
class Greeter : public alr::EndpointClass
{
public:
  static alr::Async<std::string> helloAsync(const std::string& name);
};

// service.cpp
alr::Async<std::string> Greeter::helloAsync(const std::string& name)
{
  return "Hello " + name; // Implicit async fulfillment
}

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  auto ar = Greeter::helloAsync("Async");
  ar.then([ar]{ printf("%s\n", ar.value().c_str()); });
  ar.wait(); // Or rely on process lifetime
}
```

---
## 3. Stateful Service with Instance Management

This example shows how to work with stateful remote objects. ALR automatically manages the lifetime of remote instances.

```cpp
// service.h
#include "alr/endpoint.h"

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

// client.cpp
#include "alr/endpoint.h"
#include "client_gen.h"
#include <iostream>

int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

    // Create two independent remote instances of Calculator
    Calculator calc1;       // Remote instance created with initial value 0
    Calculator calc2(20);   // Remote instance created with initial value 20

    calc1.add(10);
    calc2.add(100);
    calc1.add(5);

    // Each call goes to the correct instance on the remote service
    std::cout << "Calc 1 memory: " << calc1.getMemory() << std::endl; // Expected: 15
    std::cout << "Calc 2 memory: " << calc2.getMemory() << std::endl; // Expected: 120

    // Remote objects are destroyed when calc1 and calc2 go out of scope
    return 0;
}
```
Notes: Lifetime auto‑managed—object destroyed once all local, remote & in‑flight refs released.

---
## 4. Passing Object References (`alr::ClassRef<T> / T&`)
```cpp
// service.h
class Session : public alr::EndpointClass
{
public:
  int id() const { return _id; }
private:
  int _id = 42;
};

class SessionUser : public alr::EndpointClass {
public:
  static std::string describe1(alr::ClassRef<Session> s) {
      return "Session " + std::to_string(s->id());
  }

  // C++ lvalue references can also be used on either end, will implicitly be converted to alr::ClassRef<T>
  static std::string describe2(Session& s) {
      return "Session " + std::to_string(s.id());
  }
};

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  Session s;
  auto txt = SessionUser::describe1(s);
  printf("%s\n", txt.c_str());
}
```
The `ClassRef<T>` represents either a local or remote object transparently.

---
## 5. Chained Asynchronous Remote Computation (`alr::AsyncRef<T>`)
```cpp
// service.h
struct Image { std::vector<uint8_t> px; };

class ImageSvc : public alr::EndpointClass
{
public:
  static alr::AsyncRef<Image> scale(alr::AsyncRef<Image> in, float amtX, float amtY);
  static alr::AsyncRef<Image> sharpen(alr::AsyncRef<Image> in, float amt);
  static alr::AsyncRef<Image> encode(alr::AsyncRef<Image> in, int q);
};

// service.cpp
alr::AsyncRef<Image> ImageSvc::scale(alr::AsyncRef<Image> in, float amtX, float amtY) {
  Image& inImg = in.value(); // Blocks and waits for any prior operation to complete (none)
   /*...scale...*/
   return outImg;
}

alr::AsyncRef<Image> ImageSvc::sharpen(alr::AsyncRef<Image> in, float amt) {
  Image& inImg = in.value(); // Blocks and waits for any prior operation to complete (scale)
  /*...sharpen...*/
  return outImg;  
}

alr::AsyncRef<Image> ImageSvc::encode(alr::AsyncRef<Image> in, int q) {
  Image& inImg = in.value(); // Blocks and waits for any prior operation to complete (sharpen)
  /*...encode...*/
  return outImg;  
}

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  Image srcImg = load(/*...path...*/);
  
  // Following 3 calls return asynchronously
  auto img1 = ImageSvc::scale(srcImg, 0.5f, 0.5f);
  auto img2 = ImageSvc::sharpen(img1, 0.7f);
  auto img3 = ImgSvc::encode(img2, 90);

  // Nothing transferred back unless we call any of the results' value() methods:
  const auto& finalImg = img3.value(); // Only img3 image sent back over the wire
  printf("pixels=%zu\n", finalImg.px.size());

  // Remote images will only go out of scope until img1, img2 and img3 go out of scope here (and remotely)
}
```
Intermediate results stay remote—bandwidth saved.

---
## 6. Cancellation & Timeouts
```cpp
// service.h
class Work : public alr::EndpointClass
{
public:
  static alr::Async<int> longJob(int ms);
};

// service.cpp
alr::Async<int> Work::longJob(int ms)
{
  // Simulate progressive work + periodic cancellation checks via CallCtx
  for (int i=0; i<ms/10; i++) {
    if(alr::CallCtx::isCanceled()) {
      return -1;
    }
    /*sleep*/
  }
  return 123;
}

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  auto job = Work::longJob(5000);
  // Cancel after some condition
  job.cancel();
  job.wait();
  if (job.isCanceled()) {
    printf("Canceled\n");
  }
}
```

---
## 7. Bulk Data Transfer (`alr::ByteBuffer`)
```cpp
// service.h
class FileCopier : public alr::EndpointClass
{
public:
  static alr::AsyncVoid sendBlock(alr::ByteBuffer buf, bool last);
};

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  const size_t block = 256*1024;
  alr::ByteBuffer buf(block);
  FILE* f = fopen("big.bin","rb");
  bool last = false;

  while(!last) {
    size_t n = fread(buf.data(), 1, block, f);
    buf.setSize(n, true);
    last = n < block;
    FileCopier::sendBlock(buf, last); // Async fire-and-forget (but tracked)
    buf.waitForBufferReady(); // Reuse once prior async send done, use multiple buffers to overlap
  }
  fclose(f);
}
```

---
## 8. Remote Thread Affinity (`alr::RemoteThread` / `RemoteThreadPool`)
```cpp
// service.h
class MathSvc : public alr::EndpointClass { public: static int add(int a,int b){ return a+b; } };

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  {
    alr::RemoteThread rt; // All calls guaranteed same remote thread
    for(int i=0;i<1000;i++) MathSvc::add(i,1);
  }

  {
    alr::RemoteThreadPool pool(8); // Up to 8 remote workers concurrently
    for(int i=0;i<5000;i++) MathSvc::add(i,2);
    pool.wait();
  }
}
```

---
## 9. Service Registry & Discovery
```cpp
// Start a registry node
// registry.cpp
alr::ConnectionInfo()
  .setListenAddress("0.0.0.0:5500")
  .setAsRegistry(true)
  .listen();

// Backend service registers with registry
// service.cpp
alr::ConnectionInfo()
  .setServiceName("profile")
  .setRegion("us-east").setZone("use1a")
  .setRegistryAddress("registry-host:5500")
  .setPublicAddress("backend-host-1:5501")
  .setListenAddress("0.0.0.0:5501")
  .listen();

// Client resolves dynamically
// client.cpp
alr::Endpoint ep = alr::ConnectionInfo()
  .setServiceName("profile")
  .setConnectAddress("registry-host:5500")
  .setResolveServiceCB([](const std::vector<alr::ServiceInstance>& list) {
      // Choose lowest active thread count
      const alr::ServiceInstance* best = nullptr;
      for(auto& s: list) {
          if (!best || s.loadInfo.numActiveThreads < best->loadInfo.numActiveThreads) {
              best = &s;
          }
      }
      return best ? best->address : "";
  })
  .connect();
```

---
## 10. Explicit Error Handling (`alr::Result<T,E>`)
```cpp
// service.h
class Accounts : public alr::EndpointClass
{
public:
  static alr::Result<int,std::string> balance(int accountId);
};

// service.cpp
alr::Result<int, std::string> Accounts::balance(int id){
  if (id < 0) {
    return "invalid id"; // error
  }
  return 420; // success
}

// client.cpp
int main()
{
    alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

    auto res = Accounts::balance(5);
    if (res.isSuccess()) {
        printf("Bal=%d\n", res.value());
    } else {
        printf("Err=%s\n", res.error().c_str());
    }
}
```
Can be combined, e.g. `alr::Async<alr::Result<T,E>>`.

---
## 11. Call Context Cancellation / Timeout Awareness
```cpp
// service.h
class LongOps : public alr::EndpointClass {
public:
 static alr::Result<std::string,std::string> run();
};

// service.cpp
alr::Result<LongOpResult, std::string> LongOps::run()
{
  LongOpResult longOpResult;
  for(;;) {
    if(alr::CallCtx::isCanceled()) return "Canceled";
    if(alr::CallCtx::isTimedOut()) return "Timed out";
    // work...
  }
  return longOpResult;
}
```

---
## 12. Mixing Sync & Async Transparently
```cpp
// service.h
class Mixed : public alr::EndpointClass {
public:
  static int fast(int x) { return x*2; }
  static alr::Async<int> slow(int x) { return x*10; }
};

// client.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("service-host:55100")
    .connect();

  int a = Mixed::fast(2);
  auto b = Mixed::slow(3);
  b.then([b, a]() { printf("%d %d\n", a, b.value()); });
  b.wait();
}
```

---
## 13. Symmetric Classes for Peer-to-Peer (`alr::CommonEndpointClass`)
```cpp
// chat.h
class Chat : public alr::CommonEndpointClass {
public:
  void post(const std::string& msg) { printf("%s\n", msg.c_str()); }
};

// main.cpp
int main()
{
  alr::Endpoint ep = alr::ConnectionInfo()
    .setConnectAddress("peer-address:55100")
    .connect();
    
  Chat local;          // Lives here and callable from local and peer (if ref passed to peer)
  remote::Chat peer;   // Proxy to Chat living on the remote endpoint, also callable from local and peer

  local.post("Hi from me"); // local prints "Hi from me"
  peer.post("Hi from you"); // remote prints "Hi from you"
}
```
Notes:

- For `alr::CommonEndpointClass`, codegen places the remote declaration under a root namespace (default `remote`).
- Use this pattern for symmetric applications (e.g., peer-to-peer chat, collaborative tools) where either side can call into the other with the same interface.
- `alr::EndpointClass` and `alr::CommonEndpointClass` can be used side-by-side, allowing mixed cases.

---
## Tips & Patterns

- Keep interfaces idiomatic; avoid artificial DTO layers unless needed for domain clarity.
- Use `AsyncRef<T>` when chaining remote operations to suppress unnecessary round‑trips.
- Always rely on framework tracking—even `AsyncVoid` failures transition endpoint to fault state (no silent loss).
- Use registry filtering (region/zone/tags) to implement multi‑region active‑active without external SDN tooling.

---

## Next Steps

After reviewing these examples, you should have a good understanding of how to use ALR's core features. For more detailed guidance on project setup and advanced patterns, please consult the [User Guide](./user-guide.md). For a deep dive into the API, see the [API Reference](./api-reference.md).
