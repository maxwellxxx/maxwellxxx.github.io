---
layout: post
title: Linux设备驱动模型及其他（11)
description: 在PCI设备地址空间分配总线资源和设备资源（真正分配）
category: manual
---

>前两篇讲到已经对系统上的PCI总线和设备完成了资源的验证，对于验证成功资源，会设置其resource结构，试图建立起一棵资源树。对于没有验证验证成功的总线资源，将其flags，start，end设为0，对于没有验证成功的设备资源平移资源区间到0开始，以已经后续处理。

##真正分配资源

上篇看到这个函数：

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

已经讲过pcibios_allocate_rom_resources是对ROM资源进行验证，而pci_assign_unassigned_resources()就是对未分配（“验证”）成功的PCI总线资源，PCI设备（包括ROM）资源进行分配，现在这个函数可不像从前那么质朴了，以前的实现可以说简单明了。我们来pci_assign_unassigned_resources()样子：

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

首先是通过find_free_bus_resource验证下总想的I/O地址是不是真的没有分配成功（仅判断resource->parent是否有值），如果I/O资源分配成功了，就没什么好修正的了，直接退出函数。如果没有分配成功就说明是有冲突的，就需要重新计算区间大小，区间大小就是总线上以及下层设备需要的区间大小总和。这里需要注意__pci_bus_size_bridges是深度优先的，所以这里的计算都是从最下层的总线开始逐层计算到上层的，只有这样才能正确修正每一层的区间大小。计算过程中还有一个就是区间对齐的问题，对于总线的I/O资源是要求4K对齐的，所以长度也是4K对齐。重新计算后得到的大小就重新赋给总线的resource。

修正完I/O的区间大小，就还要修正MEM的区间，工作有pbus_size_mem来完成，大致处理和I/O修正相同，只不过pbus_size_mem会被调用两次，分配用于查找可预期的存储空间和普通的存储空间。

解决完总线资源冲突，就可以来为没有分配成功的总线和设备资源进行分配了，如果分配成功的话还需要将resource更新到PCI设备的配置空间寄存器中。回到pci_assign_unassigned_root_bus_resources中，调用__pci_bus_assign_resources来分配资源：

	//drivers/pci/setup-bus.c
	1277 void __pci_bus_assign_resources(const struct pci_bus *bus,
	1278                 struct list_head *realloc_head,
	1279                 struct list_head *fail_head)
	1280 {  
	1281     struct pci_bus *b;
	1282     struct pci_dev *dev;
	1283    
	1284     pbus_assign_resources_sorted(bus, realloc_head, fail_head);
	1285    
	1286     list_for_each_entry(dev, &bus->devices, bus_list) {
	1287         b = dev->subordinate;
	1288         if (!b)
	1289             continue;
	1290    
	1291         __pci_bus_assign_resources(b, realloc_head, fail_head);
	1292    
	1293         switch (dev->class >> 8) {
	1294         case PCI_CLASS_BRIDGE_PCI:
	1295             if (!pci_is_enabled(dev))
	1296                 pci_setup_bridge(b);
	1297             break;
	1298    
	1299         case PCI_CLASS_BRIDGE_CARDBUS:
	1300             pci_setup_cardbus(b);
	1301             break;
	1302    
	1303         default:
	1304             dev_info(&dev->dev, "not setting up bridge for bus %04x:%02x\n",
	1305                  pci_domain_nr(b), b->number);
	1306             break;
	1307         }
	1308     }
	1309 } 

看pci_assign_unassigned_root_bus_resources的注释可以看到，这个函数一反常态并不是深度由先的。它首先为上层总线调用pbus_assign_resources_sorted，来分配总线上各个设备资源，接着如果设备上存在PCI桥再递归调用__pci_bus_assign_resources，但也是先分配自身设备的资源，这样从上层到下层逐层分配。最后为每个PCI桥调用pci_setup_bridge更新PCI桥设备配置寄存器。我们一步步来看下，首先是pbus_assign_resources_sorted：

	 //drivers/pci/setup-bus.c 
	 454 static void pbus_assign_resources_sorted(const struct pci_bus *bus
	 455                      struct list_head *realloc_head,
	 456                      struct list_head *fail_head)
	 457 {   
	 458     struct pci_dev *dev;  
	 459     LIST_HEAD(head);
	 460     
	 461     list_for_each_entry(dev, &bus->devices, bus_list)
	 462         __dev_sort_resources(dev, &head);
	 463     
	 464     __assign_resources_sorted(&head, realloc_head, fail_head);
	 465 } 

