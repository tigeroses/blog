---
title: 技巧与工具04-python控制鼠标自动化点击脚本
date: 2017-06-24 17:04:28
tags: [python,pyautogui]
category: [tools]
---
## python控制鼠标自动化点击脚本

### 事情起因
今天是DNF九周年活动，出了新职业圣职者，为了快速升级需要获取疲劳药，可以用活动送的
黑钻来抽奖，每抽一次需要分别点击三次，而我能抽奖500多次，所以不想手动来点击，刚好
前几天在微信公众号看了一个简短的文章，是关于python的pyautogui库可以自动化键盘和鼠标
的操作，因此就开始动手做；台式机以前新装的系统，因此需要下载python。

### 环境搭建

- 下载 **python2.7** 并安装
- 配置python环境变量，包括python目录和scripts目录(为了pip)
- **pip install pyautogui** 安装这个控制鼠标和键盘的库

### 熟悉pyautogui库
```python
    import pyautogui as pg #导入库 
    pg.size() #返回窗口大小，比如(1920,1080)
    pg.position() #返回鼠标当前位置
    pg.moveTo(100, 100) #移动鼠标
    pg.click(100, 100) #移动鼠标并单击
    pg.press('enter') #按下回车键
    pg.keyDown('esc') #按下退出键
    pg.keyUp('esc') #松开退出键
    pg.typewrite('hello') #文本输入
    pg.dragTo(100, 100) #鼠标拖拽
```

### 脚本编写
脚本的逻辑很简单，首先10秒的时间用来让我放置鼠标到起始的位置，也就是黑钻售货机,
进行第一次点击;之后会进入循环，即每次点击三次，分别是按钮“启动”，“停止”，“确定”，
其中三次的位置均不同，但是dnf会自动将鼠标移动到下一个需要点击的位置，为了给dnf
这个移动的时间，中间有sleep操作。

最终抽奖完成，但是程序会一直运行下去，这时需要将鼠标移动到左上角，这样程序会抛出
异常，从而捕获异常，终止程序；至于为什么不用click()函数，而是用dragTo()这个鼠标
拖拽函数，下面会提到。

```python
#/usr/bin/env python
# coding: utf-8
import sys
from time import sleep

import pyautogui as pg
import pytweening

def my_click():
    l_pos, r_pos =  pg.position()
    pg.dragTo(l_pos+1, r_pos+1, duration = 0.5, button='left')
    sleep(0.1)
            
def main():
    # get current position
    print "Place your mouse in the starting position within then seconds."
    sleep(10)

    try:
        my_click()
        times = 1
        while 1:
            my_click()
            my_click()
            sleep(0.5)
            my_click()
            print "click %03d" % times
            times += 1
    except pg.FailSafeException as e:
        print "Error: %s" % e
        print "Over"
    except Exception as e:
        print "Error: %s" % e
        print "Over"
                
if __name__ == "__main__":
    main()
```
### 问题总结
- 经过测试，使用pyautogui可进行按键和文本输入，但是无法进行鼠标的单击,即click()在dnf的窗口无效
- 怀疑是游戏方有监控鼠标的滑行轨迹，如果是直线的就进行过滤，这应该算是防止作弊的一种手段
- 还好试了dragTo()，先按下鼠标再松开是可以，否则要考虑使用非直线来进行鼠标的移动，这可能要用到
    其他的库，pyautogui中没有找到对应的方法

### 参考文档
- [PyAutoGUI——让所有GUI都自动化](http://www.tuicool.com/articles/U73A7zz)
- 微信公众号 **Python程序员**
