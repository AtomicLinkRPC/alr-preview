```cpp

#include "alr/endpoint.h"

using namespace alr;


int main(int argc, char* argv[])
{
   return ConnectionInfo()
      .setOpenSsl(false)
      .setFromCliArgs(argc, argv)
      .listen();
}
```
