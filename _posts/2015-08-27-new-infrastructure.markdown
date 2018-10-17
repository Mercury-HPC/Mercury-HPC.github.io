---
layout: post
title:  "Mercury has moved to GitHub!"
date:   2015-08-27
categories: news
---

After a few weeks of testing, the mercury development infrastructure has been moved
to GitHub. One of the main reasons for us to go in that direction is to make it easier for
users and developers to collaborate and be able to administrate services
without requiring any formal access to ANL.

Mercury's _organisation page_ is now located at:
[https://github.com/mercury-hpc](https://github.com/mercury-hpc)

The main _mercury repository_ can now be found at:
[https://github.com/mercury-hpc/mercury](https://github.com/mercury-hpc/mercury)

_Issue tracking_ is now also handled through GitHub:
[https://github.com/mercury-hpc/mercury/issues](https://github.com/mercury-hpc/mercury/issues)

_Continous testing_ has been set up with [Travis](https://travis-ci.org/) so that whenever something gets pushed to the mercury master branch,
it is tested with main NA plugins:
[https://travis-ci.org/mercury-hpc/mercury](https://travis-ci.org/mercury-hpc/mercury)
and then submitted to the CMake dashboard:
[https://cdash.hdfgroup.org/index.php?project=MERCURY&date=2015-07-15](https://cdash.hdfgroup.org/index.php?project=MERCURY&date=2015-07-15)

