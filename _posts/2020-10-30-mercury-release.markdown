---
layout: post
title:  "Release of mercury v2.0.0"
date:   2020-10-30
categories: news
---

We are very pleased to announce that the release of mercury 2.0.0 is
now available and can be downloaded from:
        [https://mercury-hpc.github.io/download/][downloads]
or
        [https://github.com/mercury-hpc/mercury/releases/tag/v2.0.0][github]

Spack package update will also be available soon.

Major features include:
 * Support for immediate lookups through `HG_Addr_lookup2()` (note that
the callback version of `HG_Addr_lookup()` will be deprecated in a future
release)
 * Improved support of libfabric and support of new `tcp` provider
 * Improved shared-memory plugin with full connection-less endpoints
support
 * Improved bulk interface with more efficient handling of I/O with
small segment count
 * Improved efficiency of mercury proc routines
 * Improved polling mechanism
 * Improved cancellation of operations and error handling
 * Improved error / warning and debug logging
 * Removed CMake options (`MERCURY_USE_SM_ROUTING`,
`MERCURY_ENABLE_POST_LIMIT`, `MERCURY_USE_SELF`, `MERCURY_USE_EAGER_BULK`),
options are now controlled at runtime, through `HG_Init_opt()`.

Please see the accompanying changelog for more details:
        Please see the accompanying [changelog][changelog] for more details.

Please submit any issue you may encounter to:
        [https://github.com/mercury-hpc/mercury/issues][issues]

Thanks again to everyone who contributed to this release.

[downloads]: {{ site.baseurl }}/download
[documentation]: {{ site.baseurl }}/documentation
[github]: https://github.com/mercury-hpc/mercury/releases/tag/v2.0.0
[issues]: https://github.com/mercury-hpc/mercury/issues
[changelog]: https://github.com/mercury-hpc/mercury/blob/v2.0.0/CHANGELOG
