---
layout: post
title: Linux设备驱动模型及其他(12)
description: /proc/iommap 和/proc/ioports相关
category: manual
---

>本系列前11篇已经讲述了PCI总线的相关概念,涉及PCI设备的配置空间,PCI设备的配置,并通过代码,分析了linux内核中pci子系统的初始化,包括PCI设备的枚举和资源分配.

##回顾1

前面讲过,计算机上电时所有PCI设备是非激活的,而且仅响应配置交易,对于x86平台,如果系统中存在pci总线,那么是通过一个统一的入口地址向“HOST-PCI bridge”发出命令，由PCI桥来间接的完成对PCI设备配置寄存器的读写。PCI总线的设计者在I/O地址空间保留了8个字节来完成这个工作，那就是0xCF8~0xCFF。这8个字节实际上构成两个32位的寄存器，第一个是“地址寄存器”0xCF8，第二个是“数据寄存器”0xCFF。要访问某个设备的配置寄存器时，CPU先往地址寄存器写如目标地址，然后通过数据寄存器读写数据。而这里写入的目标地址是一种综合的“地址”：包括了总线号、设备号、功能号、以及配置寄存器地址。关于0xCF8~0xCFF的I/O口我们可以验证一下：

	$cat /proc/ioprots
	0000-0cf7 : PCI Bus 0000:00
	  0000-001f : dma1
	  0020-0021 : pic1
	  0040-0043 : timer0
	  0050-0053 : timer1
	  0060-0060 : keyboard
	.....
	0cf8-0cff : PCI conf1		//在这里！！ 
	.....

##回顾2

每个PCI设备都通过配置寄存器组提供其各个区间的起始地址和区间大小,在第五篇枚举设备时，就通过pci_read_bases（）将配置寄存器中有关区间起始地址和大小的相关数据进行了处理并存放在pci_dev->resource[]数组中，但是这些可能是设备内部的地址，或者是由BIOS分配的总线地址。对于两种情况要分别处理。
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

对于普通设备，它的pci_dev结构中的resource[]数组开头六个（0~5）地址区间是设备上可能有的区间，第七区（6）是可能的扩充ROM区间。如果设备是PCI桥，则后面还有4个区间，pci_bus结构中的4个resource指针就分别指向这4个区间（见第6篇）而pci桥设备中的resource也是通过pci_read_bases（）从配置寄存器中读到的。我们在第5篇已经说过，PCI桥本身并不“使用”这些区间中的地址，而是用这些区间作为地址过滤的窗口。其中第一个是I/O地址，第二个用于存储器地址，第三个为“可预取”存储器地址区间，另外还有一个用于扩充ROM区间窗口。次层总线上所有设备（包括PCI桥）所使用的地址都必须在这些窗口中，换言之，这些设备所需要的地址都要从这些区间中分配。所以，每个PCI桥或者说每条PCI总线，都需要从其上层“批发”下一些地址，然后“零售”分配给连接在这条总线上的所有设备。包括把其中的一部分比发给次层总线。就这样，每条PCI总线上的设备都向其所在的总线批发地址资源，而总线则向其上层总线批发。那么，顶层的PCI总线又向谁批发呢？那就是ioport_resource和iomem_resource,这是两种地址资源的终极来源。

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

结构中的start和end表示该区间的地址范围，flags表示区间的性质，比如是memory还是I/O地址。指针child，sibling和parent则用来维系可以上下两个方向攀援的树形结构。每个区间（的resource结构）都通过指针child指向其第一个子区间，而同区间的所有子区间都通过指针sibling形成一个单链接，并都通过指针parent指向其父区间。

![resource](/images/resource.png) 

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

##/proc/iomem的形成

