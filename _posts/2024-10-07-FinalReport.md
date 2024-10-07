---
layout: post
title:  "Final GSoC Report"
date:   2024-10-07 04:00 +0530
description: My final GSoC'24 Report
---

<p class="intro">In this post, I'll be discussing my GSoC'24 project, the goals set, work done, and future scope</p>

# About OpenAstronomy
<p align="center" width="100%">
  <img src="{{ site.baseurl }}/assets/img/logoOA_svg.png" alt="OpenAstronomy Logo" style="margin-bottom: 0; margin-top: 24px"> 
</p>
OpenAstronomy is a collaboration between open source astronomy and astrophysics projects to share resources, ideas, and to improve code.

OpenAstronomy consists of many different projects: astropy, sunpy, stingray, radis

and of course GNU Astronomy Utilities (Gnuastro)

# About Gnuastro
<p align="center" width="100%">
  <img src="{{ site.baseurl }}/assets/img/gnu-logo.png" alt="Gnuastro Logo" style="margin-bottom: 0; margin-top: 24px"> 
</p>

Gnuastro is an official GNU package that consists of many CLI programs as well as library functions for manipulation and analysis of astronomical data.
Something important to note about Gnuastro is that it is written entirely in C99 and shell script. 
The Gnuastro team meets every Tuesday to exchange notes and review progress

# Goals of the project

Going into this project, there were two main goals decided
- Setting up the low level wrapper infrastructure for using OpenCL within Gnuastro, putting minimal requirements on the developers/users to know OpenCL.
- Parallelizing Gnuastro subroutines using the aforementioned wrappers to offload compute-heavy tasks to the GPU.

The first goal deals with how users of the library would interact with the OpenCL modules. Ideally, you would want the users to have no knowledge about OpenCL and only interact with it through GNUAstro.

The second goal deals with analyzing parts of the Gnuastro library, identifying easy to parallelize sections and writing optimised OpenCL Kernels for them, leveraging the wrapper infrastructure for execution.

