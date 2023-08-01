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



## HW2-1 Particle Simulation Parallelization Serial VS OpenMP
### Introduction
We focused on parallelizing the code for particle simulation. In the first part, we optimized the serial code using packed doubly-linked lists of particles and achieved about a 463x speedup at 10000 particles compared to the naive serial code. In the second part, we introduced parallelism using OpenMP, resulting in a 1382x speedup at 10000 particles.

### Algorithm
The serial code utilized packed doubly-linked lists of particles to facilitate re-binning. These lists were implemented by interleaving them inside a contiguous array of particle wrapper classes. The grid was represented as a contiguous array of pointers to the head of each bin.

The algorithm consisted of two main components. First, it iterated over the bins by grid index, calculating the force on each neighbor pair. Since bins were indexed by coordinates, iterating through neighbor bins was straightforward. Each bin was a linked list, enabling easy iteration through the particles within a bin. In the second step, it iterated over the particle wrappers by particle index, calculating their new positions and re-binning them. Re-binning was achieved in constant time due to the doubly-linked list structure. The second step exhibited good cache coherence for computing new positions, as the particle wrappers maintained the same order as the particle data, allowing for efficient memory access.

However, re-binning suffered from random cache misses because particles in a bin were not stored sequentially, and their new grid index was random. With a high number of particles, the data structures for particle bins and the grid might not fit into the cache. This could lead to cache inefficiency and performance degradation.


### OpenMP Algorithm
In the parallel code, we retained the same algorithm as in the serial code, but introduced parallelism using OpenMP. We divided the parallel process into two steps:

In the first step, we distributed ranges of columns to different threads, ensuring that no range was less than three columns to avoid overlapping neighbors. This part required no synchronization or communication, as threads performed disjoint computations and data read/writes.
In the second step, we assigned ranges of particles to threads. Since particles in different ranges could still move to or come from the same grid location, it required synchronization around re-binning. We assigned an OpenMP lock to each grid cell, allowing threads to acquire locks at the source and target grid cells while re-binning a particle.


### Parallel Analysis
Considering N particles and P threads, and G grid points, we analyzed the probability of collision and mutual exclusion during synchronization:

The probability of at least two threads needing to re-bin a particle, which is independent of the number of threads, where R is the probability of a particle being re-binned to another uniformly distributed grid point.

The overall probability of mutual exclusion ended up being relatively low, indicating that synchronization was not a significant bottleneck in the parallel implementation.

### Performance Results and Other Trailed Methods
Using only a singly-linked list for the particle bins: This approach didn't perform as well because it required regenerating the grid from scratch after each step, leading to a larger bottleneck compared to synchronization savings.
Iterating by particle order rather than grid position in the first step: This approach had poor cache behavior for accessing grid pointers, leading to no observable speedup.
Periodically sorting particles by grid position: While this approach theoretically could provide better cache performance, the need to write back the particle data in original order during state saving, combined with the overhead of memory copying during counting sort, made it less practical.
Performance Analysis: Serial and Parallel Runtime Complexity
To evaluate the performance of the serial and parallel codes, we plotted the runtime against the number of particles in a log-log scale:
The serial optimized algorithm showed linear performance on the log-log scale, indicating a runtime complexity of O(n). In contrast, the naive implementation exhibited almost exponential growth with particle numbers, which highlights the effectiveness of our optimizations.

### Design Choices and Performance Impact
The design choices significantly impacted the performance of the code:

Binning Strategy: Utilizing a grid-based binning strategy allowed us to reduce unnecessary particle pair iterations and improved the performance from O(n^2) to O(n).

Packed Doubly-Linked Lists: Using packed doubly-linked lists for particles facilitated efficient re-binning, achieving an O(n) time for this operation.

Parallelism with OpenMP: Introducing parallelism using OpenMP resulted in a substantial speedup, particularly in the second step where synchronization was efficiently handled.

Synchronization Strategy: Assigning OpenMP locks to grid cells for synchronization reduced the probability of mutual exclusion, minimizing the impact on performance.

In conclusion, the optimized serial and parallel implementations achieved significant speedups by utilizing efficient data structures, parallelizing critical steps, and optimizing memory access patterns. The overall performance improvement made the particle simulation code much more scalable and efficient, enabling faster simulations with larger particle sets.




