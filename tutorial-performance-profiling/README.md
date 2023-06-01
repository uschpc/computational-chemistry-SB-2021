# Performance Profiling with Intel VTune & NVIDIA Nvprof
USC Center for Advanced Research Computing HPC

## Pre-requisite (Option 1) - Installing the profiler GUI on your local machine
Download and install the latest version of the Intel oneAPI Vtune Profiler GUI from this [link](https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler-download.html).

Upon installation, launch the GUI from the installation directory depending on your OS.
- Windows OS: [Program Files]\Intel\oneAPI\vtune\<version>
- Linux OS: /opt/intel/oneapi/vtune/\<version>
- Mac OS: /opt/intel/oneapi/vtune_profiler/\<version>

Download and install the latest version of Intel oneAPI Advisor [here](https://www.intel.com/content/www/us/en/developer/articles/tool/oneapi-standalone-components.html#advisor).

Upon installation, launch the GUI from from the installation directory depending on your OS.
- Windows OS: [Program Files]\Intel\oneAPI\advisor\<version>
- Linux OS: /opt/intel/oneapi/advisor/\<version>
- Mac OS: /opt/intel/oneapi/advisor/\<version>

Download the NVIDIA Visual Profiler from [here](https://developer.nvidia.com/nvidia-visual-profiler). You may be required to make an account with NVIDIA in order to download products.

## Pre-requisite (Option 2) - Using an Artemis Virtual Machine to launch profiler GUI
WORK IN PROGRESS

## CPU profiling with VTune

The purpose of profiling is to gain insight into the performance of a program while it runs on a given architecture. We will first try to understand how to carry out profiling on CPUs and then move on to GPUs.

Today's steps should give you an idea about how to carry out algorithm (hotspot), microarchitecture (memory access) and parallelism (threading) analysis for different architectures.

[Intel® VTune™](https://www.intel.com/content/www/us/en/develop/documentation/vtune-help/top.html) is one of the many profiler tools that is suitable for analyzing multithreaded applications on CPUs.

Some examples of other available profilers
- [Tau](http://www.cs.uoregon.edu/research/tau/home.php)
- [Arm Map](https://www.arm.com/products/development-tools/server-and-hpc/forge/map) $$$
- [HPC Toolkit](http://hpctoolkit.org/)
- [PAPI](http://icl.cs.utk.edu/papi/)

How does a profiler work ?
They make use of the performance counter hardware that is in-built to the architecture.
To check if hardware event-based sampling is enabled on your allocated compute node:
$ cat /proc/sys/kernel/perf_event_paranoid (it should give a value of 0)

The directory [cpu_profiling](/cpu_profiling) has the source code for this section of the tutorial.

We start with the hotspot analysis on Discovery. Within the [hotspot_analysis](/cpu_profiling/hotspot_analysis) directory we provide the code for serial (single-threaded) and parallel (multithreaded) calculation of pi along with the `Makefile` containing rules to build the binaries.

For VTune analysis, applications must be compiled with the Intel compiler. We can invoke the suitable compiler by loading the load the necessary modules `intel-oneapi` on Discovery cluster with the following commands:

```
salloc --nodes=1 --ntasks=1 --cpus-per-task=2 --partition=debug
module purge
module load intel-oneapi
```

We can now build the binaries using the following `make` commands:

```
make singlethreaded_pi_calc
make multithreaded_pi_calc
```
You should have two executables in your working directory.
Set the environment to limit the OpenMP threads:
```
export OMP_NUM_THREADS=2
```

Try executing the binaries to see if you get the value of pi

```
$./singlethreaded_pi_calc
PI = 3.141593
$ ./multithreaded_pi_calc
PI = 3.141593
```

Not we capture some profile reports with the following commands
```
vtune -collect hotspots -result-dir rSingleThread ./singlethreaded_pi_calc
vtune -collect hotspots -result-dir rMultiThread ./multithreaded_pi_calc
vtune -collect memory-consumption -result-dir rMultiMemory ./multithreaded_pi_calc

```
This will result in the creation of three reports named `rSingleThread`, `rMultiThread`, `rMultiMemory`. Import the the files to your local machine to view the results.

To quickly view the results from the command line try:
```
vtune -report summary -result-dir rSingleThread/
vtune -report summary -result-dir rMultiThread/
```

We prefer using the GUI for analysis since it is feature rich and helps with the top-down tree view during analysis. Launch the GUI as listed in the pre-requisite section.

To load the profile report click on the three lines displayed on the left bar and select open > Result > \<your report file>. Your report file will end with a .vtune extension.

## Inference
![omp_2_thread_summary](img/omp_2_thread_summary.png)
We see that a total of 2 threads are created in the execution
![omp_2_thread_activity](img/omp_2_thread_activity.png)
We can analyze the activity of the threads that are forked
![omp_2_function_memory_allocation](img/omp_2_function_memory_allocation.png)
We also see the memory allocations and deallocations happening across the call stack.

## Roofline analysis with Advisor
For this section move to working on Devcloud's compute node. We will be referring to the sample code provided by intel. This is made available to you under the [roofline_analysis](/cpu_profiling/roofline_analysis) directory.

We request compute resources with the following command
```
qsub -I -l nodes=1:xeon:ppn=2 -d .
```
Compile the project with the `make` command and generate a roofline report with the following command.

```
 advisor --collect=roofline --project-dir=eRooflineSample -- ./release/roofline_demo
```

Import the report to your local machine and view it with the Advisor GUI. Steps to download and open advisor are mentioned in the pre-requisite section.

## Quirks

To see the list of available architecture-specific libraries on your compute node use the `$lscpu` command. We use the `-march=core-avx2` option when compiling on Discovery's compute nodes since the compute node on debug queue support the [Advenced Vector Instructions](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)(AVX).
To precisely check for AVX compatibility try
`lscpu | grep avx` on your allocated compute node.

## Inference
![demo_roofline](img/demo_roofline.png)
We see our program sits in a region which signals it is approaching the limits of the bandwidth and compute bounds of the architecture.

## GPU hostspot analysis with Nvidia Visual Vrofiler and nvprof tool
`nvprof` is the command line tool used to profile on Nvidia GPU accelerated architectures. Visual Profiler is the GUI that is used to interactively analyze the collected data.  
We will be going back to HPC to try this part of the tutorial. The code for this part of the tutorial is available in the `gpu_profiling`(/gpu_profiling) directory.

Start by requesting the GPU compute nodes with the following command.
```
salloc --partition=debug --gres=gpu:p100:1 --time=00:30:00
```
load nvidia sdk and compile
```
module load pgi-nvhpc/20.7
nvcc -o pi pi.cu
```
Collect a report with nvprof
```
nvprof --metrics achieved_occupancy,ipc -o occupancy.prof ./pi
nvprof -o timeline.prof ./pi
```
Open up NVVP on your local system and analyze the results.

You'll notice something like the plots below.
See how malloc and memcopy take up the highest bandwidth?

![nvprof_timeline](img/nvprof_timeline.png)
