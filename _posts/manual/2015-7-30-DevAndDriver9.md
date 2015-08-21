---
layout: post
title: Linux设备驱动模型及其他(9)
description: 为PCI设备地址空间分配总线地址
category: manual
---

>上章讲到我们已经高清了有BIOS配置好的设备的中断请求去向。

##PCI设备地址空间&总线地址
弄清了中断请求的去向，下面就要为各个PCI设备中的各个地址区间分配总线地址，并设置好各个PCI设备对这些区间（到总线地址）的映射了。前面在枚举设备的时候已经看到，每个PCI设备都通过配置寄存器组提供其各个区间的起始地址和区间大小,在第五篇枚举设备时，就通过pci_read_bases（）将配置寄存器中有关区间起始地址和大小的相关数据进行了处理并存放在pci_dev->resource[]数组中，但是这些可能是设备内部的地址，或者是由BIOS分配的总线地址。对于两种情况要分别处理。
<ul>
<li>
如果是内部地址，就需要在一个统一的总线地址空间为这些区间分配地址并建立映射。
</li>
<li>
如果是由BIOS分配的总线地址，就需要对其进行验证，并建立相关的数据结构。
</li>
</ul>
对CPU来说，总线地址相当于物理地址，在此之后还要在这基础上再加上一层映射，将虚拟地址映射到总线地址。在分配过程中，只要原来已经分配的地址可用，就尽量维持原状，对这些地址只要验证一下，并为之建立相应的数据结构。那么，什么样的地址是不可用需要重新分配的呢？主要有两种情况：
<ul>
<li>
系统在初始化后，已经将物理地址的低端的一大块分配给了内核本身，这些地址当然不能用于PCI总线，而设备的内部的地址都在低端，因此需要重新分配。
</li>
<li>
PCI设备各个区间不允许互相冲突，如果发生冲突，也要做出调整。I/O地址也是一样的。
</li>
</ul>

对于存储器和I/O两种地址的资源管理，内核中有一套资源管理机制，每一个逻辑上独立的连续地址区间都由一个resource数据结构代表，这个数据结构我们以前已经有过接触了。

	 //include/linux/ioport.h
	 14 /*
	 15  * Resources are tree-like, allowing
	 16  * nesting etc..
	 17  */
	 18 struct resource {
	 19     resource_size_t start;
	 20     resource_size_t end;
	 21     const char *name;
	 22     unsigned long flags;
	 23     struct resource *parent, *sibling, *child;
	 24 };

结构中的start和end表示该区间的地址范围，flags表示区间的性质，比如是memory还是I/O地址。指针child，sibling和parent则用来维系可以上下两个方向攀援的树形结构。每个区间（的resource结构）都通过指针child只想其第一个子区间，而同区间的所有子区间都通过指针sibling形成一个单链接，并都通过指针parent指向其父区间。

我们在第四篇讲过，在创建主PCI总线时，在pci_bus结构中对resources链表添加了两个resource，分别是：

	  //kernel/resource.c
	  28 struct resource ioport_resource = { 
	  29     .name   = "PCI IO",
	  30     .start  = 0, 
	  31     .end    = IO_SPACE_LIMIT,
	  32     .flags  = IORESOURCE_IO, 
	  33 }; 
	  34 EXPORT_SYMBOL(ioport_resource); 
	  35 
	  36 struct resource iomem_resource = {
	  37     .name   = "PCI mem", 
	  38     .start  = 0,
	  39     .end    = -1,
	  40     .flags  = IORESOURCE_MEM,
	  41 };
	  42 EXPORT_SYMBOL(iomem_resource);

表示如果要使用I/O地址区间就从ioport_resource中分配，使用内存地址区间就从iomem_resource分配。不过系统在初始化阶段已经从这两个空间分配了许多资源，所以已经不再是像它们初值所表示的整个I/O地址空间或者整个内存地址空间了。

对总线地址的确认和分配是由pcibios_resource_survey()完成的，调用关系和代码如下：

	pci_subsys_init（）（subsys_initcall）
	  |
	  |
	  pcibios_init()
	    |
	    |
	     pcibios_resource_survey();


	//arch/x86/pci/i386.c 
	387 void __init pcibios_resource_survey(void)
	388 { 
	389     struct pci_bus *bus;
	390 
	391     DBG("PCI: Allocating resources\n");
	392 
	393     list_for_each_entry(bus, &pci_root_buses, node)
	394         pcibios_allocate_bus_resources(bus);
	395 
	396     list_for_each_entry(bus, &pci_root_buses, node)
	397         pcibios_allocate_resources(bus, 0);
	398     list_for_each_entry(bus, &pci_root_buses, node)
	399         pcibios_allocate_resources(bus, 1);
	400 
	401     e820_reserve_resources_late();
	402     /*
	403      * Insert the IO APIC resources after PCI initialization has
	404      * occurred to handle IO APICS that are mapped in on a BAR in
	405      * PCI space, but before trying to assign unassigned pci res.
	406      */
	407     ioapic_insert_resources();
	408 } 

