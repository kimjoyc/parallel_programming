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



## HW2-2 Particle Simulation MPI Parallelization
### Introduction
In this project, we worked on parallelizing a particle simulation using MPI (Message Passing Interface). 
The serial code uses packed doubly-linked lists of particles to facilitate re-binning of particles. The particles are organized by bins, and these lists are interleaved inside a contiguous array of particle wrapper classes. Additionally, the grid is represented as a contiguous array of pointers to the head of each bin.

The algorithm consists of two main components. In the first step, the code iterates over the bins by grid index and calculates the force on each neighbor pair. This is achieved by iterating over the neighbor bins, and within each bin, iterating through the particles. The second step involves iterating over the particle wrappers by particle index, calculating their new positions, and re-binning them. Re-binning is a constant-time operation due to the doubly-linked list structure. The algorithm exhibits relatively good cache coherence for computing the new positions because the particle wrappers maintain the same order as the particle data, ensuring efficient memory access.

However, re-binning can suffer from random cache misses since particles in a bin are not stored sequentially, and their new grid index is random. As the number of particles increases, the data structures for particle bins and the grid may exceed the cache size, leading to cache inefficiency and performance degradation.

### Algorithm
The parallelized code uses MPI to distribute the simulation among multiple ranks. Each rank runs a simulation strategy similar to the serial implementation. Particles are stored in a contiguous packed doubly-linked cell list data structure, and there is an array of pointers to the head of each cell. Cells are squares of size just a bit larger than the interaction cutoff.

The simulation steps proceed as follows:

Each rank sends the particles on its X boundaries to its X neighbor.
The rank receives ghost particles on its X boundaries.
The rank sends the particles on its Y boundaries to its Y neighbors, including any ghost particles from the previous step that are on the corners.
The rank performs the apply_force loop.
The rank performs the move loop.
When computing new cell coordinates for the particles, any particles outside the rank's subdomain are buffered to be exchanged with the appropriate neighbor.
Finally, the rank exchanges any particles that have moved to a different rank with its neighbors. The exchange is done first along the X axis, then along the Y axis, forwarding any corners.
Particle Exchange
To exchange particles, ranks use a strategy that involves storing particles in a pre-allocated buffer. Ranks call MPI_Issend in both directions to initialize a send without blocking, then probe in both directions to get the size of incoming buffers, and finally, call MPI_Recv with the size to receive the particles. Ranks then wait on the send requests to finish.

The non-blocking send strategy is employed to avoid a domino pattern of communication along the entire dimension when using blocking send, especially when there are multiple ranks along that axis.

### Other Attempted Methods
Initially, the team attempted to assign 2D subdomains to ranks and have each rank perform a linear scan through the particles to create a spatially sorted list of particles it owns. This approach was expected to improve memory coalescing and cache coherence. However, it required an MPI_Allgatherv communication on the particles at the end of each step to share updated positions with all ranks, which led to memory copy overhead and poor performance.
Another approach considered was to periodically sort particles by grid position. Although this approach could provide better cache performance, the need to write back particle data in original order during state saving, combined with the overhead of memory copying during counting sort, made it less practical.
Performance Results
The current MPI implementation divides the simulation box into rectangular subdomains assigned to a 2D topology of MPI ranks.

The symmetric apply_force in MPI gave a significant performance boost, and the implementation reliably runs in under 10 seconds for 6 million particles with 128 num_procs.

In strong scaling, keeping the problem size constant and increasing the number of processors, the performance improves significantly, as seen in the "strong scaling" plot.

In weak scaling, increasing the problem size proportionally to the number of processors, the performance remains stable as the workload per processor remains constant, as shown in the "weak scaling" plot.

### Conclusion
The MPI parallelization of the particle simulation significantly improves performance, allowing for faster simulations with larger particle sets. The strategy to divide the simulation among multiple ranks and efficiently exchange particles through non-blocking sends and receives demonstrates effective use of MPI. Overall, the optimized MPI parallelization enhances the scalability and efficiency of the particle simulation code.



## HW2-3 Particle Simulation Parallelizing Particle Simulation GPUs


### Analysis of the Performance Results
The performance results table provides insight into the computation time for different parallelization methods as the number of particles increases. It shows the execution time for each method at various particle counts, allowing us to analyze how these methods scale with the number of particles.