__dev_sort_resources将总线上所有没有分配成功的设备添加到head链表中，添加时首先计算设备资源的对齐因子，然后按照对齐因子由大到小插入到链表中。__assign_resources_sorted则统一对head链表中的资源进行分配：

	 //drivers/pci/setup-bus.c  
	 343 static void __assign_resources_sorted(struct list_head *head,
	 344                  struct list_head *realloc_head,
	 345                  struct list_head *fail_head)
	 346 {   
	 368     LIST_HEAD(save_head);
	 369     LIST_HEAD(local_fail_head);
	 370     struct pci_dev_resource *save_res;
	 371     struct pci_dev_resource *dev_res, *tmp_res;
	 372     unsigned long fail_type;
	 373     
	 374     /* Check if optional add_size is there */
	 375     if (!realloc_head || list_empty(realloc_head))
	 376         goto requested_and_reassign;
	 ..................................
	 ..................................
	 432 requested_and_reassign:
	 433     /* Satisfy the must-have resource requests */
	 434     assign_requested_resources_sorted(head, fail_head);
	 435    
	 436     /* Try to satisfy any additional optional resource
	 437         requests */
	 438     if (realloc_head)
	 439         reassign_resources_sorted(realloc_head, head);
	 440     free_list(head);
	 441 } 

	 263 /**          
	 264  * assign_requested_resources_sorted() - satisfy resource requests
	 265  *           
	 266  * @head : head of the list tracking requests for resources
	 267  * @fail_head : head of the list tracking requests that could
	 268  *      not be allocated
	 269  *           
	 270  * Satisfy resource requests of each element in the list. Add
	 271  * requests that could not satisfied to the failed_list.
	 272  */          
	 273 static void assign_requested_resources_sorted(struct list_head *head,
	 274                  struct list_head *fail_head)
	 275 {            
	 276     struct resource *res;
	 277     struct pci_dev_resource *dev_res;
	 278     int idx;  
	 279              
	 280     list_for_each_entry(dev_res, head, list) {
	 281         res = dev_res->res;
	 282         idx = res - &dev_res->dev->resource[0]; 
	 283         if (resource_size(res) &&
	 284             pci_assign_resource(dev_res->dev, idx)) { 
	 285             if (fail_head) {
	 286                 /*
	 287                  * if the failed res is for ROM BAR, and it will
	 288                  * be enabled later, don't add it to the list
	 289                  */
	 290                 if (!((idx == PCI_ROM_RESOURCE) &&
	 291                       (!(res->flags & IORESOURCE_ROM_ENABLE))))
	 292                     add_to_list(fail_head,
	 293                             dev_res->dev, res,
	 294                             0 /* don't care */,
	 295                             0 /* don't care */);
	 296             }
	 297             reset_resource(res);
	 298         }    
	 299     }        
	 300 } 
这个函数涉及到很多复杂的操作，以后再进行分析，我们直接看最后的assign_requested_resources_sorted，这个函数就是对为分配列表进行处理。

函数首先通过链表中pci_dev_resource的信息获得resource结构和该资源的索引值（line281～line282），然后通过pci_assign_resource来进行分配资源，如果不成功就加入到fail链表，并将resource的start，end，flags置0（line285~297）.

	//drivers/pci/setup-res.c
	264 int pci_assign_resource(struct pci_dev *dev, int resno)
	265 {   
	266     struct resource *res = dev->resource + resno;
	267     resource_size_t align, size;
	268     int ret;
	269     
	270     res->flags |= IORESOURCE_UNSET;
	271     align = pci_resource_alignment(dev, res);
	272     if (!align) {
	273         dev_info(&dev->dev, "BAR %d: can't assign %pR (bogus alignment)\n",
	274              resno, res);
	275         return -EINVAL;
	276     }
	277     
	278     size = resource_size(res);
	279     ret = _pci_assign_resource(dev, resno, size, align);
	280     
	281     /*
	282      * If we failed to assign anything, let's try the address
	283      * where firmware left it.  That at least has a chance of
	284      * working, which is better than just leaving it disabled.
	285      */
	286     if (ret < 0) {
	287         dev_info(&dev->dev, "BAR %d: no space for %pR\n", resno, res);
	288         ret = pci_revert_fw_address(res, dev, resno, size);
	289     }
	290     
	291     if (ret < 0) {
	292         dev_info(&dev->dev, "BAR %d: failed to assign %pR\n", resno,
	293              res);
	294         return ret;
	295     }
	296     
	297     res->flags &= ~IORESOURCE_UNSET;
	298     res->flags &= ~IORESOURCE_STARTALIGN;
	299     dev_info(&dev->dev, "BAR %d: assigned %pR\n", resno, res);
	300     if (resno < PCI_BRIDGE_RESOURCES)
	301         pci_update_resource(dev, resno);
	302     
	303     return 0;
	304 }   
	305 EXPORT_SYMBOL(pci_assign_resource);

