---
title: python-parseXML
date: 2016-03-28 22:40:33
tags: python xml
---

以前有使用过python 解析xml的内容的两种方法，先贴出来代码，具体的含义之后搞仔细了再补充上来。

xml 文件：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<collection>
<Cycle1>
	<Number>628398</Number>
	<Signal>15168.389648 19429.083984 24276.886719 18786.134766 </Signal>
	<Background>-739.025574 -691.423401 -794.166931 -1007.662659 </Background>
</Cycle1>
<Cycle2>
	<Number>482765</Number>
	<Signal>10683.573242 14735.889648 19846.058594 13917.609375 </Signal>
	<Background>-445.148132 -482.349854 -625.839417 -890.880981 </Background>
</Cycle2>
</collection>
```

使用DOM 解析xml：

``` python
# parseDOM.py

#!/usr/bin/python
#coding=utf-8

from xml.dom.minidom import parse
import xml.dom.minidom

# 使用minidom解析器打开 XML 文档
DOMTree = xml.dom.minidom.parse("test.xml")
collection = DOMTree.documentElement

trans = {'Number': 'NUMBER', 'Signal': 'SIGNAL', 'Background': 'BACKGROUND'}
resultDict = {}

for cycle in xrange(1, 3):
	cycleData = collection.getElementsByTagName("Cycle%d" % cycle)
	if not cycleData: continue
        else:
            for k in trans:
                    value = cycleData[0].getElementsByTagName(k)[0]
                    value = value.childNodes[0].data
                    value = value.strip().split()

                    resultDict.setdefault(trans[k], []).extend(map(float, value))

for k in resultDict:
	print k, resultDict[k]

# 输出
tigerose@pc ~/github/parseXml
$python parseDOM.py 
SIGNAL [15168.389648, 19429.083984, 24276.886719, 18786.134766, 10683.573242, 14735.889648, 19846.058594, 13917.609375]
NUMBER [628398.0, 482765.0]
BACKGROUND [-739.025574, -691.423401, -794.166931, -1007.662659, -445.148132, -482.349854, -625.839417, -890.880981]
```

使用 SAX解析xml

``` python
# parseSAX.py

#!/usr/bin/python
#coding=utf-8

import xml.sax

class XmlHandler( xml.sax.ContentHandler ):
   def __init__(self):
      self.CurrentData = ""
      self.Number = ""
      self.Signal = ""
      self.Background = ""

   # 元素开始事件处理
   def startElement(self, tag, attributes):
      self.CurrentData = tag
      if tag.startswith('Cycle'):
         print "*****%s*****" % tag
         #title = attributes["Number"]
         #print "Title:", title

   # 元素结束事件处理
   def endElement(self, tag):
      if self.CurrentData == "Number":
         print "NUMber:", self.Number
      elif self.CurrentData == "Signal":
         print "SIGNAL:", self.Signal
      elif self.CurrentData == "Background":
         print "BACKGROUND:", self.Background
      self.CurrentData = ""

   # 内容事件处理
   def characters(self, content):
      if self.CurrentData == "Number":
         self.Number = content
      elif self.CurrentData == "Signal":
         self.Signal = content
      elif self.CurrentData == "Background":
         self.Background = content
  
if ( __name__ == "__main__"):
   # 创建一个 XMLReader
   parser = xml.sax.make_parser()
   # turn off namepsaces
   parser.setFeature(xml.sax.handler.feature_namespaces, 0)

   # 重写 ContextHandler
   Handler = XmlHandler()
   parser.setContentHandler( Handler )
   
   parser.parse("test.xml")

# 输出
tigerose@pc ~/github/parseXml
$python parseSAX.py 
*****Cycle1*****
NUMber: 628398
SIGNAL: 15168.389648 19429.083984 24276.886719 18786.134766 
BACKGROUND: -739.025574 -691.423401 -794.166931 -1007.662659 
*****Cycle2*****
NUMber: 482765
SIGNAL: 10683.573242 14735.889648 19846.058594 13917.609375 
BACKGROUND: -445.148132 -482.349854 -625.839417 -890.880981
```



