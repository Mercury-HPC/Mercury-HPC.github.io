---
layout: page
title: Documentation
permalink: /documentation/
---

Documentation reflects Mercury's API as of v1.0.0 and can be used
as an introduction to the Mercury library. Please look at the
[see also](#see-also) section for additional documentation.

## Overview

Mercury is composed of three main layers:

1. the [network abstraction layer](#network-abstraction-layer), which provides
  a high-performance communication interface on top of lower level network
  fabrics.
2. the [RPC layer](#rpc-layer), which provides users with the necessary
  components for sending and receiving RPC metadata (small messages). This includes
  serialization and deserialization of function arguments;
3. the [bulk layer](#bulk-layer), which provides the necessary components for
  handling large arguments---this implies large data transfers through RMA;
4. the (*optional*) [high-level RPC layer](#high-level-rpc-layer), which aims at
  providing a convenience API, builds on top of the lower layers and provides
  macros for generating RPC stubs as well as serialization and deserialization
  functions.

These three main layers can be summarized in the following diagram:

<figure>
  <img src="/assets/overview.svg" alt="Caption to image" width="500px">
</figure>

By definition, an RPC call is initiated by one process, referred to as
_origin_, and forwarded to another process, which will execute the call,
and referred to as _target_. Each side, origin and target, uses an RPC
_processor_ to serialize and deserialize parameters sent through the interface.
Calling functions with relatively small arguments results in using a short
messaging mechanism exposed by the network abstraction layer, whereas functions
containing large data arguments additionally use a remote memory access (RMA)
mechanism.

## Network Abstraction Layer

The _network abstraction_ (NA) layer is internally used by both the RPC layer
and the bulk layer (it is therefore not expected to be directly used by
a regular mercury user, who may directly jump to the [RPC layer](#rpc-layer) section).
It provides a minimal set of function calls that abstract the underlying network
fabric and that can be used to provide:
*target address lookup*, *point-to-point messaging* with both unexpected and
expected messaging, *remote memory access*, *progress* and *cancellation*.
The API is non-blocking and uses a callback mechanism
so that upper layers can provide asynchronous execution more easily: when progress
is made (either internally or after a call to `NA_Progress()`) and an operation
completes, the user callback is placed onto a completion queue. The callback
can then be dequeued and separately executed after a call to `NA_Trigger()`.

The NA layer uses a plugin mechanism so that support for various network protocols
can be easily added and selected at runtime.

### NA Interface

Typically, the first step consists of
initializing the NA interface and selecting an underlying plugin that will be
used. This can be done either at the NA level or at the HG level (see the
[RPC layer](#rpc-layer) section).
Initializing the NA interface with a specified `info_string`
results in the creation of a new `na_class_t` object. Please refer to
the [available plugins](#available-plugins) section for more information on the
`info_string` format. Additionally, it is possible to specify whether the
`na_class_t` object will be listening (for incoming connections) or
not---this is the _only_ time where a "server" specific behavior is defined, all
subsequent calls do not make any distinction between a "client" and a "server"
and instead only use the concept of _origin_ and _target_.

{% highlight C %}
na_class_t *NA_Initialize(const char *info_string, na_bool_t listen);
{% endhighlight %}

If a more specific behavior is required, the following call can be used to pass
specific init options.

{% highlight C %}
struct na_init_info {
    na_progress_mode_t progress_mode;   /* Progress mode */
    na_uint8_t max_contexts;            /* Max contexts */
    const char *auth_key;               /* Authorization key */
};

na_class_t *NA_Initialize_opt(const char *info_string, na_bool_t listen,
    const struct na_init_info *na_init_info);
{% endhighlight %}

Progress mode set with `NA_NO_BLOCK` requests explicitly for busy
spinning when making progress instead of releasing the CPU.
The maximum number of contexts
option is for now specific to the libfabric/OFI plugin and scalable endpoints
(see section [progress modes][progress modes] for more details).
Authorization keys allow for passing a system-specific credential down to the
plugin, this is required on Cray<sup>®</sup> systems to enable communication between 
separate jobs (see section [DRC credentials][DRC credentials] for more details).

The `na_class_t` object created from these initialization calls can later be
released after a call to:

{% highlight C %}
na_return_t NA_Finalize(na_class_t *na_class);
{% endhighlight %}

Once the interface has been initialized, a context within this plugin must be
created, which internally creates and associates a completion queue for the
operations:

{% highlight C %}
na_context_t *NA_Context_create(na_class_t *na_class);
{% endhighlight %}

It can then be destroyed using:

{% highlight C %}
na_return_t NA_Context_destroy(na_class_t *na_class, na_context_t *context);
{% endhighlight %}

To connect to a target and start communication, one must first get the
address of the target. The most convenient and safe way of doing that is to
first call on the target (or the equivalent `HG_Addr_xxx()` function calls):

{% highlight C %}
na_return_t NA_Addr_self(na_class_t *na_class, na_addr_t *addr);
{% endhighlight %}

And then convert that address to a string using:

{% highlight C %}
na_return_t NA_Addr_to_string(na_class_t *na_class, char *buf,
                              na_size_t buf_size, na_addr_t addr);
{% endhighlight %}

The string can then be communicated to other processes through out-of-band
mechanisms (e.g., using a file),
which can then look up the target using the non-blocking function:

{% highlight C %}
typedef na_return_t (*na_cb_t)(const struct na_cb_info *callback_info);

na_return_t NA_Addr_lookup(na_class_t *na_class, na_context_t *context,
                           na_cb_t callback, void *arg, const char *name,
                           na_op_id_t *op_id);
{% endhighlight %}

This function takes a user callback. `NA_Progress()` and `NA_Trigger()` need to
be called in this case and the resulting address can be retrieved when the user
callback is executed.
All addresses must then be freed using:

{% highlight C %}
na_return_t NA_Addr_free(na_class_t *na_class, na_addr_t addr);
{% endhighlight %}

Other routines such as `NA_Msg_send_unexpected()`, `NA_Msg_send_expected()`,
`NA_Msg_recv_unexpected()`, `NA_Msg_recv_expected()`, `NA_Put()`, `NA_Get()`, etc,
are used internally. There should not be any need for using them directly.

### Available Plugins

<hr />
__BMI:__ The BMI library itself is no longer under active feature development
  beyond basic maintenance, but the BMI Mercury NA plugin provides a very stable
  and reasonably performant option for IP networks when used with BMI's TCP
  method. <br/>
  *Technical notes:*
  * Low CPU consumption (i.e., idles without busy spinning or using threads).
  * Supports dynamic client connection and disconnection.
  * RMA (for Mercury bulk operations) is emulated via point-to-point messaging.
  * Does *not* support initializing multiple instances simultaneously.
  * Other BMI methods besides TCP are not supported.
  * For general BMI information see this [paper](http://ieeexplore.ieee.org/abstract/document/1420118/).
<hr />
__MPI:__ MPI implementations are widely available for nearly any platform, and
  the MPI Mercury NA plugin provides a convenient option for prototyping and
  functionality testing. It is not optimized for performance, however, and it
  has some practical limitations when used for persistent services. <br/>
  *Technical notes:*
  * Clients can connect to a server dynamically only if the underlying MPI 
    implementation supports `MPI_Comm_connect()`.
  * RMA (for Mercury bulk operations) is emulated via point-to-point messaging
    (note: MPI window creation requires collective coordination and is not a
    good fit for RPC use).
  * Significant CPU consumption (progress function iteratively polls pending
    operations for completion).
<hr />
__SM:__ (*as of v0.9.0*) This is the integrated shared memory NA plugin included with Mercury.
  Plugin is stable and provides significantly better performance for local
  node communication.  The eventual goal of this plugin is to provide a
  transparent shortcut for other NA plugins when they connect to local services
  using the `auto_sm` initialization option (see section [shared-memory][shared-memory] for more details),
  but it is also useful as a primary transport for single-node services. <br/>
  *Technical notes:*
  * Supports dynamic client connection and disconnection.
  * Low CPU consumption (i.e., idles without busy spinning or using threads).
  * RMA (for Mercury bulk operations) is implemented natively through
    cross-memory attach ([CMA][cma]) on Linux, and there are fallback methods
    for other platforms as well. See this [page][shared-memory]
    for design details.
<hr />
__OFI:__ (*as of v1.0.0*) The NA OFI/libfabric plugin is available for general
    purpose use, but some providers (libfabric transport plugins) are still in
    an early development state. The plugin currently supports tcp, verbs, psm2
    and gni transports. <br/>
  *Technical notes:*
  * Low CPU consumption (i.e., idles without busy spinning) if supported by the libfabric provider. At present, the `sockets` and `gni` providers accomplish this by using internal progress threads.
  * Connection-less and uses reliable datagrams.
  * RMA (for Mercury bulk operations) is implemented natively on transports
    that support it (i.e., verbs, psm2 and gni).
  * ofi/tcp (`sockets` provider) should be considered the most stable transport.
  * ofi/verbs (`verbs;rxm` provider) is stable and uses the RxM layer to emulate connection-less endpoints.
  * ofi/psm2 (`psm2` provider) is stable and can be used on Intel<sup>®</sup> Omni-Path interconnect.
  * ofi/gni (`gni` provider) support within the plugin is stable and can be
    used on Cray<sup>®</sup> systems with Gemini/Aries interconnects. Note that
    it requires the use of Cray<sup>®</sup> DRC to exchange credentials when
    communication between separate jobs is required (see section [DRC credentials][DRC credentials]).
<hr />
__CCI:__ (*deprecated*) This NA plugin is still available for general purpose use,
  but is now deprecated as CCI itself is no longer actively maintained. Some transport plugins within CCI are more robust than others. <br/>
  *Technical notes:*
  * Low CPU consumption (i.e., idles without busy spinning or using threads),
    with the exception of the TCP CCI plugin, see below.
  * Supports dynamic client connection and disconnection.
  * RMA (for Mercury bulk operations) is implemented natively on transports
    that support it.
  * Some CCI transport plugins create threads internally to assist in connection
    management or communication progress.
  * cci/tcp is not stable, consumes significant CPU and may not perform as well
    as bmi/tcp or ofi/tcp.
  * cci/verbs is stable and performant but may show connection/disconnection issues.
  * cci/sm (shared memory) is stable and performant.
  * cci/gni is considered unstable at this time.
<hr />
Below is a table summarizing the protocols and expected format for each plugin
(`[ ]` means optional, in which case the plugin will select default hostnames and ports to use).

plugin | protocol             | initialization format[<sup>1</sup>](#init_format) 
------ | --------             | ---------------------
bmi    | tcp                  | `bmi+tcp[://<hostname,IP>:<port>]`
mpi    | dynamic, static[<sup>2</sup>](#mpi_static)  | `mpi+<protocol>`
na     | sm                   | `na+sm`
ofi    | tcp <br/> verbs <br/> psm2 <br/> gni | `ofi+tcp[://<hostname,IP,interface name>:<port>]` <br/> `ofi+verbs[://<hostname,IP,interface name>:<port>]`[<sup>3</sup>](#ofi_verbs_config) <br/> `ofi+psm2`[<sup>4</sup>](#ofi_psm2_config) <br/> `ofi+gni[://<hostname,IP,interface name>]` [<sup>5</sup>](#ofi_gni_config)
cci[<sup>6</sup>](#cci_config) | tcp <br/> verbs <br/> sm | `cci+tcp[://<hostname,IP,interface name>:<port>]` <br/> `cci+verbs[://<hostname,IP,interface name>:<port>]` <br/> `cci+sm[://<PID>/<ID>]`

<a name="init_format"><sup>1</sup></a> When not being initialized in listening mode,  the port specification can be elided.

<a name="mpi_static"><sup>2</sup></a> MPI static mode requires all mercury processes to
be started in the same mpirun invocation.

<a name="ofi_verbs_config"><sup>3</sup></a> The libfabric domain name can also be
passed directly to select the right adapter to use. See the output generated by the
command `fi_info` for provider name `verbs;ofi_rxm` (e.g., `mlx5_0`).

<a name="ofi_psm2_config"><sup>4</sup></a> Any hostname or port being passed will be ignored.

<a name="ofi_gni_config"><sup>5</sup></a> No port information needs to be passed,
the most common interface name is `ipogif0`, which will be used by default if
no hostname is passed.

<a name="cci_config"><sup>6</sup></a> Note that a `cci.ini` file as well as
the `CCI_CONFIG` environment variable can be provided to select the network interface to use. See the CCI `README` files for more details.

<hr />
For reference, lookup address formats are also given below but it is *very strongly
encouraged* to use `HG_Addr_to_string()` and not try to generate any of these
strings by hand.

plugin | protocol             | lookup format
------ | --------             | -------------
bmi    | tcp                  | `bmi+tcp://<hostname or IP>:<port>`
mpi    | dynamic <br/> static | `mpi+dynamic://<MPI port name>` <br/> `mpi+static://<MPI rank>`
na     | sm                   | `na+sm://<PID>/<SM ID>`
ofi    | tcp <br/> verbs <br/> psm2 <br/> gni | `ofi+sockets://fi_sockaddr_in://<IP>:<port>` <br/> `ofi+verbs;ofi_rxm://fi_sockaddr_in://<IP>:<port>` <br/> `ofi+psm2://fi_addr_psmx2://<EP native address>` <br/> `ofi+gni://fi_addr_gni://<EP native address>`
cci    | tcp <br/> verbs <br/> sm | `cci+tcp://<IP>:<port>` <br/> `cci+verbs://<IP>:<port>` <br/> `cci+sm://<PID>/<SM ID>`

## RPC Layer

The _RPC layer_ provides users with the necessary components for
sending, receiving and executing RPCs. This layer is composed of two sub-layers:
the HG core RPC
layer, which defines an RPC operation as a buffer that is sent to a target
and triggers a callback associated to that operation; and the HG RPC
layer, which includes serialization and deserialization of function arguments.

Every RPC call results in the serialization of function parameters into a
memory buffer (its size generally being limited to a couple of kilobytes, depending on
the network protocol used). This buffer is then sent to the target using the network
abstraction (NA) layer interface. One of the key requirements is to limit memory
copies at any stage of the transfer, especially when transferring large amounts
of data. Therefore, if the data sent is small, it is serialized and sent using
small messages (unexpected and expected messages); otherwise
a description of the memory region that is to be
transferred is sent within this same small message to the target, which can
then start pulling the data (if the data is the input of the remote call) or
pushing the data (if the data is the output of the remote call).

Limiting the
size of the initial RPC request to the target also helps in scalability, since
it avoids unnecessary server resource consumption if large numbers of processes
are concurrently accessing the same target. Depending on the degree of control
desired, all these steps can be transparently handled by Mercury or directly
exposed to the user.

### HG Interface

To initialize the HG RPC interface, two options are available, either by using
the default `HG_Init()` function and specifying an init info string as described
in that [section](#available-plugins), which creates its own NA class internally:

{% highlight C %}
hg_class_t *HG_Init(const char *info_string, hg_bool_t listen);
{% endhighlight %}

Or by
using the `HG_Init_opt()` function, which allows to pass
an existing NA class (created by the routine presented
in the [network abstraction layer](#network-abstraction-layer) section) through
the init options:

{% highlight C %}
struct hg_init_info {
    struct na_init_info na_init_info;   /* NA Init Info */
    na_class_t *na_class;               /* NA class */
    hg_bool_t auto_sm;                  /* Use NA SM plugin with local addrs */
    hg_bool_t stats;                    /* (Debug) Print stats at exit */
};

hg_class_t *HG_Init_opt(const char *na_info_string, hg_bool_t na_listen,
    const struct hg_init_info *hg_init_info);
{% endhighlight %}

Additional options are available such as NA init info options (see
[NA interface](#na-interface)), transparent shared-memory (see [shared-memory]).
Similar to the NA layer, the `HG_Init()` call results in the
creation of a new `hg_class_t` object. The `hg_class_t` object can later be
released after a call to:

{% highlight C %}
hg_return_t HG_Finalize(hg_class_t *hg_class);
{% endhighlight %}

Once the interface has been initialized, a context of execution must be
created, which (similar to the NA layer) internally associates a specific
queue to the operations that complete:

{% highlight C %}
hg_context_t *HG_Context_create(hg_class_t *hg_class);
{% endhighlight %}

It can then be destroyed using:

{% highlight C %}
hg_return_t HG_Context_destroy(hg_context_t *context);
{% endhighlight %}

Before an RPC can be sent, the HG class needs a way of identifying it so that a
callback corresponding to that RPC can be executed on the target. Additionally,
the functions to serialize and deserialize the function arguments associated
to that RPC must be provided. This is done through the `HG_Register_name()` function.
Note that this step can be simplified by using the
[high-level RPC layer](#high-level-rpc-layer). Registration must be done on
both the origin and the target with the same `func_name` identifier.
Alternatively `HG_Register()` can be used to pass a user-defined unique
identifier and avoid internal hashing of the provided function name.

{% highlight C %}
typedef hg_return_t (*hg_proc_cb_t)(hg_proc_t proc, void *data);
typedef hg_return_t (*hg_rpc_cb_t)(hg_handle_t handle);

hg_id_t HG_Register_name(hg_class_t *hg_class, const char *func_name,
                         hg_proc_cb_t in_proc_cb, hg_proc_cb_t out_proc_cb,
                         hg_rpc_cb_t rpc_cb);
{% endhighlight %}

In the case where an RPC does not require a response, one can indicate that
no response is expected (and therefore avoid waiting for a message to be sent
back) by using the following call (on an already registered RPC):

{% highlight C %}
hg_return_t HG_Registered_disable_response(hg_class_t *hg_class, hg_id_t id,
                                           hg_bool_t disable);
{% endhighlight %}

As mentioned in the [overview](#overview) section, there is no real distinction between client and server since
it may be desirable for a client to also act as a server for other processes.
Therefore, the interface only uses the distinction of _origin_ and _target_.

#### Origin

In a typical scenario, the origin will first lookup a target and get an
address. This can be achieved by:

{% highlight C %}
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t HG_Addr_lookup(hg_context_t *context, hg_cb_t callback, void *arg,
                           const char *name, hg_op_id_t *op_id);
{% endhighlight %}

This function takes a user callback. `HG_Progress()` and `HG_Trigger()` need to
be called in this case and the resulting address can be retrieved when the user
callback is executed. Connection to the target may occur at this time, though that
behavior is left upon the NA plugin implementation and protocol (which may or
may not be connectionless).
All addresses must then be freed using:

{% highlight C %}
hg_return_t HG_Addr_free(hg_class_t *hg_class, hg_addr_t addr);
{% endhighlight %}

In a typical scenario, the origin will initiate an RPC call by using the RPC ID
defined after a call to `HG_Register()`. Using the `HG_Create()` call will define
a new `hg_handle_t` object that can be used (and re-used) to set/get input/output
arguments.

{% highlight C %}
hg_return_t HG_Create(hg_context_t *context, hg_addr_t addr, hg_id_t id,
                      hg_handle_t *handle);
{% endhighlight %}

This handle can be destroyed with `HG_Destroy()`---and a reference count prevents
resources from being released while the handle is still in use.

{% highlight C %}
hg_return_t HG_Destroy(hg_handle_t handle);
{% endhighlight %}

The second step is to pack the input arguments within a structure, for which
a serialization function is provided with the `HG_Register()` call. The
`HG_Forward()` function can then be used to send that structure (which describes
the input arguments). This function is non-blocking. When it completes, the associated
callback can be executed by calling `HG_Trigger()`.

{% highlight C %}
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t HG_Forward(hg_handle_t handle, hg_cb_t callback, void *arg,
                       void *in_struct);
{% endhighlight %}

When `HG_Forward()` completes (i.e., when the user callback can be triggered),
the RPC has been remotely executed and a response with the output results has
been sent back. This output can then be retrieved (usually within the callback)
with the following function:

{% highlight C %}
hg_return_t HG_Get_output(hg_handle_t handle, void *out_struct);
{% endhighlight %}

Retrieving the output may result in the creation of memory objects, which
must then be released by calling:

{% highlight C %}
hg_return_t HG_Free_output(hg_handle_t handle, void *out_struct);
{% endhighlight %}

To be safe and if necessary, one must make a copy of the results before
calling `HG_Free_output()`. Note that in the case of an RPC with no response,
completion occurs after the RPC has been successfully sent (i.e., there is
no output to retrieve).

#### Target

On the target, the RPC callback passed to the `HG_Register()` function must
be defined.

{% highlight C %}
typedef hg_return_t (*hg_rpc_cb_t)(hg_handle_t handle);
{% endhighlight %}

Whenever a new RPC is received, that callback will
be invoked. The input arguments can be retrieved with:

{% highlight C %}
hg_return_t HG_Get_input(hg_handle_t handle, void *in_struct);
{% endhighlight %}

Retrieving the input may result in the creation of memory objects, which
must then be released by calling:

{% highlight C %}
hg_return_t HG_Free_input(hg_handle_t handle, void *in_struct);
{% endhighlight %}

When the input has been retrieved, the arguments contained in the input structure
can be passed to the actual function call (e.g., if that's an RPC `open()`, the
`open()` function can now be called). When the execution is done, an output structure
can be filled with the return value and/or the output arguments of the function.
It can then be sent back using:

{% highlight C %}
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t HG_Respond(hg_handle_t handle, hg_cb_t callback, void *arg,
                       void *out_struct);
{% endhighlight %}

This call is also non-blocking. When it completes, the associated callback is
placed onto a completion queue. It can then be triggered after a call to
`HG_Trigger()`. Note that in the case of an RPC with no response, calling
`HG_Respond()` will return an error.

#### Progress and Cancellation

Mercury uses a callback model. Callbacks are passed to non-blocking functions
and are pushed to the context's completion queue when the operation completes.
Explicit progress is made by calling `HG_Progress()`. `HG_Progress()` returns when
an operation completes, is in the completion queue or `timeout` is reached.

{% highlight C %}
hg_return_t HG_Progress(hg_context_t *context, unsigned int timeout);
{% endhighlight %}

When an operation completes, calling `HG_Trigger()` allows the callback
execution to be separately controlled from the main progress loop.

{% highlight C %}
hg_return_t HG_Trigger(hg_context_t *context, unsigned int timeout,
                       unsigned int max_count, unsigned int *actual_count);
{% endhighlight %}

In some cases, one may want to call `HG_Progress()` then `HG_Trigger()` or have
them execute in parallel by using separate threads, this is further describe
in the [progress modes][progress modes] section.

Note also that in cases where it is desirable to cancel an HG operation,
one can call `HG_Cancel()` on a HG handle to cancel an on-going
`HG_Forward()` or `HG_Respond()` operation. Please refer to this
[page][cancellation] for additional details
regarding cancellation of operations and handling of timeouts.

{% highlight C %}
hg_return_t HG_Cancel(hg_handle_t handle);
{% endhighlight %}

## Bulk Layer

In addition to the previous layer, some RPCs may require the transfer of larger
amounts of data. For these RPCs, the _bulk layer_ can be used. It is built on
top of the RMA protocol defined in the network abstraction layer and prevents
intermediate memory copies.

The origin process exposes a memory region to the target by creating a bulk
descriptor (which contains virtual memory address information, size of the
memory region that is being exposed, and other parameters that depend on the
underlying network implementation).
The bulk descriptor can be serialized and sent to the target along
with the RPC request arguments (using the RPC layer). When the target gets
the input parameters, it can deserialize the bulk descriptor, get the size of
the memory buffer that has to be transferred, and initiate the transfer.
Only targets should initiate one-sided transfers so that they can, as well as
controlling the data flow, protect their memory from concurrent accesses.

As no explicit ack message is sent on transfer completion, the origin process
can only assume that accesses to its local memory are completed once it
receives the RPC response from the target. Therefore, in the case of an RPC
with no response, great care should be taken when initiating a bulk transfer
to ensure that the origin gets notified when its exposed memory can be safely
released and accessed.

### HG Bulk Interface

The interface uses the class and execution context that are defined by the
HG RPC layer.
To initiate a bulk transfer, one needs to create a bulk descriptor on both the
origin and the target, which will then be passed to the `HG_Bulk_transfer()`
call.

{% highlight C %}
hg_return_t HG_Bulk_create(hg_class_t *hg_class, hg_uint32_t count,
                           void **buf_ptrs, const hg_size_t *buf_sizes,
                           hg_uint8_t flags, hg_bulk_t *handle);
{% endhighlight %}

The bulk descriptor can be released by using:

{% highlight C %}
hg_return_t HG_Bulk_free(hg_bulk_t handle);
{% endhighlight %}

For convenience, memory pointers from an existing bulk descriptor can be
accessed with:

{% highlight C %}
hg_return_t
HG_Bulk_access(hg_bulk_t handle, hg_size_t offset, hg_size_t size,
               hg_uint8_t flags, hg_uint32_t max_count, void **buf_ptrs,
               hg_size_t *buf_sizes, hg_uint32_t *actual_count);
{% endhighlight %}

When the bulk descriptor from the origin has been received, the target
can initiate the bulk transfer to/from its own bulk descriptor. Virtual offsets
can be used to transfer data pieces from a non-contiguous block transparently.
The call is non-blocking. When the operation completes, the user callback
is placed onto the context's completion queue.

{% highlight C %}
hg_return_t
HG_Bulk_transfer(hg_context_t *context, hg_bulk_cb_t callback, void *arg,
                 hg_bulk_op_t op, hg_addr_t origin_addr,
                 hg_bulk_t origin_handle, hg_size_t origin_offset,
                 hg_bulk_t local_handle, hg_size_t local_offset,
                 hg_size_t size, hg_op_id_t *op_id);
{% endhighlight %}

Note that for convenience, as the transfer needs to be realized within the
RPC callback, the routine `HG_Get_info()` enables easy retrieval of classes,
contexts and source address:

{% highlight C %}
struct hg_info {
    hg_class_t *hg_class;               /* HG class */
    hg_context_t *context;              /* HG context */
    hg_addr_t addr;                     /* HG address */
    hg_id_t id;                         /* RPC ID */
};

struct hg_info *HG_Get_info(hg_handle_t handle);
{% endhighlight %}

Additionally, it is also possible to bind the origin's address to the bulk handle
through the `HG_Bulk_bind()` function at the cost of an additional overhead
for serializing and deserializing addressing information. This should only be
necessary when the source address retrieved from the `HG_Get_info()` call is
different from the one that must be used for the transfer (e.g., multiple sources).

{% highlight C %}
hg_return_t HG_Bulk_bind(hg_bulk_t handle, hg_context_t *context);
{% endhighlight %}

In that particular case, the address information can be directly retrieved using:

{% highlight C %}
hg_addr_t HG_Bulk_get_addr(hg_bulk_t handle);
{% endhighlight %}

## High-level RPC Layer

For convenience, the high-level RPC layer provides macros and routines that can
reduce the amount of code required to send an RPC call with Mercury. For macros,
Mercury makes use of the [Boost preprocessor library](http://www.boost.org/doc/libs/release/libs/preprocessor/) so that users can generate all the boilerplate code that
is necessary to serialize and deserialize function arguments.

### Generate proc routines

The first macro, called `MERCURY_GEN_PROC()`, generates both structures and proc functions
to serialize arguments. The structure fields contain either input arguments or
output arguments. The generated proc routine uses pre-existing types to serialize
and deserialize the field.

{% highlight C %}
MERCURY_GEN_PROC(struct_type_name, fields)
{% endhighlight %}

#### Example:

The following function has two input arguments, one output argument and one
return value.

{% highlight C %}
int rpc_open(const char *path, rpc_handle_t handle, int *event_id);
{% endhighlight %}

The following macro can be used to generate boilerplate code for the input
argument (please refer to the [predefined types](#predefined-types) section for
a list of the existing types that can be passed to this macro):

{% highlight C linenos %}
MERCURY_GEN_PROC( rpc_open_in_t, ((hg_const_string_t)(path))
                                 ((rpc_handle_t)(handle)) )

/* Will generate an rpc_open_in_t struct */

typedef struct {
    hg_const_string_t path;
    rpc_handle_t handle;
} rpc_open_in_t;

/* and an hg_proc_rpc_open_in_t proc function */

hg_return_t
hg_proc_rpc_open_in_t(hg_proc_t proc, void *data)
{
    hg_return_t ret;
    rpc_open_in_t *struct_data = (rpc_open_in_t *) data;

    ret = hg_proc_hg_const_string_t(proc, &struct_data->path);
    if (ret != HG_SUCCESS) {
      /* error */
    }
    ret = hg_proc_rpc_handle_t(proc, &struct_data->handle);
    if (ret != HG_SUCCESS) {
      /* error */
    }
    return ret;
}
{% endhighlight %}

Note the parentheses that separate the name of the field and its type. Each
field is then separated by another pair of parentheses. This follows the sequence
data type of the Boost preprocessor library.

### Generate proc routines for existing structures

In some cases, however, the argument types are not known by Mercury, which is
the case of the previous example with the `rpc_handle_t` type. For these cases,
another macro, called `MERCURY_GEN_STRUCT_PROC`, can be used. It defines a
serialization function for an existing struct or type---this assumes that the
type can be mapped to already existing types; if not, users can create their
own proc function and use the `hg_proc_raw` routine that takes a stream of bytes.

{% highlight C %}
MERCURY_GEN_STRUCT_PROC(struct_type_name, fields)
{% endhighlight %}

#### Example:

The following function has one non-standard type, `rpc_handle_t`.

{% highlight C linenos %}
int rpc_open(const char *path, rpc_handle_t handle, int *event_id);

typedef struct {
    hg_uint64_t cookie;
} rpc_handle_t;
{% endhighlight %}

The following macro can then be used to generate boilerplate code for the type
by defining its fields.

{% highlight C linenos %}
MERCURY_GEN_STRUCT_PROC( rpc_handle_t, ((hg_uint64_t)(cookie)) )

/* Will generate an hg_proc_rpc_handle_t function */

static hg_return_t
hg_proc_rpc_handle_t(hg_proc_t proc, void *data)
{
    hg_return_t ret;
    rpc_handle_t *struct_data = (rpc_handle_t *) data;

    ret = hg_proc_hg_uint64_t(proc, &struct_data->cookie);
    if (ret != HG_SUCCESS) {
      /* error */
    }
    return ret;
}
{% endhighlight %}

### Predefined types

Mercury uses standard types so that the size of the type is fixed between platforms
when serializing and deserializing it. For convenience, HG types can also be
used to serialize bulk handles for example, but also strings, etc.

Standard type | Mercury type
------------- | ------------
`int8_t`      | All standard types prefixed with `hg_`
`uint8_t`     | `hg_bool_t`
`int16_t`     | `hg_ptr_t`
`uint16_t`    | `hg_size_t`
`int32_t`     | `hg_id_t`
`uint32_t`    | `hg_bulk_t`
`int64_t`     | `hg_const_string_t`
`uint64_t`    | `hg_string_t`

### Register RPC

In conjunction with the previous macros, the following macro makes the registration
of RPC calls more convenient by mapping the types to the generated proc functions.

{% highlight C %}
MERCURY_REGISTER(hg_class, func_name, in_struct_type_name, out_struct_type_name,
                 rpc_cb);
{% endhighlight %}

#### Example:

{% highlight C %}
int rpc_open(const char *path, rpc_handle_t handle, int *event_id);
{% endhighlight %}

One can use the `MERCURY_REGISTER` macro and pass the types of the input/output
structures directly. In cases where no input or no output argument is present,
the `void` type can be passed to the macro.

{% highlight C %}
rpc_open_id_g = MERCURY_REGISTER(hg_class, "rpc_open", rpc_open_in_t,
                                 rpc_open_out_t, rpc_open_cb);
{% endhighlight %}

## See also
Below is a list of more advanced topics and documentation items:

<ul>
  {% for post in site.categories.documentation reversed %}
    <li><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></li>
  {% endfor %}
  <li><a href="{{ site.baseurl }}/doxygen/index.html">Doxygen documentation</a></li>
</ul>

[cma]: https://lwn.net/Articles/405284/
[shared-memory]: {% post_url 2017-01-30-shared-memory %}
[cancellation]: {% post_url 2016-07-26-cancellation %}
[progress modes]: {% post_url 2018-10-22-progress-modes %}
[drc credentials]: {% post_url 2018-10-22-drc-credentials %}

