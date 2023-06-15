---
title: Network Abstraction Layer
---

The _network abstraction_ (NA) layer is internally used by both the RPC layer
and the bulk layer.
The NA layer uses a plugin mechanism so that support for various network
protocols can be easily added and selected at runtime.

!!! info

    The NA interface should not be directly used if you intend to use
    Mercury's RPC layer (HG calls). In that case, please
    directly jump to the [available plugins](#available-plugins)
    section for a list of plugins that can be used when initializing
    Mercury&mdash;Mercury's initialization is then further described in the
    [RPC layer](hg.md) section.

## NA Interface

NA provides a minimal set of function calls that abstract the underlying network
fabric and that can be used to provide:
*target address lookup*, *point-to-point messaging* with both unexpected and
expected messaging, *remote memory access (RMA)*, *progress* and *cancelation*.
The API is non-blocking and uses a callback mechanism
so that upper layers can provide asynchronous execution more easily: when progress
is made (either internally or after a call to `NA_Progress()`) and an operation
completes, the user callback is placed onto a completion queue. The callback
can then be dequeued and separately executed after a call to `NA_Trigger()`.

### Initialization

When using NA, the first step of a program should consist of
initializing the NA interface and selecting an underlying plugin that will be
used. Initializing the NA interface with a specified `info_string`
results in the creation of a new `na_class_t` object. Please refer to
the [available plugins](#available-plugins) section for more information on the
`info_string` format. Additionally, it is possible to specify whether the
`na_class_t` object will be listening or
not&mdash;this is the _only_ time where a "server" specific behavior is defined, all
subsequent calls do not make any distinction between a "client" and a "server"
and instead only use the concept of _origin_ and _target_. It is worth noting,
however, that the `listen` flag may have an effect on the resources that are
allocated and that the address passed through `info_string` will be used to
create an endpoint that remote peers can access.

```C
na_class_t *
NA_Initialize(const char *info_string, bool listen);
```

If a more specific behavior is required, the following call can also be used to
pass specific init options.

| Option | Description |
|-----|----|
| `ip_subnet` | Preferred IP subnet to use. |
| `auth_key` |  Authorization key that can be used for communication. All processes should use the same key in order to communicate. |
| `max_unexpected_size` | Max unexpected size hint that can be passed to control the size of unexpected messages. |
| `max_expected_size` | Max expected size hint that can be passed to control the size of unexpected messages. |
| `progress_mode` | Progress mode flag. Setting `NA_NO_BLOCK` will force busy-spin on progress and remove any wait/notification calls. |
| `addr_format` | Preferred address format. Can be set to `NA_ADDR_IPV4`, `NA_ADDR_IPV6` or `NA_ADDR_NATIVE`. |
| `max_contexts` | Maximum number of contexts that are expected to be created. |
| `thread_mode` | Thread mode flags. Setting `NA_THREAD_MODE_SINGLE` will relax thread-safety requirements. |
| `request_mem_device` | Request support for tranfers to/from memory devices (e.g., GPU, etc). |

```C
struct na_init_info {
    const char *ip_subnet;
    const char *auth_key;
    size_t max_unexpected_size;
    size_t max_expected_size;
    uint8_t progress_mode;
    enum na_addr_format addr_format;
    uint8_t max_contexts;
    uint8_t thread_mode;
    bool request_mem_device;
};

na_class_t *
NA_Initialize_opt2(const char *info_string, bool listen, unsigned int version,
    const struct na_init_info *na_init_info);
```

The `na_class_t` object created from these initialization calls should later be
released with a call to:

```C
na_return_t
NA_Finalize(na_class_t *na_class);
```

Once the interface has been initialized, a context within this plugin must be
created, which internally creates and associates a completion queue for the
operations:

```C
na_context_t *
NA_Context_create(na_class_t *na_class);
```

It can then be destroyed using:

```C
na_return_t
NA_Context_destroy(na_class_t *na_class, na_context_t *context);
```

### Target Address Lookup

To communicate with a target, one must first get its address.
The most convenient and safe way of doing that is by calling on the target:

```C
na_return_t
NA_Addr_self(na_class_t *na_class, na_addr_t **addr_p);
```

And then convert that address to a string using:

```C
na_return_t
NA_Addr_to_string(na_class_t *na_class, char *buf, size_t *buf_size_p, na_addr_t *addr);
```

The string can then be exchanged to other processes through out-of-band
mechanisms (e.g., using a file, etc),
which can then look up the target using the function:

```C
na_return_t
NA_Addr_lookup(na_class_t *na_class, const char *name, na_addr_t **addr_p);
```

All addresses must then be freed using:

```C
na_return_t
NA_Addr_free(na_class_t *na_class, na_addr_t *addr);
```

### Point-to-point Messaging

Point-to-point messaging in NA is always non-blocking with completion callbacks
being executed after a call to `NA_Trigger()` (once the operation has completed
and been placed onto the completion queue).
NA supports two separates modes for sending and receiving messages: either
*unexpected* or *expected*. Expected messages should always have their
receive pre-posted. Though messages may be dropped without notification if that
is not the case, they are usually still queued and later processed.
Unexpected messages on the other handle never require receives to be pre-posted
and messages are also
allowed to be dropped (though once again plugins usually do queue them).
Both types of messages are tagged messages that take the same arguments for sends:
```C
na_return_t
NA_Msg_send_unexpected(na_class_t *na_class, na_context_t *context,
    na_cb_t callback, void *arg, const void *buf, size_t buf_size,
    void *plugin_data, na_addr_t *dest_addr, uint8_t dest_id, na_tag_t tag,
    na_op_id_t *op_id);

na_return_t
NA_Msg_send_expected(na_class_t *na_class, na_context_t *context,
    na_cb_t callback, void *arg, const void *buf, size_t buf_size,
    void *plugin_data, na_addr_t *dest_addr, uint8_t dest_id, na_tag_t tag,
    na_op_id_t *op_id);
```

And only mostly differ in their receive operation:

```C
na_return_t
NA_Msg_recv_unexpected(na_class_t *na_class, na_context_t *context,
    na_cb_t callback, void *arg, void *buf, size_t buf_size,
    void *plugin_data, na_op_id_t *op_id);

na_return_t
NA_Msg_recv_expected(na_class_t *na_class, na_context_t *context,
    na_cb_t callback, void *arg, void *buf, size_t buf_size,
    void *plugin_data, na_addr_t source_addr, uint8_t source_id,
    na_tag_t tag, na_op_id_t *op_id);
```

One will only match with a specific `source_addr` and `tag` while the other
will match with *any* source and tag, which can then later be retrieved from the
callback info.

```C
struct na_cb_info_recv_unexpected {
    size_t actual_buf_size;
    na_addr_t *source;
    na_tag_t tag;
};
```

Note that for best performance, `NA_Msg_buf_alloc()` and `NA_Msg_buf_free()`
may be used to allocate send and receive buffers.

### Remote Memory Access

Remote memory access requires host memory that is desired to be
accessed to be first registered with the NA layer. This is done in two steps,
by first creating a handle that describes the memory buffer that is to
be registered and calling `NA_Mem_register()` on it.

```C
na_return_t
NA_Mem_handle_create(na_class_t *na_class, void *buf, size_t buf_size,
    unsigned long flags, na_mem_handle_t **mem_handle_p);

na_return_t
NA_Mem_register(na_class_t *na_class, na_mem_handle_t *mem_handle,
    enum na_mem_type mem_type, uint64_t device);
```

Similarly, `NA_Mem_deregister()` and `NA_Mem_handle_free()` must be called
to release resources.

Once memory has been registered, the handle of the target must be serialized
and exchanged with the peer that will initiate the RMA operation. This is done
by calling:

```C
na_return_t
NA_Mem_handle_serialize(na_class_t *na_class, void *buf, size_t buf_size,
    na_mem_handle_t *mem_handle);
```

The peer can then deserialize the handle using:

```C
na_return_t
NA_Mem_handle_deserialize(na_class_t *na_class, na_mem_handle_t **mem_handle_p,
    const void *buf, size_t buf_size);
```

And initiate an RMA operation using both the handle of the target that describes
its remote memory and the local handle that describes its local memory:

```C
na_return_t
NA_Put(na_class_t *na_class, na_context_t *context, na_cb_t callback, void *arg,
    na_mem_handle_t *local_mem_handle, na_offset_t local_offset,
    na_mem_handle_t *remote_mem_handle, na_offset_t remote_offset,
    size_t data_size, na_addr_t *remote_addr, uint8_t remote_id, na_op_id_t *op_id);

na_return_t
NA_Get(na_class_t *na_class, na_context_t *context, na_cb_t callback, void *arg,
    na_mem_handle_t *local_mem_handle, na_offset_t local_offset,
    na_mem_handle_t *remote_mem_handle, na_offset_t remote_offset,
    size_t data_size, na_addr_t *remote_addr, uint8_t remote_id, na_op_id_t *op_id);
```

Similar to point-to-point operations, RMA operations are non-blocking and
use a callback-based model that is triggered after a call to `NA_Trigger()`
when the operation completes.

### Progress and Cancelation

NA progress model is always explicit and users are expected to call
`NA_Progress()` followed by a call to `NA_Trigger()`:

```C
na_return_t
NA_Progress(na_class_t *na_class, na_context_t *context, unsigned int timeout);

na_return_t
NA_Trigger(na_context_t *context, unsigned int max_count, unsigned int *actual_count);
```

`NA_Trigger()` always operates on a single context while `NA_Progress()` may
operate both on a class and a context. When progress is called, it returns
as soon as an operation either completes or is already
in the completion queue so that a call to `NA_Trigger()` may be done to
empty the queue and execute the user callback.

When an operation must be canceled, users are expected to call `NA_Cancel()` on
that operation:

```C
na_return_t
NA_Cancel(na_class_t *na_class, na_context_t *context, na_op_id_t *op_id);
```

Cancelation is always asynchronous. When/if the operation is successfully
canceled, it will be pushed to the completion queue with an `NA_CANCELED` error
return code.

## Available Plugins

NA supports different backend implementations. However, OFI/libfabric is the
recommended plugin in most situations for inter-node communication, while SM
(shared-memory) is recommended for intra-node communication.

### Summary

The table below summarizes the current list of plugins along with the transports
that we currently support with those plugins.

| Plugin / Transport | `tcp`              | `verbs`            | `shm`              | `opx`              | `gni`              | `cxi`              |
| ------------------ | ------------------ | ------------------ | ------------------ | ------------------ | ------------------ | ------------------ |
| `ofi`              | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: |
| `ucx`              | :heavy_check_mark: | :heavy_check_mark: | :grey_question:    | :x:                | :grey_question:    | :x:                |
| `sm`               | :x:                | :x:                | :heavy_check_mark: | :x:                | :x:                | :x:                |

!!! Warning

    Additional transports may be supported for each plugin but we do not recommend
    their use unless explicitly mentioned in the above table as they are either
    unstable or have not been tested. Transports with :x: are not available
    for the selected plugin. Transports with :grey_question: are not supported
    but may be available in the future. Transports with :exclamation: have known issues.

### Initialization String Format

Below is a table summarizing the protocols and expected format for each plugin
(`[ ]` means optional, in which case the plugin will select default hostnames and ports to use).

Plugin | Protocol             | Initialization format[<sup>1</sup>](#init_format)
------ | --------             | ---------------------
ofi    | tcp <br/> verbs <br/> shm <br/> opx <br/> gni <br/> cxi | `ofi+tcp[://<hostname,IP,interface name>:<port>]` <br/> `ofi+verbs[://[MLX device/]<hostname,IP,interface name>:<port>]`[<sup>2</sup>](#ofi_verbs_config) <br/> `ofi+shm`[<sup>3</sup>](#no_config) <br/> `ofi+opx[://<HFI device>:<port>]` <br/> `ofi+gni[://<hostname,IP,interface name>]` [<sup>4</sup>](#ofi_gni_config) <br/> `ofi+cxi[://<CXI device>:<port ID>]`
ucx    | all <br/> tcp <br/> rc,ud,dc [<sup>5</sup>](#ucx_tls) | `ucx+all[://[net_device/]<hostname,IP,interface name>:<port>]`
na     | sm                   | `na+sm[://<shm_prefix>]`

!!! note

    Invalid port numbers that are passed may be silently ignored by the underlying implementation in which case a new port will be automatically picked up.

    <a name="init_format"><sup>1</sup></a> When initialized without listening, the port specification can be elided.

    <a name="ofi_verbs_config"><sup>2</sup></a> The libfabric domain name can also be
    passed directly to select the right adapter to use. See the output generated by the
    command `hg_info` for provider name `verbs;ofi_rxm` (e.g., `mlx5_0`).

    <a name="no_config"><sup>3</sup></a> Any hostname or port being passed will be ignored.

    <a name="ofi_gni_config"><sup>4</sup></a> No port information needs to be passed,
    the most common interface name is `ipogif0`, which will be used by default if
    no hostname is passed.

    <a name="ucx_tls"><sup>5</sup></a> Please refer to the UCX [documentation][ucx_tls_doc] for a full list
    of available transports that can be used.

[ucx_tls_doc]: https://openucx.readthedocs.io/en/master/faq.html#which-transports-does-ucx-use

### OFI

(*as of v1.0.0*)
The NA OFI/libfabric plugin is available for general
purpose use. The plugin currently supports tcp, verbs, opx
and cxi transports. The psm2 and gni protocols are deprecated.
See this [page](ofi.md) for additional implementation and
performance details.

!!! attention

    As of libfabric 1.18.0, tcp no longer uses the RxM layer. To force use of tcp
    with RxM, `tcp;ofi_rxm` must be directly passed. However, it is recommended to use
    `tcp` for improved stability and performance.

*Technical notes:*

* Low CPU consumption (i.e., idles without busy spinning) is supported by all
  supported libfabric providers.
* Connection-less and uses reliable datagrams.
* RMA (for Mercury bulk operations) is implemented natively on transports
  that support it.
* ofi/tcp (`tcp` provider) may use the RxM layer to emulate connection-less endpoints. It also emulates RMA operations.
* ofi/verbs (`verbs` provider) uses the RxM layer to emulate connection-less endpoints (the first message being sent may be slower).
* ofi/opx (`opx` provider) can be used on legacy Intel<sup>®</sup> Omni-Path 2 interconnect and Cornelis Omni-Path Express hardware.
* ofi/gni (`gni` provider) can be used on Cray<sup>®</sup> systems with Gemini/Aries interconnects. Note that
  it requires the use of Cray<sup>®</sup> DRC to exchange credentials when
  communication between separate jobs is required (see section on [DRC credentials](drc.md)).
* ofi/cxi (`cxi` provider) can be used on HPE systems with Slingshot interconnect. For systems with multiple NICs, `hwloc`
  can be used to automatically select the closest NIC to the CPU in-use (mercury must be configured with `NA_OFI_USE_HWLOC`
  in that case).

*Influential variables:*

* `RDMAV_HUGEPAGES_SAFE`: must be set when using hugepages in combination with `verbs` provider.
* `FI_UNIVERSE_SIZE`: must be set when exceeding 256 peers with `tcp` or `verbs` providers.

Please refer to the libfabric manpages for additional details for each transports.

### UCX

(*as of v2.1.0*)
The UCX plugin is available for general purpose use. By default and as opposed
to other plugins, the UCX plugin is able to automatically determine which
transport is best to be used. This is achieved by passing the `all` keyword in
lieu of a specific transport. However, note that we are only testing the
`tcp` and `rc,ud,dc` (verbs) protocols of UCX.

*Technical notes:*

* Connection-less is currently emulated on top of connected endpoints. Therefore,
  it is expected that the first message sent to a target will be slower than
  subsequent messages.
* A thread safe enabled UCX library is required to be used unless users explicitly
  tell NA, using the `thread_mode` init option (see [above](#initialization)),
  that they will not access classes and contexts with more than one thread.
* `NA_Addr_to_string()` cannot be used on non-listening processes to convert a
   self-address to a string. This is due to the fact that UCX does not expose
   endpoints prior to their connection.

*Influential variables:*

* `ucx_info -c -f` will display the default configuration. Each of these
  variables can be overridden by the user. Note, however, that the `UCX_TLS`
  and `UCX_NET_DEVICES` are currently overridden by the NA UCX plugin.

### SM

(*as of v0.9.0*) This is the integrated shared memory NA plugin.
Plugin is stable and provides significantly better performance for local
node communication. The goal of this plugin is to provide a
transparent shortcut for other NA plugins when they connect to local services
using the `auto_sm` initialization option (see the [RPC section](hg.md) for more details),
but it is also useful as a primary transport for single-node services.

*Technical notes:*

  * Uses fully connection-less communication.
  * Low CPU consumption (i.e., idles without busy spinning or using threads).
  * RMA (for Mercury bulk operations) is implemented natively through
    cross-memory attach ([CMA][cma]) on Linux, and there are fallback methods
    for other platforms as well. See this [page](sm.md)
    for additional implementation and performance details.

[cma]: https://lwn.net/Articles/405284/

## Deprecated Plugins

### CCI

(*deprecated*) This NA plugin is no longer available for general purpose use,
  and is now deprecated as CCI itself is no longer actively maintained.

### MPI

MPI implementations are widely available for nearly any platform, and
the NA MPI plugin provides a convenient option for prototyping and
functionality testing. It is not optimized for performance, however, and it
has some practical limitations when used for persistent services.

*Technical notes:*

  * Clients can connect to a server dynamically only if the underlying MPI 
    implementation supports `MPI_Comm_connect()`.
  * RMA (for Mercury bulk operations) is emulated via point-to-point messaging
    (note: MPI window creation requires collective coordination and is not a
    good fit for RPC use).
  * Significant CPU consumption (progress function iteratively polls pending
    operations for completion).

### BMI

The BMI library itself is no longer under active feature development
beyond basic maintenance, but the NA BMI plugin provides a very stable
and reasonably performant option for IP networks when used with BMI's TCP
method.

*Technical notes:*

  * Low CPU consumption (i.e., idles without busy spinning or using threads).
  * Supports dynamic client connection and disconnection.
  * RMA (for Mercury bulk operations) is emulated via point-to-point messaging.
  * Does *not* support initializing multiple instances simultaneously.
  * Other BMI methods besides TCP are not supported.
  * For general BMI information see this [paper](http://ieeexplore.ieee.org/abstract/document/1420118/).