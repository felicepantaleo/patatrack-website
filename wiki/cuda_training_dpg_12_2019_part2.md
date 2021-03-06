---
title: "CUDA Training for Tracker DPG - part 2, using CUDA in CMSSW"
author: "Andrea Bocci"
layout: wiki
resource: true
categories: wiki
---

## Set up

### Allocation of machines and GPUs

See [part 1](cuda_training_dpg_12_2019.md) for the allocation of a machine and GPU.


### Setting up CMSSW

We are going to use a special release of CMSSW to make use of the utilities
developed by the Patatrack group.  
You can find more information on the [Patatrack development](PatatrackDevelopment.md)
wiki page.

```bash
# on patatrack02 and felk40
export VO_CMS_SW_DIR=/data/cmssw

# on cmg-gpu1080
export VO_CMS_SW_DIR=/data/patatrack/cmssw

# common
export SCRAM_ARCH=slc7_amd64_gcc820
source $VO_CMS_SW_DIR/cmsset_default.sh

# create a working area
scram list CMSSW_11_0_0_pre13
cmsrel CMSSW_11_0_0_pre13_Patatrack
cd CMSSW_11_0_0_pre13_Patatrack/src

# load the environment
cmsenv

# set up a local git repository
git cms-init -x cms-patatrack
```

You should be in the `from-CMSSW_11_0_0_pre13_Patatrack` branch, and you should
be able to work as you would in a normal CMSSW development area.


### Material for the exercises

```bash
# checkout and build the CMSSW modules used in the tutorial
git merge cms-patatrack/Tutorial_December2019_part2
git cms-addpkg DataFormats/Math Patatrack/Tutorial
scram b -j 4 -k

# generate some random input data
cd Patatrack/Tutorial
cmsRun test/generateCylindricalVectors.py
```


## Simple CUDA modules in CMSSW

### Code organisation

Compilation with CUDA and `nvcc` follows different paths for different files.

A simplified distinction is that

  - **.cc** files are compiled by the host compiler (e.g. `gcc` or `clang`)
  - **.cu** files are compiled by the CUDA compiler driver (`nvcc`) which
      - compiles the host part using the host compiler
      - compiles the device part with an NVIDIA proprietary compiler, after
        preprocessing with the host compiler
      - links the host and device part, so that the host code can call the
        kernels on the device

