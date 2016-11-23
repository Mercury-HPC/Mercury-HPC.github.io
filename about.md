---
layout: page
title: About
permalink: /about/
---

Engineers and scientists working in a heterogeneous computing environment
often want to distribute the various tasks of an application---such as
computation, storage, analysis, or visualization---to different types of
resources and libraries. Often, a technique known as remote procedure call
(RPC) is used that allows local calls to be transparently executed onto remote
resources.

Using RPC on a high-performance computing system presents two limitations,
however: the inability to take advantage of the native network transport and
the inability to transfer large amounts of data.

To avoid these limitations, we define Mercury---an RPC interface specifically
designed for high-performance computing. Mercury builds on a small, easily
ported network abstraction layer, providing operations closely matched to the
capabilities of high-performance network environments. Unlike most other RPC
frameworks, Mercury directly supports handling remote calls containing large
data arguments. Moreover, Mercury’s network protocol is designed to scale to
thousands of clients.

## Collaborators

<pre style="border: 0; padding: 0; overflow-x: visible;">
<table><colgroup><col style="width: 33%" /><col style="width: 33%" /><col style="width: 33%" /></colgroup><tbody><tr>
<td>
<figure><img width="80%" src="/assets/anl_logo.png" alt="ANL Logo" /></figure>
</td>
<td>
<figure><img width="60%" src="/assets/intel_logo.svg" alt="Intel Logo" /></figure>
</td>
<td>
<figure><img width="100%" src="/assets/hdf_logo.svg" alt="HDF Logo" /></figure>
</td>
</tr></tbody></table>
</pre>

