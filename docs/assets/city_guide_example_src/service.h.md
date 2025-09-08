```cpp

#pragma once

#include "alr/async.h"
#include "alr/class_ref.h"
#include "alr/result.h"

#include "city_guide_common.h"

#include <chrono>
#include <map>
#include <mutex>
#include <string>
#include <vector>


namespace cityguide {

// Forward declare the client so the service can store a reference to it.
class CityGuideClient;


struct CityInfo
{
   std::string name;
   std::string region;
   int population;
   std::vector<std::string> landmarks;
};


struct Area
{
   Location center;
   float radius;
};


struct Review
{
   int rating; // 1-5
   std::string username;
   std::string comment;
};


struct PointOfInterestReviews
{
   PointOfInterest poi;
   std::vector<Review> reviews;
};


struct CityReviewSummary
{
   int total_reviews;
   float average_rating;
};


struct CityRouteSummary
{
   uint32 pointCount;
   uint32 featureCount;
   uint32 distance;
   uint32 elapsedTime;
};


// The following 3 structs illustrate how API evolution works by evolving the
// structs and APIs over 3 fictional versions.
//
// Version 1:
// We start out with the 'WeatherInfo' struct in the first version. It uses a
// 'conditions' field to store the weather conditions, which is a string.
//
// Version2:
// The 'WeatherConditions' struct is added and the 'WeatherInfo' struct is
// updated to include a new field of this new struct. The original 'conditions'
// string field is deprecated, but kept for backwards compatibility until
// version 1 is no longer supported.
//
// Version3:
// The new 'getWeatherForecast' method and 'WeatherForecast' struct are added.
//
struct WeatherConditions
{
   float windDirection;
   float windSpeed;
   float precipitation;

   // Added in version 3. We will get this default value if we receive this
   // struct from a remote that does not have this field.
   std::string description = "Unknown";
};


struct WeatherInfo
{
   Location location;
   float temperature;

   // Added in version 3.
   int hour; // 0-23

   // Added in version 2.
   WeatherConditions conditionsV2;

   // Deprecated. Use 'conditionsV2' instead. Everything below can safely be
   // removed once we no longer support version 1.
   std::string conditions;
};


// Added in version 3.
struct WeatherForecast
{
   std::string city;
   std::vector<WeatherInfo> forecast;
};



class CityGuideService : public alr::EndpointClass
{
public:
   CityGuideService(const std::string& userName,
                    const std::string& cityName);
   ~CityGuideService();

   static CityInfo getCityInfoAnonymous(const std::string& cityName);

   CityInfo getCityInfo();

   alr::Result<PointOfInterest, std::string>
      getPointOfInterest(const Location& location);

   alr::Async<PointOfInterest>
      getPointOfInterestAsync(const Location& location);

   alr::Async<alr::Result<PointOfInterest, std::string>>
      getPointOfInterestAsyncResult(const Location& location);

   void streamPointsOfInterest(CityGuideClient& client, const Area& area,
                               bool logResults = false);

   alr::Result<PointOfInterestReviews, std::string>
      getPointOfInterestReviews(PointOfInterest poi);

   alr::AsyncRef<alr::Result<std::vector<PointOfInterest>, std::string>>
      getPointsOfInterest(const Area& area);

   alr::AsyncRef<alr::Result<std::vector<PointOfInterestReviews>, std::string>>
   getPointsOfInterestReviews(alr::AsyncRef<alr::Result<
                              std::vector<PointOfInterest>, std::string>> pois);

   CityReviewSummary getCityReviewSummary();

   AsyncVoid cityGuideChatAsync(const cityguide::ChatMessage& chatMsg);

   WeatherInfo getWeatherInfo(const Location& location);

   WeatherForecast getWeatherForecast(const Location& location);

   static sint64 doSomeWork(uint32 millsecs);

   static uint64 add(uint64 a, uint64 b);

private:
   std::string _username;
   std::string _cityName;
   alr::ClassRef<CityGuideClient> _client;

   static std::mutex _mutex;
   static std::map<std::string, alr::EndpointWeakRef> _userSessions;
};


class RouteRecorderService : public alr::EndpointClass
{
public:
   RouteRecorderService() = default;

   void startRecord();
   void stopRecord();

   AsyncVoid recordRouteLocation(const Location& location);

   alr::Async<CityRouteSummary> getRouteSummary();

private:
   std::chrono::system_clock::time_point _startTime;
   std::chrono::system_clock::time_point _endTime;
   std::vector<PointOfInterest> _pointsOfInterest;
   Location _lastLocation = {};
   int _locationCount = 0;
   int _pointOfInterestCount = 0;
   float _totalDistance = 0;
};


} // namespace cityguide
```
