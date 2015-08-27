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
it is tested with BMI/MPI/CCI plugins:
[https://travis-ci.org/mercury-hpc/mercury](https://travis-ci.org/mercury-hpc/mercury)
and then submitted to the CMake dashboard:
[https://cdash.hdfgroup.org/index.php?project=MERCURY&date=2015-07-15](https://cdash.hdfgroup.org/index.php?project=MERCURY&date=2015-07-15)
The dashboard is now also being updated to the latest version so things should also
be nicer on this side soon.

The new mercury website is based on [Jekyll](http://jekyllrb.com/), which makes it easier to add new pages and
code-related stuff than it was with the wordpress one.
Regarding _mailing-lists_, we will keep them hosted at ANL as it has been working well so far this way.

