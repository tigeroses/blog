---
title: hdf5库使用
date: 2021-10-18 20:26:53
tags: c hdf5
---

本篇记录在centos下hdf5的C语言接口使用,包括读写h5文件,创建属性,组,数据集合等,以及使用python的h5py库来读取h5文件

## 准备

安装hdf5库

## 代码示例

引入头文件 `#include "hdf5.h"`

声明可复用的变量

```cpp
hid_t file_id, group_id, dataspace_id, dataset_id;
herr_t status;
```

### file

基础的打开文件和关闭文件操作,注意一个文件在未关闭的状态下勿重复打开,否则会报错

```cpp
// 创建文件
string filename;
hid_t file_id = H5Fcreate(filename.c_str(), H5F_ACC_TRUNC, H5P_DEFAULT, H5P_DEFAULT);
// 关闭文件
hid_t status = H5Fclose(file_id);
```

### group

组等价于目录的概念,组支持嵌套,即一个组下面可以包含别的组或数据集合

```cpp
group_id = H5Gcreate(file_id, "/myGroup", H5P_DEFAULT, H5P_DEFAULT, H5P_DEFAULT);
status = H5Gclose(group_id);
```

### dataset

数据集为存放数据的集合

#### 普通数据集合

将8个整型数作为2*4的结构写入文件

```cpp
vector<int> vec{1,2,3,4,5,6,7,8};
unsigned int rank = 2;
hsize_t dims[2] = {2, 4}; // two rows and four columns
dataspace_id = H5Screate_simple(rank, dims, NULL);
dataset_id = H5Dcreate(file_id, "/myGroup/data", H5T_NATIVE_INT, dataspace_id, H5P_DEFAULT, H5P_DEFAULT, H5P_DEFAULT);
status = H5Dwrite(dataset_id, H5T_NATIVE_INT, H5S_ALL, H5S_ALL, H5P_DEFAULT, &vec[0]);
status = H5Dclose(dataset_id);
status = H5Sclose(dataspace_id);
```

#### 组合数据集合

即C中的结构体,需要进行数据类型从内存到文件的映射

如果结构体中带有字符串,则需要特殊处理;这里使用固定长度字符串,处理起来稍麻烦些,使用变长类型 H5T_VARIABLE 则更简单

```cpp
const size_t char_len = 32;
struct MyStruct
{
    MyStruct(const char* s, unsigned int i, float f)
    {
        int p = 0;
        while (s[p] != '\0')
        {
            valStr[p] = s[p];
            ++p;
        }
        valInt = i;
        valFloat = f;
    }
    char valStr[char_len];
    unsigned int valInt;
    float   valFloat;
};
hid_t strtype, memtype, filetype;
strtype = H5Tcopy(H5T_C_S1);
status = H5Tset_size(strtype, char_len);

memtype = H5Tcreate(H5T_COMPOUND, sizeof(MyStruct));
status = H5Tinsert(memtype, "valStr", HOFFSET(MyStruct, MyStruct::valStr), strtype);
status = H5Tinsert(memtype, "valInt", HOFFSET(MyStruct, MyStruct::valInt), H5T_NATIVE_UINT);
status = H5Tinsert(memtype, "valFloat", HOFFSET(MyStruct, MyStruct::valFloat), H5T_NATIVE_FLOAT);

filetype = H5Tcreate(H5T_COMPOUND, char_len + 4 + 4);
status = H5Tinsert(filetype, "valStr", 0, strtype);
status = H5Tinsert(filetype, "valInt", char_len, H5T_STD_U32LE);
status = H5Tinsert(filetype, "valFloat", char_len+4, H5T_IEEE_F32LE);

vector<MyStruct> compoundDatas;
compoundDatas.push_back({"abc", 123, 66.88});
compoundDatas.push_back({"this is my test program", 987654321, 1.0});
rank = 1;
hsize_t compoundDims[rank] = {compoundDatas.size()};
dataspace_id = H5Screate_simple(rank, compoundDims, NULL);
dataset_id = H5Dcreate(file_id, "/myGroup/compoundData", filetype, dataspace_id, H5P_DEFAULT,
    H5P_DEFAULT, H5P_DEFAULT);
status = H5Dwrite(dataset_id, memtype, H5S_ALL, H5S_ALL, H5P_DEFAULT, &compoundDatas[0]);

status = H5Dclose(dataset_id);
status = H5Sclose(dataspace_id);
status = H5Tclose(strtype);
```

### attribute

属性可用来描述数据集,如数据的长度,统计结果等

要创建属性,首先要开辟dataspace,其为待写入数据的空间属性,一般需要设定维度,然后创建属性并写入文件

```cpp
// 使用属性标识版本号
unsigned int version = 1;
hsize_t dimsAttr[1] = {1};
dataspace_id = H5Screate_simple(1, dimsAttr, NULL);
hid_t attr = H5Acreate(file_id, "version", H5T_STD_U32LE, dataspace_id, H5P_DEFAULT, H5P_DEFAULT);
status = H5Awrite(attr, H5T_NATIVE_UINT, &version);
H5Sclose(dataspace_id);
H5Aclose(attr);
```

## 编译和运行

注意编译软件时使用的hdf5库必须和使用时的hdf5库版本完全一致,否则可能出现报错

注意替换为自己安装的库路径

```sh
hdf5Path=/root/lib/hdf5-1.12.1
g++ -std=c++17 -O2 hdf5_c.cpp -o test_hdf5 \
    -I $hdf5Path/include \
    -L $hdf5Path/lib \
    -l hdf5

export LD_LIBRARY_PATH=$hdf5Path/lib:$LD_LIBRARY_PATH
./test_hdf5
```

## h5py读取h5文件

使用C库hdf5可以写入h5文件,当然有方法来读取数据,为了方便这里使用python库h5py来读取h5文件

加载上面生成的h5文件,并依次读取普通数据集合,复合数据集合和属性

```python
import h5py

f = h5py.File("test.h5", "r")
print(list(f.keys()))

d = f['/myGroup/data']
print("dataset: /myGroup/data")
print("shape:", d.shape, " dtype:", d.dtype)
print(d[...])

d = f['/myGroup/compoundData']
print("dataset: /myGroup/compoundData")
print("shape:", d.shape, " dtype:", d.dtype)
print(d[...])

a = f['/'].attrs
print("group attributes: /")
print("version:", a.get('version')[0])
```

正常情况输出如下:

```text
['myGroup']
dataset: /myGroup/data
shape: (2, 4)  dtype: int32
[[1 2 3 4]
 [5 6 7 8]]
dataset: /myGroup/compoundData
shape: (2,)  dtype: [('valStr', 'S32'), ('valInt', '<u4'), ('valFloat', '<f4')]
[(b'abc',       123, 66.88) (b'this is my test program', 987654321,  1.  )]
group attributes: /
version: 1
```

## 参考

* [hdf5官方文档](https://portal.hdfgroup.org/display/HDF5)
* [hdf5使用示例](https://portal.hdfgroup.org/display/HDF5/Examples+by+API)
* [h5py官方文档](https://docs.h5py.org/en/stable/)