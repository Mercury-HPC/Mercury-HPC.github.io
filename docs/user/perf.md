---
title:  "Diagnosing Performance"
---

Measuring performance both at the Network Abstraction (NA) level and Mercury
(RPC) level is always a good idea to ensure proper configuration of the network
and that the expected performance can be achieved. To do that, Mercury comes
with a set of performance measurement utilities.

!!! note

    To ensure that the perf utilities are compiled, `BUILD_TESTING` and
    `BUILD_TESTING_PERF` must be turned `ON` in CMake options. Additionally, to
    enable MPI support with HG perf tests, `MERCURY_TESTING_ENABLE_PARALLEL`
    must also be set to `ON`. Performance utilities are placed into the `bin`
    directory relative to the install prefix.

## NA Perf

NA perf tests are composed of: `na_lat`, `na_bw_get`, `na_bw_put` (clients) and
`na_perf_server` (server). For all of the tests, `na_perf_server` must first
be launched:

```
na_perf_server -c ofi -p tcp -b
```

This will generate a `port.cfg` within the current working directory. This file
will be used by the client to contact the server and must reside within the same
working directory when launching the client.

To measure latency of the network, run:

```
na_lat -c ofi -p tcp -b -l 1000
```

To measure bandwidth, run:

```
na_bw_get -c ofi -p tcp -b -l 1000
```

## HG Perf

HG perf tests are composed of: `hg_rate`, `hg_bw_read`, `hg_bw_write` (clients)
and `hg_perf_server` (server). For all of the tests, `hg_perf_server` must first
be launched:

```
hg_perf_server -c ofi -p tcp -b
```

This will generate a `port.cfg` within the current working directory. This file
will be used by the client to contact the server and must reside within the same
working directory when launching the client.

To measure RPC rate, run:

```
hg_rate -c ofi -p tcp -b -l 1000
```

To measure bulk RPC bandwidth, run:

```
hg_bw_write -c ofi -p tcp -b -l 1000
``` 

## Example: DAOS Use Case

Most often it is desirable to replicate a specific workflow or communication
pattern. In the case of a storage system like DAOS that follows a
client-server model, there are two levels of parallelism with servers being both
multithreaded and distributed over multiple nodes. Clients on the other hand
may also use threads but they are in most cases distributed over multiple nodes.
The Mercury performance benchmarks (compiled with parallel mode enabled)
can be used to replicate this use case.
To simulate multithreaded servers running on multiple nodes, `hg_perf_server`
can be launched with the following parameters:

```
mpirun -np 2 --bind-to numa --map-by numa hg_perf_server -c ofi -p tcp -b -C 16
```

`-C` controls the number of Mercury classes/contexts per process (equivalent to
the number of DAOS storage targets) while `-np` controls the number of
processes (equivalent to the number of DAOS engines). Since classes/contexts
operate independently there is no restriction
on either of these values but as a general rule, it is best to restrict `-C` to
the number of available cores per NUMA node (which is also the reason why tasks
are mapped by and bound to NUMA nodes in that case).

Similarly, to launch multiple clients, one would launch:

```
mpirun -hostfile hosts.txt --bind-to hwthread --map-by numa hg_bw_write -c ofi -p tcp -b -l 100
```

The resulting bandwidth in that case is the aggregate bandwidth
(or RPC rate when using `hg_rate`). Without thread,
it is best to bind the client to HW threads or cores to ensure that tasks are
not migrated and the performance remains consistent.

!!! tip

    * `-x` can be used to control the number of RPCs in flight
    * `-b` forces busy spin and prevents any wait/sleep
    * `-w` changes the RMA window size and the number of RMAs in flight,
    increasing it will saturate bandwidth but be wary of memory usage
    * `-y` / `-z` controls the min/max buffer size, setting it to the same value
    allows for testing a specific buffer size

    See `--help` for additional options