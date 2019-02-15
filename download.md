---
layout: page
title: Download
permalink: /download/
---

Mercury is distributed under a [BSD-like license][license].

## Releases [![Latest version][mercury-release-svg]][mercury-release-link]

Filename                           | Date       | Size    | Arch | Type
---------------------------------- | ---------- | ------- | ---- | -----------
[mercury-1.0.1.tar.bz2][1.0.1] | 2019-02-15 | 584 kB  | Any  | Source .bz2

Releases can also be accessed through [GitHub][gh-releases], beware that
generated tarballs accessible from the *"Source code"* link do not include
mchecksum/kwsys submodules. Older releases are also available [here][ftp]. 

## Current development distribution

The mercury repository is hosted on GitHub at:
[https://github.com/mercury-hpc/mercury](https://github.com/mercury-hpc/mercury)

To get the source (read-only access), simply run:
{% highlight bash %}
git clone --recurse-submodules https://github.com/mercury-hpc/mercury.git 
{% endhighlight %}

[mercury-release-svg]: https://img.shields.io/github/release/mercury-hpc/mercury.svg
[mercury-release-link]: https://github.com/mercury-hpc/mercury/releases/latest
[license]: https://github.com/mercury-hpc/mercury/blob/master/COPYING
[1.0.1]: https://github.com/mercury-hpc/mercury/releases/download/v1.0.1/mercury-1.0.1.tar.bz2
[ftp]: ftp://ftp.mcs.anl.gov/pub/mercury/releases/
[gh-releases]: https://github.com/mercury-hpc/mercury/releases
