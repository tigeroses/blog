---
title: seqan库的使用
date: 2020-04-26 20:40:41
tags: seqan 基因注释
---

seqan库是进行生物序列分析的一个现代的C++库,目前有seqan2, seqan3两个版本,seqan3正在开发当中  
我打算应用seqan库实现一个简单的注释程序,因为seqan3暂时还未实现gtf文件的相关操作,因此选用seqan2  

seqan是header-only的库,因此无需添加lib,只要包含头文件即可使用  

## 定义别名

为了使用简洁,定义常用类型的别名

```cpp
typedef seqan::FragmentStore<> TStore;
typedef seqan::Value<TStore::TAnnotationStore>::Type TAnnotation;
typedef TAnnotation::TId TId;
typedef TAnnotation::TPos TPos;
typedef seqan::IntervalAndCargo<TPos, TId> TInterval;
typedef seqan::IntervalTree<TPos, TId> TIntervalTree;
typedef seqan::String<TIntervalTree> TIntervalTrees;
typedef seqan::Iterator<TStore const, seqan::AnnotationTree<> >::Type TCIterator;
typedef seqan::Iterator<TStore, seqan::AnnotationTree<> >::Type TIterator;
typedef seqan::String<TId> TIds;
typedef seqan::BamAlignmentRecord BamRecord;
```

## gtf文件的加载

直接上代码:

```cpp
seqan::FragmentStore<> store;
seqan::GffFileIn annotationFile;
if (!seqan::open(annotationFile, gtf_file.c_str()))
{
    return;
}
readRecords(store, annotationFile);
```

可以看到,只要几行代码就将gtf文件的数据读取到内存中;使用FragmentStore来管理内存

gtf数据在内存中的存储,可以被视为关系型数据库,每一行表示一个gene,因此通过唯一ID可以访问gene数据,而gene数据是树状结构,如下图:
![seqan-annotation-store](/images/intro_seqan/seqan-annotation-store.png)

想要遍历gtf数据,首先拿到根节点迭代器,然后使用树的遍历方式即可

## 构建interval tree

```cpp
seqan::String<seqan::String<TInterval>> intervals;
int numContigs = seqan::length(store.contigStore);
resize(intervals, numContigs);

TIterator it = seqan::begin(store, seqan::AnnotationTree<>());
if (!goDown(it))
{
    return;
}
do
{
    SEQAN_ASSERT_EQ(getType(it), "gene");
    TPos beginPos = getAnnotation(it).beginPos;
    TPos endPos = getAnnotation(it).endPos;
    TId contigId = getAnnotation(it).contigId;
    if (beginPos > endPos)
        std::swap(beginPos, endPos);
    appendValue(intervals[contigId], TInterval(beginPos, endPos, value(it)));
} while (goRight(it));

TIntervalTrees intervalTrees;
resize(intervalTrees, numContigs);

SEQAN_OMP_PRAGMA(parallel for)
for (int i = 0; i < numContigs; ++i)
    seqan::createIntervalTree(intervalTrees[i], intervals[i]);
```

要构建线段树intervalTrees,首先得有一组线段intervals.通过遍历gtf数据,对每个gene构建一个interval,加入intervals,这里注意chromosome之间无关联,应分别建立数据;最后通过createIntervalTree接口构建intervalTrees,利用chromosome之间独立的特性,使用openmp加速构建过程

## 注释

