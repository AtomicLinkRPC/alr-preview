# Raw Performance Results (Remote)

The results below are from a subset of ALR's tests used during development:
- This is a different set of tests than the "ALR vs. gRPC" tests.
- In all tests, a single client process and a single service process is used.
- All tests have TLS enabled (OpenSSL).

## Hardware Specifications
### Main system
- Dell Alienware Aurora R16
- Processor: Intel(R) Core(TM) i9-14900KF [Cores 24] [Logical processors 32]
- Memory: 64 GB
- OS: Windows 11

### Remote system
- Microsoft Surface Laptop Studio
- Processor: 11th Gen Intel(R) Core(TM) i7-11370H @ 3.30GHz
- Memory: 32.0 GB
- OS: Windows 11

### Network
- 2.5Gbps physical Ethernet

## Run Summary

```
---------------------------------------------
Service: x.x.x.x:51100, TLS: Yes

| Name                             |    Total RPCs |      RPC/s | Lat p99 | Send Bytes/s | Recv Bytes/s | Avg Batch |
| --------------------             | ------------- | ---------- | ------- | ------------ | ------------ | --------- |
| OneParamToSrvAsyncVoid           |   100,000,000 |  9,874,361 |         |   97,108,544 |          224 |   2,813.6 |
| OneParamToClnAsyncVoid           |   200,000,000 | 11,171,454 |         |          363 |  110,818,712 |   1,986.7 |
| MultiOneParamToSrvAsyncVoid      | 1,000,000,000 | 31,239,726 |         |  218,748,256 |        1,102 |   3,559.2 |
| SendAsyncReplyAsyncVoid          |   200,000,000 | 11,938,798 |         |  103,465,072 |  103,578,784 |   2,548.4 |
| SmallStructToSrvAsyncVoid        |   100,000,000 |  8,536,336 |         |  101,026,376 |          698 |   2,427.9 |
| MediumStructToSrvAsyncVoid       |    50,000,000 |  4,599,573 |         |  226,916,992 |          807 |     988.2 |
| MultiEndpointToSrvAsyncVoid      | 2,000,000,000 | 71,515,000 |         |  286,216,192 |        1,304 |   3,660.8 |
| MultiEndpointToClnAsyncVoid      | 2,000,000,000 | 34,086,840 |         |          960 |  136,456,736 |   2,491.8 |
| SmallStructSendReplyAsyncVoid    |   200,000,000 |  9,553,021 | 91.3 ms |  211,908,784 |  212,051,664 |   2,915.1 |
| MediumStructSendReplyAsyncVoid   |   100,000,000 |  3,302,560 | 85.3 ms |  275,597,600 |  275,691,168 |     784.4 |
| LargeStructSendReplyAsyncVoid    |    20,000,000 |  1,355,739 | 95.3 ms |  273,883,456 |  273,884,064 |     323.8 |
| SmallStructVectorParamAsyncVoid  |     2,000,000 |     18,446 |         |  295,331,520 |          302 |       4.0 |
| MediumStructVectorParamAsyncVoid |       200,000 |      3,881 |         |  295,025,856 |          263 |       1.0 |
| LargeStructVectorParamAsyncVoid  |       100,000 |      1,505 |         |  295,146,528 |          204 |       1.0 |
| SmallStructRTAsyncVal            |   100,000,000 |  7,255,553 | 30.1 ms |  167,864,624 |  160,650,032 |     627.5 |
| MediumStructRTAsyncVal           |   100,000,000 |  2,251,428 |  144 ms |  191,384,560 |  189,186,864 |     273.8 |
| LargeStructRTAsyncVal            |    50,000,000 |  1,011,787 |  455 ms |  207,438,640 |  206,483,296 |     193.7 |

---------------------------------------------
```

