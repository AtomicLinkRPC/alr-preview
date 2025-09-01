# Raw Performance Results (Localhost)

The results below are from a subset of ALR's tests used during development:

- This is a different set of tests than the "ALR vs. gRPC" tests.
- In all tests, a single client process and a single service process is used.
- All tests have TLS enabled (OpenSSL).

## Hardware Specifications

- Dell Alienware Aurora R16
- Processor: Intel(R) Core(TM) i9-14900KF [Cores 24] [Logical processors 32]
- Memory: 64 GB
- OS: Windows 11

## Run Summary

```
---------------------------------------------
Service: localhost:51100, TLS: Yes

| Name                             |    Total RPCs |      RPC/s | Lat p99 | Send Bytes/s | Recv Bytes/s | Avg Batch |
| --------------------             | ------------- | ---------- | ------- | ------------ | ------------ | --------- |
| OneParamToSrvAsyncVoid           |   100,000,000 | 15,891,463 |         |  156,281,520 |          350 |   2,922.2 |
| OneParamToClnAsyncVoid           |   200,000,000 | 17,546,332 |         |          384 |  174,032,976 |   2,987.6 |
| MultiOneParamToSrvAsyncVoid      | 1,000,000,000 |114,880,576 |         |  804,449,856 |        4,161 |   3,217.6 |
| SendAsyncReplyAsyncVoid          |   200,000,000 | 21,112,756 |         |  183,002,160 |  183,045,360 |   1,704.6 |
| SmallStructToSrvAsyncVoid        |   100,000,000 | 13,932,204 |         |  164,880,304 |        1,213 |   2,754.1 |
| MediumStructToSrvAsyncVoid       |    50,000,000 |  6,593,046 |         |  325,260,096 |        1,236 |   1,062.2 |
| MultiEndpointToSrvAsyncVoid      | 2,000,000,000 |208,980,096 |         |  836,400,704 |        3,998 |   3,477.7 |
| MultiEndpointToClnAsyncVoid      | 2,000,000,000 |217,338,864 |         |        4,254 |  869,813,568 |   3,794.6 |
| SmallStructSendReplyAsyncVoid    |   200,000,000 | 20,418,520 | 15.8 ms |  449,414,144 |  449,357,088 |     569.3 |
| MediumStructSendReplyAsyncVoid   |   100,000,000 |  9,634,936 | 84.1 ms |  799,744,896 |  799,755,648 |     788.6 |
| LargeStructSendReplyAsyncVoid    |    20,000,000 |  3,656,299 | 68.7 ms |  738,620,032 |  738,621,504 |     323.6 |
| SmallStructVectorParamAsyncVoid  |     2,000,000 |    249,940 |         |4,001,670,144 |        4,097 |       4.0 |
| MediumStructVectorParamAsyncVoid |       200,000 |     41,381 |         |3,145,413,376 |        2,807 |       1.0 |
| LargeStructVectorParamAsyncVoid  |       100,000 |     16,873 |         |3,307,369,984 |        2,289 |       1.0 |
| SmallStructRTAsyncVal            |   100,000,000 | 15,725,022 | 18.3 ms |  367,395,648 |  351,643,264 |     694.6 |
| MediumStructRTAsyncVal           |   100,000,000 | 10,318,442 | 34.7 ms |  871,568,512 |  861,261,056 |     531.0 |
| LargeStructRTAsyncVal            |    50,000,000 |  5,719,521 | 82.2 ms |1,171,037,056 |1,165,346,560 |     279.7 |

---------------------------------------------
```

