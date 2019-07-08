---
layout:     post
title:      "code-device_driver"
subtitle:   ""
date:       2019-07-05 00:28:00
author:     "刘念真"
header-img: "img/post-bg-js-module.jpg"
tags: Linux-device_driver&device
---

**Read the fucking source code!**

![device&device_driver](img\device&device_driver.gif)

![硬件拓扑](D:img\driver_model.gif)

本文简要讲述下Linux内核中的设备模型，包括kobject、kset、ktype、bus、class、device、device driver等概念，**内容借鉴自蜗窝科技[wowo](<http://www.wowotech.net/device_model/13.html/comment-page-3#comments>)**。

*A*. 基于上图的硬件拓扑先说device、device driver、bus、class比较上层一些的概念：
1. bus：可以认为bus是cpu和设备之前信息交互的通道，设备通过bus和cpu通信，从面向对象的设计思路来看这样就避免了设备和cpu的直接耦合，可以参考`include/linux/device.h中struct bus_type`的注释。
2. class：可以认为class是一类具有共同特性设备的抽象集合，基于面向对象的思想比较好理解；从属于同一class的设备驱动程序，可以直接从class中继承即可。
3. device：描述硬件的设备信息，如设备名字`const char *init_name`，设备从属的总线`struct bus_type *bus`，设备从属的类`struct class *class`等等，参考`include/linux/device.h中struct device`结构。
4. device driver：内核抽象出driver概念来描述设备的驱动程序，包含设备初始化接口`probe`、电源管理`const struct dev_pm_ops *pm`、该driver所支持设备的设备树信息`const struct of_device_id *of_match_table`等内容，参考`include/linux/device.h中struct device_driver`结构。
5. 基于device和device_driver结构派生出诸如i2c_client和i2c_driver、spi_device和spi_driver、platform_device和platform_driver等结构。
小结：在驱动程序中如何使用这些结构？

*B*. Kobject & kset & ktype
1. 