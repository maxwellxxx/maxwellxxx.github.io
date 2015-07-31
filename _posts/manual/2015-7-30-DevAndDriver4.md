---
layout: post
title: Linux设备驱动模型及其他(4)
description: Hi，cortana！
category: manual
---

>上章讲到Linux写PCI设备枚举和配置的准备工作。

##PCI子系统初始化（PCI设备枚举和设置）
完成了准备工作，接下来就是正式的枚举总线以及设备了，但是具体从哪里开始，还得寻找下。它其实是在pci子系统初始化的时候才真正开始工作的：

	 arch/x86/pci/legacy.c
	 57 int __init pci_subsys_init(void)
	 58 {
	 59     /*
	 60      * The init function returns an non zero value when
	 61      * pci_legacy_init should be invoked.
	 62      */
	 63     if (x86_init.pci.init()) 
	 64         pci_legacy_init();
	 65
	 66     pcibios_fixup_peer_bridges();
	 67     x86_init.pci.init_irq();
	 68     pcibios_init();
	 69
	 70     return 0;
	 71 }
	 72 subsys_initcall(pci_subsys_init);

相对于我们熟悉的arch_initcall，这里是subsys_initcall，从上一篇提供链接的博客可以知道，subsys_initcall的级别是低于arch_initcall的，这也印证了我们分析的流程是对的————先做了准备工作才进行这里正式的工作。那就一点一点来吧！首先是pci_legacy_init()：
	
	pci_legacy_init()（legacy.c）
	   |
	   |
	   |
	   pcibios_scan_root（0）（arch/x86/pci/common.c）
	****************************************************
	475 void pcibios_scan_root(int busnum)
	476 {   
	477     struct pci_bus *bus;
	478     struct pci_sysdata *sd;
	479     LIST_HEAD(resources);
	480     
	481     sd = kzalloc(sizeof(*sd), GFP_KERNEL);
	482     if (!sd) {
	483         printk(KERN_ERR "PCI: OOM, skipping PCI bus %02x\n", busnum);
	484         return;
	485     }
	486     sd->node = x86_pci_root_bus_node(busnum);
	487     x86_pci_root_bus_resources(busnum, &resources);
	488     printk(KERN_DEBUG "PCI: Probing PCI hardware (bus %02x)\n", busnum);
	489     bus = pci_scan_root_bus(NULL, busnum, &pci_root_ops, sd, &resources);
	490     if (!bus) {
	491         pci_free_resource_list(&resources);
	492         kfree(sd);
	493     } 
	494 } 

在这个函数里首先调用 x86_pci_root_bus_resources为0号总线“分配资源”，函数如下：

	 //arch/x86/pci/bus_numa.c
	 30 void x86_pci_root_bus_resources(int bus, struct list_head *resources)
	 31 {   
	 32     struct pci_root_info *info = x86_find_pci_root_info(bus);
	 33     struct pci_root_res *root_res;
	 34     struct pci_host_bridge_window *window;
	 35     bool found = false;
	 36     
	 37     if (!info)
	 38         goto default_resources;
	 ......
	 ......
	 66     
	 67 default_resources:
	 68     /*
	 69      * We don't have any host bridge aperture information from the
	 70      * "native host bridge drivers," e.g., amd_bus or broadcom_bus,
	 71      * so fall back to the defaults historically used by pci_create_bus().
	 72      */
	 73     printk(KERN_DEBUG "PCI: root bus %02x: using default resources\n", bus);
	 74     pci_add_resource(resources, &ioport_resource);
	 75     pci_add_resource(resources, &iomem_resource);
	 76 }   

	 //drivers/pci/bus.c
	 20 void pci_add_resource_offset(struct list_head *resources, struct resource *res,
	 21                  resource_size_t offset)
	 22 {
	 23     struct pci_host_bridge_window *window;
	 24  
	 25     window = kzalloc(sizeof(struct pci_host_bridge_window), GFP_KERNEL);
	 26     if (!window) {
	 27         printk(KERN_ERR "PCI: can't add host bridge window %pR\n", res);
	 28         return;
	 29     }         
	 30  
	 31     window->res = res;
	 32     window->offset = offset;
	 33     list_add_tail(&window->list, resources);
	 34 }   
	 35 EXPORT_SYMBOL(pci_add_resource_offset);
	 36     
	 37 void pci_add_resource(struct list_head *resources, struct resource *res)
	 38 {   
	 39     pci_add_resource_offset(resources, res, 0);
	 40 }   
	 41 EXPORT_SYMBOL(pci_add_resource);


这里一般情况下就会到default_ressources,

##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译



