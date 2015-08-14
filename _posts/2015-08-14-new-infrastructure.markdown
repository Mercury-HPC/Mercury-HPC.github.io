---
layout: post
title:  "Mercury has moved to GitHub!"
date:   2015-08-14
categories: news
---

After some experimentation, we moved all the mercury development infrastructure
to GitHub. One of the main reasons for doing that is to make it easier for
users and developers to collaborate and be able to administrate services
without requiring any formal access to ANL.

Mercury's _organisation page_ is now located there:
[https://github.com/mercury-hpc](https://github.com/mercury-hpc)

The main _mercury repository_ is there:
[https://github.com/mercury-hpc/mercury](https://github.com/mercury-hpc/mercury)

_Issue tracking_ is now also handled through GitHub:
[https://github.com/mercury-hpc/mercury/issues](https://github.com/mercury-hpc/mercury/issues)

_Continous testing_ has been set up with [Travis](https://travis-ci.org/) so that whenever we push something,
it is tested with BMI/MPI/CCI plugins:
[https://travis-ci.org/mercury-hpc/mercury](https://travis-ci.org/mercury-hpc/mercury)
and then submitted to the CMake dashboard:
[https://cdash.hdfgroup.org/index.php?project=MERCURY&date=2015-07-15](https://cdash.hdfgroup.org/index.php?project=MERCURY&date=2015-07-15)
The dashboard is also being updated to the latest version so things should also
be nicer on this side soon.

The new website is based on [Jekyll](http://jekyllrb.com/), which makes it easier to add new pages and
code-related stuff than the old wordpress one.
Regarding _mailing-lists_, we will keep them hosted at ANL for now as it seems
to be working fine this way.

