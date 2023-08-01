# Parallel Programming Directories

## HW1 Optimized Matrix Multiplication
### Introduction
This repository contains the implementation of an optimized DGEMM algorithm for matrix multiplication. Various techniques such as loop reordering, microkernel vectorization, multilevel blocking, repacking, and prefetching were utilized to improve the performance of the algorithm significantly, achieving a speedup ranging from 5% to 50%.

### Optimizations
Loop Reordering
Reordered the for loop "jki" to optimize access to the matrices and reduce large memory jumps.
Preserved spatial and temporal locality by making the i-th loop the innermost loop for matrix B.
Microkernel Vectorization
Implemented microkernel vectorization using AVX intrinsics to achieve parallelism and improve performance.
Utilized padding and unpadding functions to ensure contiguous memory access and cache alignment.
Multilevel Blocking
Performed loop reordering and matrix B unrolling to fit the cache line and eliminate load instructions for B access.
Applied unrolling to the microkernel vectorization and matrix C.
Repacking
Implemented repacking with alignment of 32 to enable simultaneous contiguous memory allocation and vectorized instructions with matrix C.
Prefetching
Utilized gcc intrinsic "__builtin_prefetch" to prefetch memory locations and speed up the memory transfer process.
Merging Algorithms
Merged the prefetching algorithm with the multilevel blocking, microkernel vectorization, and repacking algorithm for improved performance on smaller matrices.
Odd Behavior


### Performance Dips
Performance dips may be caused by lack of cut off recursion before 1x1 microkernel on small blocks and not auto-tuning hyperparameters.
Using alternate data layouts involving blocked or recursive layouts could potentially reduce performance dips.
Huge Performance Decrease Around Size 512
The performance decrease around matrix size 512 may be due to cache conflicts, reaching the cache limit of the CPU.
To address this issue, tuning parameters and utilizing memory prefetching helped achieve a better ratio of cache occupancy to cache misses.

### Getting Started
To run the optimized DGEMM algorithm, follow these steps:

Clone this repository to your local machine.
Compile the code using a C++ compiler that supports AVX intrinsics.
Run the compiled executable with the desired matrix size and other parameters.
