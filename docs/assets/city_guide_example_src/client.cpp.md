```cpp

#include "alr/endpoint.h"
#include "client.h"
#include "endpoint2_gen.h"

#include <iostream>


using namespace std;
using namespace alr;


namespace cityguide {


AsyncVoid
CityGuideClient::postChatMessage(const string& toUsername,
                                 const ChatMessage& msg)
{
   logAlways("CityGuide message from {} to {}: '{}'\n",
             msg.fromUsername, toUsername, msg.message);
}


AsyncVoid
CityGuideClient::addPointOfInterestAsync(const PointOfInterest& poi)
{
   _pois.push_back(poi);
}


void CityGuideClient::logPointsOfInterest()
{
   for (const auto& poi : _pois) {
      cout << "Point of interest: " << poi.name << " at "
           << poi.location.latitude << ", "
           << poi.location.longitude
           << endl;
   }
}


} // namespace cityguide
```
