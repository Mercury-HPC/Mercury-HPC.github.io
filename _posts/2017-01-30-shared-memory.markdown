---
layout: post
title:  "Shared-memory plugin for Mercury"
date:   2017-01-30 14:31:03
categories: documentation
---

Mercury defines a network abstraction layer that allows definition of plugins
for high-speed networks. In the case of intranode communication, an efficient
way of accessing memory between processes must be used (e.g., through shared
memory).

## Introduction

Mercury's network abstraction layer is designed to take advantage of
high-speed networks by providing two types of messaging: _small messaging_
(low-latency) and _large transfers_ (high-bandwidth).
Small messages fall into two categories: unexpected and expected. Large
transfers use a one-sided mechanism for efficient access to remote memory,
enabling transfers to be made without intermediate copy.
The interface currently provides implementations for BMI, MPI, and CCI, which
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

It is also worth noting that other libraries such as libfabric may also support
shared memory optimization in the future, though having an independent shared
memory plugin that can be used with other transports at the same time, in a
transparent manner, (and than can provide optimal performance)
is an important feature for mercury.

This document assumes that the reader already has knowledge of Mercury and its
layers (NA / HG / HG Bulk). Adding shared memory support to mercury is
referenced under the GitHub issue [#75].

## Requirements

The design of this plugin follows three main requirements:

- Reuse existing shared memory technology and concepts to the degree possible;
- Make progress on conventional and shared memory communication simultaneously
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
reserving a buffer (_atomically_) in the mmap'ed shared memory object and then
signaling to the destination that it has been copied (in a non blocking manner).
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
When the NA plugins supports it (e.g., SM, CCI), progress can be
made without any latency overhead, which would otherwise be introduced by
entering multiple `NA_Progress()` calls in order to do busy polling.

## Transparent Use

Transparent use is essential for a shared-memory plugin as users cannot pay
the cost of adding an explicit condition to switch between local and remote
operations (both in terms of performance and convenience).
One simple way of adding transparent shared-memory operations is to
determine shared memory address information on lookup calls, which will in
turn drive subsequent mercury calls to use or not use the shared memory NA
plugin class. Multiple solutions can be used to actually determine whether an
address is local or not, although the most simple solution
consists of using the addressing convention that is provided by the NA layer,
i.e., retrieve the corresponding hostname and determine whether that hostname
is a local hostname. Other solutions are considered as well.
A mode to completely disable that plugin can also be provided through a
CMake option.

## Performance

TBD

## Conclusion and Future work
Implementation of a first version of this shared memory plugin has been done and
provides good performance results. No threads are used internally and progress
is made in a non blocking manner without busy polling. RMA transfers of non
contiguous memory segments (i.e., scatter/gather operations) are also supported.

Implementation of transparent use features is still
on going and has not been completed yet.

[#75]: https://github.com/mercury-hpc/mercury/issues/75
[cma]: https://lwn.net/Articles/405284/
[YAMA]: https://www.kernel.org/doc/Documentation/security/Yama.txt
