---
layout: post
title: Linux设备驱动模型及其他(10)
description: 在PCI设备地址空间分配PCI设备资源（验证）
category: manual
---

>上章讲到所有的PCI总线所需要的地址资源已经分配好了。

##为PCI设备分配资源（验证）

回到pcibios_resource_survey中，接下来就要为PCI设备分配地址资源了，这里的分配调用了两趟（这个两趟是不是很熟悉？？可以复习下第六篇）pcibios_allocate_resources，这里的“分配”同样也是“追认”。

	//arch/x86/pci/i386.c 
	387 void __init pcibios_resource_survey(void)
	388 {
	389     struct pci_bus *bus;
	390    
	391     DBG("PCI: Allocating resources\n");
	392    
	393     list_for_each_entry(bus, &amp;pci_root_buses, node)
	394         pcibios_allocate_bus_resources(bus);
	395    
	396     list_for_each_entry(bus, &amp;pci_root_buses, node)
	397         pcibios_allocate_resources(bus, 0);
	398     list_for_each_entry(bus, &amp;pci_root_buses, node)
	399         pcibios_allocate_resources(bus, 1);
	400    
	401     e820_reserve_resources_late();
	.........................................
	408 }

	306 static void pcibios_allocate_resources(struct pci_bus *bus, int pass)
	307 { 
	308     struct pci_dev *dev;
	309     struct pci_bus *child;
	310 
	311     list_for_each_entry(dev, &amp;bus-&gt;devices, bus_list) {
	312         pcibios_allocate_dev_resources(dev, pass);
	313 
	314         child = dev-&gt;subordinate; 
	315         if (child)  
	316             pcibios_allocate_resources(child, pass);
	317     }   
	318 }

pcibios_allocate_resources为当前总线的每个设备分配地址空间，如果设备为桥设备，则递归调用pcibios_allocate_resources（）为次层总线上的设备分配地址空间，正真的工作由pcibios_allocate_dev_resources来进行：

	//arch/x86/pci/i386.c 
	248 static void pcibios_allocate_dev_resources(struct pci_dev *dev, int pass) 
	249 {
	250     int idx, disabled, i;
	251     u16 command;
	252     struct resource *r;
	253    
	254     struct pci_check_idx_range idx_range[] = {
	255         { PCI_STD_RESOURCES, PCI_STD_RESOURCE_END },
	256 #ifdef CONFIG_PCI_IOV
	257         { PCI_IOV_RESOURCES, PCI_IOV_RESOURCE_END },
	258 #endif
	259     };
	260    
	261     pci_read_config_word(dev, PCI_COMMAND, &amp;command);
	262     for (i = 0; i &lt; ARRAY_SIZE(idx_range); i++)
	263         for (idx = idx_range[i].start; idx &lt;= idx_range[i].end; idx++) {
	264             r = &amp;dev-&gt;resource[idx];
	265             if (r-&gt;parent)  /* Already allocated */
	266                 continue;
	267             if (!r-&gt;start)  /* Address not assigned at all */
	268                 continue;
	269             if (r-&gt;flags &amp; IORESOURCE_IO)
	270                 disabled = !(command &amp; PCI_COMMAND_IO);
	271             else
	272                 disabled = !(command &amp; PCI_COMMAND_MEMORY);
	273             if (pass == disabled) {
	274                 dev_dbg(&amp;dev-&gt;dev,
	275                     "BAR %d: reserving %pr (d=%d, p=%d)\n",
	276                     idx, r, disabled, pass);
	277                 if (pci_claim_resource(dev, idx) &lt; 0) {
	278                     if (r-&gt;flags &amp; IORESOURCE_PCI_FIXED) {
	279                         dev_info(&amp;dev-&gt;dev, "BAR %d %pR is immovable\n",
	280                              idx, r);
	281                     } else {
	282                         /* We'll assign a new address later */
	283                         pcibios_save_fw_addr(dev,
	284                                 idx, r-&gt;start);
	285                         r-&gt;end -= r-&gt;start;
	286                         r-&gt;start = 0;
	287                     }
	288                 }
	289             }
	290         }
	291     if (!pass) {
	292         r = &amp;dev-&gt;resource[PCI_ROM_RESOURCE];
	293         if (r-&gt;flags &amp; IORESOURCE_ROM_ENABLE) {
	294             /* Turn the ROM off, leave the resource region,
	295              * but keep it unregistered. */
	296             u32 reg;
	297             dev_dbg(&amp;dev-&gt;dev, "disabling ROM %pR\n", r);
	298             r-&gt;flags &amp;= ~IORESOURCE_ROM_ENABLE;
	299             pci_read_config_dword(dev, dev-&gt;rom_base_reg, &amp;reg);
	300             pci_write_config_dword(dev, dev-&gt;rom_base_reg,
	301                         reg &amp; ~PCI_ROM_ADDRESS_ENABLE);
	302         }
	303     }
	304 } 