The majority of my work lives [here](https://github.com/DeadSpheroid/gnuastro/tree/final)

# Pre GSoC

Prior to GSoC, my experience consisted mostly of Deep Learning and Natural Language Processing. My knowledge of GPU Processing was limited and naive.

During the proposal drafting period, candidates were asked to submit a task, which was implementing simple image convolution on both CPU and GPU using OpenCL.

# During GSoC

Work on GSoC kicked off around May 1st with the first objective being

## Build System Integration

Now, Gnuastro being a GNU project uses the GNU Build System, GNU Autotools(Autoconf, Automake, Libtool).

This was a completely new build system for me to work with and I had to get the library to include OpenCL and link against the OpenCL library at compile time.

Thanks to some helpful pointers from Mohammad, I was able to grasp the working pretty quickly and was able to set up Gnuastro to include and build with OpenCL if it was detected on the system.

For more information, you can read [here](https://deadspheroid.github.io/my-blog/post/GettingStarted/)

## Wrapper Infrastructure

The next goal was to create wrappers around the OpenCL C API, for various operations(data transfer, launching kernels, querying devices).

This was done in the form of a new Gnuastro module called `cl-utils.c` which contained

#### Initialisation
Functions dealing with initialising OpenCL and creating, destroying OpenCL objects

#### Data Transfer Functions
Functions dealing with data transfer to and from the GPU.
This is one of the biggest overheads in GPU Programming. Additionally, OpenCL presented another challenge in the form of transferring structs to the GPU, which was problematic as one of Gnuastro's most important data structures `gal_data_t` could not be directly transferred.

So, this module provides two ways of transferring data to the GPU
1. OpenCL Buffers
and
2. OpenCL Shared Virtual Memory

Buffers are intended to be used for simple data structures, while OpenCL SVM is better with more complex data structures involving internal pointers.

The interface looks something like this
{%- highlight C -%}
void
gal_cl_write_to_device (cl_mem *buffer, void *mapped_ptr,
                             cl_command_queue command_queue);

void *
gal_cl_read_to_host (cl_mem buffer, size_t size,
                           cl_command_queue command_queue);
.
.
.
gal_data_t *
gal_cl_alloc_svm (size_t size_of_array, size_t size_of_dsize,
                  cl_context context, cl_command_queue command_queue);

void
gal_cl_map_svm_to_cpu (cl_context context, cl_command_queue command_queue, 
                void *svm_ptr, size_t size);
{%- endhighlight -%}

The main advantage of using OpenCL SVM, is it allows the CPU host process and the GPU process to share the same virtual address space, which greatly simplifies the user's experience. Hence going forward, this was chose as the primary method of achieving data transfer

For more information on the two, and a comparison see [here](https://deadspheroid.github.io/my-blog/post/ExploringFurther/)

#### Executing Kernels
Now, the main code running on the GPU is the OpenCL Kernel, usually defined in a .cl file and compiled at runtime.

The idea when making this module, was to keep the interface as similar to the original pthreads `gal_threads_spin_off()` interface that Gnuastro already had. So i created a `gal_cl_threads_spinoff()` function, taking information like the kernel filepath, number of inputs, list of inputs, number of threads executed and more.

{%- highlight C -%}
typedef struct clprm
{
  char               *kernel_path; /* Path to kernel.cl file */
  char               *kernel_name; /* Name of __kernel function */
  char             *compiler_opts; /* Additional compiler options */
  cl_device_id          device_id; /* Device to be targeted */
  cl_context              context; /* Context of OpenCL in use */
  int             num_kernel_args; /* Number of total kernel arguments */
  int                num_svm_args; /* Number of SVM args*/
  void              **kernel_args; /* Array of pointers to kernel args */
  size_t       *kernel_args_sizes; /* Sizes of non SVM args */
  int          num_extra_svm_args; /* Number of implicit SVM args */
  void           **extra_svm_args; /* Array of pointers to these args */
  int                    work_dim; /* Work dimension of job - 1,2,3 */
  size_t        *global_work_size; /* Array of global sizes of size work_dim */
  size_t         *local_work_size; /* Array of local sizes of size work_dim */
} clprm;
{%- endhighlight -%}

These wrappers were not developed all at once, but rather in conjunction with the convolution implementation, writing wrappers as and when I needed them.

#### Using the OpenCL modules in your program
Finally, when a user wants to use Gnuastro's OpenCL capabilities within their own programs, the flow followed would look like:
- Intialize OpenCL
- Transfer Input to Device
- Write an OpenCL Kernel
- Spinoff Threads
- Copy Output back to Host

Lets take an example where we need to simply add two fits images.

##### Initialize OpenCl
{%- highlight C -%}
  cl_context context;
  cl_platform_id platform_id;
  cl_device_id device_id;

  gal_cl_init (CL_DEVICE_TYPE_GPU, &context, &platform_id, &device_id);
  cl_command_queue command_queue
      = gal_cl_create_command_queue (context, device_id);
{%- endhighlight -%}

This initializes and OpenCL context, among other objects for use in future function calls.

##### Transfer Input to Device
Make use of `gal_cl_copy_data_to_gpu()` to transfer the loaded fits files to the GPU, passing the previously initialized context and command queue. Make sure the command queue finishes before proceeding ahead through `gal_cl_finish_queue()`

{%- highlight C -%}
  gal_data_t *input_image1_gpu
      = gal_cl_copy_data_to_gpu (context, command_queue, input_image1);
  gal_data_t *input_image2_gpu
      = gal_cl_copy_data_to_gpu (context, command_queue, input_image2);
  gal_data_t *output_image_gpu
      = gal_cl_copy_data_to_gpu (context, command_queue, output_image);

  gal_cl_finish_queue (command_queue);
{%- endhighlight -%}

##### Write an OpenCL Kernel
First, any custom structs you use, must be defined in the kernel, here we define gal_data_t.

Then, you create the "per thread" function that will be executed, prefixed by `__kernel` and always returning `void`.

In the arguments, mention the pointers to the inputs/outputs, as well as a `__global` identifier, since your input is acessible by all threads.

Make use of OpenCl's `get_global_id(0)` to get the thread id along the 0th dimension.

Perform the core operation of your program.

Putting it all together, it looks like this:

{%- highlight C -%}
typedef struct  __attribute__((aligned(4))) gal_data_t
{
  /* Basic information on array of data. */
  void *restrict array; /* Array keeping data elements.               */
  uchar type;         /* Type of data (see 'gnuastro/type.h').      */
  size_t ndim;          /* Number of dimensions in the array.         */
  size_t *dsize;        /* Size of array along each dimension.        */
  size_t size;          /* Total number of data-elements.             */
.
.
.
  /* Pointers to other data structures. */
  struct gal_data_t *next;  /* To use it as a linked list if necessary.   */
  struct gal_data_t *block; /* 'gal_data_t' of hosting block, see above.  */
} gal_data_t;

__kernel void
add(__global gal_data_t *input_image1,
    __global gal_data_t *input_image2,
    __global gal_data_t *output_image)
{
    int id = get_global_id(0);

    float *input_array1 = (float *)input_image1->array;
    float *input_array2 = (float *)input_image2->array;
    float *output_array = (float *)output_image->array;

    output_array[id] = input_array1[id] + input_array2[id];
    return;
}
{%- endhighlight -%}


##### Spin Off Threads
Make use of the `clprm` struct defined in `gnuastro/cl-utils.h` to group all the relevant parameters.

`Kernel Path` is the filepath to the OpenCL Kernel you just wrote.

`Kernel Name` is the name of the function you defined with `__kernel` earlier.

`Compiler Options` is a string of any special compiler options like macros/debug options you wish to use for the kernel.

`Device Id & Context` are the objects intialized in the first step.

`Number of Kernel Arguments` is the number of kernel arguments.

`Number of SVM Arguments` is the number of arguments that use SVM(all the gal_data_t's)

`Kernel Arguments` is an array to void pointers of kernel arguments.

`Number of Extra SVM Arguments` is the number of arguments that are implicitly referenced with a struct. For example, `input_image1_gpu` is directly referenced as a kernel argument, but the `input_image1_gpu->array` is implicitly referenced.

`Extra SVM Arguments` is an array of void pointers to the aforementioned special arguments.

`Work Dim` is the number of dimensions of the threads (1, 2, 3)
For example, an array would have 1 dimension(0,1,2,...34,35,36) x
an image would have 2 dimensions(0:0, 0:1, 1:0, 1:1,....) x:y
a volume would have 3 dimensions x:y:z

`Global Work Size` is the total number of threads spun off

`Local Work Size` is the number of threads in a block on one GPU core. Leaving it blank lets the device choose this number.

{%- highlight C -%}
  clprm *sprm = (clprm *)malloc (sizeof (clprm));

  void *kernel_args[] = { (void *)input_image1_gpu, (void *)input_image2_gpu,
                          (void *)output_image_gpu };

  void *svm_ptrs[]
      = { (void *)input_image1_gpu->array, (void *)input_image2_gpu->array,
          (void *)output_image_gpu->array };

  size_t numactions = input_image1->size;

  sprm->kernel_path = "./lib/kernels/add.cl";
  sprm->kernel_name = "add";
  sprm->compiler_opts = "";
  sprm->device_id = device_id;
  sprm->context = context;
  sprm->num_kernel_args = 3;
  sprm->num_svm_args = 3;
  sprm->kernel_args = kernel_args;
  sprm->num_extra_svm_args = 3;
  sprm->extra_svm_args = svm_ptrs;
  sprm->work_dim = 1;
  sprm->global_work_size = &numactions;
  sprm->local_work_size = NULL;

  gal_cl_finish_queue (command_queue);
{%- endhighlight -%}
##### Copy Output back to Host

{%- highlight C -%}
gal_cl_read_data_to_cpu(context, command_queue, output_image_gpu);
{%- endhighlight -%}

The complete program can be accessed [here](https://github.com/DeadSpheroid/gnuastro/blob/final/cl-example-add-fits.c)

## Parallelized Subroutines
To achieve the goal of GPU acceleration, first we needed to identify parts of the library that could be parallelized.
Its important to note that not everything can be parallelized, and just because something can be, doesnt mean it should be.

The most obvious candidate for this of course was 2D Image Convolution, already implemented in Gnuastro in the `astconvolve` module.


#### Convolution
I got to work creating a new module `cl-convolve.c` containing the new implementation of convolution `gal_convolve_cl()`

The exact code can be viewed [here](https://github.com/DeadSpheroid/gnuastro/blob/final/lib/cl-convolve.c), but in short:
1. Transfer input, kernel and output images to GPU
2. Spin off a thread for each pixel in the input, convolving that particular pixel.
3. Copy the output image back to CPU

After integrating the OpenCL wrappers with astconvolve, the new interface looks like this

<p align="center" width="100%">
  <img src="{{ site.baseurl }}/assets/img/cl_opt.png" alt="--cl option on CLI" style="margin-bottom: 0; margin-top: 24px"> 
</p>

Note the new `--cl=INT` option, which is a CLI argument allowing users to choose the implementation they want and the device they want.

- --cl=0 corresponds to existing pthread implementation on CPU
- --cl=1 corresponds to OpenCL implementation on GPU
- --cl=2 corresponds to OpenCL implementaion on CPU

Leaving this parameter blank, defaults to the pthread implementation, making GPU acceleration opt-in, owing to the minute floating point differences that can occur. 

#### Optimised Convolution
The power of GPUs comes not from the many threads that are launched, but rather from the many optimisations possible, from organising threads into blocks, to special kinds of memory. I decided to try optimising Convolution based on Labeeb's suggestion of using shared memory.

However most of the optimisation out there are for CUDA, not OpenCL, but the principles in question were the same. Thanks to [this article](https://www.evl.uic.edu/sjames/cs525/final.html), I was able to implement an optimised 2Dconvolution kernel in OpenCL.

The results of the optimisation were surprisingly positive:
For a 5000 x 5000 image, times recorded for the convolution operation(excluding data reading/writing in seconds were)

| |Pthread|OpenCL-CPU|OpenCL-GPU|
|:-------|:--------:|:-------:|:------:|
|w/out optimisations|1.014374|0.918015|**0.025869**|
|w/ optimisations|1.053622|0.326756|**0.004184**|

Thats a speedup of **~6.2 times** over the non optimised GPU run, and **~242 times** over the existing pthread implementation in Gnuastro!

Further optimisations are possible using special native functions like MUL24 and constant memory. But the details of those and how these optimisation work is a topic for a separate post.

#### Same code on CPU and GPU
The initial idea was to have the exact same code running on both the CPU(via pthread) and the GPU(via OpenCL). This is possible because OpenCL Kernels are based on OpenCL C which is a variant(kind of a subset) of C99.

This is because Gnuastro is a "minimal dependencies" package and having two separate implementations would greatly overcomplicate the codebase.

After a discussion, it was decided that the best path forward for OpenCL in Gnuastro would be to completely replace the existing pthread implementation.

In essence, the existing "convoluted" convolution implementation would be replaced with my new one, allowing the same code to be ran in 3 different ways:
- With OpenCL on the GPU
- With OpenCL on the CPU
- With GCC+Pthreads on the CPU

It was challenging, owing to the different styles in which we write code for a CPU device versus a GPU device. But I managed to get a partially working version using some C macros here and there to do so. It still fails some Gnuastro tests, which is yet to be resolved.

# Post GSoC
Now, that the wrapper infrastructure is set up and convolution is implemented, whats left is to test the implementation against real life scenarios to make sure it lives up to the expectations of the Gnuastro users.
We also need to come up with a consistent way to execute the same kernel on both OpenCL and GCC, as mentioned earlier.

Additionally, now that work on one module is complete, it opens the scope for more modules to be implemented on the GPU (like statistics, interpolation and more).

# Acknowledgements
GSoC has been an incredible learning experience for me both from a technical view and from a personal view.

On the technical side, I learned a lot about one of my favourite domains in Low Level Programming, GPU Programming and my understanding of how to write libraries that are easy to use, performant and above and all else, FOSS, improved tremendously. It's one thing when you learn and write code for your own personal projects, but it's a completely different experience contributing to something like Gnuastro.

On the personal side, the weekly meetings with the Gnuastro team were always extremely engaging and i got to learn a lot from the team, Giacomo's work on Astrometry, Alvaro's work on Deconvolution and Ronald too. Their feedback on stuff like debugging using valgrind/gdb and references to other projects using OpenCL, alongside other topics has been invaluable.

Above and all else im thankful to my mentor [Mohammad Akhlagi](https://akhlaghi.org/). Its been amazing getting to interact with someone so experienced and I learned a lot from him, ranging from Astronomy to Hacking the GNU C Library. He was always patient and understanding of my other responsibilities and allowed me to work at my own pace. I'm grateful to him for the opportunity to be a part of the Gnuastro community.

Finally, I can't explain how indebited I am to my mentor [Labeeb Asari](https://www.linkedin.com/in/labib-asari/?originalSubdomain=in). His knowledge about GPU Programming has been vital to my work on this project and I'm grateful to him for introducing me to the Gnuastro team. From [Project X](https://github.com/ProjectX-VJTI), to GSoC to college in general, he has been a big help in everything I've done and im glad to have him as a mentor and friend.

A huge thank you to the Google Summer of Code Team for undertaking this wonderful initiative and I hope they continue this program in future years.