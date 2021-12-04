---
title:  "Cancelation and Timeouts"
---

Mercury provides two separate types of transfers: RPC and bulk data. We
present in this page how on-going operations can be canceled and
recovered from a canceled state.

## Pre-requisites

Mercury has been defined as a building block for distributed services.
In that context, adding support for cancelation of mercury operations
is a primary requirement to provide resiliency and allow services to
recover after a fault has occurred (e.g., node failure, etc). This
implies reclaiming resources that canceled operations have previously
allocated. Mercury defines remote lookup operations as well as two types
of transfer operations, RPC and bulk data transfers, which may be
interrupted if any of the party involved no longer responds or reaches a
time of *no response* (e.g., on process termination), in which case
pending operations must be canceled. Canceling an operation that cannot
complete, either because a fault has occurred or a timeout has been
reached, is necessary in order to reach proper completion.

This documentation assumes that the reader already has knowledge of Mercury
and its layers (NA / HG / HG Bulk).

## Progress model

Mercury uses a callback-based mechanism that is
built on top of the network abstraction (NA) layer’s callback mechanism.
A callback mechanism presents two advantages compared to a traditional
request based model: there is no explicit `wait()` control point;
cancelation of operations can be easily done without any additional
code branching.

Mercury’s progress is directly driven by NA’s progress. When an NA
operation completes, an internal callback is pushed to the NA’s
completion queue. To make progress on Mercury’s operations, the Mercury
layer also internally triggers these NA callback operations. Triggering
NA operations may in turn result in the completion of Mercury
operations. When these operations complete, the user callback that is
associated with these operations is in turn pushed to a completion
queue.

RPC operations are associated with a *handle* that is explicitly created
by the user and linked to an execution target. The forward call is
passed the handle along with a structure holding the input arguments,
serializes input arguments, and passes the RPC parameters to the target.
On completion, the user callback passed to the forward call is pushed to
a completion queue. When an RPC forward operation completes, the *get
output* call is passed that same handle to retrieve output arguments,
deserializes them. Note that the handle can be safely re-used after
completion (or cancelation) to issue another RPC to the same target.
When the user no longer needs to operate on a given handle, it can be
explicitly destroyed; a reference count provides operation safety.

## RPC cancelation

In that context, Mercury’s RPC cancelation can be defined so that:

-   No explicit tracking of handles is required (user calls
    `HG_Destroy()`).
-   User callbacks are pushed to the completion queue with a canceled
    state.
-   Canceled handles can be reused to retry to forward a call to a
    target.

When cancelation is done on an HG operation, cancelation is also done
internally on the NA operations that were involved in that HG operation.
When this cancelation is successful and the NA operations complete with
a canceled state, the HG callbacks associated to the NA operations are
pushed to the completion queue. When these callbacks are executed with a
canceled state, the actual HG operation is successfully canceled and the
state of the operation passed to the callback function indicates that
the operation was canceled.

!!! warning
        It is important to note that cancelation is always *local*, in the
        sense that there is no communication involved with a remote party (i.e.,
        it does not rely on the remote party being able to communicate).

It is also worth noting that the main focus of our implementation of
mercury’s RPC cancelation is the recovery from faults, and not the
support of RPC transfer scenarios that rely on cancelation (e.g.,
optimistic queries in a data service, or higher level protocol
operations, which could theoretically also be supported).

Xancelation can be supported for the following operations:
-   `HG_Forward()`, `HG_Respond()` (RPC operations)
-   `HG_Bulk_transfer()` (Bulk operations)

We define the following calls:

-   RPC operations:
    ```C
        hg_return_t HG_Cancel(hg_handle_t handle);
    ```

-   Bulk operations:
    ```C
        hg_return_t HG_Bulk_cancel(hg_op_id_t op_id);
    ```

## Use Cases

