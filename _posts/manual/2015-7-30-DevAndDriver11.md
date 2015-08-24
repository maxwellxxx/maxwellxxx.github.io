---
layout: post
title: Linux设备驱动模型及其他（待合并)
description: 为PCI设备地址空间分配总线地址
category: manual
---

首先看下

	//arch/x86/pci/i386.c 
	354 static int __init pcibios_assign_resources(void)
	355 {    
	356     struct pci_bus *bus;
	357     
	358     if (!(pci_probe & PCI_ASSIGN_ROMS))
	359         list_for_each_entry(bus, &pci_root_buses, node)
	360             pcibios_allocate_rom_resources(bus);
	361  
	362     pci_assign_unassigned_resources();
	363     pcibios_fw_addr_list_del();
	364     
	365     return 0;
	366 }  

pci_assign_unassigned_resources()就是对未分配（“验证”）成功的PCI总线资源，PCI设备（包括ROM）资源进行分配，现在这个函数可不像从前那么质朴了，以前的实现可以说简单明了，我们来看看它现在的样子：

	//drivers/pci/setup-bus.c
	1669 void __init pci_assign_unassigned_resources(void)
	1670 { 
	1671     struct pci_bus *root_bus;
	1672 
	1673     list_for_each_entry(root_bus, &pci_root_buses, node)
	1674         pci_assign_unassigned_root_bus_resources(root_bus);
	1675 } 

	1573 /*
	1574  * first try will not touch pci bridge res
	1575  * second and later try will clear small leaf bridge res
	1576  * will stop till to the max depth if can not find good one
	1577  */
	1578 void pci_assign_unassigned_root_bus_resources(struct pci_bus *bus)
	1579 {
	1580     LIST_HEAD(realloc_head); /* list of resources that
	1581                     want additional resources */
	1582     struct list_head *add_list = NULL;
	1583     int tried_times = 0;
	1584     enum release_type rel_type = leaf_only;
	1585     LIST_HEAD(fail_head);
	1586     struct pci_dev_resource *fail_res;
	1587     unsigned long type_mask = IORESOURCE_IO | IORESOURCE_MEM |
	1588                   IORESOURCE_PREFETCH | IORESOURCE_MEM_64;
	1589     int pci_try_num = 1;
	1590     enum enable_type enable_local;
	1591 
	1592     /* don't realloc if asked to do so */
	1593     enable_local = pci_realloc_detect(bus, pci_realloc_enable);
	1594     if (pci_realloc_enabled(enable_local)) {
	1595         int max_depth = pci_bus_get_depth(bus);
	1596 
	1597         pci_try_num = max_depth + 1;
	1598         dev_printk(KERN_DEBUG, &bus->dev,
	1599                "max bus depth: %d pci_try_num: %d\n",
	1600                max_depth, pci_try_num);
	1601     }
	1602 
	1603 again:
	1604     /*
	1605      * last try will use add_list, otherwise will try good to have as
	1606      * must have, so can realloc parent bridge resource
	1607      */
	1608     if (tried_times + 1 == pci_try_num)
	1609         add_list = &realloc_head;
	1610     /* Depth first, calculate sizes and alignments of all
	1611        subordinate buses. */
	1612     __pci_bus_size_bridges(bus, add_list);
	1613 
	1614     /* Depth last, allocate resources and update the hardware. */
	1615     __pci_bus_assign_resources(bus, add_list, &fail_head);
	1616     if (add_list)
	1617         BUG_ON(!list_empty(add_list));
	1618     tried_times++;
	1619 
	1620     /* any device complain? */
	1621     if (list_empty(&fail_head))
	1622         goto dump;
	1623 
	1624     if (tried_times >= pci_try_num) {
	1625         if (enable_local == undefined)
	1626             dev_info(&bus->dev, "Some PCI device resources are unassigned, try booting with pci=realloc\n"); 
	1627         else if (enable_local == auto_enabled)
	1628             dev_info(&bus->dev, "Automatically enabled pci realloc, if you have problem, try booting with pci=realloc=off\n");
	1629 
	1630         free_list(&fail_head);
	1631         goto dump;
	1632     }
	1633 
	1634     dev_printk(KERN_DEBUG, &bus->dev,
	1635            "No. %d try to assign unassigned res\n", tried_times + 1);
	1636 
	1637     /* third times and later will not check if it is leaf */
	1638     if ((tried_times + 1) > 2)
	1639         rel_type = whole_subtree;
	1640 
	1641     /*
	1642      * Try to release leaf bridge's resources that doesn't fit resource of
	1643      * child device under that bridge
	1644      */
	1645     list_for_each_entry(fail_res, &fail_head, list)
	1646         pci_bus_release_bridge_resources(fail_res->dev->bus,
	1647                          fail_res->flags & type_mask,
	1648                          rel_type);
	1649 
	1650     /* restore size and flags */
	1651     list_for_each_entry(fail_res, &fail_head, list) {
	1652         struct resource *res = fail_res->res;
	1653 
	1654         res->start = fail_res->start;
	1655         res->end = fail_res->end;
	1656         res->flags = fail_res->flags;
	1657         if (fail_res->dev->subordinate)
	1658             res->flags = 0;
	1659     }
	1660     free_list(&fail_head);
	1661 
	1662     goto again;
	1663 
	1664 dump:
	1665     /* dump the resource on buses */
	1666     pci_bus_dump_resources(bus);
	1667 }