## Run Details
```
------------------------------------------------------
Note: The reported average message size represents a
fully framed message, including all necessary metadata.
------------------------------------------------------


Hello from Client! Service: x.x.x.x:51100, TLS: Yes


#### Start of test OneParamToSrvAsyncVoid ####
Description:       A single client uses 1 thread to invoke 100,000,000 stateful
                   async calls with 1 parameter to the service.

Results for:       OneParamToSrvAsyncVoid
Duration:          10.1 s
Total test RPCs:   100,000,000
RPC/s:             9,874,361
One RPC every:     101 ns
Total msgs:        send: 100,000,005 | recv: 282
Avg msg size:      send: 9.8 | recv: 8.1

Total send bytes:  983,441,290    (97,108,544 bytes/sec)
Total recv bytes:  2,271          (224 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |            230

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   100,000,005 |        35,542 |         2813.6 |    0.04%
 Remote -> Local  |           282 |           282 |            1.0 |  100.00%

Local batch histogram:
   1        |         5
   2 - 457  |       516  **
 458 - 912  |     1,210  ******
 913 - 1367 |       629  ***
1368 - 1822 |     2,307  ***********
1823 - 2277 |       981  *****
2278 - 2732 |    10,630  ******************************************************
2733 - 3187 |    11,634  ************************************************************
3188 - 3642 |       269  *
3643 - 4096 |     7,361  *************************************

Remote batch histogram:
   1        |       282  ************************************************************
   2 - 3    |         0
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test OneParamToClnAsyncVoid ####
Description:       A single client uses 1 thread, with each thread
                   requesting the service to invoke 200,000,000 stateful
                   async calls with 1 parameter to the client.

Results for:       OneParamToClnAsyncVoid
Duration:          17.9 s
Total test RPCs:   200,000,000
RPC/s:             11,171,454
One RPC every:     89.5 ns
Total msgs:        send: 817 | recv: 200,000,029
Avg msg size:      send: 8.0 | recv: 9.9

Total send bytes:  6,505          (363 bytes/sec)
Total recv bytes:  1,983,962,257  (110,818,712 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              2 |              6
 Remote:          |              1 |              0

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |           817 |           817 |            1.0 |  100.00%
 Remote -> Local  |   200,000,029 |       100,670 |         1986.7 |    0.05%

Local batch histogram:
   1        |       817  ************************************************************
   2 - 3    |         0
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |        28
   2 - 457  |       276
 458 - 912  |     1,196  *
 913 - 1367 |     8,322  *******
1368 - 1822 |    12,007  **********
1823 - 2277 |    69,310  ************************************************************
2278 - 2732 |       466
2733 - 3187 |       346
3188 - 3642 |     1,541  *
3643 - 4096 |     7,178  ******


#### Start of test MultiOneParamToSrvAsyncVoid ####
Description:       A single client uses 50 endpoints with each using 2 threads to invoke
                   1,000,000,000 total stateful async calls with one parameter to the
                   service.

#### Results for:  MultiOneParamToSrvAsyncVoid ####
Duration:          32.0 s
Total test RPCs:   1,000,000,000
RPC/s:             31,239,726
One RPC every:     32.0 ns
Total msgs:        send: 1,000,003,126 | recv: 4,953
Avg msg size:      send: 7.0 | recv: 7.1

Total send bytes:  7,002,246,626  (218,748,256 bytes/sec)
Total recv bytes:  35,295         (1,102 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              6 |              6
 Remote:          |              6 |          1,244

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote | 1,000,003,126 |       280,960 |         3559.2 |    0.03%
 Remote -> Local  |         4,953 |         4,741 |            1.0 |   95.72%

Local batch histogram:
   1        |     2,690
   2 - 462  |     4,529  *
 463 - 923  |     1,665
 924 - 1384 |     4,990  *
1385 - 1844 |    14,340  ****
1845 - 2305 |    24,127  ******
2306 - 2766 |     5,310  *
2767 - 3226 |     5,014  *
3227 - 3687 |     3,642  *
3688 - 4147 |   214,653  ************************************************************

Remote batch histogram:
   1        |     4,549  ************************************************************
   2 - 3    |       189  **
   4 - 4    |         3
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test SendAsyncReplyAsyncVoid ####
Description:       A single client uses 4 threads to invoke a total of 200,000,000
                   async calls with 1 parameter to the service, which then invokes
                   another async call with the same parameter back to the client.

Results for:       SendAsyncReplyAsyncVoid
Duration:          16.8 s
Total test RPCs:   200,000,000
RPC/s:             11,938,798
One RPC every:     83.8 ns
Total msgs:        send: 200,002,493 | recv: 200,000,621
Avg msg size:      send: 8.7 | recv: 8.7

Total send bytes:  1,733,257,836  (103,465,072 bytes/sec)
Total recv bytes:  1,735,162,758  (103,578,784 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              2 |             74
 Remote:          |              2 |            643

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   200,002,493 |        78,483 |         2548.4 |    0.04%
 Remote -> Local  |   200,000,621 |       316,140 |          632.6 |    0.16%

Local batch histogram:
   1        |     2,181  ******
   2 - 778  |     3,550  **********
 779 - 1555 |     9,041  ***************************
1556 - 2331 |    19,028  ********************************************************
2332 - 3108 |    20,076  ************************************************************
3109 - 3884 |    11,387  **********************************
3885 - 4661 |    13,216  ***************************************
4662 - 5437 |         1
5438 - 6214 |         2
6215 - 6990 |         1

Remote batch histogram:
   1        |       385
   2 - 573  |   201,766  ************************************************************
 574 - 1144 |    89,072  **************************
1145 - 1715 |    19,226  *****
1716 - 2286 |     3,595  *
2287 - 2858 |     1,125
2859 - 3429 |       355
3430 - 4000 |       133
4001 - 4571 |       481
4572 - 5142 |         2


#### Start of test SmallStructToSrvAsyncVoid ####
Description:       A single client uses 1 thread to invoke 100,000,000 stateful
                   async calls with 1 small struct parameter to the service.

Results for:       SmallStructToSrvAsyncVoid
Duration:          11.7 s
Total test RPCs:   100,000,000
RPC/s:             8,536,336
One RPC every:     117 ns
Total msgs:        send: 100,000,008 | recv: 328
Avg msg size:      send: 11.8 | recv: 24.9

Total send bytes:  1,183,486,416  (101,026,376 bytes/sec)
Total recv bytes:  8,180          (698 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |              0


#### Start of test MediumStructToSrvAsyncVoid ####
Description:       The client uses 1 thread to invoke 50,000,000 async calls with a
                   struct containing 9 fields to the client. The struct is
                   initialized and the fields are set to different values
                   at each iteration.

Results for:       MediumStructToSrvAsyncVoid
Duration:          10.9 s
Total test RPCs:   50,000,000
RPC/s:             4,599,573
One RPC every:     217 ns
Total msgs:        send: 50,000,008 | recv: 402
Avg msg size:      send: 49.3 | recv: 21.8

Total send bytes:  2,466,717,607  (226,916,992 bytes/sec)
Total recv bytes:  8,775          (807 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |              0


#### Start of test MultiEndpointToSrvAsyncVoid ####
Description:       A single client uses 50 endpoints with each using 1 thread to invoke
                   2,000,000,000 total async calls with zero parameters to the service.

#### Results for:  MultiEndpointToSrvAsyncVoid ####
Duration:          28.0 s
Total test RPCs:   2,000,000,000
RPC/s:             71,515,000
One RPC every:     14.0 ns
Total msgs:        send: 2,000,000,300 | recv: 4,490
Avg msg size:      send: 4.0 | recv: 8.1

Total send bytes:  8,004,367,829  (286,216,192 bytes/sec)
Total recv bytes:  36,468         (1,304 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              2 |              1
 Remote:          |              0 |            608

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote | 2,000,000,300 |       546,335 |         3660.8 |    0.03%
 Remote -> Local  |         4,490 |         4,490 |            1.0 |  100.00%

Local batch histogram:
   1        |       301
   2 - 457  |     4,150
 458 - 912  |     1,686
 913 - 1367 |     3,343
1368 - 1822 |     6,075
1823 - 2277 |    46,122  ******
2278 - 2732 |     9,215  *
2733 - 3187 |    13,911  **
3188 - 3642 |    59,783  ********
3643 - 4096 |   401,749  ************************************************************

Remote batch histogram:
   1        |     4,490  ************************************************************
   2 - 3    |         0
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test MultiEndpointToClnAsyncVoid ####
Description:       A single client uses 100 endpoints to the service, using 1 thread
                   per endpoint to invoke 2,000,000,000 total async calls with zero
                   parameters to the remote endpoints.

#### Results for:  MultiEndpointToClnAsyncVoid ####
Duration:          58.7 s
Total test RPCs:   2,000,000,000
RPC/s:             34,086,840
One RPC every:     29.3 ns
Total msgs:        send: 7,127 | recv: 2,000,000,902
Avg msg size:      send: 7.9 | recv: 4.0

Total send bytes:  56,383         (960 bytes/sec)
Total recv bytes:  8,006,417,965  (136,456,736 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              2 |            104
 Remote:          |              0 |              1

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |         7,127 |         7,031 |            1.0 |   98.65%
 Remote -> Local  | 2,000,000,900 |       802,640 |         2491.8 |    0.04%

Local batch histogram:
   1        |     7,071  ************************************************************
   2 - 3    |        19
   4 - 4    |         0
   5 - 5    |         3
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |       932
   2 - 457  |     8,062  *
 458 - 912  |    97,479  *****************
 913 - 1367 |   207,110  *************************************
1368 - 1822 |    56,122  **********
1823 - 2277 |    46,847  ********
2278 - 2732 |    25,176  ****
2733 - 3187 |    17,285  ***
3188 - 3642 |    12,210  **
3643 - 4097 |   331,417  ************************************************************


#### Start of test SmallStructSendReplyAsyncVoid ####
Description:       A single client uses 10 threads to invoke 200,000,000 total async calls
                   with a small 2 field struct to the service. The service then invokes
                   another async call for each struct back to the client.

Results for:       SmallStructSendReplyAsyncVoid
Duration:          20.9 s
Total test RPCs:   200,000,000
RPC/s:             9,553,021
Total msgs:        send: 200,003,594 | recv: 200,000,593
Avg msg size:      send: 22.2 | recv: 22.2
Latency:           min: 57.5 ms | avg: 65.7 ms | max: 107 ms
Latency %:         50% in 62.5 ms | 90% in 78.4 ms | 95% in 81.7 ms | 99% in 91.3 ms

Total send bytes:  4,436,476,970  (211,908,784 bytes/sec)
Total recv bytes:  4,439,468,143  (212,051,664 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              8 |              0
 Remote:          |              6 |              0


#### Start of test MediumStructSendReplyAsyncVoid ####
Description:       A single client uses 10 threads to invoke 100,000,000 total async calls
                   with a medium sized struct containing 9 fields to the service.
                   The service then invokes another async call for each struct
                   back to the client.

Results for:       MediumStructSendReplyAsyncVoid
Duration:          30.3 s
Total test RPCs:   100,000,000
RPC/s:             3,302,560
Total msgs:        send: 100,004,357 | recv: 100,001,053
Avg msg size:      send: 83.4 | recv: 83.5
Latency:           min: 46.3 ms | avg: 51.1 ms | max: 106 ms
Latency %:         50% in 47.8 ms | 90% in 61.7 ms | 95% in 67.3 ms | 99% in 85.3 ms

Total send bytes:  8,344,968,155  (275,597,600 bytes/sec)
Total recv bytes:  8,347,800,755  (275,691,168 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              1 |              0
 Remote:          |              0 |              0


#### Start of test LargeStructSendReplyAsyncVoid ####
Description:       A single client uses 10 threads to invoke 20,000,000 total async calls
                   with a large 14 field struct (with nested structs) to the service.
                   The service then invokes another async call for each struct
                   back to the client.

Results for:       LargeStructSendReplyAsyncVoid
Duration:          14.8 s
Total test RPCs:   20,000,000
RPC/s:             1,355,739
Total msgs:        send: 20,000,545 | recv: 20,000,543
Avg msg size:      send: 202.0 | recv: 202.0
Latency:           min: 78.5 ms | avg: 87.4 ms | max: 101 ms
Latency %:         50% in 87.0 ms | 90% in 91.9 ms | 95% in 93.2 ms | 99% in 95.3 ms

Total send bytes:  4,040,355,884  (273,883,456 bytes/sec)
Total recv bytes:  4,040,365,081  (273,884,064 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |              0


#### Start of test SmallStructVectorParamAsyncVoid ####
Description:       A single client uses 10 endpoints with each using 10 threads to
                   invoke 2,000,000 total async calls with a vector parameter
                   containing 1,000 small 2 field structs.

#### Results for:  SmallStructVectorParamAsyncVoid ####
Duration:          108 s
Total test RPCs:   2,000,000
RPC/s:             18,446
One RPC every:     54.2 us
Total msgs:        send: 2,000,330 | recv: 4,130
Avg msg size:      send: 16,007.9 | recv: 7.9

Total send bytes:  32,021,002,580 (295,331,520 bytes/sec)
Total recv bytes:  32,790         (302 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             17 |             10
 Remote:          |              1 |              5

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |     2,000,330 |       500,164 |            4.0 |   25.00%
 Remote -> Local  |         4,130 |         4,115 |            1.0 |   99.64%

Local batch histogram:
   1        |       161
   2 - 3    |         3
   4 - 4    |   499,932  ************************************************************
   5 - 6    |        55
   7 - 7    |         4
   8 - 8    |         2
   9 - 10   |         2
  11 - 11   |         3
  12 - 12   |         0
  13 - 13   |         2

Remote batch histogram:
   1        |     4,103  ************************************************************
   2 - 3    |        12
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test MediumStructVectorParamAsyncVoid ####
Description:       A single client uses 10 endpoints with each using 10 threads to
                   invoke 200,000 total async calls with a vector parameter
                   containing 1,000 structs, each with 9 fields.

#### Results for:  MediumStructVectorParamAsyncVoid ####
Duration:          51.5 s
Total test RPCs:   200,000
RPC/s:             3,881
One RPC every:     258 us
Total msgs:        send: 200,333 | recv: 1,730
Avg msg size:      send: 75,883.7 | recv: 7.8

Total send bytes:  15,202,002,560 (295,025,856 bytes/sec)
Total recv bytes:  13,570         (263 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             96 |             10
 Remote:          |              1 |              6

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |       200,333 |       200,283 |            1.0 |   99.98%
 Remote -> Local  |         1,730 |         1,716 |            1.0 |   99.19%

Local batch histogram:
   1        |   200,268  ************************************************************
   2 - 3    |         7
   4 - 4    |         2
   5 - 5    |         4
   6 - 6    |         1
   7 - 7    |         2
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |     1,706  ************************************************************
   2 - 3    |        10
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test LargeStructVectorParamAsyncVoid ####
Description:       A single client uses 10 endpoints with each using 10 threads to invoke
                   100,000 total async calls with a vector parameter containing
                   1,000 large 14 field structs (with nested structs).

#### Results for:  LargeStructVectorParamAsyncVoid ####
Duration:          66.4 s
Total test RPCs:   100,000
RPC/s:             1,505
One RPC every:     664 us
Total msgs:        send: 100,330 | recv: 1,730
Avg msg size:      send: 195,365.3 | recv: 7.8

Total send bytes:  19,601,002,560 (295,146,528 bytes/sec)
Total recv bytes:  13,570         (204 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             94 |             10
 Remote:          |              0 |              5

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |       100,330 |       100,294 |            1.0 |   99.96%
 Remote -> Local  |         1,730 |         1,728 |            1.0 |   99.88%

Local batch histogram:
   1        |   100,280  ************************************************************
   2 - 3    |         7
   4 - 4    |         3
   5 - 5    |         3
   6 - 6    |         0
   7 - 7    |         1
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |     1,726  ************************************************************
   2 - 3    |         2
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test SmallStructRTAsyncVal ####
Description:       A single client uses 10 endpoints with each using 10 threads to invoke
                   100,000,000 total Async<T> calls to round-trip a small 2 field struct.
                   For each round-tripped struct, the client invokes a user-supplied
                   callback.

#### Results for:  SmallStructRTAsyncVal ####
Duration:          13.8 s
Total test RPCs:   100,000,000
RPC/s:             7,255,553
Total msgs:        send: 100,002,641 | recv: 100,002,069
Avg msg size:      send: 23.1 | recv: 22.1
Latency:           min: 899 us | avg: 14.1 ms | max: 39.1 ms
Latency %:         50% in 14.0 ms | 90% in 20.5 ms | 95% in 23.5 ms | 99% in 30.1 ms

Total send bytes:  2,313,601,990  (167,864,624 bytes/sec)
Total recv bytes:  2,214,166,647  (160,650,032 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             13 |             45
 Remote:          |              7 |             39

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   100,002,641 |       159,356 |          627.5 |    0.16%
 Remote -> Local  |   100,002,069 |       229,417 |          435.9 |    0.23%

Local batch histogram:
   1        |     4,773  ***
   2 - 320  |    77,144  ************************************************************
 321 - 638  |    30,383  ***********************
 639 - 956  |    12,618  *********
 957 - 1274 |     8,653  ******
1275 - 1592 |     6,981  *****
1593 - 1910 |     5,853  ****
1911 - 2228 |     4,570  ***
2229 - 2546 |     3,313  **
2547 - 2863 |     5,068  ***

Remote batch histogram:
   1        |     3,195  *
   2 - 334  |   110,729  ************************************************************
 335 - 666  |    78,067  ******************************************
 667 - 999  |    19,842  **********
1000 - 1331 |     8,151  ****
1332 - 1663 |     4,295  **
1664 - 1996 |     2,388  *
1997 - 2328 |     1,340
2329 - 2660 |       729
2661 - 2992 |       681


#### Start of test MediumStructRTAsyncVal ####
Description:       A single client uses 10 endpoints with each using 10 threads to
                   invoke 100,000,000 total Async<T> calls to round-trip a struct
                   containing 9 fields. For each round-tripped struct, the
                   client invokes a user-supplied callback.

#### Results for:  MediumStructRTAsyncVal ####
Duration:          44.4 s
Total test RPCs:   100,000,000
RPC/s:             2,251,428
Total msgs:        send: 100,006,633 | recv: 100,003,872
Avg msg size:      send: 85.0 | recv: 84.0
Latency:           min: 26.4 ms | avg: 63.3 ms | max: 1.27 s
Latency %:         50% in 60.5 ms | 90% in 86.5 ms | 95% in 107 ms | 99% in 144 ms

Total send bytes:  8,500,583,919  (191,384,560 bytes/sec)
Total recv bytes:  8,402,970,836  (189,186,864 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             17 |             54
 Remote:          |              7 |             43

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   100,006,633 |       365,315 |          273.8 |    0.37%
 Remote -> Local  |   100,003,872 |       722,619 |          138.4 |    0.72%

Local batch histogram:
   1        |    29,377  ***********************
   2 - 89   |    60,323  ************************************************
  90 - 176  |    45,437  ************************************
 177 - 263  |    75,361  ************************************************************
 264 - 350  |    46,997  *************************************
 351 - 437  |    29,601  ***********************
 438 - 524  |    21,413  *****************
 525 - 611  |    15,634  ************
 612 - 698  |    12,145  *********
 699 - 784  |    29,027  ***********************

Remote batch histogram:
   1        |    27,056  ****
   2 - 89   |   183,904  *****************************
  90 - 177  |   373,579  ************************************************************
 178 - 265  |    71,024  ***********
 266 - 353  |    27,273  ****
 354 - 441  |    13,305  **
 442 - 529  |     8,138  *
 530 - 617  |     5,045
 618 - 705  |     3,555
 706 - 792  |     9,740  *


#### Start of test LargeStructRTAsyncVal ####
Description:       A single client uses 10 endpoints with each using 10 threads to invoke
                   50,000,000 total Async<T> calls to round-trip a large 14 field
                   struct (with nested structs). For each round-tripped struct,
                   the client invokes a user-supplied callback.

#### Results for:  LargeStructRTAsyncVal ####
Duration:          49.4 s
Total test RPCs:   50,000,000
RPC/s:             1,011,787
Total msgs:        send: 50,006,506 | recv: 50,002,808
Avg msg size:      send: 205.0 | recv: 204.1
Latency:           min: 38.9 ms | avg: 132 ms | max: 1.40 s
Latency %:         50% in 111 ms | 90% in 222 ms | 95% in 274 ms | 99% in 455 ms

Total send bytes:  10,251,094,659 (207,438,640 bytes/sec)
Total recv bytes:  10,203,884,055 (206,483,296 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             16 |             22
 Remote:          |              6 |             69

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |    50,006,506 |       258,130 |          193.7 |    0.52%
 Remote -> Local  |    50,002,808 |       729,576 |           68.5 |    1.46%

Local batch histogram:
   1        |    11,886  *******
   2 - 38   |    25,068  ****************
  39 - 74   |    21,281  *************
  75 - 110  |    19,772  ************
 111 - 146  |    18,597  ************
 147 - 183  |    19,939  ************
 184 - 219  |    19,347  ************
 220 - 255  |    15,965  **********
 256 - 291  |    13,898  *********
 292 - 327  |    92,377  ************************************************************

Remote batch histogram:
   1        |     9,974  *
   2 - 37   |   115,795  ***************
  38 - 73   |   443,156  ************************************************************
  74 - 109  |    88,502  ***********
 110 - 145  |    24,469  ***
 146 - 180  |    10,945  *
 181 - 216  |     5,929
 217 - 252  |     3,414
 253 - 288  |     2,142
 289 - 323  |    25,250  ***
```
