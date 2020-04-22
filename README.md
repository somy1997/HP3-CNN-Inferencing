# Introduction
This project aims at implementing several convolution kernels for a CNN Inferencing Engine, analyzing the algorithms for AlexNet and VGG-19 architectures and picking the most efficient out of the implemented ones for those architectures. The following tasks were implemented in the project:
* Creating a custom specification format which can be used to store and load available pretrained models in C++. Converting pretrained VGG-19 and AlexNet to the developed format 
* Implementing the various convolution algorithms without the help of CUDNN library:
	* Direct Convolution
	* Convolution that uses IM2COL unrolling and GEMM 
	* Convolution that uses FFT computation
	* Convolution that uses Winograd Algorithm
* Implementing a forward pass library which can read the custom model and perform standard operations like Convolution (using both CUDNN and the above implemented functions), Pooling, Activation and FC/Linear Layer operation. 
* Creating a small dataset to test the correctness of the above operations and algorithms
* Profiling the convolution algorithms for the above mentioned architectures for run times and GPU memory usage

# Overview of Implementation
The following diagram represents an overview of our system:
![Implementation Overview](https://github.com/prajwal1210/HP3-CNN-Inferencing/blob/master/documentaion-files/ImplementationOverview.png)
  
# Directory Structure
Here, we briefly explain the skeleton directory structure of the project. The concrete implementation details of the various components shown in the diagram above are present within the respective directories:
```
.
├── forward/                  : Contains the main Inferencing Engine libraries 
│   ├── data/                   : Contains the dataset
│   ├── batch_test/             : Contains the Batch Image CNN Forward Test files
│   ├── cnn_forward_test/       : Contains the Single Image CNN Forward Test files
│   ├── operations_test/        : Contains the Batch Image CNN Forward Test files
│   ├── cnn_forward.h           : Library that wraps the forward pass and model loading
│   ├── cnn_forward.cc		
│   ├── data_util.h             : Data Loader Library
│   ├── data_util.cc
│   ├── operations.h            : Library containing the main CNN operations
│  	└── operations.cc 
├── kernels/                  : Contains Kernel Implementations
│   ├── Direct/                 : Contains the implementation of Direct Convolution
│   ├── FFT/                    : Contains the implementation of the FFT based Convolution
│   ├── im2col/                 : Contains the implementation of the Im2Col unrolling and GEMM based Convolution
│   └── winograd/               : Contains the implementation of the Winograd-based Convolution
├── pretrained-models/        : Directory where the pretrained VGG and AlexNet modes are saved
├── profiling/                : Contains the code for profiling
│   ├── MemoryUsageProfiling/   : Contains the code for Memory Usage profiling
│   └── TimeProfiling/          : Contains the code for Kernel Run Time profiling 
├── proto                     : Contains files related to protobuf
│   ├── * Protobuf              : The files generated by protobuf compiler for
│	│	generated files *		  C++ and Python
│   ├── network.proto           : Custom Protobuf Specification file for a CNN model 
│   ├── translator.h            : Library that translates protobuf objects to operations in forward/operations.h class 
│   └── translator.cc
├── Makefile                  : Global Makefile
├── ConvertToSpecification.py : Script to download pretrained models and save them according to the custom specification 
└── README.md
```
# Requirements
To run the entire code, there are a few requirements :
* CUDA 10.1 (This is the version  tested on. To change it some makefiles have to changed)
* CUDNN  
* Google Protobuf Complier and C++ Runtime :
* PyTorch 
* OpenCV (C++ and Python)

**[IMPORTANT]** Apart from the library requirements, we would like to mention that there are two versions of the following kernels each having different memory requirements and respective performances :
* FFT Kernel -
	* ```fft_fast.cu``` : This is the faster fft variant which is the default setting in all the makefiles. This requires a considerably large GPU memory and only works on GPUs with **memory > 12 GB** like on Colab
	*  ```fft_lowmem.cu``` : This is the much  slower fft variant requiring far less memory than the fast version and can run on GPUs with **memory >= 2 GB**
* Winograd Kernel -
	* ```winograd_fast.cu``` : This is the faster winograd variant which is the default setting in all the makefiles. This requires a considerably large GPU memory and only works on GPUs with **memory > 12 GB** like on Colab
	*  ```winograd_lowmem.cu``` : This is the much  slower winograd variant requiring far less memory than the fast version and can run on GPUs with **memory >= 3-4 GB**

We explain how to over-ride the default Makefile settings and switch to the lowmem versions in the Compile Instructions.

# Getting Started : Pre-Build Steps
After cloning the directory, there are certain steps that need to be carried out so that the code can be compiled and executed

**NOTE:** Even though we recommend carrying out the steps manually,  the global Makefile (in the main directory) now supports doing all steps mentioned below for you by running:
```
$ make ready_project
```

###  1. Compiling the proto files
Since every version of protobuf present has some differences specially in the C++ runtime, it is important to build C++ libraries again from the specification:
```
# In the main directory #
$ cd proto
$ protoc -I=. --cpp_out=. ./network.proto
```
Additionally, if you encounter problems with the proto files generated for python (it is more stable across versions than C++ files but it might change in the future), you should compile those as well: 
``` (In the proto directory) $ protoc -I=. --python_out=. ./network.proto ```
### 2. Downloading the pretrained models and saving them
```
# In the main directory #
$ python ConvertToSpecification.py
```
### 3. Unzipping the Training Data
```
# In the main directory #
$ cd forward/data
$ unzip MiniImageNet.zip
```

# Compile Instructions

> **[IMPORTANT] Changing to Low memory kernels for Winograd and FFT  while compiling** 
> Please read the Requirements section to know what low memory variants of these kernels are. By default, the Makefiles compile the faster higher memory variants of these kerenls. In order to switch to the lowmem options, use the tags: ```FFT_FILE=fft_lowmem.cu```,  ```WINOGRAD_FILE=winograd_lowmem.cu``` or both, with any of the make procedures mentioned hereafter as:
> ```
> $ make <TARGET> FFT_FILE=fft_lowmem.cu WINOGRAD_FILE=winograd_lowmem.cu
>  ``` 

**Global Make:** All the binaries mentioned mentioned below can be compiled using the global Makefile by the following commands:
```
## In the main directory ##
$ make tests 		#For the tests
$ make profilers 	#For the profiler binaries

## For cleaning ##
$ make tests TARGET=clean       #For the tests
$ make profilers TARGET=clean   #For the profiler binaries
```
However, we suggest to do it manually within each directory as the ```make run```  commands cannot be run through the global Makefile. So, manually making them within each directory is a more safer option. Another reason to do it manually is if you want to differently over-ride the Makefile defaults

## Tests
We have 3 tests that extensively check the correctness of differenet functionalities of the inferencing engine. The tests are located in different folders in the `forward/` directory. The tests are:

 1. **Test-1 Operations Test**
	> Located under forward/opearations_test

	This tests the individual component operations in the operations.h library and compares the results with Pytorch Output
	
 2. **Test-2 Single Image CNN Forward Test**
	> Located under forward/cnn_forward_test
		
	This tests the CNN Forward library by running both VGG19 and Alexnet on a single image and comparing the results with Pytorch Output
	
 3. **Test-3 Batch Image CNN Forward Test**
	> Located under forward/batch_test
	
	This tests the CNN Forward library for a batch of 8 images by running both VGG19 and Alexnet and comparing the results with Pytorch Output
### **How to run the tests**
The steps to compile and run the tests mentioned above are the same through the use of makefiles. Within the respective directories, the following steps are to be followed:
 * **Compilation:**   `$ make test`  [Can be skipped if compiled through the global makefile]
 * **Running the tests:**
	```
	$ make run_direct		# For Direct Convolution
	$ make run_fft			# For FFT Convolution
	$ make run_winograd		# For Winograd Convolution
	$ make run_im2col		# For IM2COL & GEMM Convolution
	```
 * **Clean the test directory:**  ```$ make clean```
 
## Profiling
We have two types of profiling under the `profiling/` folder in the main directory. Each profiling pipeline is split into - 
 1.  C++/Bash based logger which runs the forward inferencing engine, 
 2. Python based analyzer which generates the plots 

The loggers generate files which are used by the analyzers. We have provided the logger files generated by our experiments in the directories as well so that you can only run the analyzers.
Now, we explain the two types of profiling:

#### Run Time Profiling
> Located under profiling/TimeProfiling/

This profiling setup runs a forward pass of the VGG and AlexNet architecture over multiple batch-sizes for all the convolution algorithms multiple times each (= 3 as default), measures the different times as convolution time, algorithmic overhead time and total time and compares them [More details of the experimental setup are provided in the concerned directory]. The steps to run it are as follows:

* **Compilation:**   `$ make`  [Can be skipped if compiled through the global makefile]
* **Running the C++ based logger:**
	```
	$ make run_vgg      # For VGG
	$ make run_alex     # For AlexNet
	```
	By default, this code runs 3 iterations of each batchsize and kernel setting for more statistically sound results. This can be changed, however we faced some issues in Colab when increasing it beyon 3 so that should be kept in mind. To change this default:
	```
	$ make run_<vgg/alex> NUM_RUNS=<NUMBER> 
	```
	This will generate the log files - `logVGG.txt` and `logALEX.txt`
* **Running the Analyzer:**
	The analyzer can be run be as follows:
	```
	$ python analyzer.py
	``` 
	This will take `logVGG.txt` and/or `logALEX.txt` present in the directory and generate plots in the `profiling/TimeProfiling/Plots/` folder   
	
* **Clean the test directory:**  ```$ make clean``` [Removes only the C++ based logger binary]


#### GPU Memory Usage Profiling
> Located under profiling/MemoryUsageProfiling/

This experimental profiling setup runs a forward pass of the VGG architecture for a single image and profiles the memory being used through ``nvidia-smi`` command. It then plots a comparison of the GPU usage pattern

**[IMPORTANT] NOTE:** The logger part of this compiler cannot be run on Colab as it requires `nvidia-smi` to run in ***background*** in shell. Only analyzer can be run on the previously generated files on Colab

The steps to compile and run this profiler are:
 * **Compilation:**   `$ make`  [Can be skipped if compiled through the global makefile]
* **Running the C++ and Bash based logger:**
	```
	$ make run
	```
	By default, this code runs 1 forward pass of each kernel on VGG with batch_size = 1 while a special setting of the nvidia-smi commands logs the memory usage in background. This will generate 5 files  - `memory_cudnn.txt`, `memory_direct.txt`, `memory_fft.txt`, `memory_im2col.txt`, `memory_winograd.txt` 
	

	> In case, you encounter any issues with `make run`, manually run the `memProfiler.sh` script and try 

* **Running the Analyzer:**
	The analyzer can be run be as follows:
	```
	$ python memoryAnalyzer.py
	```
	This will take whatever of the above mentioned files are present in the directory and generate plots in the `profiling/MemoryUsageProfiling/Plots/` folder   

* **Clean the test directory:**  ```$ make clean```

## *General Compilation Issues*
We predict some issues that you can encounter during compilation and offer solutions:

* **CUBLAS Library Error:** [Will occur definitely if you try to change CUDA Version]
The path to CUBLAS library has been hardcoded to the Makefiles as: 
 `/usr/local/cuda-10.1/man/man7/libcublas.so.7`
 This may cause an error if CUBLAS is not present in that location or will most likely cause one if you are using a different version of CUDA than 10.1. You need to change this path or call all the makes as:
	```
	$ make <TARGET> CUBLAS=<PATH TO libcublas.so.7>
	```
	

# Profiling Results

The profiling results are explained in the following READMEs within appropriate directories:
* [Time Profiling](profiling/TimeProfiling/README.md)
* [GPU Memory Usage Profiling Profiling](profiling/MemoryUsageProfiling/README.md)

The explanation regarding other files like kernel implementations are available in the respective folders
***

#### Google Colab Notebook to run the Tests/TimeProfiling on - https://colab.research.google.com/drive/1yjsrzHxDNiFaRRBKb8oK6P7b-ZQCU7Qn