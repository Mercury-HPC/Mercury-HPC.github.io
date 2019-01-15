---
layout: post
title:  "Release of mercury v1.0.0"
date:   2018-11-18
categories: news
---

We are very pleased to announce that the release of mercury 1.0.0 is
now available and can be downloaded from:
        [http://mercury-hpc.github.io/download/][downloads]

Updated documentation is available at:
        [http://mercury-hpc.github.io/documentation/][documentation]

Please submit any issue you may encounter to:
        [https://github.com/mercury-hpc/mercury/issues][issues]

Major features since 0.9.0 (pre-release) include support for libfabric
on traditional IP and IB networks, as well as Intel<sup>®</sup> Omni-Path (PSM2)
and Cray<sup>®</sup> Aries (GNI) fabrics. Please see the accompanying [changelog][changelog] for
a full list of features and bug fixes. Non exhaustive list:
 * Add optional busy spin mode through `NA_NO_BLOCK` init info
 * Add support for scalable endpoints and progress (limited to
libfabric plugin) through max_contexts init info
 * Add ability to pass authorization keys to establish communication
using DRC w/Cray GNI
 * Support for response over eager message size limit
 * Improved performance of shared-memory plugin

[downloads]: {{ site.baseurl }}/download
[documentation]: {{ site.baseurl }}/documentation
[issues]: https://github.com/mercury-hpc/mercury/issues
[changelog]: https://github.com/mercury-hpc/mercury/blob/v1.0.0/CHANGELOG
