---
title: Download
---

Mercury is distributed under a [BSD 3-Clause license][license].

## Latest Release

Filename                           | Date       | Size    | Arch | Type
---------------------------------- | ---------- | ------- | ---- | -----------
[mercury-2.4.0.tar.bz2][2.4.0] | 2024-10-28 | 694 kB  | Any  | Source .bz2

Note that releases can also be accessed through [GitHub][gh-releases].

!!! warning

    Always use the tarballs named `mercury-x.tar.bz2` and not the tarballs that GitHub generates from the *"Source code"* link as they do not include `mchecksum`/`kwsys` submodules.

!!! tip

    Mercury packages are also distributed through Spack: [![Spack version][spack-release-svg]][spack-release-link]

## Current development distribution

The mercury repository is hosted on GitHub at:
[https://github.com/mercury-hpc/mercury](https://github.com/mercury-hpc/mercury)

To get the source (read-only access), simply run:
```bash
git clone --recurse-submodules https://github.com/mercury-hpc/mercury.git 
```

[license]: https://github.com/mercury-hpc/mercury/blob/master/LICENSE.txt
[2.4.0]: https://github.com/mercury-hpc/mercury/releases/download/v2.4.0/mercury-2.4.0.tar.bz2
[gh-releases]: https://github.com/mercury-hpc/mercury/releases
[spack-release-svg]: https://img.shields.io/spack/v/mercury.svg?style=plastic
[spack-release-link]: https://spack.readthedocs.io/en/latest/package_list.html#mercury
