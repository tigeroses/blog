---
title: python-pil
date: 2016-03-28 22:00:18
tags: python pil
---

工作中接触到图像，需要提取图片的像素值，python的pil库可以很方便的处理图片。

## 常用方法

这里总结的内容来自网络，加上自己的一点修改。

1 导入需要的图像库：
import Image

2 图片的新建与存取：
新建图片
Image.new(mode,size)
Image.new(mode,size,color)
如：newImg = Image.new("RGBA",(640,480),(0,255,0))

读取图片
im=Image.open('/home/Picture/test.jpg')

保存图片
im.save("save.gif","GIF") #保存图像为gif格式

显示图片
im.show()

3 图片信息
im.format, im.size, im.mode

4 操作像素点
获取某个像素位置的值：
im.getpixel((4,4))

写某个像素位置的值：
img.putpixel((4,4),(255,0,0))

im.point(function) #,这个function接受一个参数，且对图片中的每一个点执行这个函数
比如：out=im.point(lambda i:i*1.5)#对每个点进行50%的加强

5 变换图片
两张图片相加：
Image.blend(img1,img2,alpha) # 这里alpha表示img1和img2的比例参数

图片裁剪：
box=(100,100,500,500) #设置要裁剪的区域
region=im.crop(box) #此时，region是一个新的图像对象。

图像黏贴（合并）
im.paste(region,box)#粘贴box大小的region到原先的图片对象中。

通道分离：
r,g,b=im.split()#分割成三个通道，此时r,g,b分别为三个图像对象。

通道合并：
im=Image.merge("RGB",(b,g,r))#将b,r两个通道进行翻转。

改变图像的大小：
out=img.resize((128,128))#resize成128*128像素大小

旋转图像：
out=img.rotate(45) #逆时针旋转45度
region = region.transpose(Image.ROTATE_180）

图像转换：
out = im.transpose(Image.FLIP_LEFT_RIGHT) #左右对换。
out = im.transpose(Image.FLIP_TOP_BOTTOM) #上下对换

图像类型转换：
im=im.convert("RGBA")

## 使用示例

脚本主要是搜索目录下的所有图片，然后对每一张图片提取像素最高的前15%的像素点的平均值

``` python
import sys, os
reload(sys)
sys.setdefaultencoding('utf8')

from PIL import Image
import numpy as np
import glob

def extractIntensity(filename):
    if not os.path.exists(filename):
        return 0

    image = Image.open(filename)
    image_array = image.load()

    width = image.size[0]
    height = image.size[1]

    lst = []
    for h in xrange(height):
        for w in xrange(width):
            #lst.append(image.getpixel((w, h)))
            lst.append(image_array[w, h])

    lst = np.array(lst)
    
    threshold = np.percentile(lst, 85)
    return np.average(lst[lst >= threshold])

def processAllImage(imageAddr, outfile):
    fileList = glob.glob(os.path.join(imageAddr, '*.tif'))
    fileList.sort()

    fhOut = open(outfile, 'w')
    for filename in fileList:
        name = os.path.basename(filename)
        name = name.split('-')[0]
        fhOut.write(name+',')
        fhOut.write(str(round(extractIntensity(filename), 2))+'\n')

    fhOut.close()

processAllImage(imageAddr, outfile)
```
