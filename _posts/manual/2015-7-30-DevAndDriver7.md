---
layout: post
title: Linux设备驱动模型及其他(7)
description: PCI设备中断
category: manual
---

>上章讲到完成了根总线以及次层总线的扫描，并为所有pci设备完成了分配结构和串联等工作

##PCI设备中断

从这里开始就要统筹地进行对设备的设置了。这种设置包括两个方面：
<ul>
<li>1.设备的中断请求线与系统中断控制器之间的连接.</li>
<li>2.设备上各个区间在总线上的地址映射.</li>
</ul>
也就是从前需要通过跳线设置的两个内容。不过，实际上很可能BIOS已经在系统加电自检的阶段就完成了设置，所以这里往往只是加以确认。

另外，系统中除了0号总线之外可能还存在其他的主总线，也要加以扫描，但是出于效率的考虑，这种扫描要推迟到处理中断请求先才进行。在后面会看到！！！

###中断配置

前面讲过，在PCI总线上有INTA～INTD共4条中断请求线。凡是能产生中断的PCI设备，其中断请求必定连接在其中的一条上，此时其寄存器PCI_INTERRUPT_PIN的内容（1~4）表示连接在哪条线上。按照PCI总线规格书的规定，凡是单功能PCI模块（接口卡）的中断请求都应该连接在INTA上，多功能模块（多个逻辑设备共存在同一个物理模块上）才可以在使用INTA之余再使用其他的中断请求线。那么，这INTA～INTD4条中断请求线又是怎么样连接到中断控制器的中断请求输入线呢?PCI总线规格书中没有对此硬性规定。通常，系统中的中断控制器一共有16条中断请求输入线，每个PCI设备的中断请求线（如果有的话）就是要选择连接到其中的一条上去。理想的情况当然是每条中断请求输入线最多只连接一个中断源，但是一般而言这是不可能的，实际上只能对各项设备加以适当组合，让若干设备共用一条中断请求输入线。配备PCI总线的PC机母版一般都采用一种可编程中断请求路径互连器（router，通常与PCI桥集成在同一个芯片中），由软件设置4条PCI中断请求线与中断控制器的中断请求输入线之间的互连，下图为一种典型的设计示意【1】。
***
![PCI_IRQ](/images/PCI_IRQ.png)
***
从图中可以看出，在PCI接口卡上将中断请求都连接在INTA上其实只是局限与具体PCI插槽的意义，只是接口卡的结构比较划一而已。实际上，系统母版的设计自动将各种设备和中断请求分布到了路径互连器的各条输入线上。从系统软件的角度，有两个方面需要密切关注。
<ul>
<li>1.选择、确定，并通过路径互连器实施连接.</li>
<li>2.搞清楚具体插槽中具体设备的中断到底连接到了中断控制器的哪条输入线上，这样才能确定应该把设备的中断服务程序“登记”到哪一个中断服务队列中.</li>
</ul>
为此，BIOS通常提供一个“中断路径表”，为各条PCI总线的各个插槽提供其4条中断请求线的去向，就是母版上连接到了路径互连器的哪条输入线上。至于路径互连器的输出，则总是一对一地连接到中断控制器上。

