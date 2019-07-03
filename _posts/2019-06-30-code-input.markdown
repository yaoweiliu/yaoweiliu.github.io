---
layout:     post
title:      "code-input"
subtitle:   ""
date:       2019-06-30 23:41:00
author:     "刘念真"
header-img: "img/post-bg-js-module.jpg"
tags: Linux-input
---

Read the fucking source code!

![input](img\input.png)

如图示为Linux里input子系统的framework。input设备的主设备号都是13，就像miscdevice的主设备号都是10一样，参考`include/uapi/linux/major.h`文件中的INPUT_MAJOR和MISC_MAJOR宏。

input subsystem可以分为两个部分：Device drivers即`struct input_dev`和Event handlers即`struct input_handler`；其中Event handlers从Device drivers获得相应的事件并负责上报这些事件。

input_dev驱动常用数据结构和函数接口：

`struct input_dev`代表一个input设备；

`input_allocate_device()`申请input设备，一般在申请之后要初始化各成员变量，特别是`struct input_dev`的name字段，再之后设置`input_dev`支持的事件，如`evbit\keybit\relbit\absbit等`[`set_bit()`]。

`input_register_device()`注册input设备，查看input.c可知在这个接口内建立`input_dev`和`input_handler`之间的联系，如
```
list_for_each_entry(handler, &input_handler_list, node)
	input_attach_handler(dev, handler);
```

`input_report_abs()\input_report_rel()\input_report_key()\input_sync()`是上报相应的事件的接口函数，一般这些接口在中断处理函数内调用，

`input_set_drvdata()`保存设备结构体资源到`struct device`的driver_data字段中，一般在`probe`函数内完成。

`input_get_drvdata()`从`struct device`的driver_data字段获取之前保存的设备结构体资源，一般在`remove`函数内完成。

kernel内input设备采用分层的设计框架，一般的键盘、鼠标、LCD触摸屏等设备都属于input设备，当然也都是字符设备的范畴；其中的`file_operations`操作集的初始化在更底层的部分完成（这一部分抽象成公共的接口由kernel的input子系统来做），用户不用关心。可以阅读内核源码下的`drivers/input/evdev.c和drivers/input/input.c`。具体分析文章参见[input subsystem](https://www.cnblogs.com/lifexy/p/7542989.html)。

