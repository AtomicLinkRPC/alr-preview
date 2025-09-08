```cpp
#pragma once

#include "alr/endpoint_types.h"

#include <optional>
#include <string>
#include <vector>


namespace cityguide {


struct Location
{
   float latitude;
   float longitude;
};


struct PointOfInterest
{
   std::string name;
   std::string type; // e.g., "park", "restaurant"
   Location location;
};


struct ChatMessage
{
   std::optional<std::string> toUsername; // Leave unset to send to all users.
   std::string fromUsername;
   std::string message;
};


} // namespace cityguide
```