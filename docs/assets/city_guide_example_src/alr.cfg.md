```
[common]
product_name = CityGuide
product_version = 0.1.2
default_listen_address = localhost:51100
default_connect_address = localhost:51100

[endpoint_1]
cpp_include_header = city_guide_common.h
cpp_include_header = service.h
cpp_remote_caps = true

[endpoint_2]
cpp_include_header = city_guide_common.h
cpp_include_header = client.h
cpp_remote_caps = true
```