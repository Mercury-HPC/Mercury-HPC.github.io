---
layout: post
title:  "Shared-memory plugin for Mercury"
date:   2018-10-29 17:40:10
categories: documentation
---

Mercury defines a network abstraction layer that allows definition of plugins
for high-speed networks. In the case of intra-node communication, the most
efficient way of accessing memory between processes is to use shared-memory.

## Introduction

Mercury's network abstraction layer is designed to take advantage of
high-speed networks by providing two types of messaging: _small messaging_
(low-latency) and _large transfers_ (high-bandwidth).
Small messages fall into two categories: unexpected and expected. Large
transfers use a one-sided mechanism for efficient access to remote memory,
enabling transfers to be made without intermediate copy.
The interface currently provides implementations for BMI, MPI, CCI and OFI libfabric, which
are themselves abstractions on top of the common HPC network protocols.
When doing communication locally on the same node between separate processes,
it is desirable to bypass the loopback network interface and directly
access/share memory regions, thereby increasing bandwidth and
decreasing latency of the transfers. While the CCI plugin already provides a
shared memory transport, it currently has some
inherent limitations: handling is explicit and not transparent to the user; it
requires explicit use of the CCI external library; it uses a connection thread internally;
CCI only provides RMA registration per contiguous segment region which prevents
users from transferring non-contiguous regions in one single RMA operation.

It is also worth noting that other libraries such as libfabric support
shared memory optimization, though having an independent shared-memory plugin
that can be used with other transports at the same time, in a
transparent manner, (and that can provide optimal performance)
is an important feature for mercury.

This document assumes that the reader already has knowledge of Mercury and its
layers (NA / HG / HG Bulk). Adding shared memory support to mercury is
referenced under the GitHub issue [#75].

## Requirements

The design of this plugin follows three main requirements:

- Reuse existing shared-memory technology and concepts to the degree possible;
- Make progress on conventional and shared-memory communication simultaneously
  without busy waiting;
- Use plugin transparently (i.e., ability to use the same address for both
  conventional and shared memory communication).

## SM Plugin Architecture

The plugin can be decomposed into multiple parts: _lookup_, _short messages_
(unexpected and expected); _RMA transfers_; _progress_.
Lookup, connection and set up of exchange channels between processes is
initialized through UNIX domain sockets. One listening socket is created on
the listening processes, which then accept connections in a non blocking manner
and exchange address channel information between peers.

### Short Messages
For short messages, we use an mmap'ed POSIX shared memory object combined to a
signaling interface that allows either party to be notified when a new message
has been written into the shared memory space. One region is created per
listening interface that allows processes to post short messages, by first
reserving a buffer _atomically_ in the mmap'ed shared memory object and then
signaling the destination that it has been copied (in a non blocking manner).
This requires a 2-way signaling channel (based on either an `eventfd` or a named
pipe if the former is not available)
per process/connection (`eventfd` descriptors can be exchanged over
UNIX domain sockets through ancillary data).
The number of buffers that are mmap'ed is fixed and its global size pre-defined
to 64 (the size of a 64-bit atomic value) pages of 4 KB.

CCI and MPICH for instance use different methods, CCI exposes 64 lines of
64-byte buffers with a default maximum send size of 256 bytes, MPICH exposes
8 buffers of 32 KB and copies messages in a pipelined fashion.
The sender waits until there is an empty buffer, then fills it in and marks the
number of bytes copied. CCI on the other hand uses an mmap'ed ring buffer for
message headers and fails when the total number of buffer has been exhausted.
In our case, we combine both approaches, buffers are reserved and marked with
the number of bytes copied.

### Large Transfers
Large transfers do not require explicit signaling between processes and CMA
(cross-memory attach) can be used directly without mmap (on Linux platforms that support [CMA][cma] as of kernel v3.2, see also this [page][YAMA] for security
requirements).
In that case, the only arguments needed are the remote process ID as well as
a local and remote `iovec` that describe the memory segments that are to
be transferred. Note that in this case, the SM plugin performs better
by registering non-contiguous memory segments and does a single
scatter/gather operation between local and remote regions.
Note also that passing local and remote handles requires the base address of
the memory regions that needs to be accessed to be first communicated to
the issuing process, though this is part of the NA infrastructure, which also
implies serialization, exchange and deserialization of remote memory handles.

Progress is made through `epoll()`/`poll()`/`kqueue()` only on connections and
message signaling interfaces that are registered to a polling set,
RMA operations using CMA complete immediately
after the `process_vm_readv()` or `process_vm_writev()` calls complete.

## Simultaneous Progress

The second requirement is the ability to make progress simultaneously over
different plugins without busing polling. In order to do that, the easiest
and most efficient way is to make use of the kernel's existing file
descriptor infrastructure.
That method consists of exposing a file descriptor from the NA plugin and
add it to a global polling set of NA plugin file descriptors.
Making progress on that set of file descriptors is then made through
the `epoll()`/`poll()`/`kqueue()` system calls.
When one file descriptor _wakes up_, the NA layer can then determine which
plugin that file descriptor belongs to and enter the corresponding
progress functions to complete the operation.
When the NA plugins supports it (e.g., SM, CCI, OFI), progress can be
made without any latency overhead, which would otherwise be introduced by
entering multiple `NA_Progress()` calls in order to do busy polling.

## Transparent Use

Transparent use is essential for a shared-memory plugin as
adding an explicit condition to switch between local and remote
operations is not always reasonable, both in terms of performance and
convenience.
This mode of operation can be enabled within Mercury through the CMake option
`MERCURY_USE_SM_ROUTING`.

When enabled, Mercury internally creates a separate NA SM class and makes
progress on that class as well as on the requested class (e.g., OFI, CCI).
Local node detection is enabled by generating a UUID for the node and storing
that UUID in the `NA_SM_TMP_DIRECTORY/NA_SM_SHM_PREFIX` directory. When a lookup
call is issued, the string passed is parsed and the UUID compared with the
locally stored UUID.

## Performance

Below is a performance comparison of the shared-memory plugin when using both
wait and busy spin mechanisms, CCI v2.2, libfabric v1.7.0a1 and mercury v1.0.0.
The following plot shows the RPC average time with one RPC in flight:

<figure>
  <img src="/assets/shared-memory/rpc_time1.svg" alt="RPC time 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="/assets/shared-memory/rpc_time16.svg" alt="RPC time 16" width="80%">
</figure>

The following plot shows the RPC with *pull* bulk transfer performance
compared to existing plugins with various transfer sizes:

<figure>
  <img src="/assets/shared-memory/write_bw1.svg" alt="Write Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="/assets/shared-memory/write_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

The following plot shows the RPC with *push* bulk transfer performance
compared to existing plugins with various transfer sizes:

<figure>
  <img src="/assets/shared-memory/read_bw1.svg" alt="Read Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="/assets/shared-memory/read_bw16.svg" alt="Read Bandwidth 16" width="80%">
</figure>


[#75]: https://github.com/mercury-hpc/mercury/issues/75
[cma]: https://lwn.net/Articles/405284/
[YAMA]: https://www.kernel.org/doc/Documentation/security/Yama.txt
