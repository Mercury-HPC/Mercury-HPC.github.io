---
layout: page
title: Documentation
permalink: /documentation/
---

Documentation reflects Mercury's API as of v0.9.0 and can be used
as an introduction to the Mercury library. Please look at the
[see also](#see-also) section for additional documentation.

## Overview

Mercury is composed of three main layers:

1. the [network abstraction layer](#network-abstraction-layer), which provides
  a high-performance communication interface on top of lower level network
  fabrics.
2. the [RPC layer](#rpc-layer), which provides users with the necessary
  components for sending and receiving RPC metadata. This includes
  serialization and deserialization of function arguments;
3. the [bulk layer](#bulk-layer), which provides the necessary components for
  handling large arguments---this implies large data transfers;
4. (optionally) the [high-level RPC layer](#high-level-rpc-layer), which aims at
  providing a convenience API, builds on top of the lower layers and provides
  macros for generating RPC stubs as well as serialization and deserialization
  functions.

These three main layers can be summarized in the following diagram:

<figure>
  <img src="/assets/overview.svg" alt="Caption to image" width="500px">
</figure>

By definition, an RPC call is initiated by one process, referred to as
_origin_, and forwarded to another process, which will execute the call,
and referred to as _target_. Each side uses an RPC
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

### Interface

Typically, the first step consists of
initializing the NA interface and selecting an underlying plugin that will be
used. Initializing the NA interface with a specified `info_string`
results in the creation of a new `na_class_t` object. Please refer to
the [available plugins](#available-plugins) section for more information on the
`info_string` format. Additionally, it is possible to specify whether the
`na_class_t` object will be listening (for incoming connections) or
not---this is the _only_ time where a server specific behavior is defined, all
subsequent calls do not make any distinction between a client and a server
and instead only use the concept of _origin_ and _target_.

{% highlight C %}
na_class_t *NA_Initialize(const char *info_string, na_bool_t listen);
{% endhighlight %}

The `na_class_t` object can later be released after a call to:

{% highlight C %}
na_return_t NA_Finalize(na_class_t *na_class);
{% endhighlight %}

Once the interface has been initialized, a context within this plugin must be
created, which internally associates a specific completion queue to the
operations:

{% highlight C %}
na_context_t *NA_Context_create(na_class_t *na_class);
{% endhighlight %}

It can then be destroyed using:

{% highlight C %}
na_return_t NA_Context_destroy(na_class_t *na_class, na_context_t *context);
{% endhighlight %}

To connect to a target and start communication, one must first get the
address of the target. The most convenient way (it can also be manually passed
if the address is fixed) of doing it is to first call on the target:

{% highlight C %}
na_return_t NA_Addr_self(na_class_t *na_class, na_addr_t *addr);
{% endhighlight %}

And then convert that address to a string using:

{% highlight C %}
na_return_t NA_Addr_to_string(na_class_t *na_class, char *buf, na_size_t buf_size, na_addr_t addr);
{% endhighlight %}

The string can then be communicated to other processes (e.g., using a file),
which can then look up the target using the non-blocking function:

{% highlight C %}
typedef na_return_t (*na_cb_t)(const struct na_cb_info *callback_info);

na_return_t NA_Addr_lookup(na_class_t *na_class, na_context_t *context, na_cb_t callback, void *arg, const char *name, na_op_id_t *op_id);
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

* _BMI_: flexible connection and disconnection. Provides stable TCP implementation,
  other protocols are not supported. Plugin is stable but it may
  not perform as well as other plugins. Remote memory access is emulated on top
  of point-to-point messaging.
* _MPI_: static connection and disconnection, although connection can be
  established using either `MPI_Comm_split()` or `MPI_Comm_connect()`. Plugin
  provides relatively good performance depending on the underlying transport
  used by the MPI implementation. Remote memory access is emulated on top of
  point-to-point messaging.
* _SM_: shared-memory plugin. Plugin is stable and shows good performance for
  local node communication. Remote memory access is done through
  cross-memory attach ([CMA][cma]) on Linux. See this
  [page]({% post_url 2017-01-30-shared-memory %}) for design details
  about that plugin (Transparent usage of this plugin has not been implemented yet).
* _CCI_: flexible connection and disconnection. Provides support for TCP, UDP,
  verbs, uGNI and shared memory. Plugin shows good performance and supports
  InfiniBand as well as Cray<sup>®</sup> Gemini/Aries interconnect but is more
  experimental.

Below is a table summarizing the protocols and expected format for each plugin
([ ] means optional).

plugin | protocol             | initialization format[<sup>1</sup>](#init_format)  |  lookup format
------ | --------             | ---------------------          |  -------------
bmi    | tcp                  | `bmi+tcp[://<hostname>:<port>]`| `[bmi+]tcp://<hostname>:<port>`
mpi    | dynamic, static[<sup>2</sup>](#mpi_static)  | `mpi+<protocol>` | `[mpi+]<protocol>://<port>`
na     | sm                   | `na+sm[://<PID>/<ID>]`         | `[na+]sm://<PID>/<ID>`
cci    | tcp, verbs, gni <br/> sm | `cci+<protocol>[://<hostname>:<port>]`[<sup>3</sup>](#cci_config) <br/> `cci+sm[://<PID>/<ID>]` | `[cci+]<protocol>://<hostname>:<port>` <br/> `[cci+]sm://<cci shmem path>/<PID>/<ID>`[<sup>4</sup>](#cci_sm_config)

<a name="init_format"><sup>1</sup></a> When not being initialized in listening mode, the
port specification should be elided.

<a name="mpi_static"><sup>2</sup></a> MPI static mode requires all mercury processes to
be started in the same mpirun invocation.

<a name="cci_config"><sup>3</sup></a> Note that CCI ignores for now the hostname that is passed. Instead please use a `cci.ini` file as well as the `CCI_CONFIG` environment
variable to select the network interface to use. See the CCI `README` files for more
details.

<a name="cci_sm_config"><sup>4</sup></a> The default CCI config uses `/tmp/cci/sm/<hostname>`.
This is not configurable on a per-endpoint basis. When in doubt, use
`HG_Addr_to_string() or NA_Addr_to_string()`. `<ID>` is a 32-bit integer.

## RPC Layer

The _RPC layer_ provides users with the necessary components for
sending and receiving RPCs. This layer is composed of two sub-layers: a core RPC
layer, which defines an RPC operation as a buffer that is sent to a target
and triggers a callback associated to that operation; and a higher level RPC
layer, which includes serialization and deserialization of function arguments.

Every RPC call results in the serialization of function parameters into a
memory buffer (its size generally being limited to a couple of kilobytes, depending on
the interconnect). This buffer is then sent to the target using the network
abstraction layer interface. One of the key requirements is to limit memory
copies at any stage of the transfer, especially when transferring large amounts
of data. Therefore, if the data sent is small, it is serialized and sent by using
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

### Interface

Two choices are offered to start using this layer, either by passing
an existing NA class and an NA context (created by the routines presented
in the [network abstraction layer](#network-abstraction-layer) section):

{% highlight C %}
hg_class_t *HG_Init_na(na_class_t *na_class, na_context_t *na_context);
{% endhighlight %}

Or by letting Mercury initialize its own NA class and context internally:

{% highlight C %}
hg_class_t *HG_Init(const char *info_string, hg_bool_t listen);
{% endhighlight %}

Similarly to the NA layer, initializing the HG interface results in the
creation of a new `hg_class_t` object. The `hg_class_t` object can later be
released after a call to:

{% highlight C %}
hg_return_t HG_Finalize(hg_class_t *hg_class);
{% endhighlight %}

Once the interface has been initialized, a context of execution must be
created, which (similarly to the NA layer) internally associates a specific
completion queue to the operations:

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

hg_id_t HG_Register_name(hg_class_t *hg_class, const char *func_name, hg_proc_cb_t in_proc_cb, hg_proc_cb_t out_proc_cb, hg_rpc_cb_t rpc_cb);
{% endhighlight %}

In the case where an RPC does not require a response, one can indicate that
no response is expected (and therefore avoid waiting for a message to be sent
back) by using the following call (on an already registered RPC):

{% highlight C %}
hg_return_t HG_Registered_disable_response(hg_class_t *hg_class, hg_id_t id, hg_bool_t disable);
{% endhighlight %}

As mentioned previously, there is no distinction between client and server since
it may be desirable for a client to also act as a server for other processes.
Therefore the interface only uses the distinction of _origin_ and _target_.

#### Origin

In a typical scenario, the origin will first lookup a target and get an
address. This can be achieved by:

{% highlight C %}
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t HG_Addr_lookup(hg_context_t *context, hg_cb_t callback, void *arg, const char *name, hg_op_id_t *op_id);
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
hg_return_t HG_Create(hg_context_t *context, hg_addr_t addr, hg_id_t id, hg_handle_t *handle);
{% endhighlight %}

This handle can be destroyed with `HG_Destroy()`---a reference count prevents
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

hg_return_t HG_Forward(hg_handle_t handle, hg_cb_t callback, void *arg, void *in_struct);
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

To be safe, it is therefore recommended to make a copy of the results before
calling `HG_Free_output()`. Note that in the case of an RPC with no response,
completion occurs after the RPC has been successfully sent (i.e., there is
no output to retrieve).

#### Target

On the target, the RPC callback passed to the `HG_Register()` function must be
defined.

{% highlight C %}
typedef hg_return_t (*hg_rpc_cb_t)(hg_handle_t handle);
{% endhighlight %}

When the callback is called, a new RPC has been received. The input arguments
can hence be retrieved with:

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
can then be filled with the return value and/or the output arguments of the function.
It can then be sent back using:

{% highlight C %}
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t HG_Respond(hg_handle_t handle, hg_cb_t callback, void *arg, void *out_struct);
{% endhighlight %}

This call is also non-blocking. When it completes, the associated callback is
placed onto a completion queue. It can then be triggered after a call to
`HG_Trigger()`. Note that in the case of an RPC with no response, calling
`HG_Respond()` will return an error.

#### Progress and Cancellation

Mercury uses a callback model. Callbacks are passed to non-blocking functions
and are pushed to the context's completion queue when the operation completes.
Explicit progress is made by calling `HG_Progress()`. `HG_Progress()` returns when
an operation completes or `timeout` is reached.

{% highlight C %}
hg_return_t HG_Progress(hg_class_t *hg_class, hg_context_t *context, unsigned int timeout);
{% endhighlight %}

When an operation completes, calling `HG_Trigger()` allows the callback
execution to be separately controlled from the main progress loop.

{% highlight C %}
hg_return_t HG_Trigger(hg_class_t *hg_class, hg_context_t *context, unsigned int timeout, unsigned int max_count, unsigned int *actual_count);
{% endhighlight %}

In some cases, one may want to call `HG_Progress()` then `HG_Trigger()` or have
them execute in parallel by using separate threads.

Note also that in cases where it is desirable to cancel an HG operation,
one can call `HG_Cancel()` on a HG handle to cancel an on-going
`HG_Forward()` or `HG_Respond()` operation. Please refer to this
[page]({% post_url 2016-07-26-cancellation %}) for additional details
regarding cancellation of operations.

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
The bulk descriptor can then be serialized and sent to the target along
with the RPC request arguments (using the RPC layer). When the target gets
the input parameters, it can deserialize the bulk descriptor, get the size of
the memory buffer that has to be transferred, and initiate the transfer.
Only the target initiates one-sided transfers so that it can, as well as
controlling the data flow, protect its memory from concurrent accesses.

As no explicit ack message is sent on transfer completion, the origin process
can only assume that accesses to its local memory are completed once it
receives the RPC response from the target. Therefore, in the case of an RPC
with no response, extreme care should be taken when initiating a bulk transfer
by ensuring that the origin gets notified when its exposed memory can be safely
released/accessed.

### Interface

The interface uses the class and execution context that are defined by the
RPC layer.
To initiate a bulk transfer, one needs to create a bulk descriptor on both the
origin and the target, which will then be passed to the `HG_Bulk_transfer()`
call.

{% highlight C %}
hg_return_t HG_Bulk_create(hg_class_t *hg_class, hg_uint32_t count, void **buf_ptrs, const hg_size_t *buf_sizes, hg_uint8_t flags, hg_bulk_t *handle);
{% endhighlight %}

The bulk descriptor can be released by using:

{% highlight C %}
hg_return_t HG_Bulk_free(hg_bulk_t handle);
{% endhighlight %}

For convenience, memory pointers from an existing bulk descriptor can be accessed with:

{% highlight C %}
hg_return_t HG_Bulk_access(hg_bulk_t handle, hg_size_t offset, hg_size_t size, hg_uint8_t flags, hg_uint32_t max_count, void **buf_ptrs, hg_size_t *buf_sizes, hg_uint32_t *actual_count);
{% endhighlight %}

When the bulk descriptor from the origin has been received, the target
can initiate the bulk transfer to/from its own bulk descriptor. Virtual offsets
can be used to transfer data pieces from a non-contiguous block transparently.
The call is non-blocking. When the operation completes, the user callback
is placed onto the context's completion queue.

{% highlight C %}
hg_return_t HG_Bulk_transfer(hg_context_t *context, hg_bulk_cb_t callback, void *arg, hg_bulk_op_t op, hg_addr_t origin_addr, hg_bulk_t origin_handle, hg_size_t origin_offset, hg_bulk_t local_handle, hg_size_t local_offset, hg_size_t size, hg_op_id_t *op_id);
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

## High-level RPC Layer

For convenience, the high-level RPC layer provides macros and routines that can
reduce the amount of code required to send an RPC call with Mercury. For macros,
Mercury makes use of the [Boost preprocessor library](http://www.boost.org/doc/libs/release/libs/preprocessor/) so that users can generate all the boilerplate code that
is necessary to serialize and deserialize function arguments.

### Generate proc routines

The first macro, called `MERCURY_GEN_PROC`, generates both structures and proc functions
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

The following macro can then be used to generate boilerplate code for the input
argument (please refer to the [predefined types](#predefined-types) section for
a list of the existing types that can be passed to this macro):

{% highlight C linenos %}
MERCURY_GEN_PROC( rpc_open_in_t, ((hg_const_string_t)(path)) ((rpc_handle_t)(handle)) )

/* Will generate an rpc_open_in_t struct */

typedef struct {
    hg_const_string_t path;
    rpc_handle_t handle;
} rpc_open_in_t;

/* and an hg_proc_rpc_open_in_t function */

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
when serializing and deserializing it. For convenience, Mercury types can also be
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
MERCURY_REGISTER(hg_class, func_name, in_struct_type_name, out_struct_type_name, rpc_cb)
{% endhighlight %}

#### Example:

{% highlight C %}
int rpc_open(const char *path, rpc_handle_t handle, int *event_id);
{% endhighlight %}

One can use the `MERCURY_REGISTER` macro and pass the types of the input/output
structures directly. In cases where no input or no output argument is present,
the `void` type can be passed to the macro.

{% highlight C %}
rpc_open_id_g = MERCURY_REGISTER(hg_class, "rpc_open", rpc_open_in_t, rpc_open_out_t, rpc_open_cb);
{% endhighlight %}

## See also
<ul>
  {% for post in site.categories.documentation reversed %}
    <li><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></li>
  {% endfor %}
  <li><a href="http://dx.doi.org/10.1109/CLUSTER.2013.6702617">Cluster 2013 paper</a></li>
  <li><a href="{{ site.baseurl }}/doxygen/index.html">Doxygen documentation</a></li>
</ul>

[cma]: https://lwn.net/Articles/405284/

