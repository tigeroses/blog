---
title: 技巧与工具03-调用百度翻译API进行中英文翻译
date: 2016-10-26 21:30:38
tags: [自动化翻译]
category: [tools]
---

工作中有时会遇到需要中英文互相翻译的情况，词数少的话可以手动使用翻译软件进行
翻译，如果量很大，编写程序自动翻译会是个更好的选择．
<!--more-->

本篇使用python编写脚本调用百度翻译API进行自动化翻译，依次读取文本文件的每一行，
翻译之后输出到结果文件中．

## 百度翻译API
当需要进行自动化翻译的时候，首先想到谷歌翻译，毕竟是公认的翻译最准确的平台，
在网上找到脚本实验，使用的是http请求来调用[谷歌翻译]( http://translate.google.cn/ )的主页，程序填入字段从而
获取到翻译后的结果，测试发现不可行，无法抓取翻译后的内容，查看网页源代码发现
应该是谷歌将结果放到其他位置而不是当前页面;谷歌到也提供翻译API，不过收费的，
暂时不考虑．

然后自然找到了百度翻译,其翻译平台在这里:[百度翻译开放平台]( http://api.fanyi.baidu.com/api/trans/product/index ).
它的好处就是每个月翻译字数低于200万是免费的，超过了再收费，对我这种偶尔翻译
下的人来说，基本是可以免费使用了．

使用前需要在主页点击*申请接入*，进行注册，它会给APPID和密钥，这些东西是之后
调用API翻译必须要得．官方文档有详细的使用说明和示例，不多说，直接上我的脚本的代码．

```python
# translate_en2zh.py

#/usr/bin/env python
#coding=utf8
 
import sys
import httplib
import md5
import urllib
import random
import json

reload(sys)
sys.setdefaultencoding('utf-8')

def trans_line(line, fromLang, toLang):
    result = []

    # replace with your id and key
    appid = ''
    secretKey = ''

    httpClient = None
    myurl = '/api/trans/vip/translate'
    q = line
    salt = random.randint(32768, 65536)

    sign = appid+q+str(salt)+secretKey
    m1 = md5.new()
    m1.update(sign)
    sign = m1.hexdigest()
    myurl = myurl+'?appid='+appid+'&q='+urllib.quote(q)+'&from='+fromLang+'&to='+toLang+'&salt='+str(salt)+'&sign='+sign
     
    try:
        httpClient = httplib.HTTPConnection('api.fanyi.baidu.com')
        httpClient.request('GET', myurl)
     
        #response是HTTPResponse对象
        response = httpClient.getresponse()
        jsonData = json.loads(response.read())
        # print jsonData
        result.append(jsonData['trans_result'][0]['src'])
        result.append(jsonData['trans_result'][0]['dst'])
    except Exception, e:
        print e
    finally:
        if httpClient:
            httpClient.close()

    return result

if __name__ == "__main__":
    fromLang = 'en'
    toLang = 'zh'

    if len(sys.argv) != 2:
        print "Enter a file name: "
        exit()

    filename = sys.argv[1]
    tmp = filename.split(".")
    if len(tmp) != 2:
        print "Error filename"
        exit()
    out_name = tmp[0] + '_en2zh.' + tmp[1]

    translated_list = []
    with open(filename, 'r') as fh_in:
        for line in fh_in:
            line = line.strip()
            if not line: continue

            translated_list.append(trans_line(line, fromLang, toLang))

    with open(out_name, 'w') as fh_out:
        for line_result in translated_list:
            #fh_out.write(line_result[1].encode("GBK") + "\n")
            fh_out.write(line_result[1] + "\n")


```

测试文本*en.txt*如下，功率相关的英文．
```
Active power P
Reactive power Q
Apparent power S
Power factoer
```

命令行输入:`python translate_en2zh.py en.txt`,没有任何输出则运行成功．

打开新生成的*en_en2zh.txt*:
```
有功功率P
无功功率Q
视在功率S
功率因素
```

翻译完全正确．

可以在[管理控制台]( http://api.fanyi.baidu.com/api/trans/product/desktop )查看使用字符的详细情况
