---
title: Mercury Bulk Layer
---

In addition to the previous layer, some RPCs may require the transfer of larger
amounts of data. For these RPCs, the _bulk layer_ can be used. It is built on
top of the RMA protocol defined in the network abstraction layer and prevents
intermediate memory copies.

## HG Bulk Interface

The interface allows origin processes to expose a memory region to the target by creating a bulk
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

### Descriptors

The interface uses the class and execution context that are defined by the
HG RPC layer.
To initiate a bulk transfer, one needs to create a bulk descriptor on both the
origin and the target, which will later be passed to the `HG_Bulk_transfer()`
call.

```C
hg_return_t
HG_Bulk_create(hg_class_t *hg_class, hg_uint32_t count,
               void **buf_ptrs, const hg_size_t *buf_sizes,
               hg_uint8_t flags, hg_bulk_t *handle);
```

The bulk descriptor can be released by using:

```C
hg_return_t HG_Bulk_free(hg_bulk_t handle);
```

For convenience, memory pointers from an existing bulk descriptor can be
accessed with:

```C
hg_return_t
HG_Bulk_access(hg_bulk_t handle, hg_size_t offset, hg_size_t size,
               hg_uint8_t flags, hg_uint32_t max_count, void **buf_ptrs,
               hg_size_t *buf_sizes, hg_uint32_t *actual_count);
```

Additionally, it is also possible to bind the origin's address to the bulk handle
through the `HG_Bulk_bind()` function at the cost of an additional overhead
for serializing and deserializing addressing information. This should only be
necessary when the source address retrieved from the `HG_Get_info()` call is
different from the one that must be used for the transfer (e.g., multiple sources).

```C
hg_return_t
HG_Bulk_bind(hg_bulk_t handle, hg_context_t *context);
```

In that particular case, the address information can be directly retrieved using:

```C
hg_addr_t
HG_Bulk_get_addr(hg_bulk_t handle);
```

### Serialization

Serialization and deserialization of bulk handles should never be explicitly done
by users and we instead encourage the use of the mercury proc routines that
provide an `hg_proc_hg_bulk_t` routine:

```C
hg_return_t
hg_proc_hg_bulk_t(hg_proc_t proc, void *data);
```

For more details on RPC argument serialization, please refer to this [section](hg_macros.md).

### Bulk Transfer

When the bulk descriptor from the origin has been received, the target
can initiate the bulk transfer to/from its own bulk descriptor. Virtual offsets
can be used to transfer data pieces from a non-contiguous block transparently.
The call is non-blocking. When the operation completes, the user callback
is placed onto the context's completion queue.

```C
hg_return_t
HG_Bulk_transfer(hg_context_t *context, hg_bulk_cb_t callback, void *arg,
                 hg_bulk_op_t op, hg_addr_t origin_addr,
                 hg_bulk_t origin_handle, hg_size_t origin_offset,
                 hg_bulk_t local_handle, hg_size_t local_offset,
                 hg_size_t size, hg_op_id_t *op_id);
```

Note that for convenience, as the transfer needs to be realized within the
RPC callback on the RPC target, the routine `HG_Get_info()` enables easy
retrieval of classes, contexts and source address:

```C
struct hg_info {
    hg_class_t *hg_class;               /* HG class */
    hg_context_t *context;              /* HG context */
    hg_addr_t addr;                     /* HG address */
    hg_id_t id;                         /* RPC ID */
};

struct hg_info *
HG_Get_info(hg_handle_t handle);
```
