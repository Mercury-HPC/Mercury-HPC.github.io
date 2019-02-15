---
layout: post
title:  "Release of mercury v1.0.1"
date:   2019-02-15
categories: news
---

We are very pleased to announce that the release of mercury 1.0.1 is
now available and can be downloaded from:
        [http://mercury-hpc.github.io/download/][downloads]

Updated documentation is available at:
        [http://mercury-hpc.github.io/documentation/][documentation]

Please submit any issue you may encounter to:
        [https://github.com/mercury-hpc/mercury/issues][issues]

This is mostly a bug fix release, which includes:
 * Update CMake policy for CMake 3.12 and above
 * Fix potential race when forcing a handle to be destroyed
 * Fix cancelation of HG operations (fix #267)
 * Fix HG_Reset() to reset NA resources upon NA class change (fix #272)
 * NA OFI: fix cancelation of operations that cannot be canceled
 * NA OFI: remove extra fi_addr_xxx prefix from generated URI
 * NA SM: remove page size check that would prevent to run on system w pages larger than 4KB (fix #268)

 Please see the accompanying [changelog][changelog] for more details.

[downloads]: {{ site.baseurl }}/download
[documentation]: {{ site.baseurl }}/documentation
[issues]: https://github.com/mercury-hpc/mercury/issues
[changelog]: https://github.com/mercury-hpc/mercury/blob/v1.0.1/CHANGELOG
