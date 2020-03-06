---
title: python服务器客户端通信
date: 2016-05-03 21:37:21
tags: [python,flask]
category: [programming]
---

这里主要使用python的flask框架搭建一个简易服务器端，然后使用httplib库作为客户端与服务端进行通信，传输json数据并统计打包，网络传输，解包的时间。

## 代码

``` python
##### http_server.py

#!/usr/bin/env python
# -*- coding:utf-8 -*-

from flask import Flask
from flask.ext.restful import Resource, Api
from flask import request, make_response
import datetime
import json

app = Flask(__name__)
api = Api(app)

class data(Resource):
    def get(self):
        start_time = datetime.datetime.now()

        data = {}
        tmp_data = {}
        for i in xrange(1000):
            tmp_data[i] = {}
            tmp_data[i]["name"] = "tigerrose"
            tmp_data[i]["tag"] = "cool"
        data['data'] = tmp_data
        data = json.dumps(data)

        end_time = datetime.datetime.now()
        print 'Size: %s' % len(data)
        print 'Time: %s' % (end_time - start_time)
        
        return data
    
    def post(self):
        data = request.stream.read()
        json_data = json.loads(data)

        start_time = datetime.datetime.now()
        tmp_data = {}
        for d in json_data:
            tmp_data[d] = json_data[d]

        end_time = datetime.datetime.now()
        print 'Data Size: %s' % len(data)
        print 'Unpack Time: %s' % (end_time - start_time)
        
        return make_response('sucess')

api.add_resource(data, '/data/')

if __name__ == '__main__':
    app.run(debug=True)
```

``` python
##### http_client.py    
#!/usr/bin/env python
# -*- coding:utf-8 -*-

import httplib
import datetime
import json
import urllib
import multiprocessing

def getData():
    url = 'http://127.0.0.1:5000/data/'
    conn = httplib.HTTPConnection('127.0.0.1', port=5000)

    start_time = datetime.datetime.now()
    conn.request(method="GET", url=url)
    response = conn.getresponse()
    res = response.read()

    end_time = datetime.datetime.now()
    print 'Size: %s' % len(res)
    print 'Time: %s' % (end_time-start_time)

def sendData():
    url = 'http://127.0.0.1:5000/data/'
    conn = httplib.HTTPConnection('127.0.0.1', port=5000)

    start_time = datetime.datetime.now()

    # prepare the data
    data = {}
    for i in xrange(100000):
        data[i] = {'name': 'tigerrose'}
    data = json.dumps(data)

    data_time = datetime.datetime.now()

    conn.request("POST", url, data)
    response = conn.getresponse()

    end_time = datetime.datetime.now()
    res = response.read()

    print 'Data Size: %s' % len(data)
    print 'Pack Time: %s' % (data_time-start_time)
    print 'Transform Time: %s' % (end_time-data_time)
    print res

if __name__ == '__main__':
    # 测试 POST 方法
    sendData()
```

## 运行结果

``` bash
$python http_server.py 
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
* Restarting with stat
* Debugger is active!
* Debugger pin code: 328-155-210
```

``` bash
$python http_cilent.py 
Data Size: 3188890
Pack Time: 0:00:00.368087
Transform Time: 0:00:01.012829
sucess
```

此时，服务器端也多了几行输出

``` bash
Data Size: 3188890
Unpack Time: 0:00:00.106405
127.0.0.1 - - [03/May/2016 22:00:58] "POST http://127.0.0.1:5000/data/ HTTP/1.1" 200 -
```

## 结果说明

首先运行http_server, 开启服务，然后运行http_client, 运行客户端，发送数据给服务端并获取返回值，可以看到结果显示了数据打包，解包和网络传输以及数据大小的具体数值。

## 原理

1 服务端的搭建。服务端用的是flask restful做的简易app， 其中，api.add_resource(data, '/data/') 就是将继承自Resource并实现来get或者post方法的类映射到`http://host:port/data/`这个网络资源上。
  app.run()方法是开启服务，debug参数为True代表是debug模式，好处是输出一些调试信息，并且当你修改http_server代码后它会自动重启服务，但是注意不要在实际项目中使用，会有安全隐患，默认的host地址是127.0.0.1, 端口是5000，可以传参数修改。

2 客户端搭建。 客户端使用httplib的HTTPConnection进行创建连接, request函数发送POST请求，如果是get请求将method改成GET即可。

3 数据传输。 我个人理解的数据传输就是发送POST请求到获取response返回结果的时间，而打包时间是生成json数据串的时间，解包是将传输的json数据读取到内存的过程。使用datetime.datetime.now()来获取当前时间，两个时间相减即是一段python代码所运行的时间。

## 参考
http://dormousehole.readthedocs.io/en/latest/
http://www.pythondoc.com/Flask-RESTful/quickstart.html
http://www.jb51.net/article/66763.htm
http://www.tuicool.com/articles/J3maU3F
http://www.01happy.com/python-httplib-get-and-post/