!!! example

    _Cancelation of an RPC operation without bulk transfer involved_

    In the case described below, a handle is created and forwarded to a
    target. After reaching timeout, the current operation referenced by the
    handle is canceled. The `forward_cb` user callback is pushed to the
    context’s completion queue with a canceled return state.

    ```C
    static hg_return_t
    forward_cb(const struct hg_cb_info *cb_info)
    {
        if (cb_info->ret == HG_CANCELED) {
            /* Canceled */
        }
        if (cb_info->ret == HG_SUCCESS) {
            /* Sucessfully executed */
        }
    }

    int
    main(void)
    {
        hg_handle_t handle;

        /* Create new HG handle */
        HG_Create(hg_context, target, rpc_id, &hg_handle);
        /* Encode RPC */
        ...
        /* Forward call */
        HG_Forward(hg_handle, forward_cb, forward_cb_args, in_struct);
        /* No progress after timeout */
        ...
        /* Cancel operation */
        HG_Cancel(hg_handle);
        /* Trigger user callback */
        HG_Trigger(hg_context);
        /* Destroy handle */
        HG_Destroy(hg_handle);
    }
    ```

!!! example

    _Cancelation of an RPC operation with bulk transfer involved_

    In that case, the RPC operation additionally involves a bulk transfer
    that is initiated by a remote target. A bulk handle that describes the
    memory region is first created and registered to the NIC. This bulk
    handle is then serialized along with the RPC arguments, thereby
    exposing/publishing the memory region to the target. The RPC handle is
    similarly forwarded to the target. After reaching timeout, the handle is
    canceled. The `forward_cb` user callback is pushed to the context’s
    completion queue with a canceled return state.

    It is worth noting that when the RPC operation is canceled, the target
    may be in the process of accessing the exposed bulk region. To guarantee
    prevention of further remote accesses to the region involved in the bulk
    transfer, the bulk transfer must be separately destroyed and unpublished
    (note that this guaranty is only valid if the underlying NA plugin
    supports it). The bulk handle is part of the input data structure and
    can be referenced within the RPC handle callback function for the
    purpose of separately canceling the remote bulk operation.

    ```C
    static hg_return_t
    forward_cb(const struct hg_cb_info *cb_info)
    {
        if (cb_info->ret == HG_CANCELED) {
            /* Canceled */
        }
        if (cb_info->ret == HG_SUCCESS) {
            /* Sucessfully executed */
        }
    }

    int
    main(void)
    {
        hg_handle_t handle;
        hg_bulk_t bulk_handle;

        /* Create new HG bulk handle */
        HG_Bulk_create(hg_class, buffer_ptrs, buffer_sizes,
                    &hg_bulk_handle);
        /* Create new HG handle */
        HG_Create(hg_context, target, rpc_id, &hg_handle);
        /* Encode RPC and bulk handle */
        in_struct.bulk_handle = bulk_handle;
        ...
        /* Forward call */
        HG_Forward(hg_handle, forward_cb, forward_cb_args, &in_struct);
        /* No progress after timeout */
        ...
        /* Cancel operation */
        HG_Cancel(hg_handle);
        /* Trigger user callback */
        HG_Trigger(hg_context);
        /* Destroy HG handle */
        HG_Destroy(hg_handle);
        /* Destroy HG bulk handle */
        HG_Bulk_destroy(hg_bulk_handle);
    }
    ```

!!! example

    _Cancelation of a bulk operation_

    Bulk cancelation follows a similar model. However, bulk operations are
    initiated by an RPC target, not by the origin. Bulk operations are
    identified by an operation ID, which gets freed when the bulk callback
    is executed.

    ```C
    static hg_return_t
    bulk_cb(const struct hg_cb_info *cb_info)
    {
        if (cb_info->ret == HG_CANCELED) {
            /* Canceled */
        }
        if (cb_info->ret == HG_SUCCESS) {
            /* Sucessfully executed */
        }
    }

    static hg_return_t
    rpc_cb(hg_handle_t handle)
    {
        hg_bulk_t origin_handle, local_handle;

        /* Setup handles etc */
        ...
        /* Start the transfer */
        HG_bulk_transfer(context, bulk_cb, bulk_cb_args, HG_BULK_PULL,
                        origin_addr, origin_handle, origin_offset,
                        local_handle, local_offset, size, &op_id);
        /* No progress after timeout */
        ...
        /* Cancel operation */
        HG_Bulk_cancel(op_id);
        /* Trigger user callback */
        HG_Trigger(context);
    }
    ```

