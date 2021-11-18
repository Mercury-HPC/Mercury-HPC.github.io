---
# Only edit this file to modify description.
# Edit theme's home layout instead if you want to make layout changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: home
---

Mercury is a Remote Procedure Call (RPC) framework specifically designed for
use in High-Performance Computing (HPC) systems with high-performance fabrics.
Its network implementation is abstracted to make efficient use of native transports
and allow easy porting to a variety of systems (including future systems).
Mercury supports asynchronous transfer of parameters and execution requests,
and has dedicated support for large data arguments that are transferred using
Remote Memory Access (RMA). Its interface is generic and allows any function call to be serialized.
Since code generation is done using the C preprocessor, no external tool is required.

See [here][doc-link] for a quick introduction on how to use Mercury in your projects!

<table style="border: 0;"><colgroup><col style="width: 80%" /><col style="width: 20%" /></colgroup><tbody><tr>
<td style="color:#333;font-weight:light;font-style:oblique;border: 0;">
Mercury is partially supported by DOE Office of Science Advanced
Scientific Computing Research (ASCR) research and by NSF Directorate
for Computer &amp; Information Science &amp; Engineering (CISE)
Division of Computing and Communication Foundations (CCF) core
program funding. Mercury is a core component of the
<a href="//www.mcs.anl.gov/research/projects/mochi/">Mochi</a>
ecosystem of microservices, an R&D 100 winning project.
</td>
<td style="border: 0;">
<figure><img height="150" src="/assets/RD100_2021_Winner_Logo.svg" alt="2021 R&D 100" /></figure>
</td>
</tr></tbody></table>

[doc-link]: {{ site.baseurl }}/documentation
[mochi-link]: //www.mcs.anl.gov/research/projects/mochi/