### CUDA Parallelization
The "Cuda" method exhibits the best performance across all particle counts, demonstrating the highest level of scalability. The execution time remains consistently low as the number of particles increases, indicating that the CUDA approach efficiently utilizes the GPU's parallel processing capabilities. CUDA leverages data-parallelism, allowing the GPU to perform simultaneous computations on multiple particles, which leads to superior scalability and reduced computational costs.

### Serial and Improved Serial
The "Serial" and "Improved Serial O(n)" methods show significant increases in execution time as the number of particles increases. Both methods exhibit poor scalability, as the execution time rises substantially with larger particle sets. The "Improved Serial O(n)" method was intended to achieve linear time complexity, but it still demonstrates a sharp increase in computation time, indicating that it may not be truly linear for very large particle counts.

### OpenMP
The "OpenMP" method shows relatively good scalability compared to the serial implementations. As the number of particles increases, the execution time grows, but the rate of increase is not as steep as in the serial methods. OpenMP parallelization efficiently distributes work among threads, leading to improved performance compared to purely serial implementations.

### MPI
The "MPI" method also demonstrates decent scalability. However, the execution time increases more rapidly as the number of particles grows compared to the OpenMP approach. MPI involves communication and synchronization overhead, which becomes more prominent with larger particle sets, leading to a reduction in performance efficiency.

### Naive CUDA
The "Naive Cuda" method appears to have better performance than the serial implementations but is outperformed by the other parallelization methods, including "Cuda." The "Naive Cuda" method may not be fully optimized to take advantage of the GPU's capabilities, leading to suboptimal performance compared to the more sophisticated "Cuda" approach.

### Conclusion
In conclusion, the performance analysis shows that the "Cuda" parallelization method utilizing the GPU offers the best scalability and computational efficiency for large-scale particle simulations. It outperforms other parallelization methods, including OpenMP and MPI, which rely on CPU-based parallel processing and communication/synchronization. However, the parallelization method chosen depends on the available hardware and the specific requirements of the simulation. For GPU-accelerated systems, the "Cuda" approach is highly recommended, while for CPU-based clusters, OpenMP or MPI can still provide reasonable scalability and performance. The "Naive Cuda" approach, while faster than the serial methods, does not fully exploit the potential of the GPU and should be further optimized for improved performance.


## HW3: Parallelizing Genome Assembly UPC++


### Distributed Memory of Hash MapsAlgorithm

We achieved the distributed memory of hash maps by using `upcxx::global_ptr` , pointing to distributed data. We store `upcxx::global_ptr<kmer_pair>` as an array to store K-mers and an array of`upcxx::global_ptr<int>` to store the used slots. It also initializes global pointers to local portions, making them accessible across all processes. The implementation of the hash map uses one-sided communication, which allows processes to read and write data in the hash table without explicit synchronization. This is achieved through the `upcxx::rput`, `upcxx::rget`, and `upcxx::atomic_domain` functions. The HashMap structure allows efficient insertion and retrieval of k-mer pairs in a distributed manner across multiple processes.

### Parallel Computation

With the data distributed, we are able to compute data in a parallel fashion. The steps are as follows:

1. Initialize the UPC++ runtime and process input arguments.
2. Check the k-mer size to ensure compatibility with the compiled binary.
3. Calculate the hash table size based on the number of k-mers
4. Initialize the HashMap data structure and read the k-mers from the input file, partitioning the data among the available ranks.
5. Insert the k-mers into the hash table.
6. Identify the starting k-mers
7. Assemble the contigs by following the forward extensions of the starting k-mers through the hash table

## A discussion of how using UPC++ implemented your design choices.

UPC++ has greatly simplified and structured the coding of this homework by abstracting parallel programming complexities and providing a powerful framework for efficient and scalable parallel code development. By taking advantage of UPC++ features such as global pointers, one-sided communication, and synchronization barriers, the homework implementation has been designed to fully utilize the power of UPC++ for distributed computing.  The use of UPC++ to implement the k-mer hash table provides several benefits over other parallel programming models, such as MPI or OpenMP. First, UPC++ provides a more expressive programming model than MPI or OpenMP, making it easier to write and maintain parallel code. Second, UPC++ provides a number of performance optimizations that are tailored to distributed memory systems, such as non-blocking remote memory access and efficient one-sided communication primitives. Finally, UPC++ provides a higher level of abstraction than MPI or OpenMP, which can simplify the programming of complex parallel algorithms.

