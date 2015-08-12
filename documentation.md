---
layout: page
title: Documentation
permalink: /documentation/
---

Documentation reflects mercury API as of v0.9.0 and can be used as an introduction to
the mercury library. Please look at the [see also](#see-also) section for
additional documentation.

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
Calling functions with relatively small arguments results in using the short
messaging mechanism exposed by the network abstraction layer, whereas functions
containing large data arguments additionally use the remote memory access (RMA)
mechanism.

## Network Abstraction Layer

The _network abstraction_ (NA) layer is used by both the RPC layer and the bulk layer.
It provides a minimal set of function calls that abstract the underlying network
fabric, and that can be used to provide:
connection/disconnection to/from a target, point-to-point messaging,
remote memory access. The API is non-blocking and uses a callback mechanism
so that upper layers can provide asynchronous execution more easily: when progress
is made (either internally or after a call to `NA_Progress()`) and an operation
completes, the user callback is placed onto a completion queue. The callback
can then be dequeued and separately executed after a call to `NA_Trigger()`.

The NA layer uses a plugin mechanism so that support for various network protocols
can be easily added and selected at runtime.

### Interface

Typically, the first step before being able to use the mercury layer is to
initialize the NA interface and select an underlying plugin that will be used
by the upper layers. Initializing the NA interface with a specified `info_string`
string results in the creation of a new `na_class_t` object. Please refer to
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

After the interface initialized, one may also get its own address, using:

{% highlight C %}
na_return_t NA_Addr_self(na_class_t *na_class, na_addr_t *addr);
{% endhighlight %}

Convert it to a string using:

{% highlight C %}
na_return_t NA_Addr_to_string(na_class_t *na_class, char *buf, na_size_t buf_size, na_addr_t addr);
{% endhighlight %}

And communicate it to other processes (e.g., using a file), which can then look
it up using:

{% highlight C %}
na_return_t NA_Addr_lookup_wait(na_class_t *na_class, const char *name, na_addr_t *addr);
{% endhighlight %}

Or the non-blocking callback version:

{% highlight C %}
na_return_t NA_Addr_lookup(na_class_t *na_class, na_context_t *context, na_cb_t callback, void *arg, const char *name, na_op_id_t *op_id);
{% endhighlight %}

All addresses must then be freed using:

{% highlight C %}
na_return_t NA_Addr_free(na_class_t *na_class, na_addr_t addr);
{% endhighlight %}

Other routines such as `NA_Msg_send_unexpected()`, `NA_Msg_send_expected()`,
`NA_Msg_recv_unexpected()`, `NA_Msg_recv_expected()`, `NA_Put()`, `NA_Get()`, etc,
are used internally. There should not be any need to use them directly.


### Available Plugins

* _BMI_: flexible connection and disconnection. Provides stable TCP implementation,
  other protocols are not supported. Plugin is very stable but it may
  not perform as well as other plugins. Remote memory access is emulated on top
  of point-to-point messaging.
* _CCI_: flexible connection and disconnection. Provides support for TCP, UDP,
  verbs, uGNI and shared memory. Plugin shows best performance but is more
  experimental.
* _MPI_: static connection and disconnection, although connection can be
  established using either `MPI_Comm_split` or `MPI_Comm_connect`. Plugin provides
  relatively good performance depending on the underlying transport used by the
  MPI implementation. Remote memory access is emulated on top of point-to-point
  messaging.
* _SSM_: experimental and unsupported.

Plugins can be selected by passing a string of the form: `plugin+protocol://host:port`.
Below is a table summarizing the values that are available for each plugin:

plugin | protocol
------ | --------
bmi    | tcp
cci    | tcp, verbs, gni, sm
mpi    | default

## RPC Layer

## Bulk Layer

## High-level RPC Layer

## See also
<ul>
  {% for post in site.categories.documentation  reversed %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
  <li><a href="ftp://ftp.mcs.anl.gov/pub/mercury/documents/doxygen_doc/index.html">Doxygen documentation</a></li>
</ul>

