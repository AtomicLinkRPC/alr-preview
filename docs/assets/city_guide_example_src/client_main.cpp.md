```cpp

#include "alr/endpoint.h"
#include "alr/result.h"
#include "client.h"
#include "endpoint2_gen.h"


#include <atomic>
#include <chrono>
#include <iostream>
#include <string>
#include <thread>
#include <cstdlib>

using namespace std;
using namespace alr;
using namespace cityguide;


// See 'GetCityInfo' below for details.
void LogCityInfo(const CityInfo& cityInfo)
{
   cout << "City info received: " << endl;
   cout << "  Name:       " << cityInfo.name << endl;
   cout << "  Region:     " << cityInfo.region << endl;
   cout << "  Population: " << cityInfo.population << endl;

   for (const string& landmark : cityInfo.landmarks) {
      cout << "  Landmark:   " << landmark << endl;
   }
}


// This example shows simple stateless and stateful unary RPC calls. The client
// connects to the service, retrieves city info, and then logs the city info to
// the console.
//
void GetCityInfo()
{
   // Connect to the service.
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   // We can make stateless remote calls by calling static methods. In this
   // example, we retrieve the city info without the user first "logging in".
   CityInfo cityInfo = CityGuideService::getCityInfoAnonymous("San Francisco");

   LogCityInfo(cityInfo);

   // We can also make stateful remote calls by creating an instance of a
   // remote class. In this example, we retrieve the city info after the user
   // "logs in".
   CityGuideService service("John Doe", "San Francisco"); // A *remote* instance
   cityInfo = service.getCityInfo();

   LogCityInfo(cityInfo);
}


// This example shows how to get a point of interest using the Result<T, E>
// class. This is useful when you want to handle errors in a more explicit way.
// Note that the method returning the Result class can simply return either a
// value or an error, and does not need to explicitly create a Result<T, E>
// instance.
//
// To prevent silently treating the value as an error or vice versa, a compile-
// time error will occur if T and E are the same type or if one is convertible
// to the other.
//
void GetPointOfInterestResult()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   CityGuideService service("John Doe", "San Francisco");
   Location location = { 50, 100 };

   // Retrieve the point of interest using the Result<T, E> class.
   Result<PointOfInterest, string> poi = service.getPointOfInterest(location);

   if (poi.isSuccess()) {
      cout << "Point of interest received: " << poi.value().name << endl;
   } else {
      cout << "Failed to get point of interest: " << poi.error() << endl;
   }
}


// This example shows how to get a point of interest using the Async<T> class.
// This is useful when you want to handle the result in an async manner. The
// async result can be waited on or a callback can be set to handle the result
// when it becomes available.
//
// Note that we don't need to explicitly call 'wait', since the 'poi.value()'
// call will also wait for the async result to complete, but it is shown to
// illustrate its usage. The 'wait' and 'value' calls can also take a timeout
// value, and will return 'true' if the value was retrieved successfully, or
// 'false' if the call timed out.
//
void GetPointOfInterestAsync()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   CityGuideService service("John Doe", "San Francisco");
   Location location = { 50, 100 };

   AsyncVal<PointOfInterest> poi = service.getPointOfInterestAsync(location);

   poi.wait();
   cout << "Point of interest received: " << poi.value().name << endl;
}


// This example shows how to combine Async<T> and Result<T, E> to get both async
// behavior as well as handle errors more explicitly.
void GetPointOfInterestAsyncResult()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   CityGuideService service("John Doe", "San Francisco");
   Location location = { 50, 100 };

   AsyncVal<Result<PointOfInterest, string>> result =
      service.getPointOfInterestAsyncResult(location);

   // This will wait for the async result for up to 10 seconds.
   auto& poi = result.value(10'000);

   if (poi.isSuccess()) {
      cout << "Point of interest received: " << poi.value().name << endl;
   } else {
      cout << "Failed to get point of interest: " << poi.error() << endl;
   }
}


// This example shows how to stream points of interest to the client. The
// client will receive each point of interest as it is streamed from the service
// asynchronously.
//
// Note that unlike some other RPC frameworks, there is no contract that first
// needs to be established between the client and service to "stream" data.
// Instead, both the client and service are free to call into any static or
// instance method of any class on the other side at any time. This allows for
// a more flexible and dynamic communication model. For instance, if an error
// occurs on the service side, it can call into a method on the client side to
// notify the client of the error. Or the service can call into a method on the
// client side with progress updates, etc. At the same time, the client can
// call into the service at any time to cancel an operation if needed etc.
//
// Another important concept that this example illustrates... Even though the
// service is streaming data to the client asynchronously, the client is blocked
// until the service has finished streaming all the data since the
// 'streamPointsOfInterest' method is a synchronous method. The blocked client
// thread will actually handle the async results as they come in due to the
// Thread Continuation mechanism. Essentially what this means is that the
// service is sending the messages to the blocked thread's message queue, and
// the client thread itself is processing those messages as they come in,
// waiting for the final async reply which will indicate the end of the original
// call.
//
// This simplifies many scenarios, since the client thread needs no extra
// thread synchronization to handle the async results. In this example it
// simply appends the POIs to its internal list as they come in, with no
// locking required. This also means that the streaming is bounded by the
// client side call. Therefore, when 'streamPointsOfInterest' returns locally,
// all streaming is guaranteed to have completed.
//
void StreamPointsOfInterest()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   CityGuideService service("John Doe", "San Francisco"); // <- remote instance
   CityGuideClient client;                                // <- local instance
   Area area = {};
   area.center.latitude = 50;
   area.center.longitude = 100;
   area.radius = 100;

   // Block and wait for the service to stream all POIs to the specified client.
   // Note that we pass the local client to remote here. The remote will
   // indirectly work with a ClassRef<CityGuideClient> to the local client
   // instance here.
   service.streamPointsOfInterest(client, area);

   // We now have all the POIs locally, so log them.
   client.logPointsOfInterest();
}

// See 'ChainAsyncCalls' below for details.
void OnReviewsRetrieved(const Result<vector<PointOfInterestReviews>, string>&
                        result)
{
   if (result.isError()) {
      cout << "Failed to get POI reviews: " << result.error() << endl;
      return;
   }

   auto& reviews = result.value();

   for (const PointOfInterestReviews& review : reviews) {
      cout << "POI: " << review.poi.name << endl;
      for (const Review& r : review.reviews) {
         cout << "  Review: " << r.username << " rating: " << r.rating << endl;
      }
   }
}

// This example shows how to chain async calls together. In this case, we first
// get a list of points of interest, then pass that async result to the next
// async call in order to get the reviews for each POI. The 'then' method is
// used to set a callback that will be called once the async result becomes
// available from the final async call.
//
// Note that since we did not try to access the list of POIs locally, the
// actual list of POIs will not be sent across the wire to this endpoint.
// However, the reference to this async result can be used locally to pass into
// another remote method that takes an AsyncRef<T> parameter. When that remote
// method tries to access the list of POIs, it will first block until the list
// become available before it starts executing. A method can take any number of
// AsyncRef<T> (or AsyncRef<Result<T, E>>) parameters.
//
// Also note that neither of the calls here will block the current thread.
// Therefore, to handle errors that can occur as part of the chained calls,
// each async result should be checked for errors by each method on the remote
// side. If an error occurs somewhere in the call chain, that remote method
// should return an error result. That error will then propagate through the
// remaining chained methods to the final chained method, at which time we can
// check locally for an error by the callback that was set with the 'then'
// method.
//
void ChainAsyncCalls()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   CityGuideService service("John Doe", "San Francisco");
   Area area = {{ 50, 100 }, 100 };

   AsyncRef<Result<vector<PointOfInterest>, string>> pois =
      service.getPointsOfInterest(area);

   // Note that 'pois' is a reference representing an async result. The actual
   // list of POIs by default will not be sent across the wire to this
   // endpoint. However, if we were now to do the following locally between
   // these two calls:
   // 
   // Result<vector<PointOfInterest>, string> localPois = pois.value();
   //
   // ...then the 'value' call would block and request that the actual list of
   // POIs (or error) be sent across the wire to this endpoint.

   AsyncRef<Result<vector<PointOfInterestReviews>, string>> reviews =
      service.getPointsOfInterestReviews(pois);

   reviews.then([reviews]() {
      OnReviewsRetrieved(reviews);
   });
}


// This example shows a chat service where users can post messages to each
// other, or to everyone that is currently online. The service will send the
// chat message to the appropriate user(s) asynchronously.
//
// Note in this example we use multiple threads to simulate multiple users, but
// this example will also work if each client was instead on a different system.
//
// For simplicity in this example, we sleep for a short time in each thread to
// make sure everyone is "online" at roughly the same time.
//
// The remote service instance will automatically be disposed once all remaining
// local, remote and in-flight references to it have been released. Therefore,
// when the local reference to each one here goes out of scope in each thread,
// each will notify the remote, resulting in it being disposed.
//
void CityGuideChat()
{
   vector<thread> threads;

   for (int idx = 0; idx < 3; ++idx) {
      threads.emplace_back([idx]() {
         Endpoint ep = ConnectionInfo().connect();

         if (!ep.isConnected()) {
            cout << "Failed to connect to service: " << ep.getError() << endl;
            return;
         }

         string username = "User #" + to_string(idx);
         CityGuideService service(username, "San Francisco");

         // Wait so everyone is "logged in" at more or less the same time.
         this_thread::sleep_for(500ms);

         // Leave the "toUsername" field unset to send the message to everyone.
         ChatMessage msg = {};
         msg.fromUsername = username;
         msg.message = "Hi, what a beautiful day in the city!";

         service.cityGuideChatAsync(msg);

         // Don't log out too quick as we might miss some messages from others!
         this_thread::sleep_for(500ms);
      });
   }

   for (auto& thread : threads) {
      thread.join();
   }
}


// This example shows streaming from the client to the service. The client will
// record a route by sending a series of locations to the service. The service
// will then calculate the total distance of the route and return it to the
// client when requested.
//
// Similar to above, it is worth mentioning that no explicit contract needs to
// be established between the client and service to stream data. The client can
// call into any static or instance method of any class on the service side at
// any time, and vice versa. This allows for a more flexible and dynamic
// communication model.
//
// Since we sent the locations asynchronously, we need to ensure that all
// async operations have completed before requesting the summary. We can do
// this by calling 'waitForIdle' on the endpoint, which will block until all
// pending operations have completed.
//
// Alternatively, we can explicitly configure a remote thread by creating an
// instance of 'RemoteThread' on the current thread's stack. This will ensure
// that all subsequent sync or async calls from this thread will be handled by
// that reserved remote thread. In that case we would not need to call
// 'waitForIdle' since async calls will be bound by the lifetime of the
// 'RemoteThread' instance. When we then call 'getRouteSummary', the call will
// also be queued onto that reserved thread's message queue, which will then
// guarantee that all of the prior async calls have been processed by the time
// that thread handles the 'getRouteSummary' call.
//
void RecordRoute()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   RouteRecorderService recorder; // A remote instance
   recorder.startRecord();

   for (int idx = 0; idx < 10; ++idx) {
      Location location = { 50.0f + idx, 100.0f + idx };
      recorder.recordRouteLocation(location);
   }

   ep.waitForIdle();
   recorder.stopRecord();

   AsyncVal<CityRouteSummary> summary = recorder.getRouteSummary();

   cout << "Route summary: distance: " << summary.value().distance << endl;
}


// This example shows multiple concurrent synchronous calls to the service from
// multiple threads using the same connection. The service will simulate doing
// some blocking work during each call, in this case sleeping for 1 second
// during each call. It will then return the number of elapsed milliseconds to
// the client.
//
// The client will add up the total CPU time and also measure the wall clock
// time. If the threads were to interfere or block each other, the total CPU
// time would be around 10 seconds. However, since it can handle multiple
// synchronous calls from different threads at the same time, the total CPU
// time will be around 1 second.
//
// Typical output could be:
//   CPU time:        10.01 seconds
//   Wall clock time: 1.004 seconds
//
void Multithreaded()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   vector<thread> threads;
   atomic<int64_t> cpuTimeMs = 0;
   auto startTime = chrono::system_clock::now();

   for (int idx = 0; idx < 10; ++idx) {
      threads.emplace_back([&ep, &cpuTimeMs]() {
         // We need to put the endpoint context on the new thread's stack so
         // that it knows which connection to use.
         EndpointCtx epCtx(ep);

         // While this is a synchronous call, it will not interfere with other
         // threads using the same connection, even if other threads are making
         // synchronous or asynchronous calls. This is completely transparent
         // to the application developer, and the framework will handle all the
         // necessary thread synchronization.
         cpuTimeMs += CityGuideService::doSomeWork(1000);
      });
   }

   // Wait for all threads to finish.
   for (auto& thread : threads) {
      thread.join();
   }

   auto endTime = chrono::system_clock::now();
   float wcTime = (float)chrono::duration_cast<chrono::milliseconds>
      (endTime - startTime).count() * 0.001f;
   float cpuTime = (float)cpuTimeMs * 0.001f;
   cout << "CPU time:        " << cpuTime << " seconds" << endl;
   cout << "Wall clock time: " << wcTime << " seconds" << endl;
}


// A description of how API evolution is supported:
//
// Serialization and deserialization are performed using mutable visitor lookup
// tables. For instance, a struct might have 5 fields, and during serialization
// and deserialization, those field entries would be iterated over using this
// lookup table.
//
// During the handshake phase, both endpoints share their schemas with each
// other, which include the classes, methods, parameters, return values, structs
// and fields that they support. The "tag" or ID of each parameter or field is
// its type + name. For a return value, it is just its type. Therefore, changing
// a parameter or field's type or name will make it be considered to be a
// different parameter or field.
//
// Still during the handshake phase, both endpoints then filter out the
// classes, methods, parameters, structs and fields that the other endpoint
// does not support from their own visitor lookup tables. Endpoint 1 (usually
// the service side) then sorts the remaining entries in its visitor tables to
// match the order of the entries in the visitor table of endpoint 2 (usually
// the client side).
//
// As a final handshake step, each endpoint then updates it bool table that
// indicates what is supported by the other endpoint. The check to see whether
// the remote supports a particular class, method, parameter or field is then a
// very cheap lookup into this bool table.
//
// The sorting allows method parameters and struct fields to be re-ordered
// between versions without breaking forwards or backwards compatibility. For
// instance, deprecated and soon-to-be-removed fields can be moved to the end
// of the struct with a comment that they can be safely removed after version X
// is no longer supported.
//
// By the time messages start being sent between the two endpoints, only those
// fields and parameters that both endpoints support will be sent on the wire,
// and in the exact order that both sides expect. This also eliminates the need
// for embedded "tags" to identify each value within the payload.
// 
// The below example shows an evolving API over time. In this case, we have
// three versions of the 'WeatherInfo' struct. The first version has a single
// string field for the weather conditions. The second version adds a new
// struct for the weather conditions, but keeps the original field for
// backwards compatibility. The third version adds a new method to get the
// weather forecast.

void ApiEvolution()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   CityGuideService service("John Doe", "San Francisco");
   Location location = { 50, 100 };
   WeatherInfo weather = service.getWeatherInfo(location);

   // Both older and newer versions support the following fields.
   cout << "Weather info received: " << endl;
   cout << "  Location:      " << weather.location.latitude << ", "
      << weather.location.longitude << endl;
   cout << "  Temperature:   " << weather.temperature << " F" << endl;

   // Use the 'hasStructField' method to check if the remote supports a
   // particular field. If the remote does not support the field, the default
   // value for that field will be returned.
   if (remoteCaps::hasStructField::cityguide::WeatherInfo::conditionsV2()) {
      // The remote supports v2 of the weather conditions, which is a struct.
      cout << "  Wind:          " << weather.conditionsV2.windDirection
           << " deg, " << weather.conditionsV2.windSpeed << " mph" << endl;
      cout << "  Precipitation: " << weather.conditionsV2.precipitation
           << " in" << endl;
      cout << "  Description:   " << weather.conditionsV2.description << endl;
   }
   else {
      // Fall back to using the old version, which has just a single string.
      cout << "  Conditions:    " << weather.conditions << endl;
   }

   // Check if the remote service supports the new 'WeatherForecast' method.
   if (remoteCaps::hasMethod::cityguide::CityGuideService::getWeatherForecast()) {
      WeatherForecast forecast = service.getWeatherForecast(location);
      cout << endl << "Weather forecast received: " << endl;
      cout << "  City: " << forecast.city << endl;
      for (const WeatherInfo& w : forecast.forecast) {
         cout << "  Forecast: " << w.hour << ":00, " << w.temperature << " F"
            << endl;
      }
   }
   else {
      cout << endl << "Weather forecast not supported by remote service" << endl;
   }
}


// This example shows how to log messages sent and/or received by the client
// and/or service. In this case, we log messages sent and received on the client
// side.
//
// This can be useful to debug issues with the client/service communication, or
// to log messages for auditing purposes.
//
// Below is the output from running the sample. The first message is the one
// sent by the client to the service, and the second message is the reply from
// the service.
//
// Note that the output shown is that of the complete framed messages sent and
// received for the request/reply. There was no additional hidden metadata that
// was sent, illustrating the compact message sizes (10 bytes round-trip total).

/*
--------- LogMessages ---------

Message: Send: epId: 13
--->
  0      uint:     6          Message size
  1      ctrl:     18         Message type: Sync Request
  2      uint:     4          Src context ID
  3      uint:     16         Invoke: CityGuideService::add(...)
  4      uint:     20
  5      uint:     22
<---

Message: Recv: epId: 13
--->
  0      uint:     4          Message size
  1      ctrl:     35         Message type: Sync Reply
  2      uint:     4          Dst context ID
  3      uint:     42
<---

CityGuideService::add: 20 + 22 = 42

*/

void LogMessages()
{
   Endpoint ep = ConnectionInfo().connect();

   if (!ep.isConnected()) {
      cout << "Failed to connect to service: " << ep.getError() << endl;
      return;
   }

   // Log messages from/to the client.
   ep.setLocalDebugMode(true, true);

   uint64_t result = CityGuideService::add(20, 22);
   cout << "CityGuideService::add: 20 + 22 = " << result << endl;

   // Turn off logging.
   ep.setLocalDebugMode(false, false);
}


int main(int argc, char* argv[])
{
   cout << endl << "--------- GetCityInfo ---------" << endl;
   GetCityInfo();

   cout << endl << "--------- GetPointOfInterestResult ---------" << endl;
   GetPointOfInterestResult();

   cout << endl << "--------- GetPointOfInterestAsync ---------" << endl;
   GetPointOfInterestAsync();

   cout << endl << "--------- GetPointOfInterestAsyncResult ---------" << endl;
   GetPointOfInterestAsyncResult();

   cout << endl << "--------- StreamPointsOfInterest ---------" << endl;
   StreamPointsOfInterest();

   cout << endl << "--------- ChainAsyncCalls ---------" << endl;
   ChainAsyncCalls();

   cout << endl << "--------- CityGuideChat ---------" << endl;
   CityGuideChat();

   cout << endl << "--------- RecordRoute ---------" << endl;
   RecordRoute();

   cout << endl << "--------- Multithreaded ---------" << endl;
   Multithreaded();

   cout << endl << "--------- ApiEvolution ---------" << endl;
   ApiEvolution();

   cout << endl << "--------- LogMessages ---------" << endl;
   LogMessages();

   cout << endl << "Press Enter to exit..." << endl;
   cin.get();

   return 0;
}
```