内核会对系统中每一条根总线执行一次pci_assign_unassigned_root_bus_resources，而这个函数主要就是负责对未分配资源的总线和设备进行真正意义上的资源分配，而复杂的地方就在这里。在2.4.x的时候，对没有分配成功资源的总线是不加以处理的，也可以说是内核的一个bug，而在3.x版本中早对这个问题进行了比较好的修复。函数开头的一些操作用用于设置分配尝试次数，具体作用没有搞清楚。我们直接来看关键的部分。

首先是pci_bus_size_bridges()，还记得第九篇中提到过，如果为总线（也可以说是桥）“分配”资源没有成功的话，会将其相应的resource的flags、start、end置为0，以便后面重新分配资源。而pci_bus_size_bridges()就是为没有分配成功的桥设备资源（总线）解决冲突的。我们来看下这个函数：

	//drivers/pci/setup-bus.c

	1271 void pci_bus_size_bridges(struct pci_bus *bus)
	1272 {   
	1273     __pci_bus_size_bridges(bus, NULL); 
	1274 }   


	1153 void __pci_bus_size_bridges(struct pci_bus *bus, struct list_head *realloc_head)
	1154 {   
	1155     struct pci_dev *dev;
	1156     unsigned long mask, prefmask, type2 = 0, type3 = 0;
	1157     resource_size_t additional_mem_size = 0, additional_io_size = 0;
	1158     struct resource *b_res;
	1159     int ret;
	1160     
	1161     list_for_each_entry(dev, &bus->devices, bus_list) {
	1162         struct pci_bus *b = dev->subordinate;
	1163         if (!b)
	1164             continue;
	1165     
	1166         switch (dev->class >> 8) {
	1167         case PCI_CLASS_BRIDGE_CARDBUS:
	1168             pci_bus_size_cardbus(b, realloc_head);
	1169             break;
	1170     
	1171         case PCI_CLASS_BRIDGE_PCI:
	1172         default:
	1173             __pci_bus_size_bridges(b, realloc_head);
	1174             break;
	1175         }
	1176     }
	1177     
	1178     /* The root bus? */
	1179     if (pci_is_root_bus(bus))
	1180         return;
	1181     
	1182     switch (bus->self->class >> 8) {
	1183     case PCI_CLASS_BRIDGE_CARDBUS:
	1184         /* don't size cardbuses yet. */
	1185         break;
	1186     
	1187     case PCI_CLASS_BRIDGE_PCI:
	1188         pci_bridge_check_ranges(bus);
	1189         if (bus->self->is_hotplug_bridge) {
	1190             additional_io_size  = pci_hotplug_io_size;
	1191             additional_mem_size = pci_hotplug_mem_size;
	1192         }
	1193         /* Fall through */
	1194     default:
	1195         pbus_size_io(bus, realloc_head ? 0 : additional_io_size,
	1196                  additional_io_size, realloc_head);
	1197     
	1198         /*
	1199          * If there's a 64-bit prefetchable MMIO window, compute  
	1200          * the size required to put all 64-bit prefetchable
	1201          * resources in it.
	1202          */
	1203         b_res = &bus->self->resource[PCI_BRIDGE_RESOURCES];
	1204         mask = IORESOURCE_MEM;
	1205         prefmask = IORESOURCE_MEM | IORESOURCE_PREFETCH;
	1206         if (b_res[2].flags & IORESOURCE_MEM_64) {
	1207             prefmask |= IORESOURCE_MEM_64;
	1208             ret = pbus_size_mem(bus, prefmask, prefmask,
	1209                   prefmask, prefmask,
	1210                   realloc_head ? 0 : additional_mem_size,
	1211                   additional_mem_size, realloc_head);
	1212     
	1213             /*
	1214              * If successful, all non-prefetchable resources
	1215              * and any 32-bit prefetchable resources will go in
	1216              * the non-prefetchable window.
	1217              */
	1218             if (ret == 0) {
	1219                 mask = prefmask;
	1220                 type2 = prefmask & ~IORESOURCE_MEM_64;
	1221                 type3 = prefmask & ~IORESOURCE_PREFETCH;
	1222             }
	1223         }
	1224     
	1225         /*
	1226          * If there is no 64-bit prefetchable window, compute the
	1227          * size required to put all prefetchable resources in the
	1228          * 32-bit prefetchable window (if there is one).
	1229          */
	1230         if (!type2) {
	1231             prefmask &= ~IORESOURCE_MEM_64;
	1232             ret = pbus_size_mem(bus, prefmask, prefmask,
	1233                      prefmask, prefmask,
	1234                      realloc_head ? 0 : additional_mem_size,
	1235                      additional_mem_size, realloc_head);
	1236    
	1237             /*
	1238              * If successful, only non-prefetchable resources
	1239              * will go in the non-prefetchable window.
	1240              */
	1241             if (ret == 0)
	1242                 mask = prefmask;
	1243             else
	1244                 additional_mem_size += additional_mem_size;
	1245    
	1246             type2 = type3 = IORESOURCE_MEM;
	1247         }
	1248    
	1249         /*
	1250          * Compute the size required to put everything else in the
	1251          * non-prefetchable window.  This includes:
	1252          *
	1253          *   - all non-prefetchable resources
	1254          *   - 32-bit prefetchable resources if there's a 64-bit
	1255          *     prefetchable window or no prefetchable window at all
	1256          *   - 64-bit prefetchable resources if there's no
	1257          *     prefetchable window at all
	1258          *
	1259          * Note that the strategy in __pci_assign_resource() must
	1260          * match that used here.  Specifically, we cannot put a
	1261          * 32-bit prefetchable resource in a 64-bit prefetchable
	1262          * window.
	1263          */
	1264         pbus_size_mem(bus, mask, IORESOURCE_MEM, type2, type3,
	1265                 realloc_head ? 0 : additional_mem_size,
	1266                 additional_mem_size, realloc_head);
	1267         break;
	1268     }
	1269 }

