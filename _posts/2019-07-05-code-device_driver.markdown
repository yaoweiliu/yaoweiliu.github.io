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
6. 小结：在驱动程序中如何使用这些结构？

*B*. Kobject & kset & ktype

1. Kobject是基本数据类型，每个Kobject都会在"/sys/“文件系统中以目录的形式出现。
2. Ktype代表Kobject（严格地讲，是包含了Kobject的数据结构）的属性操作集合（由于通用性，多个Kobject可能共用同一个属性操作集，因此把Ktype独立出来了）；**可以通过Ktype的release字段可以将包含该种类型kobject的数据结构的内存空间释放掉**。
3. Kset是一个特殊的Kobject（因此它也会在"/sys/“文件系统中以目录的形式出现），它用来集合相似的Kobject（这些Kobject可以是相同属性的，也可以不同属性的）。
4. Kobject的分配和释放：
   a. kmalloc动态分配接口：`kobject_init_and_add()`，这种方式分配的kobject，会在引用计数变为0时，由kobject_put调用其ktype的release接口，释放内存空间，具体可参考后面有关kobject_put的讲解。
   b. 由Kobject模块分配：`kobject_create_and_add()`，使用kobject_create自行分配空间，并内置了一个ktype（dynamic_kobj_ktype），用于在计数为0是释放空间。
   c. 通过kobject_get和kobject_put可以修改kobject的引用计数，并在计数为0时，调用ktype的release接口，释放占用空间。
5. 总结，Ktype以及整个Kobject机制的理解:
   Kobject的核心功能是：保持一个引用计数，当该计数减为0时，自动释放（由本文所讲的kobject模块负责） Kobject所占用的meomry空间。这就决定了Kobject必须是动态分配的（只有这样才能动态释放）。 
   而Kobject大多数的使用场景，是内嵌在大型的数据结构中（如Kset、device_driver等），因此这些大型的数据结构，也必须是动态分配、动态释放的。那么释放的时机是什么呢？是内嵌的Kobject释放时。但是Kobject的释放是由Kobject模块自动完成的（在引用计数为0时），那么怎么一并释放包含自己的大型数据结构呢？ 
   这时Ktype就派上用场了。我们知道，Ktype中的release回调函数负责释放Kobject（甚至是包含Kobject的数据结构）的内存空间，那么Ktype及其内部函数，是由谁实现呢？是由上层数据结构所在的模块！因为只有它，才清楚Kobject嵌在哪个数据结构中，并通过Kobject指针以及自身的数据结构类型，找到需要释放的上层数据结构的指针，然后释放它。 
   讲到这里，就清晰多了。所以，每一个内嵌Kobject的数据结构，例如kset、device、device_driver等等，都要实现一个Ktype，并定义其中的回调函数。同理，sysfs相关的操作也一样，必须经过ktype的中转，因为sysfs看到的是Kobject，而真正的文件操作的主体，是内嵌Kobject的上层数据结构！ 
   顺便提一下，Kobject是面向对象的思想在Linux kernel中的极致体现，但C语言的优势却不在这里，所以Linux kernel需要用比较巧妙（也很啰嗦）的手段去实现。------引用自[蜗窝科技](http://www.wowotech.net/device_model/kobject.html)
参考内核：`drivers/base/bus.c`、`include/linux/kobject.h`、`lib/kobject.c`

*C*. sysfs

1. kobject和kset在/sys/下的表现形式都是目录，目录名分别是两个结构体的name字段。
2. 驱动中使用sysfs：
   a)定义：使用DEVICE_ATTR(_name, _mode, _show, _store)定义一个名字为dev_attr_##_name的struct device_attribute变量，_mode可以使只写0444或者只读0222或者读写0666，_show为cat该文件时的回调函数，_store为echo该文件时的回调函数。
   b)创建文件：device_create_file()函数利用该对象(dev_attr_##_name)在device下创建文件。
   c)示例：https://github.com/yaoweiliu/work/blob/master/device_attr/device_attr.c
   d)基于low level的attribute请参考源码(include/linux/sysfs.h)，还没理解明白。

*D*. device&device_driver

1. device_driver的作用：获得与相应的硬件进行信息交互的能力，也即驱动硬件正常工作(时序等等因素)。
   **platform_driver中的suspend/resume/shutdown接口分别在系统挂起/唤醒/关机时被调用，probe/remove接口分别在设备和设备驱动匹配/移除时调用！**

*E*. bus&class

1. 

*F*. device resource management

1. 源码位置：driver/base/devres.c
2. 作用：
3. 接口：devm_xxx()