---
title:  "Libfabric Plugin"
---

!!! note

    The implementation notes in this page are provided for reference and
    may be slightly out of date. If you are only looking for user documentation,
    please refer to the NA plugin documentation on this [page](na.md#ofi).

## OFI Capabilities

The OFI plugin makes use of the following libfabric capabilities:

* `FI_EP_RDM`
* `FI_DIRECTED_RECV`
* `FI_READ`
* `FI_RECV`
* `FI_REMOTE_READ`
* `FI_REMOVE_WRITE`
* `FI_RMA`
* `FI_SEND`
* `FI_TAGGED`
* `FI_WRITE`
* `FI_SOURCE`
* `FI_SOURCE_ERR`
* `FI_ASYNC_IOV`
* `FI_CONTEXT`
* `FI_MR_BASIC`
* Scalable endpoints

## Feature Matrix

 * Supported :heavy_check_mark:
 * Limited support or emulated :exclamation: (see footnote)
 * Not supported :x:

Feature | tcp | verbs | psm2 | gni
--------|-----|-------|------|-------------
Source Addressing | :exclamation:[<sup>1</sup>](#addressing_emul) | :exclamation:[<sup>1</sup>](#addressing_emul) | :heavy_check_mark: | :heavy_check_mark:
Manual Progress | :exclamation:[<sup>2</sup>](#progress_emul) | :heavy_check_mark: | :exclamation:[<sup>2</sup>](#progress_emul) | :exclamation:[<sup>2</sup>](#progress_emul)
`FI_WAIT_FD` | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :x:
Scalable Endpoints | :exclamation:[<sup>3</sup>](#scal_emul) | :x: | :heavy_check_mark: | :heavy_check_mark:

<a name="addressing_emul"><sup>1</sup></a> *Emulated:* source address is encoded
in the message header.

<a name="progress_emul"><sup>2</sup></a> *Emulated:* the provider is using a thread to drive background progress.

<a name="scal_emul"><sup>3</sup></a> *Emulated:* provider resources are shared.

## Performance

Below is a performance comparison of the libfabric plugin  with available libfabric providers when using both
wait and busy spin mechanisms, mercury v1.0.0 and libfabric v1.7.0a1.

### InfiniBand `verbs;rxm`

Performance is measured between two nodes on ALCF's [cooley](https://www.alcf.anl.gov/user-guides/cooley) cluster using the FDR InfiniBand interface `mlx5_0`.
The following plot shows the RPC average time
compared to `ofi+tcp` when using one single RPC in-flight:

<figure>
  <img src="../../assets/ofi/verbs/rpc_time1.svg" alt="RPC time 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/verbs/rpc_time16.svg" alt="RPC time 16" width="80%">
</figure>

The following plot shows the RPC with *pull* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="../../assets/ofi/verbs/write_bw1.svg" alt="Write Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/verbs/write_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

The following plot shows the RPC with *push* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="../../assets/ofi/verbs/read_bw1.svg" alt="Read Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/verbs/read_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

### Omni-Path `psm2`

Performance is measured between two nodes on LCRC's [bebop](https://www.lcrc.anl.gov/systems/resources/bebop/) cluster using the Omni-Path interface with PSM2 v11.2.23.
The following plot shows the RPC average time
compared to `ofi+tcp` when using one single RPC in-flight:

<figure>
  <img src="../../assets/ofi/psm2/rpc_time1.svg" alt="RPC time 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/psm2/rpc_time16.svg" alt="RPC time 16" width="80%">
</figure>

The following plot shows the RPC with *pull* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="../../assets/ofi/psm2/write_bw1.svg" alt="Write Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/psm2/write_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

The following plot shows the RPC with *push* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="../../assets/ofi/psm2/read_bw1.svg" alt="Read Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/psm2/read_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

### Aries `gni`

Performance is measured between two Haswell nodes (debug queue in exclusive mode) on Nersc's [cori](http://www.nersc.gov/users/computational-systems/cori/) system using the interface `ipogif0` with uGNI v6.0.14.0.
The following plot shows the RPC average time
compared to `ofi+tcp` when using one single RPC in-flight:

<figure>
  <img src="../../assets/ofi/gni/rpc_time1.svg" alt="RPC time 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/gni/rpc_time16.svg" alt="RPC time 16" width="80%">
</figure>

The following plot shows the RPC with *pull* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="../../assets/ofi/gni/write_bw1.svg" alt="Write Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/gni/write_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

The following plot shows the RPC with *push* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="../../assets/ofi/gni/read_bw1.svg" alt="Read Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="../../assets/ofi/gni/read_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>
