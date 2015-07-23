---
layout: page
title: Documentation
permalink: /documentation/
---

## Introduction

## Overview

<figure>
  <img src="/assets/overview.svg" alt="Caption to image" width="500px">
  <figcaption>
    Architecture overview.
  </figcaption>
</figure>

Each side uses an RPC *processor* to serialize and deserialize parameters sent
through the interface. Calling functions with relatively small arguments results
in using the short messaging mechanism exposed by the network abstraction layer,
whereas functions containing large data arguments additionally use the remote
memory access (RMA) mechanism.


## Network Abstraction Layer

### Interface

### Available Plugins

* BMI
* MPI
* CCI

### Adding a new plugin

## Mercury RPC Layer

{% highlight C linenos %}
MERCURY_GEN_PROC( open_in_t, ((hg_const_string_t)(path)) ((hg_int32_t)(flags)) ((hg_uint32_t)(mode)) )
MERCURY_GEN_PROC( open_out_t, ((hg_int32_t)(ret)) )
{% endhighlight %}

## Mercury Bulk Layer

## Conclusion

## See also
<ul>
  {% for post in site.categories.documentation  reversed %}
    <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
  <li><a
href="ftp://ftp.mcs.anl.gov/pub/mercury/documents/doxygen_doc/index.html">Doxygen</a></li>
</ul>

