```cpp

#include "service.h"
#include "endpoint1_gen.h"

#include <chrono>
#include <string>
#include <thread>

using namespace std;
using namespace alr;


namespace cityguide {


struct ChatMsgRecipient
{
   string toUsername;
   EndpointWeakRef epRef;
};


mutex CityGuideService::_mutex;
map<string, EndpointWeakRef> CityGuideService::_userSessions;


// CityGuideService ctor, which will be called on the service endpoint side when
// a client creates a CityGuideService instance from the client endpoint side.
CityGuideService::CityGuideService(const string& username,
                                   const string& cityName) :
   _username(username),
   _cityName(cityName)
{
   if (!_username.empty()) {
      // Simulate user login by adding this session to the list of active users.
      lock_guard<mutex> lock(_mutex);
      _userSessions[_username] = EndpointCtx::getWeakRef();
   }
}


// Will be called once all remote, local and in-flight references to this
// instance are gone.
CityGuideService::~CityGuideService()
{
   if (!_username.empty()) {
      lock_guard<mutex> lock(_mutex);
      // This user is now logging out. Remove their session.
      _userSessions.erase(_username);
   }
}


CityInfo CityGuideService::getCityInfo()
{
   CityInfo city = {};
   city.name = _cityName;
   city.region = "Region";
   city.population = 1'000'000;
   city.landmarks = { "Landmark1", "Landmark2" };
   return city;
}


// A static method, can be called without an instance of CityGuideService.
CityInfo CityGuideService::getCityInfoAnonymous(const string& cityName)
{
   CityInfo city = {};
   city.name = cityName;
   city.region = "Region";
   city.population = 1'000'000;
   city.landmarks = { "Landmark1", "Landmark2" };
   return city;
}


// Simulate looking up a point of interest at the specified location. We just
// create a dummy point of interest for demonstration purposes.
Result<PointOfInterest, string>
CityGuideService::getPointOfInterest(const Location& location)
{
   PointOfInterest poi = {};

   poi.location = location;
   poi.name = "Point of interest: latitude: " +
              to_string(location.latitude) +
              ", longitude: " +
              to_string(location.longitude);

   return poi;
}


// An async method that returns a PointOfInterest.
alr::Async<PointOfInterest>
CityGuideService::getPointOfInterestAsync(const Location& location)
{
   PointOfInterest poi = {};

   poi.location = location;
   poi.name = "Point of interest: latitude: " +
              to_string(location.latitude) +
              ", longitude: " +
              to_string(location.longitude);

   return poi;
}


// An async method that returns a Result<PointOfInterest, string>. Used to
// handle error cases more explicitly.
alr::Async<Result<PointOfInterest, string>>
CityGuideService::getPointOfInterestAsyncResult(const Location& location)
{
   PointOfInterest poi = {};

   poi.location = location;
   poi.name = "Point of interest: latitude: " +
              to_string(location.latitude) +
              ", longitude: " +
              to_string(location.longitude);

   return poi;
}


// Note that ALR is not constrained to a "streaming" mode compared to some
// other RPC frameworks. Instead, ALR uses a call-from-anywhere-to-anywhere
// model, which allows for much more natural and dynamic interactions between
// endpoints. To simulate "streaming" calls, we can simply loop and call into
// the client multiple times.
void CityGuideService::streamPointsOfInterest(CityGuideClient& client,
                                             const Area& area,
                                             bool logResults)
{
   for (int idx = 0; idx < 10; idx++) {
      Location location = { 50.0f + idx, 100.0f + idx };
      auto poi = getPointOfInterest(location);

      if (poi.isSuccess()) {
         client.addPointOfInterestAsync(poi.value());
      }
   }

   // Note that we can call methods on an endpoint class from either endpoint.
   // If 'logResults' is true, the service side will call it from here. In the
   // example, the client leaves 'logResults' at the default 'false' value, and
   // instead logs the results locally after this synchronous call returns.
   //
   // In either case, 'logPointsOfInterest' will be called on the client side on
   // the same thread that is currently blocked on this synchronous call.
   if (logResults) {
      client.logPointsOfInterest();
   }
}


Result<PointOfInterestReviews, string>
CityGuideService::getPointOfInterestReviews(PointOfInterest poi)
{
   PointOfInterestReviews reviews = {};
   reviews.poi = poi;
   return reviews;
}


AsyncRef<Result<vector<PointOfInterest>, string>>
CityGuideService::getPointsOfInterest(const Area& area)
{
   vector<PointOfInterest> pois;

   for (int idx = 0; idx < 10; idx++) {
      Location location = { 50.0f + idx, 100.0f + idx };
      auto poi = getPointOfInterest(location);

      if (poi.isSuccess()) {
         pois.emplace_back(poi.value());
      } else {
         // We can simply return a string here, which will implicitly set
         // the `E` in `AsyncRef<Result<T, E>>`.
         return poi.error();
      }
   }

   // We can return the vector here, which will implicitly set the `T` in
   // `AsyncRef<Result<T, E>>`.
   return pois;
}


AsyncRef<Result<vector<PointOfInterestReviews>, string>>
CityGuideService::getPointsOfInterestReviews(
   AsyncRef<Result<vector<PointOfInterest>, string>> pois)
{
   vector<PointOfInterestReviews> reviews;

   for (auto poi : pois.value().value()) {
      auto result = getPointOfInterestReviews(poi);

      if (result.isError()) {
         return result.error();
      }

      reviews.emplace_back(result.value());
   }

   return reviews;
}


CityReviewSummary CityGuideService::getCityReviewSummary()
{
   CityReviewSummary summary = {};
   summary.total_reviews = 100;
   summary.average_rating = 4.5;
   return summary;
}


AsyncVoid CityGuideService::cityGuideChatAsync(const ChatMessage& chatMsg)
{
   vector<ChatMsgRecipient> recipients;

   if (chatMsg.toUsername.has_value()) {
      // Send the message only to the specified user.
      lock_guard<mutex> lock(_mutex);
      auto it = _userSessions.find(chatMsg.toUsername.value());
      if (it != _userSessions.end()) {
         recipients.emplace_back(ChatMsgRecipient{it->first, it->second});
      }
   }
   else {
      // Send it as a public post to all other online users.
      lock_guard<mutex> lock(_mutex);

      for (auto& it : _userSessions) {
         if (it.first != chatMsg.fromUsername) {
            recipients.emplace_back(ChatMsgRecipient{it.first, it.second});
         }
      }
   }

   for (auto& recipient : recipients) {
      Endpoint ep = recipient.epRef.lock();

      if (ep.isConnected()) {
         // This user is currently logged in. Set their endpoint as the current
         // endpoint context (connection) and send the chat message.
         EndpointCtx epCtx(ep);
         CityGuideClient::postChatMessage(recipient.toUsername, chatMsg);
      }
   }
}


WeatherInfo CityGuideService::getWeatherInfo(const Location& location)
{
   WeatherInfo weather = {};
   weather.location = location;
   weather.temperature = 75;

   if (remoteCaps::hasStructField::cityguide::WeatherInfo::conditionsV2()) {
      // The remote supports the new version of the struct.
      weather.conditionsV2.windDirection = 180;
      weather.conditionsV2.windSpeed = 10;
      weather.conditionsV2.precipitation = 0;
      weather.conditionsV2.description = "Sunny";
   }
   else {
      // Fall back to using the old version of the struct.
      weather.conditions = "Sunny";
   }

   return weather;
}


WeatherForecast CityGuideService::getWeatherForecast(const Location& location)
{
   WeatherForecast forecast = {};
   forecast.city = "San Francisco";

   for (int idx = 0; idx < 5; idx++) {
      WeatherInfo weather = {};
      weather.location = location;
      weather.hour = 13 + idx;
      weather.temperature = (float)(70 + idx);
      forecast.hourly.emplace_back(weather);
   }

   return forecast;
}


int64_t CityGuideService::doSomeWork(uint32_t millsecs)
{
   auto startTime = chrono::system_clock::now();
   this_thread::sleep_for(chrono::milliseconds(millsecs));
   auto endTime = chrono::system_clock::now();
   return chrono::duration_cast<chrono::milliseconds>(endTime - startTime).count();
}


uint64_t CityGuideService::add(uint64_t a, uint64_t b)
{
   return a + b;
}


void RouteRecorderService::startRecord()
{
   _pointsOfInterest.clear();
   _lastLocation = {};
   _locationCount = 0;
   _pointOfInterestCount = 0;
   _totalDistance = 0;
   _startTime = chrono::system_clock::now();
}


void RouteRecorderService::stopRecord()
{
   _endTime = chrono::system_clock::now();
}


AsyncVoid RouteRecorderService::recordRouteLocation(const Location& location)
{
   if (_locationCount > 0) {
      float distance = (float)sqrt(
         pow(location.latitude - _lastLocation.latitude, 2) +
         pow(location.longitude - _lastLocation.longitude, 2));
      _totalDistance += distance;
   }
   _lastLocation = location;
   _locationCount++;
}


Async<CityRouteSummary> RouteRecorderService::getRouteSummary()
{
   CityRouteSummary summary = {};
   summary.pointCount = (uint32_t)_pointsOfInterest.size();
   summary.featureCount = 0;
   summary.distance = (uint32_t)_totalDistance;
   summary.elapsedTime = (uint32_t)chrono::duration_cast<chrono::seconds>
                         (_endTime - _startTime).count();
   return summary;
}


} // namespace cityguide
```
