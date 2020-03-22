---
title: 工作中遇到的压缩方式总结
date: 2020-03-22 21:00:52
tags: 压缩 zlib igzip
---

本文总结工作中使用过的数据压缩方法,主要有zlib,qatzip,igzip等  
最后还进行了针对大规模数据多线程解压缩加速的分析

## zlib库

zlib是用于数据压缩的函数库,使用deflate算法  
**deflate算法**是同时使用了*LZ77算法*和*霍夫曼编码*的一个无损压缩算法

主要函数有:
* `int compress (Bytef *dest, uLongf *destLen, const Bytef *source, uLong sourceLen);`  
    压缩方法,将源缓冲中的数据压缩并放入目的缓冲区  
    注意目的缓冲区的大小有可能比压缩前还要大,因此destLen要留够空间,至少比sourceLen加12字节之后还大0.1%  
    返回Z_OK表示成功;Z_MEM_ERROR表示没有足够内存;Z_BUF_ERROR表示目的缓冲区不够大
* `int compress2 (Bytef *dest, uLongf *destLen,const Bytef *source, uLong sourceLen,int level);`  
    功能与compress函数一样,增加了level参数,范围0-9,从0-9速度变慢,但压缩率提高,设置0表示不压缩,Z_DEFAULT_COMPRESSION 表示设置一个适中的参数  
    返回值与compress相同,多出Z_STREAM_ERROR 表示level参数无效
* `int uncompress (Bytef *dest, uLongf *destLen,const Bytef *source, uLong sourceLen);`  
    解压缩  
    返回值与compress相同,多出一个Z_DATA_ERROR 表示输入数据被破坏
* `deflateInit() + deflate() + deflateEnd()`  
    三个函数结合完成compress功能,参考zlib仓库example.c compress.c
* `inflateInit() + inflate() + inflateEnd()`  
    三个函数完成uncompress功能
* gz开头的函数,用来操作gz文件,类似stdio调用,如果gzopen,gzwrite等

简单的压缩示例代码:
```cpp
#include <zlib.h>

int gzCompress(Bytef *data, uLong ndata, Bytef *zdata, uLong *nzdata, int level)
{
	// error code 
	// Z_OK if success, Z_MEM_ERROR if there was not enough memory, 
	// Z_BUF_ERROR:-5 if there was not enough room in the output buffer, 
	// Z_STREAM_ERROR if the level parameter is invalid.
	z_stream c_stream;
	int err = 0;
	if (data && ndata > 0) 
	{
		c_stream.zalloc = NULL;
		c_stream.zfree = NULL;
		c_stream.opaque = NULL;
		if (deflateInit2(&c_stream, level, Z_DEFLATED,
			MAX_WBITS + 16, 8, Z_DEFAULT_STRATEGY) != Z_OK) return -1;
		c_stream.next_in = data;
		c_stream.avail_in = ndata;
		c_stream.next_out = zdata;
		c_stream.avail_out = *nzdata;
		while (c_stream.avail_in != 0 && c_stream.total_out < *nzdata) 
		{
			try
			{
				if ((err = deflate(&c_stream, Z_BLOCK)) != Z_OK)
					return err;
			}
			catch (...)
			{
				cout << "deflate error: " << endl;
				return -1;
			}
		}
		if (c_stream.avail_in != 0) 
			return c_stream.avail_in;
		for (;;) 
		{
			if ((err = deflate(&c_stream, Z_FINISH)) == Z_STREAM_END) break;
			if (err != Z_OK) return err;
		}
		if (deflateEnd(&c_stream) != Z_OK) return -1;
		*nzdata = c_stream.total_out;
		return 0;
	}
	return -1;
}
```

关于压缩等级,从0-9,速度越来越慢,随之而来的是更低的压缩率

压缩文件是二进制的,由三部分组成
1. 头信息
2. 数据主体 
3. 校验

以下为标准格式的简要说明,详细解释可以看参考文档:
![](/images/compress_method/header.png)

## qatzip库

通过硬件加速的方式进行压缩,即需要插入一张单独的intel的QAT卡;好处显而易见,正常压缩是消耗CPU资源,用另一张卡单独进行压缩,空闲出CPU资源可以进行其他计算,提高整体效率,缺点就是费钱,并占用一个PCIE插槽位置  
另外只能运行于linux系统,不支持windows