对于每一个PCI设备的6个常规的区间，如果这些区间已经生效（可以接受访问，line269～line272中得到的disabled为0）就在第一趟（pass=0）扫描时候进行分配地址资源，否则就在第二趟（pass=1）时分配。如果时扩充ROM区间（第七个区间）则要在第一趟（pass=0）时进行处理。

这些设备区间存在几种可能。第一种时已经分配过资源的（parent指向父节点，line265），对于这种直接跳过；第二种是起始地址为0（line267），这种区间可能不需要分配资源，或者其父节点不能满足其要求，也跳过；第三种时要分配地址的区间了，在上一段条件的约束下，在pass0或者1时直接通过pci_claim_resource分配资源。这个函数在上篇已经讲过了。

如果发现不能在当前的起始地址从父节点上分配所需的资源，即范围不符或者发生冲突，就先将该区间的平移到起始地址为0的地方（line281~line286）,这些区间的起始地址需要加以变更才行。这里需要说明，这些区间的地址资源之所以分配失败并不是由于父节点中的资源短缺，而是因为对起始地址的要求不能满足。BIOS在确定父节点窗口大小时是经过计算的，加上对窗口位置的对齐，父节点的窗口一般都要比实际需要的大。所以，只要允许将区间适当的平移，就一定能分配到所需要的地址资源。

对于ROM区间，则在第一次扫描时进行关闭（pass=0）。ROM区间一般只是在初始化时由BIOS或者具体的设备驱动程序使用，所以现在可以关闭了。如果需要时还可以在设备驱动程序中打开。但是它的地址空间其实还没有分配，对ROM的地址空间分配是和上面没有分配成功的0~6个地址区间一起进行的。

pcibios_resource_survey的工作到这里就完全结束了（这里忽略了e820，和APCI的处理），也意味着pci_subsys_init()的工作也大致结束了。对于不能在原有起始地址上分配所需地址资源的区间，即起始地址已经变成0的区间，以及ROM区间的地址空间要通过后续的工作来分配，注意，资源没有分配成功的总线也是在后面一起处理的。但是具体这个工作在哪里执行呢？以前版本内核确实是在pcibios_resource_survey里面直接调用的，新的内核好像改了，也有交给initcall了，具体如下：

	//arch/x86/pci/i386.c
	354 static int __init pcibios_assign_resources(void)
	355 {
	356     struct pci_bus *bus;
	357  
	358     if (!(pci_probe &amp; PCI_ASSIGN_ROMS))
	359         list_for_each_entry(bus, &amp;pci_root_buses, node)
	360             pcibios_allocate_rom_resources(bus);
	361  
	362     pci_assign_unassigned_resources();
	363     pcibios_fw_addr_list_del();
	364  
	365     return 0;
	366 }
	367  
	368 /**
	369  * called in fs_initcall (one below subsys_initcall),
	370  * give a chance for motherboard reserve resources
	371  */  
	372 fs_initcall(pcibios_assign_resources);

看注释的话已经很明了了，对于pcibios_allocate_rom_resources（）就是进行ROM区间的地址分配，大致工作和分配0~6区间的流程差不多，这里就不分析了。重要的时这里的pci_assign_unassigned_resources()，如果说前面所谓的“分配”时一种追认，这里这个才是真正的分配！下篇见！

##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

