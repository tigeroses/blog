---
title: 基因注释
date: 2020-04-30 20:49:05
tags: 基因注释
---

# 基因注释

记录下自己对RNA-seq基因注释的学习,并对Drop-seq软件包中的注释模块进行代码研读

## 什么是基因注释

一句话概况注释:*找到与reads有overlap的基因片段,并进行标记*

这里reads指bam文件中的每一行数据,即测序下机文件fastq与参考基因组进行比对之后生成的数据,其中记录了每条read在参考基因组中的位置,有起始位置和终止位置,表示一段区间

基因注释文件记录了每个基因片段在参考基因组上的位置,也是一段区间,因此与bam文件结合,通过find overlapping我们可以查找到每条read属于哪个基因片段,将其标记在bam格式的tags中,这对后续的生信分析是有帮助的

## 基因注释文件

GTF/GFF格式是基因注释的常用格式  
GTF是Gene Transfer Format的缩写,其文件由九列数据组成,以tab分割,示例如下:

| seq_id     | source   | type | start                | end      | score                     | strand               | phase                                | attributes                                                         |
| ---------- | -------- | ---- | -------------------- | -------- | ------------------------- | -------------------- | ------------------------------------ | ------------------------------------------------------------------ |
| chr1       | HAVANA   | exon | 11869                | 12227    | .                         | +                    | .                                    | gene_id "ENSG00000223972.5"; transcript_id "ENST00000456328.2";... |
| 染色体编号 | 注释来源 | 类型 | 在参考序列的起始位置 | 终止位置 | 得分,说明注释信息的可能性 | 位于参考序列的正负链 | 仅对类型为CDS有效,表示起始编码的位置 | 包含众多属性的列表                                                 |

虽然数据有九列之多,但并不是所有都会用到,常用的有:

1. seq_id. 要查找overlap,首先得是同一条染色体
2. type. 有多种类型,如gene/transcript/exon/CDS/UTR等,它们之间有层级关系,一般gtf文件中多行数据对应一条基因的完整信息,以type为gene的行为起始;每条gene可以表示为树状结构,gene为根节点,第二层为transcript,第三层为诸如exon CDS等
3. start/end. 根据起始终止位置可以建立interval,这是find overlapping的基础
4. strand. 正负链可以作为过滤条件,假如一条read与多个基因有overlap,可以根据方向是否相同过滤掉部分基因
5. attributes. 一些列键值对属性,常用的信息包括名称,id之类

## 注释流程分析

流程可分为三步:

1. **读入gtf文件**. 从磁盘将gtf文件加载进内存,并提取需要的信息,毕竟gtf有许多信息是我们不需要的
2. **建立区间树**. 即interval tree,使用区间树是为了高效查询,为了达到最佳性能,一般使用基于红黑树的区间树实现,因为红黑树是平衡树,查找时间复杂度O(lgN),不会出现退化成链表的最坏情况
3. **查找区间并注释**. 遍历bam文件中每条read,根据其在参考序列中的位置构建interval,与前面建立的interval tree进行overlap的查找,找到之后,进行一些逻辑计算,并更新read的tags,输出到bam

## Drop-seq代码研读

Drop-seq是使用java开发的程序包,其中的*TagReadWithGeneExonFunction* 模块实现了添加注释的功能

### 主流程

主流程核心代码为:

```java
// 加载基因注释文件,并构建区间树
final OverlapDetector<Gene> geneOverlapDetector = GeneAnnotationReader.loadAnnotationsFile(ANNOTATIONS_FILE, bamDict);
// 打开输出bam文件
SAMFileWriter writer= new SAMFileWriterFactory().makeSAMOrBAMWriter(header, true, OUTPUT);
// 遍历输入bam中的每条read
for (SAMRecord r: inputSam) {
    // 对完成比对的read,进行find overlapping操作并添加注释
    if (!r.getReadUnmappedFlag())
        r=setAnnotations(r, geneOverlapDetector);
    // 输出注释后的read
    writer.addAlignment(r);
}
```

其结果是根据overlap的genes信息,添加三个Tag,示例:
> GE:Z:WASH7P    XF:Z:CODING        GS:Z:-

* GE为gene name  
* XF为locus function  
* GS为正负链

### 加载gtf并构建interval tree

核心代码:

```java
// 用来解析gtf每行数据,提取需要的字段
final FilteringGTFParser parser = new FilteringGTFParser(gtfFlatFile);
// gene name相同的gtf行,代表它们是一个gene的数据,使用它们构建GeneFromGTF数据类型,
// 此类型继承自Interval类型
final GeneFromGTFBuilder geneBuilder = new GeneFromGTFBuilder(parser);
// 初始化interval tree
final OverlapDetector<GeneFromGTF> overlapDetector = new OverlapDetector<>(0, 0);
while (geneBuilder.hasNext()){
        // 将每条gene添加到interval tree,其内部按照chromosome进行分类
        GeneFromGTF gene = geneBuilder.next();
        overlapDetector.addLhs(gene, gene);
    }
```

代码实现使用了迭代器,减少内存的消耗

将gtf每行数据以gene_name为key,放入`map<gene_name, List<GTFRecord>>`中,这样就将每条gene的数据分类好了  
geneBuilder 是个`iter<List<GTFRecord>>`,迭代时,对每个gene将其数据`List<GTFRecord>` 按gene_version分类成`map<gene_version, List<GTFRecord>>`,对key也就是所有的gene_version进行排序,取最大的gene_version对应的`List<GTFRecord>`来构建GeneFromGTF(对部分gtf文件它们的gene_version是null,则对gene_version分类和没做一样)

