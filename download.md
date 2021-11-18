---
layout: page
title: Download
permalink: /download/
---

Mercury is distributed under a [BSD 3-Clause license][license].

## Releases

Filename                           | Date       | Size    | Arch | Type
---------------------------------- | ---------- | ------- | ---- | -----------
[mercury-2.0.1.tar.bz2][2.0.1] | 2021-05-07 | 601 kB  | Any  | Source .bz2

Releases can also be accessed through [GitHub][gh-releases], beware that
generated tarballs accessible from the *"Source code"* link do not include
mchecksum/kwsys submodules.

Mercury packages are also distributed through Spack: [![Spack version][spack-release-svg]][spack-release-link]

## Current development distribution

The mercury repository is hosted on GitHub at:
[https://github.com/mercury-hpc/mercury](https://github.com/mercury-hpc/mercury)

To get the source (read-only access), simply run:
{% highlight bash %}
git clone --recurse-submodules https://github.com/mercury-hpc/mercury.git 
{% endhighlight %}

[license]: https://github.com/mercury-hpc/mercury/blob/master/LICENSE.txt
[2.0.1]: https://github.com/mercury-hpc/mercury/releases/download/v2.0.1/mercury-2.0.1.tar.bz2
[gh-releases]: https://github.com/mercury-hpc/mercury/releases
[spack-release-svg]: https://img.shields.io/spack/v/mercury.svg?style=plastic
[spack-release-link]: https://spack.readthedocs.io/en/latest/package_list.html#mercury
