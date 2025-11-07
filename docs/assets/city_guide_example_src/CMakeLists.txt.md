```cmake
cmake_minimum_required(VERSION 3.15)
project(CityGuide)

set(USE_OPENSSL ON CACHE BOOL "Enable OpenSSL support for ${PROJECT_NAME}")

# Define the output executables' names
set(EP1_EXEC_NAME "city_guide_service")
set(EP2_EXEC_NAME "city_guide_client")

# Set the languages that need to be generated for each endpoint
set(EP1_LANGUAGES cpp)
set(EP2_LANGUAGES cpp)

set(ALR_CONFIG_FILE "${CMAKE_CURRENT_SOURCE_DIR}/alr.cfg")

set(EP1_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/city_guide_common.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/service.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/service.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/service_main.cpp"
)

set(EP2_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/city_guide_common.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/client.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/client.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/client_main.cpp"
)

# Note that `alr_add_endpoints` is imported from `cmake/ALRCore.cmake`
alr_add_endpoints(
    ${PROJECT_NAME}
    ${EP1_EXEC_NAME}
    ${EP2_EXEC_NAME}
    EP1_SOURCES
    EP2_SOURCES
    ${ALR_CONFIG_FILE}
)
```