```cpp
// 打开输入bam
seqan::BamFileIn inFile;
seqan::open(inFile, inputBamFilename.c_str());
seqan::BamHeader header;
seqan::readHeader(header, inFile);

// 打开输出bam,注意初始化header
seqan::BamFileOut fileOut;
fileOut.context = seqan::context(inFile);
seqan::setFormat(fileOut, seqan::Bam());
seqan::open(fileOut, outputBamFilename.c_str(), seqan::OPEN_WRONLY |seqan::OPEN_CREATE);
seqan::writeHeader(fileOut, header);

// 遍历bam中每条read
while (!seqan::atEnd(inFile))
{
    seqan::readRecord(record, inFile);
    TPos queryBegin = record.beginPos+1;
    // 这里是根据read的cigar信息计算出长度,省略部分代码
    TPos queryEnd = queryBegin + getReferenceLength(record.cigar);

    TIds result;
    if (record.rID < seqan::length(intervalTrees))
        seqan::findIntervals(result, intervalTrees[record.rID], queryBegin, queryEnd);

    /*
    *result记录了与当前read有overlap的gene在数据库中的唯一ID,由于计算逻辑实现过长
    *接下来省略对locusFunction等的计算代码,result的使用简略记录下,通过迭代器访问原始gtf数据
    *TIterator it;
    *for (unsigned j = 0; j < seqan::length(result); ++j)
    *{
    *   int id = result[j];
    *   goTo(it, id);
    *   ...
    *}
    */

    // 更新read的tag信息,示例会生成tag信息: GE:Z:gene_name (中间的Z表示value为字符串)
    seqan::BamTagsDict tags(record.tags);
    seqan::setTagValue(tags, "GE", "gene_name");

    // 输出bam
    seqan::writeRecord(fileOut, record);
}
```

不同的注释逻辑自然实现不同,所以这里仅给出代码结构,更多细节要多阅读seqan库的文档,还是挺详细的

## 优化

一些预定义宏可能有加速效果

* SEQAN_ASYNC_IO=1 允许异步输入输出操作
* SEQAN_BGZF_NUM_THREADS=value 读写bam文件使用的线程数

其他的就是使用性能分析工具如valgrind,gprof等找出瓶颈并针对性优化

## 问题总结

### 编译问题

Q:error MSB8036: The Windows SDK version 8.1 was not found  
A:控制面板-应用程序-修改vs studio-勾选上通用工具中的win10SDK,重新安装

Q:No CMAKE_CXX_COMPILER could be found  
A:删掉缓存,重新编译

Q:windows下的项目配置  
A:配置属性-C/C++-语言 复合模式选择否,启用运行时类型信息选择是(/GR) OpenMP支持选择是;字符集选择多字节字符集;警告等级选择/W2;添加zlib,用于读取bam文件,注意x86和x64不要搞混

Q:预处理设置  
A:
>WIN32_WINDOWS  
SEQAN_ENABLE_DEBUG=1  
SEQAN_GLOBAL_EXCEPTION_HANDLER=1  
_WIN32_WINNT=0x0600  
WINVER=0x0600  
_SCL_SECURE_NO_WARNINGS  
_CRT_SECURE_NO_WARNINGS  
NOMINMAX  
SEQAN_HAS_EXECINFO=0  
SEQAN_HAS_OPENMP=1  
SEQAN_APP_VERSION="1.5.8"  
SEQAN_REVISION="f5f6583"  
SEQAN_DATE="2019-08-02_14:42:28_+0000"  
CMAKE_INTDIR="Debug"  

### 代码错误

Q:getValueByKey接口调用异常  
A:修改代码

```cpp
// 注释掉此接口
//template <typename TFragmentStore, typename TSpec, typename TKey>
//inline CharString
//getValueByKey(
//    Iter<TFragmentStore, AnnotationTree<TSpec> > const & it,
//    TKey const & key)
//{
//    return annotationGetValueByKey(*it.store, getAnnotation(it), key);
//}

// FragmentStore<TSpec, TConfig> & fragStore, 参数加const
template <typename TSpec, typename TConfig, typename TAnnotation, typename TKey, typename TValue>
inline bool
annotationGetValueByKey (
    TValue & value,
    FragmentStore<TSpec, TConfig> const & fragStore,
    TAnnotation const & annotation,
    TKey const & key)
```

## 参考

* [seqan官网](http://www.seqan.de/)
* [github仓库](https://github.com/seqan/seqan)
* [API文档](http://docs.seqan.de/seqan/master/)
* [Simple RNA-Seq](https://seqan.readthedocs.io/en/master/Tutorial/HowTo/UseCases/SimpleRnaSeq.html)