首先通过pcibios_allocate_bus_resources（）为每条PCI总线分配地址资源，这里的pci_root_buses还记得么？它是个全局变量，系统中所有的根总线（即所有通过“HOST-PCI”桥链接的总线，通常情况下只有一条）都链接到这里。实际上就是个pci_bus结构队列cibios_allocate_bus_resources代码我们来分析下：

	//arch/x86/pci/i386.c
	232 static void pcibios_allocate_bus_resources(struct pci_bus *bus)  
	233 { 
	234     struct pci_bus *child;
	235 
	236     /* Depth-First Search on bus tree */
	237     if (bus->self)
	238         pcibios_allocate_bridge_resources(bus->self);
	239     list_for_each_entry(child, &bus->children, node)
	240         pcibios_allocate_bus_resources(child);
	241 } 

我们在前几篇讲过扫描根总线的时是递归调用pci_scan_child_bus来扫描的，它是一个深度优先的算法，会扫描根总线上所有通过“PCI—PCI”桥连接的总线。当时我们也看到过总线与总线的关系通过队列children还有指针parent来描述。而队列头children维持了次层PCI总线的pci_bus结构队列，在完成PCI总线和设备的枚举后，这些数据结构就已经构建好了。那既然PCI总线的系统结构是递归的，对整个PCI结构的资源分配就应该也是递归的。所以看这里的line239，就是对次层的pci_bus结构队列递归调用了pcibios_allocate_bus_resources，做深度优先遍历。而对于总线本身，也就相当于连接总线的PCI桥（详情见pci_bus结构定义），则用pcibios_allocate_bridge_resources来分配资源。

	//arch/x86/pci/i386.c
	208 static void pcibios_allocate_bridge_resources(struct pci_dev *dev) 
	209 {
	210     int idx;
	211     struct resource *r;
	212 
	213     for (idx = PCI_BRIDGE_RESOURCES; idx < PCI_NUM_RESOURCES; idx++) {
	214         r = &dev->resource[idx];
	215         if (!r->flags)
	216             continue;
	217         if (r->parent)  /* Already allocated */
	218             continue;
	219         if (!r->start || pci_claim_bridge_resource(dev, idx) < 0) {
	220             /*
	221              * Something is wrong with the region.
	222              * Invalidate the resource to prevent
	223              * child resource allocations in this
	224              * range.
	225              */
	226             r->start = r->end = 0;
	227             r->flags = 0;
	228         }
	229     } 
	230 } 

	//include/linux/pci.h  
	 81 enum {
	 82     /* #0-5: standard PCI resources */
	 83     PCI_STD_RESOURCES,
	 84     PCI_STD_RESOURCE_END = 5,
	 85      
	 86     /* #6: expansion ROM resource */
	 87     PCI_ROM_RESOURCE,		//6
	 88      
	 89     /* device specific resources */
	 90 #ifdef CONFIG_PCI_IOV
	 91     PCI_IOV_RESOURCES,
	 92     PCI_IOV_RESOURCE_END = PCI_IOV_RESOURCES + PCI_SRIOV_NUM_BARS - 1,     
	 93 #endif
	 94      
	 95     /* resources assigned to buses behind the bridge */
	 96 #define PCI_BRIDGE_RESOURCE_NUM 4
	 97      
	 98     PCI_BRIDGE_RESOURCES,		//7
	 99     PCI_BRIDGE_RESOURCE_END = PCI_BRIDGE_RESOURCES +
	100                   PCI_BRIDGE_RESOURCE_NUM - 1,
	101     
	102     /* total resources associated with a PCI device */
	103     PCI_NUM_RESOURCES,		//11
	104     
	105     /* preserve this for compatibility */
	106     DEVICE_COUNT_RESOURCE = PCI_NUM_RESOURCES,
	107 }; 	

对于普通设备，它的pci_dev结构中的resource[]数组开头六个（0~5）地址区间是设备上可能有的区间，第七区（6）是可能的扩充ROM区间。如果设备是PCI桥，则后面还有4个区间，pci_bus结构中的4个resource指针就分别指向这4个区间（见第6篇）而pci桥设备中的resource也是通过pci_read_bases（）总配置寄存器中读到的。我们在第5篇已经说过，PCI桥本身并不“使用”这些区间中的地址，而是用这些区间作为地址过滤的窗口。其中第一个是I/O地址，第二个用于存储器地址，第三个为“可预取”存储器地址区间，另外还有一个用于扩充ROM区间窗口。次层总线上所有设备（包括PCI桥）所使用的地址都必须在这些窗口中，换言之，这些设备所需要的地址都要从这些区间中分配。所以，每个PCI桥或者说每条PCI总线，都需要从其上层“批发”下一些地址，然后“零售”分配给连接在这条总线上的所有设备。包括把其中的一部分比发给次层总线。就这样，每条PCI总线上的设备都向其所在的总线批发地址资源，而总线则向其上层总线批发。那么，顶层的PCI总线又向谁批发呢？那就是ioport_resource和iomem_resource,这是两种地址资源的终极来源。


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

