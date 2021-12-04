---
title: About
hide:
  - navigation
  - toc
---

Engineers and scientists working in a heterogeneous computing environment
often want to distribute the various tasks of an application&mdash;such as
computation, storage, analysis, or visualization&mdash;to different types of
resources and libraries. Often, a technique known as Remote Procedure Call
(RPC) is used that allows local calls to be transparently executed onto remote
resources.

Using common RPC frameworks on a High-Performance Computing (HPC) system presents
two limitations, however: the inability to take advantage of the native network
transport and the inability to transfer large amounts of data.

To avoid these limitations, we developed Mercury&mdash;an RPC interface specifically
designed for HPC. Mercury builds on a small, easily
ported network abstraction layer, providing operations closely matched to the
capabilities of high-performance network environments. Unlike most other RPC
frameworks, Mercury directly supports handling remote calls containing large
data arguments. Moreover, Mercury’s network protocol is designed to scale to
thousands of clients.

## Collaborators

<table style="border: 0;">
    <colgroup>
        <col style="width: 33%" />
        <col style="width: 33%" />
        <col style="width: 33%" />
    </colgroup>
    <tbody>
        <tr>
            <td style="border: 0;">
                <figure>
                    <img width="200px" src="../assets/images/anl_logo.svg" alt="ANL Logo" />
                </figure>
            </td>
            <td style="border: 0;">
                <figure>
                    <img width="150px" src="../assets/images/intel_logo.svg" alt="Intel Logo" />
                </figure>
            </td>
            <td style="border: 0;">
                <figure>
                    <img width="200px" src="../assets/images/hdf_logo.svg" alt="HDF Logo" />
                </figure>
            </td>
        </tr>
    </tbody>
</table>