重点是构建GeneFromGTF类实例:  
`List<GTFRecord>`转GeneFromGTF调用接口`makeGeneFromMultiVersionGTFRecords()`

1. 使用list中第一条GTFRecord的信息初始化GeneFromGTF(因为第一条的类型永远是gene),只有start end属性是取得list中所有数据的最小start,最大end  
2. 进行一致性检查. 检查list中所有数据,如正反链必须都一致,chr一致等,否则抛出异常
3. 将所有的非gene数据进行统计处理,更新GeneFromGTF成员变量`Map<String, TranscriptFromGTF> transcripts`, TranscriptFromGTF类型包括transcript, CDS, coding等相关信息

最后调用*htsjdk*库的`OverlapDetector.addLhs()`将GeneFromGTF作为节点加入线段树中

### 注释逻辑

核心代码:

```java
public SAMRecord setAnnotations (final SAMRecord r, final OverlapDetector<Gene> geneOverlapDetector) {
        Map<Gene, LocusFunction> map = AnnotationUtils.getInstance().getLocusFunctionForReadByGene(r, geneOverlapDetector);
        Set<Gene> exonsForRead = AnnotationUtils.getInstance().getConsistentExons (r, map.keySet(), ALLOW_MULTI_GENE_READS);

        List<Gene> genes = new ArrayList<>();

        for (Gene g: exonsForRead) {
            LocusFunction f = map.get(g);
            if (f==LocusFunction.CODING || f==LocusFunction.UTR)
                genes.add(g);
        }

        List<LocusFunction> allPassingFunctions = new ArrayList<>();
        if (USE_STRAND_INFO) {
            // constrain gene exons to read strand.
            genes = getGenesConsistentWithReadStrand(genes, r);
            // only retain functional map entries that are on the correct strand.
            for (Gene g: map.keySet()) {
                boolean strandCheck=readAnnotationMatchStrand(g, r);
                if (strandCheck) allPassingFunctions.add(map.get(g));
            }
        } else
            allPassingFunctions=new ArrayList<>(map.values());

        // if strand tag is used, only add locus function values for passing genes.
        for (Gene g: genes)
            allPassingFunctions.add(map.get(g));

        LocusFunction f = AnnotationUtils.getInstance().getLocusFunction(allPassingFunctions, false);

        if (genes.size()>1 && this.ALLOW_MULTI_GENE_READS==false)
            log.error("There should only be 1 gene assigned to a read for DGE purposes.");

        String finalGeneName = getCompoundGeneName(genes);
        String finalGeneStrand = getCompoundStrand(genes);

        if (f!=null)
            r.setAttribute(this.FUNCTION_TAG, f.toString());
        if (finalGeneName!=null && finalGeneStrand!=null) {
            r.setAttribute(this.TAG, finalGeneName);
            r.setAttribute(this.STRAND_TAG, finalGeneStrand);
        } else {
            r.setAttribute(this.TAG, null);
            r.setAttribute(this.STRAND_TAG, null);
        }
        return (r);
    }
```

概况一下注释逻辑:对read构建interval,查找overlap的所有基因,计算三个tag: locusFunction, geneName, geneStrand,并更新read  

对每个gene计算locusFunction的核心代码:

```java
private LocusFunction getLocusFunctionForRead (final SAMRecord rec, final Gene g) {
    List<AlignmentBlock> alignmentBlocks = rec.getAlignmentBlocks();

    LocusFunction [] blockSummaryFunction = new LocusFunction[alignmentBlocks.size()];
    Set<Gene> temp = new HashSet<>();
    temp.add(g);

    for (int i=0; i<alignmentBlocks.size(); i++) {
        AlignmentBlock alignmentBlock =alignmentBlocks.get(i);

        LocusFunction [] blockFunctions=getLocusFunctionsByBlock(alignmentBlock, temp);
        LocusFunction blockFunction = getLocusFunction(blockFunctions, false);
        blockSummaryFunction[i]=blockFunction;
    }

    LocusFunction readFunction = getLocusFunction(blockSummaryFunction, false);
    return readFunction;
}
```

计算过程:

1. 对read每个位点都计算一个LocusFunction(计算过程具体为是否与exons有overlap)
2. 将一条alignmentBlocks中所有位点的LocusFunctions合并为一个LocusFunction
3. 将所有alignmentBlocks的LocusFunctions合并为一个LocusFunction

每次合并都是取最大的LocusFunction,其是一个枚举变量,由小到大为:
`INTERGENIC, INTRONIC, UTR, CODING, RIBOSOMAL`

### 输出统计信息

| 统计项                    | 解释                                          |
| ------------------------- | --------------------------------------------- |
| AMBIGUOUS_READS_REJECTED  | 有多条同方向的overlaped gene                  |
| READ_AMBIGUOUS_GENE_FIXED | overlaped gene中同向的只有一条,但是还有反向的 |
| READS_RIGHT_STRAND        | 只有一条overlaped gene,并且与read同向         |
| READS_WRONG_STRAND        | overlaped gene没有同向,只有反向               |
| TOTAL_READS               | 处理的总reads数                               |

## 参考

* [GTF基因注释文件详解](https://blog.csdn.net/sinat_38163598/article/details/72851239)
* [液滴微流控获取单细胞及Drop-seq_tools分析流程](https://www.jianshu.com/p/0800a07cfa37)
* [Drop-seq_github](https://github.com/broadinstitute/Drop-seq)