[qatzip_github代码仓库](https://github.com/intel/QATzip.git)

简单的压缩代码:
```cpp
#ifdef COMPILE_QAT
#include <cpa.h>
#include <cpa_dc.h>

#include <qatzip.h>
#include <qatzip_internal.h>
#include <qz_utils.h>
extern QzSessionParams_T g_params_th;
#endif

int qzipCompress(Bytef *data, uLong ndata, Bytef *zdata, uLong *nzdata, int level)
{
#ifdef COMPILE_QAT
    QzSession_T session;
    int rc;

    rc = qzInit(&session, g_params_th.sw_backup);
    if (rc != QZ_OK && rc != QZ_DUPLICATE)
    {
        cout<<"qzInit failed, rc:"<<rc<<endl;
        return -1;
    }

    rc = qzSetupSession(&session, &g_params_th);
    if (rc != QZ_OK && rc != QZ_NO_INST_ATTACH)
    {
        cout<<"qzSetupSession failed, rc:"<<rc<<endl;
        return -1;
    }

    rc = qzCompress(&session, data, (uint32_t *)&ndata, zdata, (uint32_t *)nzdata, 1);
    if (rc != QZ_OK)
    {
        cout<<"qzCompress failed, rc:"<<rc<<endl;
        return -2;
    }

    qzTeardownSession(&session);
    return 0;
#else
    return gzCompress(data, ndata, zdata, nzdata, level);
#e
```


## igzip库

intel工程师使用指令集优化zlib,针对genomic data比如**bam sam**数据,在几乎不降低压缩率的情况下,速度提升约4倍  
[igzip_github代码仓库](https://github.com/intel/isa-l.git)  
igzip的代码和isa-l代码仓库在一起

igzip使用代码示例:
```cpp
#include "igzip_lib.h"
int level_size_buf[10] = {
#ifdef ISAL_DEF_LVL0_DEFAULT
	ISAL_DEF_LVL0_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL1_DEFAULT
	ISAL_DEF_LVL1_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL2_DEFAULT
	ISAL_DEF_LVL2_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL3_DEFAULT
	ISAL_DEF_LVL3_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL4_DEFAULT
	ISAL_DEF_LVL4_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL5_DEFAULT
	ISAL_DEF_LVL5_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL6_DEFAULT
	ISAL_DEF_LVL6_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL7_DEFAULT
	ISAL_DEF_LVL7_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL8_DEFAULT
	ISAL_DEF_LVL8_DEFAULT,
#else
	0,
#endif
#ifdef ISAL_DEF_LVL9_DEFAULT
	ISAL_DEF_LVL9_DEFAULT,
#else
	0,
#endif

typedef unsigned long mylong;
typedef unsigned char mychar;

int igzipCompress(mychar* source, mylong source_len, mychar* dest, mylong* dest_len, 
	int level)
{
	struct isal_zstream stream; /* Holds stream information */
	struct isal_gzip_header gz_hdr;

	isal_gzip_header_init(&gz_hdr); /* Set gzip header default values */

	isal_deflate_init(&stream); /* Initialize compression stream data structure */

	int level_size = level_size_buf[level];
	unsigned char * level_buf = NULL;
	level_buf = (unsigned char*)malloc(level_size);

	stream.avail_in = 0; // Number of bytes available at next_in.
	stream.flush = NO_FLUSH;    // Flush type can be NO_FLUSH,SYNC_FLUSH or FULL_FLUSH.
	stream.level = level;   // Compression level to use.
	stream.level_buf = level_buf; // User allocated buffer required for different compression levels.
	stream.level_buf_size = level_size;   // Size of level_buf.
	stream.gzip_flag = IGZIP_GZIP_NO_HDR;   // Indicate if gzip compression is to be performed.
	stream.next_out = dest; // Next output byte.
	stream.avail_out = *dest_len;   // Number of bytes avaliable at next_out.

	isal_write_gzip_header(&stream, &gz_hdr);   /* Write gzip header to output stream. */

	stream.next_in = source;
	stream.avail_in = source_len;
	stream.end_of_stream = 1;

	int ret = isal_deflate(&stream);
	if (ret != ISAL_DECOMP_OK) {
		printf("igzip: Error encountered while compressing file\n");
	}
	if (level_buf != NULL)
		free(level_buf);
	*dest_len = stream.next_out - dest;
	return ret;
}

```

## bgzip库及多线程解压缩

bgzip:Block compression/decompression utility  
用于bam/sam文件的格式,核心是将压缩数据分块(64KB),从而通过索引可以快速查询数据  

注:bam/sam文件是高通量测序的标准格式文件,存储内容为fastq文件与参考基因组reference进行mapping之后的数据;其中sam为文本格式,bam为二进制格式,两者可以通过*samtools*工具相互转换;bam文件可以通过建立index,快速定位数据位置,从而加速访问

考虑这样一种情况,有一千个block的数据需要压缩并存放在一个文件中,这个文件可能很大,几百GB;假如我只想要分析某几个block的数据,传统的压缩方式需要将整个文件全部解压之后才能获取想要的数据,效率很低

而通过自定义压缩block的head信息,使用其中的**extra filed** 和 **comment** 字段就可以实现index功能,步骤如下:
1. 压缩前,首先添加字段:comment添加block的ID,extra field添加压缩前和后的bytes大小;以zlib压缩举例
```cpp
const uint8_t EXTRA_LEN = 8;
const uint8_t EXTRA_BUF_LEN = 4;
const uint8_t SI1 = 100;
const uint8_t SI2 = 101;

gz_header header;
// 添加comment信息,如指定当前block的ID,如1,2,3等
// 用于后续快速获取想要的block数据
header.comment = (Bytef *)&comment[0];
header.comm_max = comment.size() + 1;
// 按照标准格式指定extra field的长度信息
header.extra_len = EXTRA_LEN;
header.extra_max = EXTRA_BUF_LEN;
uint8_t extra_buf[EXTRA_LEN] = { 0 };
// SI1 SI2为自定义字段,用来标识我们自定义的头文件格式
extra_buf[0] = SI1;
extra_buf[1] = SI2;
extra_buf[2] = EXTRA_BUF_LEN;
extra_buf[3] = 0;
header.extra = extra_buf;
err = deflateSetHeader(&c_stream, &header);
```

2. 压缩后,更新extra filed中压缩前后数据长度
```cpp
*nzdata = c_stream.total_out;

mylong dest_size = *nzdata; // 压缩后大小,必须压缩完之后才能获取
mylong raw_size = ndata; // 压缩前大小,是输入参数,我们是知道的
// 将两个长度按顺序写入字节流
memcpy(zdata + 16, &dest_size, sizeof(dest_size));
memcpy(zdata + 24, &raw_size, sizeof(raw_size));
```

3. 解压缩的时候,首先找到第一个block,读入头信息,获取当前block的标识ID,如果是想要的数据,则通过extra field获取数据长度,按照长度直接读取即可,然后跳到下一个block  
   因为对于无用的block数据,我们只要解析头信息,并**根据长度进行偏移**即可,所以遍历速度会很快  
   然后还可以通过**多线程**进行解压缩,主线程进行block的遍历,如果遇到目标数据,则从线程池中拿一个线程处理当前block  
   如果不需要解压缩,只是从1000个block中采样10个block进行后续的快速分析,则直接将10个block的二进制数据连续输出到磁盘文件即可,多个block可以直接cat到一起而不影响解压缩

## 参考文档

[zlib压缩数据](https://www.cnblogs.com/zhuyf87/archive/2013/02/21/2920522.html)  
[zlib官网](http://zlib.net/)  
[High Performance DEFLATE Compression with Optimizations for Genomic Data Sets](https://software.intel.com/en-us/articles/igzip-a-high-performance-deflate-compressor-with-optimizations-for-genomic-data)  
[GZIP file format specification version 4.3](http://www.zlib.org/rfc-gzip.html)  
[GZIP文件格式简介](https://zlib.net/manual.html)  
[zlib 1.2.11 Manual](http://www.htslib.org/doc/bgzip.html)