可见，对于PCI总线中断机制的初始化，中断路径表和路径互连器是两个关键。事实上处理正是从中断路径表开始的。从pci_legacy_init()返回后，就来到了 x86_init.pci.init_irq()，实际上是调用了pcibios_irq_init。

	 //arch/x86/kernel/x86_init.c
	 81     .pci = {
	 82         .init           = x86_default_pci_init,
	 83         .init_irq       = x86_default_pci_init_irq,
	 84         .fixup_irqs     = x86_default_pci_fixup_irqs,
	 85     },

	 //arch/x86/include/asm/pci_x86.h  
 	 204 # define x86_default_pci_init_irq   pcibios_irq_init  

	1117 void __init pcibios_irq_init(void)
	1118 {  
	1119     DBG(KERN_DEBUG "PCI: IRQ init\n");
	1120    
	1121     if (raw_pci_ops == NULL)
	1122         return;
	1123    
	1124     dmi_check_system(pciirq_dmi_table);
	1125    
	1126     pirq_table = pirq_find_routing_table();
	1127    
	1128 #ifdef CONFIG_PCI_BIOS
	1129     if (!pirq_table && (pci_probe & PCI_BIOS_IRQ_SCAN))
	1130         pirq_table = pcibios_get_irq_routing_table();
	1131 #endif
	1132     if (pirq_table) {
	1133         pirq_peer_trick();
	1134         pirq_find_router(&pirq_router);
	1135         if (pirq_table->exclusive_irqs) {
	1136             int i;
	1137             for (i = 0; i < 16; i++)
	1138                 if (!(pirq_table->exclusive_irqs & (1 << i)))
	1139                     pirq_penalty[i] += 100;
	1140         }
	1141         /*
	1142          * If we're using the I/O APIC, avoid using the PCI IRQ
	1143          * routing table
	1144          */
	1145         if (io_apic_assign_pci_irqs)
	1146             pirq_table = NULL;
	1147     }
	1148    
	1149     x86_init.pci.fixup_irqs();
	1150    
	1151     if (io_apic_assign_pci_irqs && pci_routeirq) {
	1152         struct pci_dev *dev = NULL;
	1153         /*
	1154          * PCI IRQ routing is set up by pci_enable_device(), but we
	1155          * also do it here in case there are still broken drivers that
	1156          * don't use pci_enable_device().
	1157          */
	1158         printk(KERN_INFO "PCI: Routing PCI interrupts for all devices because \"pci=routeirq\" speci
	1159         for_each_pci_dev(dev)
	1160             pirq_enable_irq(dev);
	1161     }
	1162 }

前面讲过，PCI总线的4根中断请求线与路径互连器的连接是因母版的设计而异的，所以由母版上的BIOS提供一个PCI“中断请求路径表”。其结构为irq_routing_table：

	 //arch/x86/include/asm/pci_x86.h 
	 74 struct irq_routing_table {
	 75     u32 signature;          /* PIRQ_SIGNATURE should be here */
	 76     u16 version;            /* PIRQ_VERSION */
	 77     u16 size;           /* Table size in bytes */
	 78     u8 rtr_bus, rtr_devfn;      /* Where the interrupt router lies */
	 79     u16 exclusive_irqs;     /* IRQs devoted exclusively to
	 80                        PCI usage */
	 81     u16 rtr_vendor, rtr_device; /* Vendor and device ID of
	 82                        interrupt router */
	 83     u32 miniport_data;      /* Crap */
	 84     u8 rfu[11];
	 85     u8 checksum;            /* Modulo 256 checksum must give 0 */
	 86     struct irq_info slots[0];
	 87 } __attribute__((packed));

	 #define PIRQ_SIGNATURE  (('$' << 0) + ('P' << 8) + ('I' << 16) + ('R' << 24))
	 #define PIRQ_VERSION 0x0100  
	 //××××××××××××××××××××××××××××××××××××
	 63 struct irq_info {
	 64     u8 bus, devfn;          /* Bus, device and function */
	 65     struct {   
	 66         u8 link;        /* IRQ line ID, chipset dependent,
	 67                        0 = not routed */
	 68         u16 bitmap;     /* Available IRQs */
	 69     } __attribute__((packed)) irq[4];
	 70     u8 slot;            /* Slot number, 0=onboard */
	 71     u8 rfu;
	 72 } __attribute__((packed));

“中断请求路径表”的第一个个长字一定是一个特殊的值PIRQ_SIGNATURE，版本号必须为PIRQ_VERSION。这两个值其实是为了可以从内存中扫描到这张表。“路径表”的起点一定是与16字节边界对齐，但是不在固定的位置上，所以需要扫描寻找。而其实irq_routing_table只是“中断请求路径表”的头部，表的主体slots是一个irq_info结构数组。

对于系统中每条PCI总线上的每个模块，路径表中都有一个对应的irq_info结构。结构中给出了其4条中断请求线与路径互连器输入线的连接，同时还有个位图，说明了可供选择的连接对象，即中断控制器的各条输入线。

在pcibios_irq_init首先通过调用pirq_find_routing_table（）在BIOS所在的内存区间扫描，以找到中断请求路径表，并将其指针给全局变量pirq_table。

	  //arch/x86/pci/irq.c 
	  90  *  Search 0xf0000 -- 0xfffff for the PCI IRQ Routing Table.
	  91  */ 
	  92
	  93 static struct irq_routing_table * __init pirq_find_routing_table(void) 
	  94 { 
	  95     u8 *addr;
	  96     struct irq_routing_table *rt; 
	  97 
	  98     if (pirq_table_addr) {
	  99         rt = pirq_check_routing_table((u8 *) __va(pirq_table_addr));
	 100         if (rt)
	 101             return rt;
	 102         printk(KERN_WARNING "PCI: PIRQ table NOT found at pirqaddr\n");
	 103     }
	 104     for (addr = (u8 *) __va(0xf0000); addr < (u8 *) __va(0x100000); addr += 16) {
	 105         rt = pirq_check_routing_table(addr);
	 106         if (rt)
	 107             return rt;
	 108     }
	 109     return NULL;
	 110 }

	  64 static inline struct irq_routing_table *pirq_check_routing_table(u8 *addr) 
	  65 {
	  66     struct irq_routing_table *rt;
	  67     int i;
	  68     u8 sum;  
	  69
	  70     rt = (struct irq_routing_table *) addr;
	  71     if (rt->signature != PIRQ_SIGNATURE ||
	  72         rt->version != PIRQ_VERSION ||
	  73         rt->size % 16 ||
	  74         rt->size < sizeof(struct irq_routing_table))
	  75         return NULL;
	  76     sum = 0; 
	  77     for (i = 0; i < rt->size; i++)
	  78         sum += addr[i];
	  79     if (!sum) {  
	  80         DBG(KERN_DEBUG "PCI: Interrupt Routing Table found at 0x%p\n",
	  81             rt); 
	  82         return rt;
	  83     } 
	  84     return NULL;
	  85 }  

可以看到，实际就是以16字节为步长，在BIOS内存空间去匹配irq_routing_table的前两个固定成员，如果匹配到，就肯定就是要找的“中断路径表”。并使内核全局指针pirq_table指向该表。找到这张表后，就要对它进行进一步的处理了，在pcibios_irq_init中调用pirq_peer_trick:

	 //arch/x86/pci/irq.c 
	 112 /*  
	 113  *  If we have a IRQ routing table, use it to search for peer host
	 114  *  bridges.  It's a gross hack, but since there are no other known
	 115  *  ways how to get a list of buses, we have to go this way.
	 116  */ 
	 117     
	 118 static void __init pirq_peer_trick(void)
	 119 {   
	 120     struct irq_routing_table *rt = pirq_table;
	 121     u8 busmap[256];
	 122     int i;
	 123     struct irq_info *e;
	 124     
	 125     memset(busmap, 0, sizeof(busmap));
	 126     for (i = 0; i < (rt->size - sizeof(struct irq_routing_table)) / sizeof(struct irq_info); i++) {
	 127         e = &rt->slots[i];
	 128 #ifdef DEBUG
	 129         {
	 130             int j;
	 131             DBG(KERN_DEBUG "%02x:%02x slot=%02x", e->bus, e->devfn/8, e->slot);
	 132             for (j = 0; j < 4; j++)
	 133                 DBG(" %d:%02x/%04x", j, e->irq[j].link, e->irq[j].bitmap);
	 134             DBG("\n");
	 135         }
	 136 #endif
	 137         busmap[e->bus] = 1;
	 138     }
	 139     for (i = 1; i < 256; i++) {
	 140         if (!busmap[i] || pci_find_bus(0, i))
	 141             continue;
	 142         pcibios_scan_root(i);
	 143     }
	 144     pcibios_last_bus = -1;
	 145 }  

这个函数的工作是，利用for循环对“中断请求路径表”中的各个表项所涉及的总线做一次统计，循环结束后凡是busmap[]（以总线号为下标）中为1的项相对应的总线都是在路径表中有的表项。一般情况下，这些总线应该都已经在前一阶段中完成了扫描枚举，但是从pcibios_scan_root()扫描的是从0号总线开始的一棵树，如果系统中除此之外还有其他的PCI主总线，则还没有进行扫描。而busmap则是所有PCI总线的清单，因为这份清单是来自与对pirq_table的统计，而pirq_table中的数据又是直接由BIOS提供的，所以busmap[]具有权威性！！在busmap的引导下，再次有目标的通过pcibios_scan_root(i)进行其他主总线的扫描。（这里与本文开始时呼应）而已经扫描过的总线不受此函数影响。待全部结束后将pcibios_last_bus置为-1，表示这次真的是将系统中所有的主总线都扫描完毕了，以后不用担心害怕了。

回到pcibios_irq_init（）中，这个时候所有遗漏的总线已经都扫描了，接下来就要处理路径互连器了。中断路径互连器通常与PCI桥集成在同一个芯片中，并且也作为PCI设备连接到PCI总线上。路径表头部的rtr_bus, rtr_devfn两个字段指明了该设备所在的位置，rtr_vendor, rtr_device两个字段则为芯片提供者以及产品编号。调用函数pirq_find_router(&pirq_router)的作用就是找到这个设备的pci_dev数据结构，并根据其提供者以及编号找到相应的驱动函数。

	 //arch/x86/pci/irq.c
	 818 static void __init pirq_find_router(struct irq_router *r)
	 819 {
	 820     struct irq_routing_table *rt = pirq_table;
	 821     struct irq_router_handler *h;
	 822    
	 823 #ifdef CONFIG_PCI_BIOS
	 824     if (!rt->signature) {
	 825         printk(KERN_INFO "PCI: Using BIOS for IRQ routing\n");
	 826         r->set = pirq_bios_set;
	 827         r->name = "BIOS";
	 828         return;
	 829     }
	 830 #endif
	 831    
	 832     /* Default unless a driver reloads it */
	 833     r->name = "default";
	 834     r->get = NULL;
	 835     r->set = NULL;
	 836    
	 837     DBG(KERN_DEBUG "PCI: Attempting to find IRQ router for [%04x:%04x]\n",
	 838         rt->rtr_vendor, rt->rtr_device);
	 839    
	 840     pirq_router_dev = pci_get_bus_and_slot(rt->rtr_bus, rt->rtr_devfn);
	 841     if (!pirq_router_dev) {
	 842         DBG(KERN_DEBUG "PCI: Interrupt router not found at "
	 843             "%02x:%02x\n", rt->rtr_bus, rt->rtr_devfn);
	 844         return;
	 845     }
	 846    
	 847     for (h = pirq_routers; h->vendor; h++) {
	 848         /* First look for a router match */
	 849         if (rt->rtr_vendor == h->vendor &&
	 850             h->probe(r, pirq_router_dev, rt->rtr_device))
	 851             break;
	 852         /* Fall back to a device match */
	 853         if (pirq_router_dev->vendor == h->vendor &&
	 854             h->probe(r, pirq_router_dev, pirq_router_dev->device))
	 855             break;
	 856     }
	 857     dev_info(&pirq_router_dev->dev, "%s IRQ router [%04x:%04x]\n",
	 858          pirq_router.name,
	 859          pirq_router_dev->vendor, pirq_router_dev->device);
	 860    
	 861     /* The device remains referenced for the kernel lifetime */
	 862 }  

首先通过“中断请求路径表”的头部记录的信息，利用pci_get_bus_and_slot来找到其pci_dev结构赋给全局变量pirq_router_dev，如果找不到就歇菜了。

中断路径互连器由一个irq_router的数据结构来表示，结构中的函数指针get和set用于读出和改变芯片中的互连，不同芯片可能由不同的get和set函数。PC机中可能采用的此类芯片有好几种，内核中定义了一个irq_router_handler结构的数组pirq_routers[]，里面包括了各种常用的芯片以及其probe函数。

	 //arch/x86/pci/irq.c 
	 43 struct irq_router { 
	 44     char *name;
	 45     u16 vendor, device;
	 46     int (*get)(struct pci_dev *router, struct pci_dev *dev, int pirq);
	 47     int (*set)(struct pci_dev *router, struct pci_dev *dev, int pirq,
	 48         int new);
	 49 }; 

	 51 struct irq_router_handler { 
	 52     u16 vendor;
	 53     int (*probe)(struct irq_router *r, struct pci_dev *router, u16 device);
	 54 }; 
	 
	 794 static __initdata struct irq_router_handler pirq_routers[] = {  
	 795     { PCI_VENDOR_ID_INTEL, intel_router_probe },
	 796     { PCI_VENDOR_ID_AL, ali_router_probe },
	 797     { PCI_VENDOR_ID_ITE, ite_router_probe },
	 798     { PCI_VENDOR_ID_VIA, via_router_probe },
	 799     { PCI_VENDOR_ID_OPTI, opti_router_probe },
	 800     { PCI_VENDOR_ID_SI, sis_router_probe },
	 801     { PCI_VENDOR_ID_CYRIX, cyrix_router_probe },
	 802     { PCI_VENDOR_ID_VLSI, vlsi_router_probe },
	 803     { PCI_VENDOR_ID_SERVERWORKS, serverworks_router_probe },
	 804     { PCI_VENDOR_ID_AMD, amd_router_probe },
	 805     { PCI_VENDOR_ID_PICOPOWER, pico_router_probe },
	 806     /* Someone with docs needs to add the ATI Radeon IGP */
	 807     { 0, NULL }
	 808 }; 

回到pirq_find_router()中，根据pirq_table中的vendor就可以去pirq_routers去匹配想要的“驱动”了，匹配工作具体由pirq_routers中的probe来完成，代码很好理解，就是通过对Device ID的判断来完成对irq_router结构的初始化，下面代码摘自intel_router_probe：

	 arch/x86/pci/irq.c
	 537 static __init int intel_router_probe(struct irq_router *r, struct pci_dev *r
	 538 {
	 539     static struct pci_device_id __initdata pirq_440gx[] = {
	 540         { PCI_DEVICE(PCI_VENDOR_ID_INTEL, PCI_DEVICE_ID_INTEL_82443GX_0) },
	 541         { PCI_DEVICE(PCI_VENDOR_ID_INTEL, PCI_DEVICE_ID_INTEL_82443GX_2) },
	 542         { },
	 543     };
	 544    
	 545     /* 440GX has a proprietary PIRQ router -- don't use it */
	 546     if (pci_dev_present(pirq_440gx))
	 547         return 0;
	 548    
	 549     switch (device) {
	 550     case PCI_DEVICE_ID_INTEL_82371FB_0:
	 551     case PCI_DEVICE_ID_INTEL_82371SB_0:
	 552     case PCI_DEVICE_ID_INTEL_82371AB_0:
	 ..............
	 590     case PCI_DEVICE_ID_INTEL_PATSBURG_LPC_1:
	 591         r->name = "PIIX/ICH";
	 592         r->get = pirq_piix_get;
	 593         r->set = pirq_piix_set;
	 594         return 1;
	 595     }
 	 596 
	 597     if ((device >= PCI_DEVICE_ID_INTEL_5_3400_SERIES_LPC_MIN && 
	 598          device <= PCI_DEVICE_ID_INTEL_5_3400_SERIES_LPC_MAX) 
	 599     ||  (device >= PCI_DEVICE_ID_INTEL_COUGARPOINT_LPC_MIN && 
	 600          device <= PCI_DEVICE_ID_INTEL_COUGARPOINT_LPC_MAX)
	 601     ||  (device >= PCI_DEVICE_ID_INTEL_DH89XXCC_LPC_MIN &&
	 602          device <= PCI_DEVICE_ID_INTEL_DH89XXCC_LPC_MAX)
	 603     ||  (device >= PCI_DEVICE_ID_INTEL_PANTHERPOINT_LPC_MIN &&
	 604          device <= PCI_DEVICE_ID_INTEL_PANTHERPOINT_LPC_MAX)) {
	 605         r->name = "PIIX/ICH";
	 606         r->get = pirq_piix_get;
	 607         r->set = pirq_piix_set;
	 608         return 1;
	 609     }
	 610     
	 611     return 0;
	 612 }   

上面代码就将中断路径互连器irq_router结构的get和set分别设置为相应的函数。这些都结束后就回到pcibios_irq_init中，先来看其中的line1132～line1147.

	1132     if (pirq_table) {
	1133         pirq_peer_trick();
	1134         pirq_find_router(&pirq_router);
	1135         if (pirq_table->exclusive_irqs) {
	1136             int i;
	1137             for (i = 0; i < 16; i++)
	1138                 if (!(pirq_table->exclusive_irqs & (1 << i)))
	1139                     pirq_penalty[i] += 100;
	1140         }
	1141         /*
	1142          * If we're using the I/O APIC, avoid using the PCI IRQ
	1143          * routing table
	1144          */
	1145         if (io_apic_assign_pci_irqs)
	1146             pirq_table = NULL;
	1147     }

在中断路径表pirq_table中还有个位图exclusive_irqs，位图中为1的位表示中断控制器的相应输入应该专用，而避免为多个中断源共用。所以对于这些中断请求输入线，代码中在一个数组pirq_penalty[]的相应元素上增加一个惩罚量“100”，使得中断路径互连时不太会选择这些作为对象。

还有个问题，看line1145，中断路径互连器仅在采用8259A中断控制器时才会使用，如果系统采用的时“高级可编程中断控制器”APIC，那么就需要把pirq_table设为NULL，因为APIC本身就是“可编程”的。接下就调用x86_init.pci.fixup_irqs()来进行有关中断请求路径互连了。下篇见！
##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

