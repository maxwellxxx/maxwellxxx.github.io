---
layout: post
title: Linux设备驱动模型及其他(13)
description: 设备驱动模型初探1
category: manual
---

##为什么需要设备驱动模型[1]
为什么需要设备驱动模型?这个问题的答案非常奇葩,答案就是:节能!

linux自2.6内核增加了一个引人注目的新特性——统一设备模型(device model)。设备模型提供了一个独立的机制专门来表示设备，并描述其在系统中的拓扑结构，从而使得系统具有以下优点：
<ul>
<li>代码重复最小化。</li>
<li>提供诸如引用计数这样的统一机制。</li>
<li>可以列举系统中所有的设备，观察它们的状态，并且查看它们连接的总线。</li>
<li>可以将系统中的全部设备结构以树的形式完整、有效的展 现出来——包括所有的总线和内部连接。</li>
<li>可以将设备和其对应的驱动联系起来，反之亦然。</li>
<li>可以将设备按照类型加以归类，比如分类为输入设备，而无需理解物理设备的拓扑结构。</li>
<li>可以沿设备树的叶子向其根的方向依次遍历，以保证能以正确顺序关闭各设备的电源。</li>
</ul>

最后一点是实现设备模型的最初动机。若想在内核中实现智能的电源管理，就需要来建立表示系统中设备拓扑关系的树结构。当在树上端的设备关闭电源时，内核必须首先关闭该设备节点以下的（处于叶子上的）设备电源。比如内核需要先关闭一个USB鼠标，然后才可关闭USB控制器；同样内核也必须在关闭PCI总线前先关闭USB控制器。简而言之，若要准确而又高效的完成上述电源管理目标，内核无疑需要一颗设备树。

>(注：设备模型与电源管理相关联，貌似匪夷所思，可实际上，一个新观点或模型出现，在其背后都是需求的刺激。最近以来，节电一直是Intel、IBM等大公司孜孜追求的目标，虚拟化技术的本质其实就是节能。或者说，在能源日趋紧张的今日，节能是人类共同追求的目标)。

为什么驱动模型有助于电源管理，再说两句：
<ul>系统中所有硬件设备由内核全权负责电源管理。例如，在以电池供电的计算机进入“待机”状态时，内核应立刻强制每个硬件设备（硬盘，显卡，声卡，网卡，总线控制器等等）处于低功率状态。因此，每个能够响应“待机”状态的设备驱动程序必须包含一个回调函数，它能够使得硬件设备处于低功率状态。而且，硬件设备必须按准确的顺序进入“待机”状态，否则一些设备可能会处于错误的电源状态。例如，内核必须首先将硬盘置于“待机”状态，然后才是它们的磁盘控制器，因为若按照相反的顺序执行，磁盘控制器就不能向硬盘发送命令。</ul>

##设备驱动带来的便利

linux设备驱动模型为内核建立一个统一的设备模型，从而有一个对系统结构的一般性抽象描述。换句话说，Linux设备模型提取了*硬件设备*操作的共同属性，进行抽象，并将这部分共同的属性在内核中实现，而为需要新添加*设备*或驱动提供一般性的统一接口，这使得驱动程序的开发变得更简单了，而程序员只需要去学习接口就行了。(注意区分硬件设备和设备!在下面会进行区分)

那设备驱动模型又为我们抽象了硬件设备哪些东西呢?
<ul>
<li>电源管理</li>
<li>即插即用设备支持</li>
<li>与用户空间的通信</li>
</ul>

##印记
好吧,因为马上找工作,不得不直接转换到网络子系统,关于设备模型,包括块设备子系统等问题留在以后再分析...

总结下需要弄懂的问题:
<ul>
<li>硬件设备和linux设备的区别</li>
<li>do_bseic_setup所做的工作,包括设备驱动模型子系统的初始化</li>
<li>do_initcall所做的工作包括块设备子系统初始化,网络设备初始化,pci子系统初始化</li>
<li>bus,device,driver三者关系已经在内核中的表示</li>
<li>sysfs文件系统的形成和作用</li>
<li>/dev的形成/proc/devices的形成</li>
<li>一个硬件驱动设备所做的工作:各种enable(包括各个资源,io口,mem),映射iomap,驱动硬件(初始化,中断处理),暴露设备(块,字,网),对设备文件的请求进行处理(各种ops)</li>
<li>看module_init的定义,与initcall的关系,驱动模块的工作流程(可分析e1000网卡和nvme驱动)</li>
<li>udev进程的工作原理,uevent,哪些设备会进入到/dev,register_blkdev的工作(仅暴露到/proc/devices)</li>
</ul>

系统启动时,因为还没有udev进程,uevent不能被udev接收,由系统启动结束后udev去/sys中去遍历

PCI总线扫描结束后,只扫描到PCI设备层次,一些控制器(SATA,NVME控制器)连接的设备还没有被发现,需要加载控制器驱动后才能发现设备,比如扫描到的硬盘会为之调用add_disk().

"调用device_create和直接调用device_add的区别，就只是在参数devt上，如果devt为0（即直接调用device_add），那么devtmpfs就不会创建节点udev也同理，如果uevent中没有devt，那么udev也不会创建这个节点


	device_create
	    |
	   + -- kzalloc struct device
	    |
	   +---device_register
		        |
		       +-----device_initialize
		        |
		       +-----device_add

device_create比device_add多做的事情非常清楚了呀。多一：
1. 新建struct device， device_add是不会新建的，只会加。
2. 进行了初始化， 如果不调device_register， 就得自己去调用device_initiali初始化。


关于第二点，device_register的函数说明说得很清楚：
/**
*        device_register - register a device with the system.
*        @dev:        pointer to the device structure
*
*        This happens in two clean steps - initialize the device
*        and add it to the system. The two steps can be called
*        separately, but this is the easiest and most common.
*        I.e. you should only call the two helpers separately if
*        have a clearly defined need to use and refcount the device
*        before it is added to the hierarchy.
*/

关于正真设备文件的创建（不是指sys下的文件）， 最终是由device_add函数里头的kobject_uevent(&dev->kobj, KOBJ_ADD)完成的对hotplug_helper的调用的。"----"保留"书签中的内容


##我们有什么

我们首先来看看到现在为止,我们有什么了?
<ul>
<li>1.两棵资源树(iomem,ioports)</li>
<li>2.根总线以及其他次层总线的pci_bus结构,由于总线也属于设备,pci_bus中也包含了相应的device结构.</li>
<li>3.各个PCI设备的pci_dev结构,该结构中也包含了device结构.</li>
</ul>


##参考目录
[1]《Linux内核之旅》http://www.kerneltravel.net/?p=409

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