我们首先来看下iomem中的信息

	~ cat /proc/iomem 
	00000000-00000fff : reserved
	00001000-0009fbff : System RAM
	0009fc00-0009ffff : reserved
	000a0000-000bffff : PCI Bus 0000:00
	000c0000-000c7fff : Video ROM
	000ce000-000cefff : Adapter ROM
	000d0000-000dffff : PCI Bus 0000:00
	000e0000-000fffff : reserved
	  000f0000-000fffff : System ROM
	00100000-bff9ffff : System RAM
	  01000000-017364f3 : Kernel code
	  017364f4-01d1e6bf : Kernel data
	  01e77000-01fdffff : Kernel bss
	bffa0000-bffadfff : ACPI Tables
	bffae000-bffeffff : ACPI Non-volatile Storage
	bfff0000-bfffffff : reserved
	c0000000-dfffffff : PCI Bus 0000:00
	  c0000000-c01fffff : PCI Bus 0000:02
	  c0200000-c03fffff : PCI Bus 0000:02
	  c0400000-c05fffff : PCI Bus 0000:03
	  ce000000-dfffffff : PCI Bus 0000:01
	    ce000000-cfffffff : 0000:01:00.0
	    d0000000-dfffffff : 0000:01:00.0
	e0000000-efffffff : PCI MMCONFIG 0000 [bus 00-ff]
	  e0000000-efffffff : pnp 00:0e
	f0000000-ffffffff : PCI Bus 0000:00
	  fcffbc00-fcffbfff : 0000:00:1d.7
	    fcffbc00-fcffbfff : ehci_hcd
	  fcffc000-fcffffff : 0000:00:1b.0
	    fcffc000-fcffffff : ICH HD audio
	  fd000000-feafffff : PCI Bus 0000:01
	    fd000000-fdffffff : 0000:01:00.0
	    fea7c000-fea7ffff : 0000:01:00.1
	      fea7c000-fea7ffff : ICH HD audio
	    fea80000-feafffff : 0000:01:00.0
	  feb00000-febfffff : PCI Bus 0000:03
	    febc0000-febdffff : 0000:03:00.0
	    febfc000-febfffff : 0000:03:00.0
	      febfc000-febfffff : sky2
	  fec00000-fec003ff : IOAPIC 0
	  fed00000-fed003ff : HPET 0
	  fed14000-fed19fff : pnp 00:00
	  fed1c000-fed1ffff : pnp 00:09
	    fed1f410-fed1f414 : iTCO_wdt
	  fed20000-fed8ffff : pnp 00:09
	  fed90000-fed93fff : pnp 00:00
	  fee00000-fee00fff : Local APIC
	    fee00000-fee00fff : reserved
	      fee00000-fee00fff : pnp 00:0d
	  ffc00000-ffefffff : pnp 00:0c
	  fff00000-ffffffff : reserved
	100000000-13fffffff : System RAM

其实iomem中记录的信息可以理解为是当前系统中"物理内存""的使用情况,这些"物理地址"最终都需要映射成"虚拟地址(线性地址)"才能真正的被使用.而这里面这些信息是怎么形成的呢?其实只要跟踪iomem_resource就能得到整棵资源树了.

PCI所使用的资源如何形成我们已经清楚了,那其他资源是怎么分配的呢?我们以内核代码所占资源是如何分配为例.

	00100000-bff9ffff : System RAM
	  01000000-017364f3 : Kernel code
	  017364f4-01d1e6bf : Kernel data
	  01e77000-01fdffff : Kernel bss

	 //arch/x86/kernel/setup.c  
	 149 static struct resource data_resource = {       
	 150     .name   = "Kernel data",
	 151     .start  = 0,
	 152     .end    = 0,
	 153     .flags  = IORESOURCE_BUSY | IORESOURCE_MEM
	 154 };
	 155   
	 156 static struct resource code_resource = {
	 157     .name   = "Kernel code",
	 158     .start  = 0,
	 159     .end    = 0,
	 160     .flags  = IORESOURCE_BUSY | IORESOURCE_MEM
	 161 };
	 162   
	 163 static struct resource bss_resource = {
	 164     .name   = "Kernel bss",
	 165     .start  = 0,
	 166     .end    = 0,
	 167     .flags  = IORESOURCE_BUSY | IORESOURCE_MEM
	 168 };

	 965     code_resource.start = __pa_symbol(_text);
	 966     code_resource.end = __pa_symbol(_etext)-1;
	 967     data_resource.start = __pa_symbol(_etext);
	 968     data_resource.end = __pa_symbol(_edata)-1;
	 969     bss_resource.start = __pa_symbol(__bss_start);
	 970     bss_resource.end = __pa_symbol(__bss_stop)-1;

	 1036     insert_resource(&iomem_resource, &code_resource);
	 1037     insert_resource(&iomem_resource, &data_resource);
	 1038     insert_resource(&iomem_resource, &bss_resource);

	 //kernel/resource.c  
	 822 int insert_resource(struct resource *parent, struct resource *new)
	 823 {              
	 824     struct resource *conflict;
	 825                
	 826     conflict = insert_resource_conflict(parent, new);
	 827     return conflict ? -EBUSY : 0;
	 828 }  


	 805 struct resource *insert_resource_conflict(struct resource *parent, struct resource *new)
	 806 {              
	 807     struct resource *conflict;
	 808                
	 809     write_lock(&resource_lock);
	 810     conflict = __insert_resource(parent, new);
	 811     write_unlock(&resource_lock);
	 812     return conflict;
	 813 }  

代码很好理解,最终还是调用__insert_resource来将resource插入到资源树中,本质上和第9篇分析的__request_resource是一样的.至于ioports和iomem基本是一样的处理方式,这里就不讨论了.

至此,我们就弄清楚 /proc/iommap 和/proc/ioports这两个东西是怎么形成的了,通过前面的分析,我们也明白了这两棵资源树是如何建立的.

##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

