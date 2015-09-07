---
layout: post
title: Linux设备驱动模型及其他(12)
description: 设备驱动模型初探1
category: manual
---

##



##我们有什么

我们首先来看看到现在为止,我们有什么了?
<ul>
<li>1.两棵资源树(iomem,ioports)</li>
<li>2.根总线以及其他次层总线的pci_bus结构,由于总线也属于设备,pci_bus中也包含了相应的device结构.</li>
<li>3.各个PCI设备的pci_dev结构,该结构中也包含了device结构.</li>
</ul>


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

