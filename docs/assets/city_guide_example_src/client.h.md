```cpp

#pragma once

#include "alr/class_ref.h"

#include "city_guide_common.h"

#include <string>
#include <vector>


namespace cityguide {

class RouteRecorderService;


class CityGuideClient : public alr::EndpointClass
{
public:
   CityGuideClient() {}

   static AsyncVoid postChatMessage(const std::string& toUsername,
                                    const ChatMessage& msg);

   AsyncVoid addPointOfInterestAsync(const PointOfInterest& poi);

   void logPointsOfInterest();

private:
   std::vector<PointOfInterest> _pois;
};


} // namespace cityguide
```
