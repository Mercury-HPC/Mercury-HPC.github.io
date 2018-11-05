---
layout: post
title:  "Libfabric plugin for Mercury"
date:   2018-11-02 17:21:15
categories: documentation
---

## Introduction

This document assumes that the reader already has knowledge of Mercury and its
layers (NA / HG / HG Bulk).

## Requirements

## OFI Plugin Architecture

## Performance

Below is a performance comparison of the libfabric plugin  with available libfabric providers when using both
wait and busy spin mechanisms, mercury v1.0.0 and libfabric v1.7.0a1.

<hr/>
### verbs (rxm)

The following plot shows the RPC average time
compared to `ofi+tcp` when using one single RPC in-flight:

<figure>
  <img src="/assets/ofi_verbs/rpc_time1.svg" alt="RPC time 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="/assets/ofi_verbs/rpc_time16.svg" alt="RPC time 16" width="80%">
</figure>

The following plot shows the RPC with *pull* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="/assets/ofi_verbs/write_bw1.svg" alt="Write Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="/assets/ofi_verbs/write_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

The following plot shows the RPC with *push* bulk transfer performance
compared to `ofi+tcp` with various transfer sizes:

<figure>
  <img src="/assets/ofi_verbs/read_bw1.svg" alt="Read Bandwidth 1" width="80%">
</figure>

Same plot but with 16 RPCs in-flight:

<figure>
  <img src="/assets/ofi_verbs/read_bw16.svg" alt="Write Bandwidth 16" width="80%">
</figure>

<hr/>
### psm2

TBD

<hr/>
### gni

TBD
