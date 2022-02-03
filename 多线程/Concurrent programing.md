Concurrent programing

basic concepts

1. resource contention
2. synchronization
3. scalability & performance

concurrency in Java

1. Java Threads
2. synchronization in Java

Design strategies

OpenMP(C, C++)

OpenCL

Formal Specification of Concurrent Systems

第一节课

1. **什么是concurrent programing？**

A concurrent program is the interleaving of sets of sequential atomic instructions

2. **Partial correctness和Total correctness**

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220111192709123.png" alt="image-20220111192709123" style="zoom:50%;" />

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220111192730806.png" alt="image-20220111192730806" style="zoom:50%;" />

In Short Partial tells if the semantics are true or not while total correctness mean semantics are true along while termination is must.

<img src="C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220111193104292.png" alt="image-20220111193104292" style="zoom:50%;" />

3. **2 types of correctness properties**

**Safety properties: must always be true.**

1. **Mutual exclusion**（互斥）: Two processes must not **interleave** certain sequences of instructions. Typically this involves **changing the state of a shared resource**, e.g. 

   • Updating the value of a shared variable. 

   • 2 or more processes updating a shared file.

2. **Absence of deadlock**（避免死锁）: Deadlock is when a **non terminating** system **cannot respond** 		to any signal.

**Liveness properties: These must eventually be true.**

1. Absence of starvation: Information sent is delivered. 

2. Fairness: That any contention must be resolved.

**There 4 different ways to specify fairness.**

**Weak Fairness；Strong fairness；Linear waiting；FIFO**