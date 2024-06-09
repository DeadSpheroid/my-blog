---
layout: post
title:  "OpenCL, meet the Gnuastro Build System"
date:   2024-06-09 06:15 +0530
description: Delving into OpenCL, and why it was chosen for Gnuastro, along with its integration into the build system.
---

<p class="intro">In this post, I hope to summarize the work done so far towards my GSoC project for integrating OpenCL with the Gnuastro library and my relatively limited understanding of OpenCL.</p>

# What is OpenCL?
<p align="center" width="100%">
  <img src="{{ site.baseurl }}/assets/img/opencl-logo.png" alt="OpenCL Logo" style="margin-bottom: 0; margin-top: 24px"> 
</p>
**[Open Computing Language](https://www.khronos.org/opencl/)** is a framework for writing programs that execute across **heterogenous** platforms. In simpler terms, OpenCL provides a standard interface for programmers to execute the **same** code across **multiple** devices, be it a CPU or a GPU or **any** other accelerator.

It comprises of the OpenCL standard which is maintained by [Khronos](https://www.khronos.org/opencl/), and implemented by the various hardware **manufacturers** and by the **open source community** across a wide variety of devices.

Most modern devices all support OpenCL in some format or the other. **Intel/Nvidia** for example provide their own **propietary** implementations. On the other hand, **POCL** an **open source** project provides implementations for those that dont have actively maintained propietary ones, like **AMD**.

# Why OpenCL?

<p align="center" width="100%">
  <img src="{{ site.baseurl }}/assets/img/cl-cuda.jpeg" alt="OpenCL versus CUDA" style="margin-bottom: 0; margin-top: 24px"> 
</p>

Unlike certain **propietary** frameworks *cough* [CUDA](https://developer.nvidia.com/about-cuda) *cough*, OpenCL is not constrained to any particular **manufacturer**. You can target **any GPU/CPU** as long as you get the OpenCL implementation for that device. This is made easy thanks to projects like [POCL](https://portablecl.org/).

The performance of **CUDA versus OpenCL** is heavily debated and leans towards CUDA for Nvidia hardware, but the difference depends on the use case and isn't too much of a concern as compared to the way they are used.

The **downside** of OpenCL is the **smaller** community and the lack of many **modern features** that CUDA brings.

<blockquote>
The inner workings of OpenCL, how I managed to set it up, how the OpenCL C API works is another long story and is deserving of its own post.
</blockquote>


# Kickoff with Gnuastro

The first goal for the project was to figure out a way to **integrate** OpenCL with the Gnuastro build system.

Gnuastro like many other free software uses the **GNU Build System** also called [GNU Autotools](https://www.gnu.org/software/automake/faq/autotools-faq.html)

<p align="center" width="100%">
  <img src="{{ site.baseurl }}/assets/img/gnu-logo.png" alt="GNU Autotools" style="margin-bottom: 0; margin-top: 24px"> 
</p>

The **three** major components of Autotools are:

### Autoconf
At the heart of Autotools, we have [Autoconf](https://www.gnu.org/software/autoconf/), which generates a **single** `configure` **script** from a `configure.ac` file.

This `configure` script scans the **environment** for various files and **libraries**, specific versions of them, the **hardware** being used, and more. Then, it **configures** the build of the project in certain ways enabling/disabling certain parts depending on what was found and what wasnt.

In this way, the **portability** of any project can be ensured by simply distributing the **configure** script, along with the `Makefile.in`s.

### Automake
[Automake](https://www.gnu.org/software/automake/) makes use of information found by `configure` and **generates** the `Makefile`s necessary to **build** the project.

To be more precise, it **parses** `Makefile.am`s into `Makefile.in`s which are in turn **parsed** by `configure` to produce the final `Makefile`s. Automake also performs **automatic dependency tracking**, so that recompilling isn't done unless **required**.

All you have to do, is specify the **name** and each of the **sources** involved in the library/binary, and Automake does the rest.

### Libtool
[Libtool](https://www.gnu.org/software/libtool/) is responsible for abstracting the **library** creation process, since different platforms handle static/dynamic libraries **differently**.

<blockquote>
I mainly worked with Automake and Autoconf during integration and didn't really touch Libtool.
</blockquote>

# Stepping into Integration

### Inside `configure.ac`
Without getting into detail, when checking for the **presence** of OpenCL, it suffices to check for `libOpenCL.so` and the `CL.h` header file.

That is, Gnuastro should be able to **include** the OpenCL header file to use its C API, and then later **link** against the OpenCL library.

Luckily for us, [Gnulib](https://www.gnu.org/software/gnulib/) provides a simple `AC_LIB_HAVE_LINKFLAGS` [macro](https://www.gnu.org/software/gnulib/manual/html_node/Searching-for-Libraries.html) which takes as input, a library **name** and a **test code** and tries to find the **library** and **compile/link** the test code.

Upon successfully executing, it **sets certain variables**, so we can modify further building on the basis of **finding OpenCL**.
```bash
AC_LIB_HAVE_LINKFLAGS([OpenCL], [], [#include <CL/cl.h>])
AS_IF([test "x$LIBOPENCL" = x],
      [
        if successfull ...
      ],
      [
        LIBS="$LIBOPENCL $LIBS"
        has_ocl=1;
        if unsuccessfull ...
      ])
AM_CONDITIONAL([COND_HASOPENCL], [test "x$has_ocl" = "x1"])
```
After making these modifications to `configure.ac`, we can now **test** whether OpenCL was found inside the various `Makefile.am`s and accordingly change the **build**.

### Inside `Makefile.am`
Now, we can use the **variable** we set previously in `configure.ac` and either include or exclude the OpenCL modules from being compiled and included in the **library**.

```bash
if COND_HASOPENCL
  $(info "Found OpenCL")
  MAYBE_CL = cl_utils.c
  MAYBE_CL_H = $(headersdir)/cl_utils.h
  MAYBE_CONVOLVE_CL = cl_convolve.c
else
  $(info "What is Opencl?")
endif
```
```bash
libgnuastro_la_SOURCES = \
  $(MAYBE_NUMPY_C) \
  $(MAYBE_WCSDISTORTION) \
  $(MAYBE_CL) \
  $(MAYBE_CONVOLVE_CL) \
  arithmetic.c \
  arithmetic-and.c \
  ...
```

Additionally, we need to **save** this variable in Gnuastro's `config.h` file for later use to **prevent** other modules from mistakenly including the OpenCL ones incase OpenCL was **not compiled**.

# checking for build system... yes
Now when someone builds Gnuastro, if OpenCL is **present** on their system, then the OpenCL relevant files are **compiled and included in the library**.

On the other hand, if OpenCL is **absent**, then the library is **built as normal**, as if OpenCL never existed.

Finally, we can get started with the **actual** OpenCl part and we'll have a look at Image Convolution(astconvolve) in the next post...