看line279调用_pci_assign_resource来进行分配工作，如果分配成功那么就需要将resource更新到PCI配置寄存器，但PCI桥设备是例外，这个工作由pci_setup_bridge负责。（line301）

	_pci_assign_resource
	   |
	   |
	   __pci_assign_resource	

	//drivers/pci/setup-res.c
	200 static int __pci_assign_resource(struct pci_bus *bus, struct pci_dev *dev,
	201         int resno, resource_size_t size, resource_size_t align)
	202 {   
	203     struct resource *res = dev->resource + resno;
	204     resource_size_t min;
	205     int ret;
	206     
	207     min = (res->flags & IORESOURCE_IO) ? PCIBIOS_MIN_IO : PCIBIOS_MIN_MEM;
	208     
	209     /*
	210      * First, try exact prefetching match.  Even if a 64-bit
	211      * prefetchable bridge window is below 4GB, we can't put a 32-bit
	212      * prefetchable resource in it because pbus_size_mem() assumes a
	213      * 64-bit window will contain no 32-bit resources.  If we assign
	214      * things differently than they were sized, not everything will fit.
	215      */
	216     ret = pci_bus_alloc_resource(bus, res, size, align, min,
	217                      IORESOURCE_PREFETCH | IORESOURCE_MEM_64,
	218                      pcibios_align_resource, dev);
	219     if (ret == 0)
	220         return 0;
	221     
	222     /*
	223      * If the prefetchable window is only 32 bits wide, we can put
	224      * 64-bit prefetchable resources in it.
	225      */
	226     if ((res->flags & (IORESOURCE_PREFETCH | IORESOURCE_MEM_64)) ==
	227          (IORESOURCE_PREFETCH | IORESOURCE_MEM_64)) {
	228         ret = pci_bus_alloc_resource(bus, res, size, align, min,
	229                          IORESOURCE_PREFETCH,
	230                          pcibios_align_resource, dev);
	231         if (ret == 0)
	232             return 0;
	233     }
	234     
	235     /*
	236      * If we didn't find a better match, we can put any memory resource
	237      * in a non-prefetchable window.  If this resource is 32 bits and
	238      * non-prefetchable, the first call already tried the only possibility
	239      * so we don't need to try again.
	240      */
	241     if (res->flags & (IORESOURCE_PREFETCH | IORESOURCE_MEM_64))
	242         ret = pci_bus_alloc_resource(bus, res, size, align, min, 0,
	243                          pcibios_align_resource, dev);
	244     
	245     return ret;
	246 } 

	 // arch/x86/include/asm/pci.h   
	 60 extern unsigned long pci_mem_start;           
	 61 #define PCIBIOS_MIN_IO      0x1000            
	 62 #define PCIBIOS_MIN_MEM     (pci_mem_start) 

