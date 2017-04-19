# Overview

## What is included

This repository includes the following features for use with Vivado HLS and/or SDAccel projects:

* `DataPack` class to ease SIMD parallelization in HLS kernels (`include/hlslib/DataPack.h`) 
* Utility classes for SDAccel host side integration, located at `include/hlslib/SDAccel.h`, including support for specifying FPGA memory banks
* Thread-safe stream implementation, with blocking semantics to mimic hardware behavior (`include/hlslib/Stream.h`)
* Macros to easily simulate dataflow functions using multithreading when compiled as C++, while being compatible with high-level synthesis when building for hardware (`include/hlslib/Simulation.h`)
* Various compile-time functions commonly used when designing hardware, such as log2, in `include/hlslib/Utility.h`
Some of these headers are interdependent, while others can be included standalone. Refer to the source code for details.
* CMake module files that locate SDAccel, Vivado, Vivado HLS and Vivado Lab on the host machine, setting relevant variables for build integration with CMake
* A template tcl-file that can be used with CMake or another templating engine to produce a high-level synthesis script
* `CMakeLists.txt` that builds a number of tests to verify hlslib functionality, doubling as a reference for how to integrate HLS projects with CMake using the provided module files 
* An example of how to use the Simulation and Stream headers, at `kernels/MultiStageAdd.cpp`, both as a host-only simulation (`test/TestMultiStageAdd.cpp`), and as a hardware kernel (`host/RunMultiStageAdd.cpp`) 

## Installation

The source code provided in this repository can be used on a file-by-file basis, and as such does not need to be compiled/installed. Rather, header files (located in `include/hlslib`) and CMake module files (located in `cmake`) can simply be copied to the source directory of an HLS project.

When an hlslib header is included, compilation must allow C++11 features, and the macro `HLSLIB_SYNTHESIS` must be set whenever high-level synthesis is run, using `-cflags "-std=c++11 -DHLSLIB_SYNTHESIS"` and `--xp prop:kernel.<entry function>.kernel_flags="-std=c++11 -DHLSLIB_SYNTHESIS"` for Vivado HLS and SDAccel, respectively. See the included `CMakeLists.txt` for reference.

## Compatibility

The code within has been tested with SDx 2016.3 and Vivado HLS 2016.4 for the HLS code, and GCC 5.3.0 for host/simulation code. There is no plan to accommodate older versions of any of these tools, although there is no principal reason why older versions should not work. 

# Useful information

### Running SDAccel

It is beneficial to make a custom script for setting up the SDAccel environment, as the default one generated by Xilinx sets `LD_LIBRARY_PATH`, which generally is bad practice, and will conflict with binaries compiled under a different environment.

The following is sufficient as of SDx 2016.3:

```shell
#!/bin/sh
export XILINX_OPENCL=<path to DSA folder>/xbinst/pkg/pcie
export XCL_PLATFORM=<DSA string (e.g. "xilinx_adm-pcie-7v3_1ddr_3_0")>
unset XILINX_SDACCEL
unset XCL_EMULATION_MODE
```

### Bundled libc++

When linking against the Xilinx OpenCL libraries, the linker must add the runtime library folder to the library search path. This folder contains an ancient version of libc++.so, which will break compilation when using a newer compiler.
To avoid this, backup or delete the library file located at:
```
<SDx or SDAccel folder>/runtime/lib/<architecture>/libstdc++.so
```

### relax\_ii\_for_timing

SDAccel sets the option `relax_ii_for_timing`, and a conservative clock uncertainty of 27% of the target timing. This means that it will silently increase the initiation interval if this more conservative constraint is not met, resulting a slowdown of 2x to the resulting performance. Check the HLS log file to see if your design was throttled, located at:
```
_xocc_<source file>_<kernel file>.dir/impl/build/system/<kernel file>/bitstream/<kernel file>_ipi/ipiimpl/ipiimpl.runs/impl_1/<"vivado_hls" or "runme">.log
```

### Power report crash

While building a project, Vivado generates a number of GUI-reports, even when running `xocc` on the command line. These reports have an internal limit of 64 MB per _section_. For very large projects these sections can exceed 64 MB, causing Vivado to crash with an error like "Unable to write <report name>.rpx as it exceeds maximum size of 64 MB".
So far there does not seem to be a solution to this error.

### Stream pragma error

As of SDx 2016.3, there seems to be an error when trying to set `#pragma HLS STREAM` for stream variables that are redirected through `typedef` or `using`. So far the only (and rather tedious) workaround we have found is to use `#ifdef`s to include two versions of the stream definitions, such as below:

```C++
#ifdef HLSLIB_SYNTHESIS
hls::stream<int> pipe;
#pragma HLS STREAM variable=pipe depth=kPipeDepth
#else
hlslib::Stream<int> pipe;
#endif
```