## Run Details
```
------------------------------------------------------
Note: The reported average message size represents a
fully framed message, including all necessary metadata.
------------------------------------------------------


Hello from Client! Service: localhost:51100, TLS: Yes


#### Start of test OneParamToSrvAsyncVoid ####
Description:       A single client uses 1 thread to invoke 100,000,000 stateful
                   async calls with 1 parameter to the service.

Results for:       OneParamToSrvAsyncVoid
Duration:          6.29 s
Total test RPCs:   100,000,000
RPC/s:             15,891,463
One RPC every:     62.9 ns
Total msgs:        send: 100,000,005 | recv: 272
Avg msg size:      send: 9.8 | recv: 8.1

Total send bytes:  983,430,707    (156,281,520 bytes/sec)
Total recv bytes:  2,205          (350 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |            382

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   100,000,005 |        34,221 |         2922.2 |    0.03%
 Remote -> Local  |           272 |           272 |            1.0 |  100.00%

Local batch histogram:
   1        |         6
   2 - 457  |        27
 458 - 912  |       737  *
 913 - 1367 |       211
1368 - 1822 |       802  *
1823 - 2277 |       661  *
2278 - 2732 |       723  *
2733 - 3187 |    26,650  ************************************************************
3188 - 3642 |       265
3643 - 4096 |     4,139  *********

Remote batch histogram:
   1        |       272  ************************************************************
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
Duration:          11.4 s
Total test RPCs:   200,000,000
RPC/s:             17,546,332
One RPC every:     57.0 ns
Total msgs:        send: 553 | recv: 200,000,029
Avg msg size:      send: 7.9 | recv: 9.9

Total send bytes:  4,387          (384 bytes/sec)
Total recv bytes:  1,983,696,390  (174,032,976 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |            221
 Remote:          |              0 |              0

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |           553 |           551 |            1.0 |   99.64%
 Remote -> Local  |   200,000,029 |        66,943 |         2987.6 |    0.03%

Local batch histogram:
   1        |       549  ************************************************************
   2 - 3    |         2
   4 - 4    |         0
   5 - 5    |         0
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |        25
   2 - 457  |       195
 458 - 912  |     2,465  ***
 913 - 1367 |       627
1368 - 1822 |     1,855  **
1823 - 2277 |     2,387  ***
2278 - 2732 |     1,340  *
2733 - 3187 |    43,380  ************************************************************
3188 - 3642 |       512
3643 - 4096 |    14,157  *******************


#### Start of test MultiOneParamToSrvAsyncVoid ####
Description:       A single client uses 50 endpoints with each using 2 threads to invoke
                   1,000,000,000 total stateful async calls with one parameter to the
                   service.

#### Results for:  MultiOneParamToSrvAsyncVoid ####
Duration:          8.70 s
Total test RPCs:   1,000,000,000
RPC/s:             114,880,576
One RPC every:     8.70 ns
Total msgs:        send: 1,000,003,014 | recv: 5,053
Avg msg size:      send: 7.0 | recv: 7.2

Total send bytes:  7,002,488,477  (804,449,856 bytes/sec)
Total recv bytes:  36,225         (4,161 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              6 |              6
 Remote:          |              4 |          1,209

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote | 1,000,003,014 |       310,795 |         3217.6 |    0.03%
 Remote -> Local  |         5,052 |         4,727 |            1.1 |   93.57%

Local batch histogram:
   1        |     2,006
   2 - 457  |     3,432  *
 458 - 912  |     7,922  **
 913 - 1368 |    26,829  ********
1369 - 1823 |    35,591  **********
1824 - 2278 |    15,263  ****
2279 - 2734 |     6,730  **
2735 - 3189 |     6,920  **
3190 - 3644 |     7,191  **
3645 - 4099 |   198,911  ************************************************************

Remote batch histogram:
   1        |     4,466  ************************************************************
   2 - 3    |       254  ***
   4 - 4    |         6
   5 - 5    |         1
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
Duration:          9.47 s
Total test RPCs:   200,000,000
RPC/s:             21,112,756
One RPC every:     47.4 ns
Total msgs:        send: 200,001,331 | recv: 200,000,935
Avg msg size:      send: 8.7 | recv: 8.7

Total send bytes:  1,733,569,501  (183,002,160 bytes/sec)
Total recv bytes:  1,733,978,911  (183,045,360 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |              0

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   200,001,331 |       117,332 |         1704.6 |    0.06%
 Remote -> Local  |   200,000,935 |       167,848 |         1191.6 |    0.08%

Local batch histogram:
   1        |       942
   2 - 640  |     1,824  *
 641 - 1279 |    56,727  ************************************************************
1280 - 1918 |    19,602  ********************
1919 - 2556 |    18,479  *******************
2557 - 3195 |     7,272  *******
3196 - 3834 |     5,364  *****
3835 - 4472 |     7,118  *******
4473 - 5111 |         2
5112 - 5749 |         2

Remote batch histogram:
   1        |       599
   2 - 599  |     6,819  ***
 600 - 1197 |   110,087  ************************************************************
1198 - 1794 |    20,175  **********
1795 - 2392 |    19,959  **********
2393 - 2989 |     4,235  **
2990 - 3587 |     2,973  *
3588 - 4184 |     2,996  *
4185 - 4782 |         3
4783 - 5379 |         2


#### Start of test SmallStructToSrvAsyncVoid ####
Description:       A single client uses 1 thread to invoke 100,000,000 stateful
                   async calls with 1 small struct parameter to the service.

Results for:       SmallStructToSrvAsyncVoid
Duration:          7.18 s
Total test RPCs:   100,000,000
RPC/s:             13,932,204
One RPC every:     71.8 ns
Total msgs:        send: 100,000,008 | recv: 290
Avg msg size:      send: 11.8 | recv: 30.0

Total send bytes:  1,183,447,384  (164,880,304 bytes/sec)
Total recv bytes:  8,709          (1,213 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |              0


#### Start of test MediumStructToSrvAsyncVoid ####
Description:       The client uses 1 thread to invoke 50,000,000 async calls with a
                   struct containing 9 fields to the client. The struct is
                   initialized and the fields are set to different values
                   at each iteration.

Results for:       MediumStructToSrvAsyncVoid
Duration:          7.58 s
Total test RPCs:   50,000,000
RPC/s:             6,593,046
One RPC every:     152 ns
Total msgs:        send: 50,000,007 | recv: 373
Avg msg size:      send: 49.3 | recv: 25.1

Total send bytes:  2,466,690,386  (325,260,096 bytes/sec)
Total recv bytes:  9,377          (1,236 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |              0


#### Start of test MultiEndpointToSrvAsyncVoid ####
Description:       A single client uses 50 endpoints with each using 1 thread to invoke
                   2,000,000,000 total async calls with zero parameters to the service.

#### Results for:  MultiEndpointToSrvAsyncVoid ####
Duration:          9.57 s
Total test RPCs:   2,000,000,000
RPC/s:             208,980,096
One RPC every:     4.79 ns
Total msgs:        send: 2,000,000,345 | recv: 4,715
Avg msg size:      send: 4.0 | recv: 8.1

Total send bytes:  8,004,596,701  (836,400,704 bytes/sec)
Total recv bytes:  38,270         (3,998 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              2 |              1
 Remote:          |              0 |             97

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote | 2,000,000,345 |       575,098 |         3477.7 |    0.03%
 Remote -> Local  |         4,715 |         4,715 |            1.0 |  100.00%

Local batch histogram:
   1        |       407
   2 - 457  |     5,643
 458 - 912  |    10,869  *
 913 - 1367 |    14,276  **
1368 - 1822 |    27,092  ***
1823 - 2277 |    63,155  *********
2278 - 2732 |    21,236  ***
2733 - 3187 |    10,491  *
3188 - 3642 |    11,188  *
3643 - 4097 |   410,745  ************************************************************

Remote batch histogram:
   1        |     4,715  ************************************************************
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
Duration:          9.20 s
Total test RPCs:   2,000,000,000
RPC/s:             217,338,864
One RPC every:     4.60 ns
Total msgs:        send: 4,990 | recv: 2,000,000,902
Avg msg size:      send: 7.8 | recv: 4.0

Total send bytes:  39,149         (4,254 bytes/sec)
Total recv bytes:  8,004,215,771  (869,813,568 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              2 |            171
 Remote:          |              0 |              1

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |         4,990 |         4,883 |            1.0 |   97.86%
 Remote -> Local  | 2,000,000,900 |       527,062 |         3794.6 |    0.03%

Local batch histogram:
   1        |     4,921  ************************************************************
   2 - 3    |        14
   4 - 4    |         2
   5 - 5    |         6
   6 - 6    |         0
   7 - 7    |         0
   8 - 8    |         0
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |       937
   2 - 457  |     2,333
 458 - 912  |     5,121
 913 - 1367 |     5,291
1368 - 1822 |    18,195  **
1823 - 2277 |    19,658  **
2278 - 2732 |     9,141  *
2733 - 3187 |     5,930
3188 - 3642 |     7,176
3643 - 4097 |   453,280  ************************************************************


#### Start of test SmallStructSendReplyAsyncVoid ####
Description:       A single client uses 10 threads to invoke 200,000,000 total async calls
                   with a small 2 field struct to the service. The service then invokes
                   another async call for each struct back to the client.

Results for:       SmallStructSendReplyAsyncVoid
Duration:          9.80 s
Total test RPCs:   200,000,000
RPC/s:             20,418,520
Total msgs:        send: 200,002,834 | recv: 200,003,381
Avg msg size:      send: 22.0 | recv: 22.0
Latency:           min: 89.0 us | avg: 1.12 ms | max: 31.5 ms
Latency %:         50% in 551 us | 90% in 1.83 ms | 95% in 2.92 ms | 99% in 15.8 ms

Total send bytes:  4,402,024,974  (449,414,144 bytes/sec)
Total recv bytes:  4,401,465,699  (449,357,088 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              1 |              0
 Remote:          |              0 |              0


#### Start of test MediumStructSendReplyAsyncVoid ####
Description:       A single client uses 10 threads to invoke 100,000,000 total async calls
                   with a medium sized struct containing 9 fields to the service.
                   The service then invokes another async call for each struct
                   back to the client.

Results for:       MediumStructSendReplyAsyncVoid
Duration:          10.4 s
Total test RPCs:   100,000,000
RPC/s:             9,634,936
Total msgs:        send: 100,001,157 | recv: 100,001,053
Avg msg size:      send: 83.0 | recv: 83.0
Latency:           min: 4.20 ms | avg: 28.6 ms | max: 95.0 ms
Latency %:         50% in 22.3 ms | 90% in 65.3 ms | 95% in 74.2 ms | 99% in 84.1 ms

Total send bytes:  8,300,468,529  (799,744,896 bytes/sec)
Total recv bytes:  8,300,580,257  (799,755,648 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              0 |            969


#### Start of test LargeStructSendReplyAsyncVoid ####
Description:       A single client uses 10 threads to invoke 20,000,000 total async calls
                   with a large 14 field struct (with nested structs) to the service.
                   The service then invokes another async call for each struct
                   back to the client.

Results for:       LargeStructSendReplyAsyncVoid
Duration:          5.47 s
Total test RPCs:   20,000,000
RPC/s:             3,656,299
Total msgs:        send: 20,000,543 | recv: 20,000,543
Avg msg size:      send: 202.0 | recv: 202.0
Latency:           min: 14.7 ms | avg: 37.2 ms | max: 70.9 ms
Latency %:         50% in 33.5 ms | 90% in 56.5 ms | 95% in 62.2 ms | 99% in 68.7 ms

Total send bytes:  4,040,260,327  (738,620,032 bytes/sec)
Total recv bytes:  4,040,268,217  (738,621,504 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |              0 |              0
 Remote:          |              1 |              0


#### Start of test SmallStructVectorParamAsyncVoid ####
Description:       A single client uses 10 endpoints with each using 10 threads to
                   invoke 2,000,000 total async calls with a vector parameter
                   containing 1,000 small 2 field structs.

#### Results for:  SmallStructVectorParamAsyncVoid ####
Duration:          8.00 s
Total test RPCs:   2,000,000
RPC/s:             249,940
One RPC every:     4.00 us
Total msgs:        send: 2,000,345 | recv: 4,130
Avg msg size:      send: 16,007.7 | recv: 7.9

Total send bytes:  32,021,002,706 (4,001,670,144 bytes/sec)
Total recv bytes:  32,790         (4,097 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             13 |             10
 Remote:          |              5 |            366

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |     2,000,345 |       500,146 |            4.0 |   25.00%
 Remote -> Local  |         4,130 |         4,026 |            1.0 |   97.48%

Local batch histogram:
   1        |       125
   2 - 3    |        28
   4 - 4    |   499,903  ************************************************************
   5 - 5    |        42
   6 - 6    |        30
   7 - 8    |        12
   9 - 9    |         2
  10 - 10   |         3
  11 - 11   |         0
  12 - 12   |         1

Remote batch histogram:
   1        |     3,956  ************************************************************
   2 - 3    |        63
   4 - 4    |         1
   5 - 5    |         2
   6 - 6    |         1
   7 - 7    |         2
   8 - 8    |         1
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0


#### Start of test MediumStructVectorParamAsyncVoid ####
Description:       A single client uses 10 endpoints with each using 10 threads to
                   invoke 200,000 total async calls with a vector parameter
                   containing 1,000 structs, each with 9 fields.

#### Results for:  MediumStructVectorParamAsyncVoid ####
Duration:          4.83 s
Total test RPCs:   200,000
RPC/s:             41,381
One RPC every:     24.2 us
Total msgs:        send: 200,345 | recv: 1,730
Avg msg size:      send: 75,879.1 | recv: 7.8

Total send bytes:  15,202,002,581 (3,145,413,376 bytes/sec)
Total recv bytes:  13,570         (2,807 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             89 |             10
 Remote:          |              5 |            104

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |       200,345 |       200,236 |            1.0 |   99.95%
 Remote -> Local  |         1,730 |         1,669 |            1.0 |   96.47%

Local batch histogram:
   1        |   200,213  ************************************************************
   2 - 3    |         7
   4 - 4    |         5
   5 - 5    |         1
   6 - 6    |         1
   7 - 7    |         3
   8 - 8    |         2
   9 - 9    |         2
  10 - 10   |         2
  11 - 11   |         0

Remote batch histogram:
   1        |     1,621  ************************************************************
   2 - 3    |        45  *
   4 - 4    |         2
   5 - 5    |         1
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
Duration:          5.93 s
Total test RPCs:   100,000
RPC/s:             16,873
One RPC every:     59.3 us
Total msgs:        send: 100,333 | recv: 1,730
Avg msg size:      send: 195,359.5 | recv: 7.8

Total send bytes:  19,601,002,581 (3,307,369,984 bytes/sec)
Total recv bytes:  13,570         (2,289 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             89 |             10
 Remote:          |              4 |            455

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |       100,333 |       100,280 |            1.0 |   99.95%
 Remote -> Local  |         1,730 |         1,683 |            1.0 |   97.28%

Local batch histogram:
   1        |   100,253  ************************************************************
   2 - 3    |        21
   4 - 4    |         3
   5 - 5    |         1
   6 - 6    |         1
   7 - 7    |         0
   8 - 8    |         1
   9 - 9    |         0
  10 - 10   |         0
  11 - 11   |         0

Remote batch histogram:
   1        |     1,642  ************************************************************
   2 - 3    |        40  *
   4 - 4    |         1
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
Duration:          6.36 s
Total test RPCs:   100,000,000
RPC/s:             15,725,022
Total msgs:        send: 100,001,941 | recv: 100,002,147
Avg msg size:      send: 23.4 | recv: 22.4
Latency:           min: 137 us | avg: 4.91 ms | max: 33.3 ms
Latency %:         50% in 4.19 ms | 90% in 9.55 ms | 95% in 11.6 ms | 99% in 18.3 ms

Total send bytes:  2,336,375,939  (367,395,648 bytes/sec)
Total recv bytes:  2,236,202,102  (351,643,264 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             12 |             52
 Remote:          |              8 |             50

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   100,001,941 |       170,078 |          588.0 |    0.17%
 Remote -> Local  |   100,002,147 |       143,962 |          694.6 |    0.14%

Local batch histogram:
   1        |     2,245  *
   2 - 319  |    86,873  ************************************************************
 320 - 637  |    41,707  ****************************
 638 - 955  |    10,738  *******
 956 - 1273 |     5,687  ***
1274 - 1591 |     4,524  ***
1592 - 1909 |     4,223  **
1910 - 2227 |     4,065  **
2228 - 2545 |     3,191  **
2546 - 2862 |     6,825  ****

Remote batch histogram:
   1        |     1,686  **
   2 - 334  |    48,884  ************************************************************
 335 - 667  |    46,255  ********************************************************
 668 - 999  |    18,096  **********************
1000 - 1332 |     8,674  **********
1333 - 1664 |     5,797  *******
1665 - 1997 |     4,311  *****
1998 - 2329 |     3,276  ****
2330 - 2662 |     2,406  **
2663 - 2994 |     4,577  *****


#### Start of test MediumStructRTAsyncVal ####
Description:       A single client uses 10 endpoints with each using 10 threads to
                   invoke 100,000,000 total Async<T> calls to round-trip a struct
                   containing 9 fields. For each round-tripped struct, the
                   client invokes a user-supplied callback.

#### Results for:  MediumStructRTAsyncVal ####
Duration:          9.69 s
Total test RPCs:   100,000,000
RPC/s:             10,318,442
Total msgs:        send: 100,002,388 | recv: 100,002,269
Avg msg size:      send: 84.5 | recv: 83.5
Latency:           min: 174 us | avg: 10.7 ms | max: 86.4 ms
Latency %:         50% in 9.03 ms | 90% in 20.6 ms | 95% in 25.2 ms | 99% in 34.7 ms

Total send bytes:  8,446,706,352  (871,568,512 bytes/sec)
Total recv bytes:  8,346,813,017  (861,261,056 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             10 |             58
 Remote:          |             10 |             59

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |   100,002,388 |       188,328 |          531.0 |    0.19%
 Remote -> Local  |   100,002,269 |       203,792 |          490.7 |    0.20%

Local batch histogram:
   1        |     2,184  *
   2 - 88   |    10,202  ******
  89 - 175  |    10,457  ******
 176 - 262  |    22,008  **************
 263 - 349  |    18,626  ************
 350 - 436  |    10,454  ******
 437 - 523  |     8,590  *****
 524 - 610  |     7,573  ****
 611 - 697  |     6,777  ****
 698 - 783  |    91,457  ************************************************************

Remote batch histogram:
   1        |       967
   2 - 90   |    11,416  ********
  91 - 178  |    18,503  *************
 179 - 266  |    38,853  ***************************
 267 - 354  |    14,654  **********
 355 - 443  |    11,396  ********
 444 - 531  |     9,088  ******
 532 - 619  |     7,690  *****
 620 - 707  |     6,822  ****
 708 - 795  |    84,403  ************************************************************


#### Start of test LargeStructRTAsyncVal ####
Description:       A single client uses 10 endpoints with each using 10 threads to invoke
                   50,000,000 total Async<T> calls to round-trip a large 14 field
                   struct (with nested structs). For each round-tripped struct,
                   the client invokes a user-supplied callback.

#### Results for:  LargeStructRTAsyncVal ####
Duration:          8.74 s
Total test RPCs:   50,000,000
RPC/s:             5,719,521
Total msgs:        send: 50,002,511 | recv: 50,002,200
Avg msg size:      send: 204.7 | recv: 203.7
Latency:           min: 259 us | avg: 21.7 ms | max: 141 ms
Latency %:         50% in 14.6 ms | 90% in 46.6 ms | 95% in 56.2 ms | 99% in 82.2 ms

Total send bytes:  10,237,195,218 (1,171,037,056 bytes/sec)
Total recv bytes:  10,187,449,416 (1,165,346,560 bytes/sec)

Max Alloc Msgs:   | Max Send Alloc | Max Recv Alloc
 Local:           |             16 |            117
 Remote:          |             12 |             79

Batching:         | Messages Sent | Socket Writes | Avg Batch Msgs |  Percent
 Local  -> Remote |    50,002,511 |       178,801 |          279.7 |    0.36%
 Remote -> Local  |    50,002,198 |       218,863 |          228.5 |    0.44%

Local batch histogram:
   1        |     1,613
   2 - 38   |     4,193  *
  39 - 75   |     4,187  *
  76 - 111  |     4,217  *
 112 - 148  |     4,458  *
 149 - 184  |     5,716  **
 185 - 221  |     7,310  ***
 222 - 257  |     6,079  **
 258 - 294  |     5,027  **
 295 - 330  |   136,001  ************************************************************

Remote batch histogram:
   1        |       862
   2 - 38   |     9,108  ****
  39 - 75   |    13,177  ******
  76 - 111  |    29,237  ***************
 112 - 148  |    20,330  **********
 149 - 184  |     9,647  ****
 185 - 221  |     7,952  ****
 222 - 257  |     6,323  ***
 258 - 294  |     5,574  **
 295 - 330  |   116,653  ************************************************************
```
