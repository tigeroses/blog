---
title: 在cuda中使用哈希表
date: 2020-03-15 20:08:38
tags: cuda hash cudpp
---

关于在cuda中使用哈希表的一些经验总结

## cuda中哈希方法
目前已知的在cuda中使用哈希的方法:
1. **数组**  
   适用于较小的数据规模,如键的范围是int,或者能转化为整型,值类型最长为long等

2. **cudpp**  
   可接受的键值范围均为32bit,相比数组好处是占用内存小,不用存储无用数据  
   其内部使用布谷鸟过滤,核心思想是多个hash算法生成多个映射值,如果有一个位置是空的,就将元素放入,否则踢走其中一个,被踢走的再去踢别人,依次类推  
   缺点是无法动态插入,即必须把键值对先准备好;主要用来查询  
   [cudpp_github](https://github.com/cudpp/cudpp)

3. **huge-CTR**  
   这是英伟达开发的一个点击率推荐系统的库,其中实现了哈希功能  
   优点是官方文档写了支持动态插入  
   [huge-CTR_github](https://github.com/NVIDIA/HugeCTR)

## cudpp hash使用
使用步骤:
1. 获取GPU卡信息  
   这也是任何cuda程序的第一步,检查有没有卡,以及卡的计算能力等;使用`cudaGetDeviceCount() cudaGetDeviceProperties()`等API来获取信息
```cpp
    int deviceCount;
    cudaGetDeviceCount(&deviceCount);
    if (deviceCount == 0)
    {
        fprintf(stderr, "Error (main): no devices supporting CUDA.\n");
        exit(EXIT_FAILURE);
    }
    int dev = 0;
    cudaSetDevice(dev);
    cudaDeviceProp prop;
    if (!quiet && cudaGetDeviceProperties(&prop, dev) == 0)
    {
        printf("Using device %d:\n", dev);
        printf("%s; global mem: %uB; compute v%d.%d; clock: %d kHz\n",
               prop.name, (unsigned int)prop.totalGlobalMem, (int)prop.major,
               (int)prop.minor, (int)prop.clockRate);
    }
    if (prop.major < 2)
    {
        fprintf(stderr, "ERROR: CUDPP hash tables are only supported on "
                "devices with compute\n  capability 2.0 or greater; "
                "exiting.\n");
        exit(1);
    }
```

2. 创建*CUDPP Handle*  
   CUDPPHandle 在每个cuda上下文都要建立一个
```cpp
    CUDPPHandle theCudpp;
    CUDPPResult result = cudppCreate(&theCudpp);
    if (result != CUDPP_SUCCESS)
    {
        fprintf(stderr, "Error initializing CUDPP Library.\n");
        return retval;
    }
```

3. 准备数据  
    准备两个unsigned int* 数组, 分别存放keys和values
    也可以从一个std::unordered_map获取数据  
    将keys和values从host拷贝到device

4. 创建*CUDPPHandle*
```cpp
    CUDPPHashTableConfig config;
    config.type = CUDPP_BASIC_HASH_TABLE;
    config.kInputSize = kInputSize;
    config.space_usage = space_usage;// 测试值有 1.05f, 1.15f, 1.25f, 1.5f, 2.0f
    CUDPPHandle hash_table_handle;
    CUDPPResult result;
    result = cudppHashTable(theCudpp, &hash_table_handle, &config);
    if (result != CUDPP_SUCCESS)
    {
        fprintf(stderr, "Error in cudppHashTable call in"
                "testHashTable (make sure your device is at"
                "least compute version 2.0\n");
    }
```

5. 插入数据
```cpp
result = cudppHashInsert(hash_table_handle, d_test_keys, d_test_vals, kInputSize);
cudaThreadSynchronize();
if (result != CUDPP_SUCCESS)
{
    fprintf(stderr, "Error in cudppHashInsert call in testHashTable\n");
}
```

6. 使用哈希表查询数据
```cpp
result = cudppHashRetrieve(hash_table_handle, d_test_keys, d_test_vals, kInputSize);
cudaThreadSynchronize();
if (result != CUDPP_SUCCESS)
{
    fprintf(stderr, "Error in cudppHashRetrieve call in testHashTable\n");
}
```

7. 验证数据  
   将查询的结果由GPU内存拷贝回CPU内存,进行数据的验证
   
8. 释放资源
```cpp
result = cudppDestroyHashTable(theCudpp, hash_table_handle);
if (result != CUDPP_SUCCESS)
{
    fprintf(stderr, "Error in cudppDestroyHashTable call in testHashTable\n");
}
result = cudppDestroy(theCudpp);
if (result != CUDPP_SUCCESS)
{
    printf("Error shutting down CUDPP Library.\n");
}
```

## 问题和改进

### cudpp内存泄漏问题
cudpp在更新的cuda版本如cuda10,更新的显卡架构如TitanV下出现内存泄漏问题  
情况就是只要使用cudpp的lib,代码经过第一个cuda API调用之后就会卡死,内存不断增长,直到内存爆掉  
经过测试,我发现是计算能力配置问题,新的显卡架构支持更高的计算能力,只要在编译选项中增加**compute_60;compute_70**即可解决问题  
详见[cudpp_issues_187](https://github.com/cudpp/cudpp/issues/187)


### 扩展cudpp哈希表

> 修改CUDPP库中哈希功能支持更长的键类型.

> 原库支持32bit键值对,将其编码在64bit的long long类型中;我实际工作中需要对碱基序列进行哈希查找,每一个碱基可能有ACGTN五种类型,最开始只处理单barcode是10bp,所以有5^10(9765625)种可能序列,不到10M数据,在cuda中使用数组就可以了;后来需要处理双barcode,20bp,有5^20(95367431640625)种可能序列,需要约95T数据,数组显然不够,只能用哈希,因此将键类型从32bit扩展到48bit,可以支持5^20的键,剩下16bit存储值,依然编码到64bit的long long类型,达到最小改动满足需求的目的.

[仓库地址](https://github.com/tigeroses/cudpp)