## Timeouts and Retries

Mercury does not currently support timeouts on operations because this
would require tracking handles / operation IDs, which could lead to
unnecessary overheads within the Mercury layer. For cases where a
timeout is desired, the application can wrap around Mercury calls. The
example below shows how one can use the HG request emulation library to
issue an RPC call with timeout.

!!! example

    ```C
    static hg_return_t
    forward_cb(const struct hg_cb_info *cb_info)
    {
        hg_request_t *request = (hg_request_t *) cb_info->arg;
        if (cb_info->ret == HG_CANCELED) {
            /* Canceled */
        }
        if (cb_info->ret == HG_SUCCESS) {
            /* Sucessfully executed */
        }
        hg_request_complete(request);
    }

    int
    main()
    {
        hg_handle_t handle;
        hg_request_t *request;

        /* Create new HG handle */
        HG_Create(hg_context, target, rpc_id, &hg_handle);
        /* Encode RPC */
        ...
        /* Create new request */
        request = hg_request_create(request_class);
        /* Forward call */
        HG_Forward(hg_handle, forward_cb, request, &in_struct);
        /* Wait for completion */
        hg_request_wait(request, timeout, &completed);
        if (!completed)
            HG_Cancel(hg_handle);
        /* Wait for completion */
        hg_request_wait(request, timeout, &completed);
        /* Destroy request */
        hg_request_destroy(request);
        /* Destroy handle */
        HG_Destroy(hg_handle);
    }
    ```

## Cancelation in NA

Cancelation of HG operations can only be realized by first supporting
cancelation at the NA layer. Cancelation is supported via the
following call:

```C
na_return_t
NA_Cancel(na_class_t *na_class, na_context_t *context, na_op_id_t op_id);
```

An additional `cancel` callback is added to the NA layer that allows
plugin developers to support cancelation of non-blocking operations.
These include:

-   `NA_Msg_send_unexpected()`, `NA_Msg_recv_unexpected()`
-   `NA_Msg_send_expected()`, `NA_Msg_recv_expected()`
-   `NA_Put()`, `NA_Get()`

Cancelation of NA operations is internally progressed and actually
completes when internal plugin cancelation has been successfully
completed (which may or may not be immediate). When an NA operation is
successfully canceled, the internal callback associated to the HG
operation is placed onto the NA context’s completion queue with a
`NA_CANCELED` return state.

Cancelation is supported for all plugins. It is worth
noting that BMI and MPI emulate one-sided operations (`NA_Put()`,
`NA_Get()`) on top of two-sided operations and cancelation for these
operations is not yet supported. To emulate these operations, send/recv
requests are sent to the remote target which may issue a send in the
case of a get, or issue a recv in the case of a put. Cancelation of
these operations implies cancelation at the target of the potentially
issued send/recv operations, which can only be done using timeouts
(remote notification not being an option because the caller has no
guaranty that the target is still alive).

Cancelation capabilities and execution scenarios where cancelation is
supported (i.e., not only for fault tolerance) only depend on the
underlying NA plugins and their capabilities to support cancelation of
on-going transfers. The BMI plugin for example may not recover well from
cancelation of transfers when using TCP and a rendez-vous protocol, as
the TCP channel must be entirely flushed before one can do further
communication. This may be compromising if other operations were also in
the pipe at that time as these operations may consequently fail.

## Conclusion

Adding cancelation to mercury is an important step for building
resilient services and a required component for the definition of future
high-level mercury features such as group membership and pub/sub
services, where fault tolerance must be considered in order to prevent
collective failure.
