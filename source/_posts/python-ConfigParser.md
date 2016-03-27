---
title: python-ConfigParser
date: 2015-06-14 20:33:43
tags: python
---

几乎所有的应用程序真正运行起来的时候，都会读取一个或多个配置文件。
配置文件的作用是：用户不需要修改代码，就可以改变应用程序的行为，让它更好的为应用服务。
本篇主要介绍python中ConfigParser模块的API以及使用示例。

## ConfigParser - 解析配置文件

此模块定义类 ConfigParser. 它实现了一个基本的配置文件解析语言，提供一个类似微软INI文件的结构。
使用它可以更容易的被用户自定义。
在python 3.0中ConfigParser 更名为 configparser
配置文件包括由[section] 开头的选项和name: value(name=value)等条目。其中value可以包含格式化字符串。
由’#’ ‘;’开头的行被忽略并作为注释。
在查找配置项时，如果读取的配置项不在指定的section中，将会在[DEFAULT]中查找

## RawConfigParser Objects

``` python
RawConfigParser.defaults()
返回一个包含实例范围默认的字典
RawConfigParser.sections()
返回一个可用section的列表，不包含DEFAULT
RawConfigParser.add_section(section)
增加section
RawConfigParser.has_section(section)
显示是否包含此section
RawConfigParser.options(section)
返回一个在此section中的可用选项的列表
RawConfigParser.has_option(section, option)
如果给定section存在，并包含option，返回True，否则返回False
RawConfigParser.read(filenames)
试图去读并解析一组文件名，返回被成功解析的文件名列表。
RawConfigParser.readfp(fp)
从文件或者类似文件对象中读取配置数据
RawConfigParser.get(section, option)
获取section的option的值
RawConfigParser.getint(section, option) 
            .getfloat(section, option)
            .getboolean(section, option) 可使用0/1 yes/no true/false on/off 将会转化为True, False，其他值
                            会抛出ValueError异常
                            RawConfigParser.items(section)
返回指定section的(name, value)的列表
RawConfigParser.set(section, option, value)
设置选项及值
RawConfigParser.write(fileobject)
将配置信息写入指定文件对象
RawConfigParser.remove_option(section, option)
删除给定section的option
RawConfigParser.remove_section(section)
删除section
RawConfigParser.optionxform(option)
转换option的形式。如更改选项名称为大小写敏感
```

## ConfigParser Objects

ConfigParser 继承自RawConfigParser,并且扩展了它的接口，加入一些可选参数：
ConfigParser.get(section， option， raw, vars)
    获取给定section的选项值。raw为0，返回转换后的结果，为1返回原始字符形式, vars是一个字典，用来更改选项值
ConfigParser.items(section, option， raw, vars)
    返回指定section的(name, value, raw, vars)的列表

## 代码示例

### 新建config文件

``` python
import ConfigParser

config = ConfigParser.RawConfigParser()

config.add_section('Section1')
config.set('Section1', 'int', '15')
config.set('Section1', 'bool', 'true')
config.set('Section1', 'float', '3.1415')
config.set('Section1', 'baz', 'fun')
config.set('Section1', 'bar', 'Python')
config.set('Section1', 'foo', '%(bar)s is %(baz)s!')

# Writing our configuration file to 'example.cfg'
with open('example.cfg', 'wb') as configfile:
    config.write(configfile)
```

### 获取config文件内容

``` python
import ConfigParser

config = ConfigParser.RawConfigParser()
config.read('example.cfg')

# getfloat() raises an exception if the value is not a float
# getint() and getboolean() also do this for their respective types
float = config.getfloat('Section1', 'float')
int = config.getint('Section1', 'int')
print float + int

# Notice that the next output does not interpolate '%(bar)s' or '%(baz)s'.
# This is because we are using a RawConfigParser().
if config.getboolean('Section1', 'bool'):
        print config.get('Section1', 'foo')
```

``` python
import ConfigParser

config = ConfigParser.ConfigParser()
config.read('example.cfg')

# Set the third, optional argument of get to 1 if you wish to use raw mode.
print config.get('Section1', 'foo', 0) # -> "Python is fun!"
print config.get('Section1', 'foo', 1) # -> "%(bar)s is %(baz)s!"

# The optional fourth argument is a dict with members that will take
# precedence in interpolation.
print config.get('Section1', 'foo', 0, {'bar': 'Documentation',
                                                'baz': 'evil'})
```

``` python
import ConfigParser

# New instance with 'bar' and 'baz' defaulting to 'Life' and 'hard' each
config = ConfigParser.SafeConfigParser({'bar': 'Life', 'baz': 'hard'})
config.read('example.cfg')

print config.get('Section1', 'foo') # -> "Python is fun!"
config.remove_option('Section1', 'bar')
config.remove_option('Section1', 'baz')
print config.get('Section1', 'foo') # -> "Life is hard!"
```

## 参考文献

python library reference
编写高质量代码:改善Python程序的91个建议


