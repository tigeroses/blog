---
title: CUDA C/C++总结
date: 2020-03-01 21:51:37
tags: CUDA C
---

本篇为学习笔记,学习内容为2019年参加英伟达GTC会议的课程

需要提下学习CUDA的目的,就是为了加速自己的应用,相比于CPU-only的应用程序,可以用GPU实现较大加速,当然程序首先是计算密集型而非IO密集型

# 基础

GPU加速系统,又被称**异构系统**(Heterogeneous),由CPU和GPU组成

如果熟悉C编程,可以很快上手CUDA编程,两者在代码形式上有很多类似地方,一个比较重要概念是GPU的*launch kernel*

C代码用gcc编译,cuda代码用*nvcc*编译,nvcc内部会调用gcc

启动核函数的配置 <<<blocks,threads>>> thread是最小执行单位,由threads组成block,多个block组成grid;kernel只能运行在一个grid

一般最简单的加速示例就是一个CPU的循环,执行简单的算术运算;主要是暗示我们什么类型的程序适合GPU加速

关于threads:

* 每个block中的threads个数上限是1024
* 一个block中的threads个数由内置变量*blockDim.x*给出,多个block中计算thread索引公式为:*threadIdx.x + blockIdx.x * blockDim.x*
* grid中block的个数*gridDim.x*,因此一个grid的线程总数就是*gridDim.x * blockDim.x*
* 一般一个thread一次处理一个数据
* 注意数据与threads个数很多时候都不是一一对应的,所以要特别注意索引越界问题;一般方式是将数组长度N传入kernel,算出thread索引,与N比较
* block中的threads个数为32的倍数时最优化
* 当多个block的threads总数依然无法覆盖待处理数据长度时,在kernel中用loop来重复利用threads处理后续数据;如数据有2048个,线程总数只有1024,则每一个线程处理两个数据

cuda6之后的版本可以分配出CPU/GPU都能访问的内存,API接口为:*cudaMallocManaged*

关于异常处理:

* 一些cuda函数的返回值类型为cudaError_t, 可用来检查错误*cudaGetErrorString(err)*
* 无返回值的kernel, 使用*cudaGetLastError()* 返回cudaError_t类型
* 另外,如果有一组kernel出错,因为kernel执行是异步的,为了排查错误,可以调用同步函数如cudaDeviceSynchronize() 会返回kernel执行的错误
* 自己封装一个宏来进行错误检查是有必要的

# 统一内存管理

迭代设计过程:

APOD:Assess Parallelize Optimize Deploy

评估->并行->优化->部署

使用Nsight命令行工具**nsys**来评估性能,确定优化机会

nsys基本用法: nsys profile --stats=true exe

它会生成qdrep报告,包含诸多信息:API统计,kernel执行统计,内存拷贝的大小和时间等

程序优化方法之一:更改kernel launch的**配置参数**

Streaming Multiprocessor 流处理单元SM

* 一个block中的threads被调度到SM上执行;多个block可以被调度到同一个SM上
* 为了尽可能并行,提高性能:将grid size设置为给定GPU上的SM个数的倍数,防止不对齐导致的资源浪费
* SMs创建,管理,调度和执行的单位是一个block中的一组32个threads,叫做wraps;由有half-wrap的概念,16个线程为一组,更细粒度的并行

为了获取SM的数量,调用API:
```c
int deviceId;
cudaGetDevice(&deviceId);
cudaDeviceProp props;
cudaGetDeviceProperties(&props, deviceId);
SMs = props.multiProcessorCount;
```

基础知识:CUDA's Unified Memory

* 关于Unified Memory,当UM分配之后,不管host还是device都无法访问,访问时会发生page fault,然后触发内存的迁移,将需要的数据按batches迁移
* 使用UM要注意避免不必要的时间开销,比如需要大的连续内存块,避免页错误


Asynchronous Memory Prefetching
异步内存预取:减小页错误和按需内存迁移的间接开销的技术,提高性能
```c
cudaMemPrefetchAsync(pointerToSomeUMData, size, deviceId);        // Prefetch to GPU device.
cudaMemPrefetchAsync(pointerToSomeUMData, size, cudaCpuDeviceId); // Prefetch to host. `cudaCpuDeviceId`
```

# 流异步和性能分析

Nsight Systems 可视化的性能分析工具,其可以直接打开nsys生成的qdrep文件

Concurrent CUDA Streams 并发流;流是一系列顺序执行的命令,kernel的执行,和许多内存迁移都是发生在流内,不指定的情况下使用default stream

关于控制流的几个规则:
* 流内的操作是顺序的
* 不同流内的操作相互之间不保证有任何顺序,即可认为不相关
* 默认流执行前会阻塞直到其他所有流都执行完成,反之亦然,默认流执行的时候其他流也被阻塞

API:
```c
cudaStream_t stream;       // CUDA streams are of type `cudaStream_t`.
cudaStreamCreate(&stream); // Note that a pointer must be passed to `cudaCreateStream`.

someKernel<<<number_of_blocks, threads_per_block, 0, stream>>>(); // `stream` is passed as 4th EC argument.

cudaStreamDestroy(stream); // Note that a value, not a pointer, is passed to `cudaDestroyStream`.
```

第三个参数是每个block允许使用的shared memory的bytes,默认为0

profile driven and iterative 配置文件驱动和迭代

当确定数据只在device使用,最好只分配device的内存,减小数据迁移的开销,API:
* cudaMalloc()  only GPU
* cudaMallocHost() only CPU 锁页内存,允许异步拷贝到GPU;过多的锁页内存会影响CPU性能,使用 cudaFreeHost()来释放
* cudaMemcpy() device与host相互拷贝

Using Streams to Overlap data transfers and code execution

只要CPU内存是锁页内存,就可以使用cudaMemcpyAsync()来进行异步拷贝,另一个条件就是使用非默认流

默认情况下GPU函数执行时对CPU函数是异步的,而异步拷贝,不仅对CPU,对GPU的kernel也是异步的,可以达到边计算边拷贝数据的目的,从而掩盖数据传输时间,尽量挖掘GPU计算能力