可以看出来，这个函数也是深度优先的，扫描根总线上的所有总线，递归调用__pci_bus_size_bridges，对于每一条不是根总线的总线，还需要针对桥类型做不同的操作。如果设备类型是PCI_CLASS_BRIDGE_PCI，则调用pci_bridge_check_ranges。另外后面的default是一定要处理的。首先还是来看pci_bridge_check_ranges：

	 684 /* Check whether the bridge supports optional I/O and
	 685    prefetchable memory ranges. If not, the respective
	 686    base/limit registers must be read-only and read as 0. */
	 687 static void pci_bridge_check_ranges(struct pci_bus *bus)
	 688 {   
	 689     u16 io;
	 690     u32 pmem;
	 691     struct pci_dev *bridge = bus->self;
	 692     struct resource *b_res;
	 693     
	 694     b_res = &bridge->resource[PCI_BRIDGE_RESOURCES];
	 695     b_res[1].flags |= IORESOURCE_MEM;
	 696     
	 697     pci_read_config_word(bridge, PCI_IO_BASE, &io);
	 698     if (!io) {
	 699         pci_write_config_word(bridge, PCI_IO_BASE, 0xe0f0);
	 700         pci_read_config_word(bridge, PCI_IO_BASE, &io);
	 701         pci_write_config_word(bridge, PCI_IO_BASE, 0x0);
	 702     }
	 703     if (io)
	 704         b_res[0].flags |= IORESOURCE_IO;
	 705     
	 706     /*  DECchip 21050 pass 2 errata: the bridge may miss an address
	 707         disconnect boundary by one PCI data phase.
	 708         Workaround: do not use prefetching on this device. */
	 709     if (bridge->vendor == PCI_VENDOR_ID_DEC && bridge->device == 0x0001)
	 710         return;
	 711     
	 712     pci_read_config_dword(bridge, PCI_PREF_MEMORY_BASE, &pmem);
	 713     if (!pmem) {
	 714         pci_write_config_dword(bridge, PCI_PREF_MEMORY_BASE,
	 715                            0xffe0fff0);
	 716         pci_read_config_dword(bridge, PCI_PREF_MEMORY_BASE, &pmem);
	 717         pci_write_config_dword(bridge, PCI_PREF_MEMORY_BASE, 0x0);
	 718     }
	 719     if (pmem) {
	 720         b_res[2].flags |= IORESOURCE_MEM | IORESOURCE_PREFETCH;
	 721         if ((pmem & PCI_PREF_RANGE_TYPE_MASK) ==
	 722             PCI_PREF_RANGE_TYPE_64) {
	 723             b_res[2].flags |= IORESOURCE_MEM_64;
	 724             b_res[2].flags |= PCI_PREF_RANGE_TYPE_64;
	 725         }
	 726     }
	 727     
	 728     /* double check if bridge does support 64 bit pref */
	 729     if (b_res[2].flags & IORESOURCE_MEM_64) {
	 730         u32 mem_base_hi, tmp;
	 731         pci_read_config_dword(bridge, PCI_PREF_BASE_UPPER32,
	 732                      &mem_base_hi);
	 733         pci_write_config_dword(bridge, PCI_PREF_BASE_UPPER32,
	 734                            0xffffffff);
	 735         pci_read_config_dword(bridge, PCI_PREF_BASE_UPPER32, &tmp);
	 736         if (!tmp)
	 737             b_res[2].flags &= ~IORESOURCE_MEM_64;
	 738         pci_write_config_dword(bridge, PCI_PREF_BASE_UPPER32,
	 739                        mem_base_hi);
	 740     }
	 741 }

