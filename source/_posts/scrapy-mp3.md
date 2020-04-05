---
title: 爬取英文演讲资源
date: 2020-04-05 14:55:51
tags: 爬虫 python
---

记录下使用python爬取网页并下载资源的过程.

## 动机
前段时间制定计划,每天上下班路上听点英语演讲音频练练听力,用的手机App是喜马拉雅,上面资源很丰富,但是有两个问题,一是**有广告**,想想你快睡着的时候突然来15秒字正腔圆的广告是什么感觉,二是**费流量**,我都是在线听的.  

因此考虑在PC上提前下载好部分音频,导出到手机,再切换到一个精简去广告的手机App来听,岂不美哉.  

学习英语的网站有不少,也可以提供下载,但一个一个右键另存为就不符合咱程序员的身份了,因此爬虫搞起!

## 基础知识
爬取之前,复习下需要的知识,当然这次任务很简单,这里只是总结下.  

1. python基础. 如文件存取,正则表达式re,多进程multiprocessing
2. html网页结构. 如常见的标签tag,CSS中的class
3. 爬虫相关的库.
   * urllib. 提供接口来打开网页,下载资源
   * BeautifulSoup. 解析网页,提取信息

缺少哪个py库,用`pip install xx` 来安装

## 分析与设计

### 分析过程
人工打开几个目标网页,查看网页源代码来分析下规律,即如何通过主网页,一步步跳转到最终的资源链接.  

打开主页,上面显示几十个链接,每一个链接分别是一个具体的演讲页面,其中一个表示如下:
```html
<td align="center" class="titlepic">
    <a href="/tingli/speech/mxlzyj/326635.html">
        <img src="/d/file/201906/6fffc975854bf3f3136637cd69cc6397.jpg" alt="库克杜兰大学演讲：勇于尝试 敢于做先行者(全文)" width="145" height="125" border="0" />
    </a>
</td>
```
因此只要匹配到align属性为'center',class属性为['titlepic']的td标签,获取第一个href即是一个演讲的链接地址  
这里要注意给出的链接是需要补齐前缀的

针对每一个具体的演讲的网页,基本都提供了一个音频的播放器

![](/images/scrapy_mp3/speech_player.png)

只要点击下载图标按钮,就会切换到另一个网页,内容为

![](/images/scrapy_mp3/download_icon.png)

这里两个图标分别对应mp3和lrc歌词的资源地址

分析音频播放器下载按钮的链接,不出意料,是一个js函数,如下:
```js
<SCRIPT type=text/javascript>
$(function(){
    $(".jp-title").hover(
            function(){
                $(this).addClass("jp-title-hover");
            },function(){
                $(this).removeClass("jp-title-hover");
            });
    $(".jp-title").click(function(){
        $(".help").slideToggle();
    });
    $(".jp-download").click(function(){
        window.open('/e/action/down.php?classid=10467&id=326635&mp3=http://mp3.en8848.com/speech/2019tim-cook-tulane.mp3') ;
    });
    $(".anniu").click(function(){
        $(".download").hide();
    });
    $("a[id^='jplayer_tc_']").click(function(){location.href=$(this).attr('href')});
});
</SCRIPT>
```
重点就是*window.open* 后的内容,指向最终下载页面的链接.  
这里可以通过正则表达式来解析链接地址

分析最终页面,发现内容如下:
```html
<a id="dload" target="_blank" href="http://mp3.en8848.com/speech/2019tim-cook-tulane.mp3" class="download"></a>
<a id="dloadword" href="http://mp3.en8848.com/speech/2019tim-cook-tulane.lrc" class="download"></a>
```

即mp3资源链接即是从播放器下载图标中提取出来的链接中的 **mp3=xxx**的地址  
lrc歌词改下后缀即可

### 提炼总结
1. 根据提供的主页,通过**特定的td标签**解析出来每一个演讲的链接,即是一个单独的任务
2. 对每个任务,解析js中**window.open**后跟的链接,即是最终的资源所在;分别下载mp3和lrc即可

### 伪码

```text
main_url = "xxx.html"
for td_tag in main_url:
    check if td_tag is valid
    get speech_url from td_tag
    extract rescource_url from speech_url
    download resource_url+'.mp3' and resource_url+'.lrc'
```

## 代码实现

### 代码

```py
#-- codeding:utf-8 --
import os
import urllib
import re
import multiprocessing as mp
from bs4 import BeautifulSoup

def crawl(url, dest_dir):
    speech = urllib.urlopen(url).read().decode("utf-8")
    speech_soup = BeautifulSoup(speech, "lxml")
    # Get speech name.
    speech_name = ""
    for div_tag in speech_soup.find_all('div'):
        if 'class' in div_tag.attrs and div_tag['class'] == ["sean_title"]:
            speech_name = div_tag.get_text()
            break
    if speech_name == "": return
    # Remove extra spaces.
    speech_name = speech_name.replace(" ", "")
    for script_tag in speech_soup.find_all('script'):
        if 'type' in script_tag.attrs and script_tag['type'] == "text/javascript":
            p = re.compile(r'mp3=(.*).mp3', re.DOTALL)
            #print script_tag.get_text()
            match = p.findall(script_tag.get_text())
            if match:
                urllib.urlretrieve(match[0]+'.mp3', dest_dir+speech_name+'.mp3')
                urllib.urlretrieve(match[0]+'.lrc', dest_dir+speech_name+'.lrc')
                break

def scrapy_map3():
    origin_url = "http://www.en8848.com.cn/tingli/speech/mxlzyj/index.html"
    prefix_url = "http://www.en8848.com.cn"
    l = urllib.urlopen(origin_url).read().decode("utf-8")
    soup = BeautifulSoup(l, "lxml")
    dest_dir = './speech/'
    if not os.path.exists(dest_dir): os.mkdir(dest_dir)
    # Format: <td align="center"><a href=xxx></td>
    pool = mp.Pool(4)
    for td_tag in soup.find_all('td'):
        if 'align' not in td_tag.attrs or 'class' not in td_tag.attrs: continue
        if td_tag['align'] == 'center' and td_tag['class'] == ['titlepic']:
            url = prefix_url + td_tag.a.get("href")
            pool.apply_async(crawl, args=(url,dest_dir))
    pool.close()
    pool.join()

if __name__ == "__main__":
    scrapy_map3()
```

### 分析

代码实现是在设计的伪码基础上填充了细节,诸如具体的判断,以及文件名的获取等未提到的细节

考虑到网页获取,文本解析,资源下载速度较慢,而每一个演讲都是独立的,可以使用多进程进行加速

除了多进程,还有异步IO,协程等方式可以加速

## 参考

* [小e英语_英语演讲](http://www.en8848.com.cn/tingli/speech/mxlzyj/index.html)
* [莫烦python_爬虫基础](https://morvanzhou.github.io/tutorials/data-manipulation/scraping/)
* [BeautifulSoup4.2.0中文文档](https://www.crummy.com/software/BeautifulSoup/bs4/doc/index.zh.html)