## How might you have implemented this if you were using MPI? If you were using OpenMP?

The MPI implementation would require a similar UPC++  implementation that utilizes MPI-3 one-sided communication. However, UPC++ provides a more expressive programming model that enables more efficient communication patterns, especially for irregular and dynamic communication patterns. Thus, the MPI implementation would require more explicit communication code to be written and the code may be less efficient. Specifically, the k-mers would need to be distributed across the MPI processes, with each process responsible for a portion of the k-mers. Then the communication would need to be utilized to ensure that each process had all of the k-mers that it needed to insert into the hash table. Lastly, synchronization would be employed to ensure that multiple processes did not attempt to insert the same k-mer into the hash table simultaneously.

Alternatively, OpenMP implementation would involve parallelizing loop-level parallelism and require a shared memory parallelism approach that would be incompatible with distributed memory parallelism. Consequently, the k-mers would be loaded into memory on a single node, and the hash table would be shared among the threads on that node. The synchronization step would need to ensure that multiple threads did not attempt to insert the same k-mer into the hash table simultaneously. Thus, this implementation would need to be heavily modified to distribute the workload across multiple nodes, using a combination of OpenMP and MPI or another parallel programming model.

## Any optimizations you tried and how they impacted performance.

Here are some optimizations we tried performance:

1. Change the hash function - If the hash function used is not well-suited for the data being stored, it can result in an uneven distribution of keys across the hash table, causing a higher rate of collisions. Changing the hash function could lead to better distribution and fewer collisions.
2. Reduce network communication - Since this is a distributed hash table, communication overhead between the nodes can be a bottleneck. One way to reduce communication is to increase the size of each node's hash table. This would decrease the number of requests to other nodes for data. However, this approach would increase the memory requirements of each node.


### Graphs and discussion of the **scaling experiments**

**Inter-node using 64 tasks per node** 

There could be several reasons why adding more nodes leads to an increase in the total runtime of a parallel program. Here are a few possible explanations:

1. Communication overhead: In a distributed memory parallel program, nodes must communicate with each other in order to exchange data and synchronize their operations. As the number of nodes increases, the amount of communication needed to coordinate their activities also increases. This can lead to increased communication overhead, which slows down the overall program performance.
2. Load balancing: In a parallel program, it's important to ensure that the workload is evenly distributed across all the available nodes. If some nodes are idle while others are overloaded, the overall performance of the program can suffer. As the number of nodes increases, load balancing becomes more difficult, and it's possible that some nodes may become underutilized, which can slow down the overall performance.

**Intra-node varying number of tasks per node**

Increasing the number of tasks per node while keeping the number of nodes fixed can lead to a decrease in the overall time taken because it allows for better utilization of the available resources. When there are more tasks per node, each node can handle a larger workload, which reduces the amount of time that each task has to wait for available resources. This can result in better load balancing and reduced overhead due to communication between nodes. Additionally, it can reduce the amount of data transfer between nodes, which can be a bottleneck in distributed systems. As a result, increasing the number of tasks per node can lead to better performance and reduced overall time taken.

### **Perform multinode experiments using 1, 2, 4, and 8 nodes with 60 tasks per node**

As we can see in the graph, the insertion time and the assembly time are closely correlated regardless of Node Number or Ntasks, with insertion time slightly lower than assembly time.

### How the performance change when moving from **60 tasks per node** to **64 tasks per node**?

When we move from 60 ntasks per node to 64 ntasks, the performance increases with less insertion and assembly time overall, with an exception at 2 nodes. We suspect this is due to communication between the nodes are slowed down when using a fixed ntasks of 64.

## **Run intra-node experiments using 1 node and varying the number of ranks per node**



## I**ntra-node scaling on 1 node** and [1-64] tasks per node



As ntasks per node increases on a single node, the insertion or assembly time decreases, with the exception at Ntasks=32 and assembly time is more than that of Ntasks=16. We think it’s because of the increased communication overhead of the Ntasks=32.