Things become more complicated when we need to split the CUDA code across multiple files, with
[separate compilation and linking](https://devblogs.nvidia.com/separate-compilation-linking-cuda-device-code/) steps:
![compilation trajectory](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/graphics/cuda-compilation-from-cu-to-executable.png)

Things become even more complicated when dealing with shared libraries (device
code does not support them) and plugins (_here be dragons_).
SCRAM should support all use cases, but it will need some improvements in this
area, as we understand better the constraints.

The option that seems to be working so far is

  - CUDA library calls (e.g. `cudaMalloc()`) can be used anywhere¹
  - CUDA code (e.g. `__global__` and `__device__` functions) should only go in
    **.cu** files (and in **.h** files included by **.cu** files)
  - **.cu** files should only go in plugins, not in standard shared libraries

We are looking for alternatives, but so far having CUDA kernels in libraries
causes wanton chaos and destruction, so don't do it.

___
¹ as long as CUDA is available, which today means: on Intel/AMD and ARMv8
architectures, with CentOS 7 and CentOS 8, with GCC 7.x and 8.x; support for
the IBM Power architecture is going to be added, and GCC 9.x should be 
supported sometimes next year.  
**June 2020 Update**: `CMSSW_11_1_0_Patatrack` should use CUDA 11.0, which
includes support for GCC 9.x; it will also be built and made available for
the IBM Power architecture.


### `EDProducer`s and other framework plugins

A second limitation is that `nvcc` supports c++03, c++11 and c++14 - but not
c++17 yet. Since part of the CMS framework and ROOT are already using features
from c++17, we cannot `#include` their headers in **.h** and **.cu** files that
are going to be compiled by `nvcc`.

So, we cannot define a framework plugin (e.g. an `EDProducer`) in a ".cu" file;
instead we need to split it further, for example in:

  - `plugins/MyEDProducer.cc`: declaration and definition of the plugin;
  - `plugins/MyCUDAStuff.h`: declaration of any CUDA data structures, and
    plain-C++ wrappers that invoke the kernels;
  - `plugins/MyCUDAStuff.cu`: implementation of device functions, kernels,
    and of the plain-C++ wrappers that invoke the kernels.

If "MyCUDAStuff" is used only by "MyEDProducer", one can also use the files
`plugins/MyEDProducer.h` and `plugins/MyEDProducer.cu` forthe CUDA code.


**June 2020 Update**: CUDA 11.0 does support c++17 in CUDA code.
We are not taking advantage of it yet, but it should ship together with
`CMSSW_11_1_0_Patatrack` later on.

### Error checking

All CUDA library functions have an error code as their return value.
The safe approach is to wrap *all* calls tu CUDA library function in a wrapper
that checks the return value, and throws an exception if there is an error:
```c++
#include "HeterogeneousCore/CUDAUtilities/interface/cudaCheck.h"

...
  // allocate memory buffers on the GPU
  cudaCheck(cudaMalloc(&gpu_input, sizeof(Input) * input.size()));
  cudaCheck(cudaMalloc(&gpu_output, sizeof(Output) * input.size()));

  // copy the input data to the GPU
  cudaCheck(cudaMemcpy(gpu_input, input.data(), sizeof(Input) * input.size(), cudaMemcpyHostToDevice));
...
```

### Exercise B.1

The `Patatrack/Tutorial` package contains various plugins to

  - generate random 3D vectors in cylindrical coordinates (`GenerateCylindricalVectors`)
  - convert vectors from cylindrical to cartesian coordinates (`ConvertToCartesianVectors`)
  - dump vectors in cylindrical (`PrintCylindricalVectors`) or cartesian (`PrintCartesianVectors`)
    coordinates
  - compare two collections of vectors in cartesian coordinates (`CompareCartesianVectors`)
  
as well as some configuration files that can be use to run them. For example:
```bash
cd Patatrack/Tutorial

# generate 1200 events wih 10k random vectors
cmsRun test/generateCylindricalVectors.py

# print the cylindrical vectors in the first event
cmsRun test/printCylindricalVectors.py

# convert the vectors from cylindrical to cartesian in all events
cmsRun test/benchmarkCartesianVectors.py

# convert the vectors from cylindrical to cartesian, and print the cartesian vectors in the first event
cmsRun test/printCartesianVectors.py

# convert the vectors from cylindrical to cartesian using two different modules, and compare their results
cmsRun test/compareCartesianVectors.py
```

For the first exercise:

  - read the `ConvertToCartesianVectors` `EDProducer`
  - using the skeleton provided in `Patatrack/Tutorial/plugins/ConvertToCartesianVectorsCUDA.cc`
    and `Patatrack/Tutorial/plugins/cudavectors.h`, write a `ConvertToCartesianVectorsCUDA`
    `EDProducer` that does the same conversion on the GPU
  - use `cmsRun test/compareCartesianVectors.py` to compare the results of the
    conversion of the CPU and on the GPU; what do you expect ? 
  - vary the precision of the comparison; what do you see ?

#### Bonus points

The data structures in `Patatrack/Tutorial/plugins/cudavectors.h` follow the
**Array of Structures** (**AOS**) approach; on GPUs it may be beneficial to use
a [**Structure of Arrays**](https://en.wikipedia.org/wiki/AoS_and_SoA)
(**SOA**) approach.

How would you change the `ConvertToCartesianVectorsCUDA` module to use a SOA
approach, without changing the format of the DataFormats used by CMSSW ?

How much of the "reconstruction" time is spent in the conversion between the
AOS (for CMSSW) and SOA (for processing on GPU) structures ?


## CUDA framework in CMSSW

During the Patatrack development some common patterns emerged:

  - all CUDA functions should be checked for errors, and the preferred way to
    report errors in CMSSW is to throw an exception (see `cudaCheck()` above);
  - reusing host and GPU memory is more efficient than calling `cudaMalloc()`/`cudaFree()`
    for every event;
  - GPU operations should be **asynchronous** with respect to CPU operations,
    to avoid blocking the CPU and allow the framework to schedule other modules
    on the CPU while the GPU is busy;
  - consecutive GPU operations should be scheduled in the same "CUDA stream"
    and on the same device, with minimal synchronisation with the CPU tasks;
  - it is highly desirable to have a CPU and GPU version of the same module
    available in the configuration, and let `cmsRun` choose at run time which
    one to use.

These will be the topic of the next exercises, together with the tools and
utilities that have been (and are being) developed to address them.

For more information, details and examples, please read
[the documentation](https://github.com/cms-patatrack/cmssw/blob/master/HeterogeneousCore/CUDACore/README.md)
prepared by Matti.


## CUDA memory management

### CUDA caching allocators

See [Memory management](https://github.com/cms-patatrack/cmssw/blob/master/HeterogeneousCore/CUDACore/README.md#memory-allocation).

### Exercise B.2

Following the documentation about [Memory management](https://github.com/cms-patatrack/cmssw/blob/master/HeterogeneousCore/CUDACore/README.md#memory-allocation),
update the `ConvertToCartesianVectorsCUDA` to allocate

  - automatically reused GPU memory for GPU buffers, using `cms::cuda::make_device_unique`
  - [Write-Combining Memory](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#write-combining-memory)
    for CPU-to-GPU copy buffers, using `cms::cuda::make_host_noncached_unique`

For the moment, use the [CUDA default stream](https://docs.nvidia.com/cuda/cuda-runtime-api/stream-sync-behavior.html)
(`0` or `NULL` or `nullptr` or `cudaDefaultStream`) for the `stream` parameter
of the memory allocations.

## Asynchronous execution

### CMSSW ExternalWork

CMSSW provides a generic [`ExternalWork`](https://twiki.cern.ch/twiki/bin/view/CMSPublic/FWMultithreadedFrameworkStreamModuleInterface#edm_ExternalWork)
mechanism to run part of an `EDProducer` asynchronously with respect of the
framework.

An `EDProducer` that inherits from `edm::ExternalWork` splits the work usually
run within `produce()` in three parts:

  - a new method, `acquire()`, is called to setup and start an asynchronous
    operation;
  - a callback mechanism is used to signal the framework when the operation is
    complete;
  - the `produce()` method is called to finalise the operation and "put" the
    results in the `Event`.

A simple `EDProducer` can use [this mechanism](https://github.com/cms-patatrack/cmssw/blob/master/HeterogeneousCore/CUDACore/README.md#isolated-producer-no-cuda-input-nor-output)
to efficiently offload computations to a GPU, while letting the framework
schedule some other work on the CPU.

### Multiple GPUs and CUDA streams

On a single machine there can bu multiple GPUs available. When offloading
consecutive algorithms (e.g. the unpacker followed by the local reconstruction)
we want to run them on the same GPU, to avoid transferring the intermediate
data across different GPUs. We also want to schedule them in the same "CUDA
stream", to guarantee that the second algorithm runs only after the first one
has completed, without an explicit synchronisation on the CPU side.

To set the GPU device and CUDA stream one should use a "cms::cuda::ScopedContext",
using the `cms::cuda::ScopedContextAcquire` and `cms::cuda::ScopedContextProduce`
classes in the `acquire()` and `produce()` methods, respectively.


### Exercise B.3

Following the documentation for an [isolated producer](https://github.com/cms-patatrack/cmssw/blob/master/HeterogeneousCore/CUDACore/README.md#isolated-producer-no-cuda-input-nor-output),
update the `ConvertToCartesianVectorsCUDA` to

  - set the CUDA device and obtain a CUDA stream
  - perform the memory copies and conversion asynchronously


## CPU vs GPU modules

### SwitchProducer mechanism

See [the documenation](https://github.com/cms-patatrack/cmssw/blob/master/HeterogeneousCore/CUDACore/README.md#automatic-switching-between-cpu-and-gpu-modules)
of the `SwitchProducer` mechanism.

### Exercise B.4

Write a python configuration that can automatically choose to run `ConvertToCartesianVectorsCUDA`
if there is a GPU available, and fall back to `ConvertToCartesianVectors` otherwise.

Test it by setting `CUDA_VISIBLE_DEVICES` to an empty value, e.g.
```bash
# this should run without any GPUs
CUDA_VISIBLE_DEVICES= cmsRun ...
```

## Solutions to the exercises

Possible solutions for the four exercises are available in the same repository:

  - [solution](https://github.com/cms-patatrack/cmssw/commits/Tutorial_December2019_part2-solution1) to the first exercise;
  - [solution](https://github.com/cms-patatrack/cmssw/commits/Tutorial_December2019_part2-solution2) to the second exercise;
  - [solution](https://github.com/cms-patatrack/cmssw/commits/Tutorial_December2019_part2-solution3) to the third exercise;
  - [solution](https://github.com/cms-patatrack/cmssw/commits/Tutorial_December2019_part2-solution4) to the fourth exercise.

**June 2020 Update**: The names and namespaces of some of the CUDA utilities
used in CMSSW changed during the the CMSSW 11.1.x release cycle. The solutions
may not reflect those changes.