__pci_assign_resource调用pci_bus_alloc_resource（）来为给定的设备从其所在的总线分配资源。line207，在分配时，还有个问题就是按照要求I/O区间的地址不能低于4K，MEM的地址也不能低于某个值（以前是256M）。还有就是，对于地址区间还有个是否必须满足“可预取”的问题，所以line216，line226先以IORESOURCE_PREFETCH为参调用一次pci_bus_alloc_resource（）如果失败在到line241放宽条件。看pci_bus_alloc_resource具体做了什么：

	pci_bus_alloc_resource
	  |
	  |
	  pci_bus_alloc_from_region

	// drivers/pci/bus.c
	132 static int pci_bus_alloc_from_region(struct pci_bus *bus, struct resource *res,   
	133         resource_size_t size, resource_size_t align,
	134         resource_size_t min, unsigned long type_mask,
	135         resource_size_t (*alignf)(void *,
	136                       const struct resource *,
	137                       resource_size_t,
	138                       resource_size_t),
	139         void *alignf_data,
	140         struct pci_bus_region *region)
	141 {  
	142     int i, ret;
	143     struct resource *r, avail;
	144     resource_size_t max;
	145    
	146     type_mask |= IORESOURCE_TYPE_BITS;
	147    
	148     pci_bus_for_each_resource(bus, r, i) {
	149         if (!r)
	150             continue;
	151    
	152         /* type_mask must match */
	153         if ((res->flags ^ r->flags) & type_mask)
	154             continue;
	155    
	156         /* We cannot allocate a non-prefetching resource
	157            from a pre-fetching area */
	158         if ((r->flags & IORESOURCE_PREFETCH) &&
	159             !(res->flags & IORESOURCE_PREFETCH))
	160             continue;
	161    
	162         avail = *r;
	163         pci_clip_resource_to_region(bus, &avail, region);
	164    
	165         /*
	166          * "min" is typically PCIBIOS_MIN_IO or PCIBIOS_MIN_MEM to
	167          * protect badly documented motherboard resources, but if
	168          * this is an already-configured bridge window, its start
	169          * overrides "min".
	170          */
	171         if (avail.start)
	172             min = avail.start;
	173    
	174         max = avail.end;
	175    
	176         /* Ok, try it out.. */
	177         ret = allocate_resource(r, res, size, min, max,
	178                     align, alignf, alignf_data);
	179         if (ret == 0)
	180             return 0;
	181     }

 	669 /**
 	670  * allocate_resource - allocate empty slot in the resource tree given range & alignment.
 	671  *  The resource will be reallocated with a new size if it was already allocated
 	672  * @root: root resource descriptor
 	673  * @new: resource descriptor desired by caller
 	674  * @size: requested resource region size
 	675  * @min: minimum boundary to allocate
	676  * @max: maximum boundary to allocate
 	677  * @align: alignment requested, in bytes
 	678  * @alignf: alignment function, optional, called if not NULL
 	679  * @alignf_data: arbitrary data to pass to the @alignf function
 	680  */
 	681 int allocate_resource(struct resource *root, struct resource *new,
 	682               resource_size_t size, resource_size_t min,
 	683               resource_size_t max, resource_size_t align,
 	684               resource_size_t (*alignf)(void *,
 	685                         const struct resource *,
 	686                         resource_size_t,
 	687                         resource_size_t),
 	688               void *alignf_data)
 	689 {  
 	690     int err;
 	691     struct resource_constraint constraint;
 	692    
 	693     if (!alignf)
 	694         alignf = simple_align_resource;
 	695    
 	696     constraint.min = min;
 	697     constraint.max = max;
 	698     constraint.align = align;
 	699     constraint.alignf = alignf;
 	700     constraint.alignf_data = alignf_data;
 	701    
 	702     if ( new->parent ) {
 	703         /* resource is already allocated, try reallocating with
 	704            the new constraints */
 	705         return reallocate_resource(root, new, size, &constraint);
 	706     }
 	707    
 	708     write_lock(&resource_lock);
 	709     err = find_resource(root, new, size, &constraint);
 	710     if (err >= 0 && __request_resource(root, new))
 	711         err = -EBUSY;
 	712     write_unlock(&resource_lock);
 	713     return err;
 	714 }

遍历总线上类型相同的资源区，如果符合条件就由allocate_resource来执行分配，比较简单，可以看到是直接用__request_resource来进行分配的（忽略重新分配的情况reallocate_resource），这个函数我们已经介绍过了。

分配成功的话就返回到pci_assign_resource中，到line297～line301，设置资源区标志位，如果对非PCI桥设备，调用pci_update_resource（）将新的resource信息写入到设备配置寄存器，主要就是组合寄存器的信息，然后用大家熟悉的pci_wriite_config_dword等操作设备配置寄存器。

至此，就已经为总线上的设备分配好资源了，回到__pci_bus_assign_resources中，如果大家还记得的话，PCI桥设备的resource虽然已经分配了，但还没有将信息写入到配置寄存器，这部分工作就由pci_setup_bridge负责，它主要的工作是：
<ul>
<li>启用PCI桥中I/O和MEM访问。</li>
<li>设置PCI桥对总线的控制能力。</li>
</ul>
注意这里仅仅是开启了PCI桥的相关设备，虽然前面已经将资源信息写入到PCI设备，但是对与普通PCI设备，并不意味这可以这些资源可以被访问了。配置寄存器中还有两个控制位，即PCI_COMMAND_IO和PCI_COMMAND_MEMORY，控制着设备所有的I/O和存储的地址空间。最终要将这两位置为1才能使这些区间真正的连接到PCI总线上。但是这里内核并没有做这一步，实际上是留给了具体的设备驱动程序。

到此对PCI总线的初始化就全部完成了。内存中已经建立了多棵数记录这PCI总线和PCI设备的关系（一般只有一大棵，因为一般只有一条跟总线），而每个设备的每个地址区间已经有了总线地址，而且也有了资源树。

接下来就需要分析这些树的关系，设计到驱动，驱动模型，资源树各个细节！下篇见！


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