这个函数的功能很诡异。pci桥设备的resources[]7,8,9属于过滤窗口，也就是总线资源。这个函数是用来检查当前的总线是否支持I/O和“可预取”memory类型的窗口。如果不支持，则相应的寄存器是只读的，而且值为0。是不是感觉有点多此一举？其实不是的，还记得在第9篇中flags被设置为0的那些桥设备么？它们的flags就是从这里再次被设置回来的，所以这个工作必不可少。从代码可以看到对I/O,prefetch memory,还有64位的prefetch memory都做了相应的处理。还有就是所有的所有的bus都是支持memory窗口的，所以看line 695，是没有经过判断直接设置的。还有就是读取过寄存器，需要将其清空。。。。到这里，以前flags被设为0的PCI桥资源的flags就恢复了。

接下来就回到__pci_bus_size_bridges的default分支了，首先调用了pbus_size_io，这个函数是用来计算当前总线需要的I/O空间大小，主要用来修正那些有冲突的bus。

前面讲过，其实每个资源是肯定能够被成功分配的，分配或者说验证失败主要是因为冲突的原因，只要平移一下区间，总是能够找到资源的。但是含有一种情况，就是BIOS出了问题，可能导致bus资源区间长度出现了错误（过长）。对于这两种情况引起的bus资源分配不成功，就由pbus_size_io来进行修正，包括平移区间，如果有必要还要重新计算区间长度。

	 831 /** 
	 832  * pbus_size_io() - size the io window of a given bus
	 833  *  
	 834  * @bus : the bus
	 835  * @min_size : the minimum io window that must to be allocated
	 836  * @add_size : additional optional io window
	 837  * @realloc_head : track the additional io window on this list
	 838  *  
	 839  * Sizing the IO windows of the PCI-PCI bridge is trivial,
	 840  * since these windows have 1K or 4K granularity and the IO ranges
	 841  * of non-bridge PCI devices are limited to 256 bytes.
	 842  * We must be careful with the ISA aliasing though.
	 843  */ 
	 844 static void pbus_size_io(struct pci_bus *bus, resource_size_t min_size,
	 845         resource_size_t add_size, struct list_head *realloc_head)
	 846 {   
	 847     struct pci_dev *dev;
	 848     struct resource *b_res = find_free_bus_resource(bus, IORESOURCE_IO,
	 849                             IORESOURCE_IO);
	 850     resource_size_t size = 0, size0 = 0, size1 = 0;
	 851     resource_size_t children_add_size = 0;
	 852     resource_size_t min_align, align;
	 853     
	 854     if (!b_res)
	 855         return;
	 856     
	 857     min_align = window_alignment(bus, IORESOURCE_IO);
	 858     list_for_each_entry(dev, &bus->devices, bus_list) {
	 859         int i;
	 860     
	 861         for (i = 0; i < PCI_NUM_RESOURCES; i++) {
	 862             struct resource *r = &dev->resource[i];
	 863             unsigned long r_size;
	 864    
	 865             if (r->parent || !(r->flags & IORESOURCE_IO))
	 866                 continue;
	 867             r_size = resource_size(r);
	 868    
	 869             if (r_size < 0x400)
	 870                 /* Might be re-aligned for ISA */
	 871                 size += r_size;
	 872             else
	 873                 size1 += r_size;
	 874    
	 875             align = pci_resource_alignment(dev, r);
	 876             if (align > min_align)
	 877                 min_align = align;
	 878    
	 879             if (realloc_head)
	 880                 children_add_size += get_res_add_size(realloc_head, r);
	 881         }
	 882     }
	 883    
	 884     size0 = calculate_iosize(size, min_size, size1,
	 885             resource_size(b_res), min_align);
	 886     if (children_add_size > add_size)
	 887         add_size = children_add_size;
	 888     size1 = (!realloc_head || (realloc_head && !add_size)) ? size0 :
	 889         calculate_iosize(size, min_size, add_size + size1,
	 890             resource_size(b_res), min_align);
	 891     if (!size0 && !size1) {
	 892         if (b_res->start || b_res->end)
	 893             dev_info(&bus->self->dev, "disabling bridge window %pR to %pR (unused)\n",
	 894                  b_res, &bus->busn_res);
	 895         b_res->flags = 0;
	 896         return;
	 897     }
	 898    
	 899     b_res->start = min_align;
	 900     b_res->end = b_res->start + size0 - 1;
	 901     b_res->flags |= IORESOURCE_STARTALIGN;
	 902     if (size1 > size0 && realloc_head) {
	 903         add_to_list(realloc_head, bus->self, b_res, size1-size0,
	 904                 min_align);
	 905         dev_printk(KERN_DEBUG, &bus->self->dev, "bridge window %pR to %pR add_size %llx\n",
	 906                b_res, &bus->busn_res,
	 907                (unsigned long long)size1-size0);
	 908     }
	 909 }


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

