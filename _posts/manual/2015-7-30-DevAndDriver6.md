---
layout: post
title: Linux设备驱动模型及其他(6)
description: 扫描次层总线（枚举与配置设备）
category: manual
---

>上章讲到完成了根总线的扫描，并对根总线上的设备完成了基础的配置

##扫描次层总线（枚举设备）

完成了根总线的扫描后，如果总线上存在PCI桥，就必须对次层总线做扫描，代码如下：

	//drivers/pci/probe.c （pci_scan_child_bus（））
	1854     for (pass = 0; pass < 2; pass++)
	1855         list_for_each_entry(dev, &bus->devices, bus_list) {
	1856             if (pci_is_bridge(dev))
	1857                 max = pci_scan_bridge(bus, dev, max, pass);
	1858         }

这里循环了两次，可以为啥要循环两次呢？因为在这两次循环中，针对的情况是不同的，第一次扫描是针对由BIOS进行过处理的PCI桥，第二趟扫描则是针对未进过BIOS处理过的PCI桥。

###PCI桥设备上的“窗口”[1]

判断总线上的每个设备，如果是PCI桥设备，则对其执行pci_scan_bridge()，在分析代码前，还是先了解下PCI桥设备。PCI桥设备配置寄存器组与一般PCI设备不同。一般的PCI设备可以有6个地址区间，外加一个ROM区间，代表着设备实际存在的存储器或寄存器区间；而PCI桥，则本身不一定有存储器或寄存器区间，但是却有三个用于地址过滤的区间，每个地址过滤区间决定了一个地址窗口，从靠近CPU一侧发出的地址，如果落在PCI桥的某个窗口内，就可以穿过PCI桥而到达其连接的总线上。反过来，从总线一侧由PCI设备发出的地址（如果有的话），则要在相应的PCI桥的所有窗口之外才能穿过PCI桥到达靠近CPU的一侧。

此外，PCI桥的命令寄存器中还有“memory access enable”和“I/O access enable”两个控制位，当这个两个控制位为0时，这些窗口就全部关上了（两个方向都关上）。在未完成对PCI总线初始化前，还没有为PCI设备上的各个区间分配合适的总线地址时，正是因为这两个控制位为0，才不会对CPU一侧造成干扰。

显然，每个窗口的大小和位置应该是总想上各个相应区间的总和，而总线上的各个区间则应该基本上互相连续且互不重叠。因此，一个窗口实际上也是一个区间，所以也用一个resource数据结构表示。不过，说是相应各个区间的总和，实际上窗口一定是按照某种边界对齐的，其大小可能会大于实际的需要（从而造成了少量浪费）。下面就来看看这三个区间：
<ul>
<li>第一个区间是I/O地址的窗口。PCI桥上有两个8位的寄存器，即PCI_IO_BASE和PCI_IO_LIMIT,用来确定具体窗口的起点和终点。其中高4位就是16位I/O地址的最高4位，低4位固定为0（对于16位I/O地址而言，见下）。窗口大小为4kb的倍数，并与4KB边界对齐。其起点为PCI_IO_BASE的高4位添12个0，终点是PCI_IO_LIMIT的高4位添12位1。而PCI总线并不是专为i386处理器设计的，有些处理器可能用32位I/O地址空间。所以，如果具体的PCI设备支持32位I/O地址，则进一步通过PCI_IO_BASE_UPPER16和PCI_IO_LIMIT_UPPER16两个寄存器提供窗口起点的和终点的高16位。</li>
<li>第二个区间是存储器地址的窗口。寄存器PCI_MEMORY_BASE和PCI_MEMORY_LIMIT的构造与作用与上述的PCI_IO_BASE,PCI_IO_LIMIT相似，只不过是16位的寄存器，除最低4位为0外，其高位12位为32位存储器地址中的高12位。窗口大小则为1MB的倍数，并与1MB边界对齐。</li>
<li>第三个区间是“可预取”存储器地址的窗口。寄存器PCI_PREF_MEMORY_BASE和PCI_PREF_MEMORY_LIMIT的构造与作用根PCI_MEMORY_BASE,PCI_MEMORY_LIMIT相似，但是其最低4位并不总是0，而是以0表示32位地址，以1表示64位地址。对于64位地址，PCI桥上还有PCI_PREF_BASE_UPPER32和PCI_PREF_LIMIT_UPPER32两个32位的寄存器用于地址的高32位。</li>
</ul>

###扫描次层总线

内核通过调用pci_scan_bridge()来扫描PCI桥连接的总线，代码比较长：

	 //drivers/pci/probe.c
	 753 /*
	 754  * If it's a bridge, configure it and scan the bus behind it.
	 755  * For CardBus bridges, we don't scan behind as the devices will
	 756  * be handled by the bridge driver itself.
	 757  *
	 758  * We need to process bridges in two passes -- first we scan those
	 759  * already configured by the BIOS and after we are done with all of
	 760  * them, we proceed to assigning numbers to the remaining buses in
	 761  * order to avoid overlaps between old and new bus numbers.
	 762  */
	 763 int pci_scan_bridge(struct pci_bus *bus, struct pci_dev *dev, int max, int pass) 
	 764 {
	 765     struct pci_bus *child;
	 766     int is_cardbus = (dev->hdr_type == PCI_HEADER_TYPE_CARDBUS);
	 767     u32 buses, i, j = 0;
	 768     u16 bctl;
	 769     u8 primary, secondary, subordinate;
	 770     int broken = 0;
	 771  
	 772     pci_read_config_dword(dev, PCI_PRIMARY_BUS, &buses);
	 773     primary = buses & 0xFF;
	 774     secondary = (buses >> 8) & 0xFF;
	 775     subordinate = (buses >> 16) & 0xFF;
	 776  
	 .........................
	 802     if ((secondary || subordinate) && !pcibios_assign_all_busses() &&
	 803         !is_cardbus && !broken) {
	 804         unsigned int cmax;
	 805         /*
	 806          * Bus already configured by firmware, process it in the first
	 807          * pass and just note the configuration.
	 808          */
	 809         if (pass)
	 810             goto out;
	 811     
	 812         /*
	 813          * The bus might already exist for two reasons: Either we are
	 814          * rescanning the bus or the bus is reachable through more than
	 815          * one bridge. The second case can happen with the i450NX
	 816          * chipset.
	 817          */
	 818         child = pci_find_bus(pci_domain_nr(bus), secondary);
	 819         if (!child) {
	 820             child = pci_add_new_bus(bus, dev, secondary);
	 821             if (!child)
	 822                 goto out;
	 823             child->primary = primary;
	 824             pci_bus_insert_busn_res(child, secondary, subordinate);
	 825             child->bridge_ctl = bctl;
	 826         }
	 827     
	 828         cmax = pci_scan_child_bus(child);
	 829         if (cmax > subordinate)
	 830             dev_warn(&dev->dev, "bridge has subordinate %02x but max busn %02x\n",
	 831                  subordinate, cmax);
	 832         /* subordinate should equal child->busn_res.end */
	 833         if (subordinate > max)
	 834             max = subordinate;
	 835     } else {
	 836         /*
	 837          * We need to assign a number to this bus which we always
	 838          * do in the second pass.
	 839          */
	 840         if (!pass) {
	 841             if (pcibios_assign_all_busses() || broken || is_cardbus)
	 842                 /* Temporarily disable forwarding of the
	 843                    configuration cycles on all bridges in
	 844                    this bus segment to avoid possible
	 845                    conflicts in the second pass between two
	 846                    bridges programmed with overlapping
	 847                    bus ranges. */
	 848                 pci_write_config_dword(dev, PCI_PRIMARY_BUS,
	 849                                buses & ~0xffffff);
	 850             goto out;
	 851         }
	 852     
	 853         /* Clear errors */
	 854         pci_write_config_word(dev, PCI_STATUS, 0xffff);
	 855     
	 856         /* Prevent assigning a bus number that already exists.
	 857          * This can happen when a bridge is hot-plugged, so in
	 858          * this case we only re-scan this bus. */
	 859         child = pci_find_bus(pci_domain_nr(bus), max+1);
	 860         if (!child) {
	 861             child = pci_add_new_bus(bus, dev, max+1);
	 862             if (!child)
	 863                 goto out;
	 864             pci_bus_insert_busn_res(child, max+1, 0xff);
	 865         }
	 866         max++;
	 867         buses = (buses & 0xff000000)
	 868               | ((unsigned int)(child->primary)     <<  0)
	 869               | ((unsigned int)(child->busn_res.start)   <<  8)
	 870               | ((unsigned int)(child->busn_res.end) << 16);
	 871     
	 872         /*
	 873          * yenta.c forces a secondary latency timer of 176.
	 874          * Copy that behaviour here.
	 875          */
	 876         if (is_cardbus) {
	 877             buses &= ~0xff000000;
	 878             buses |= CARDBUS_LATENCY_TIMER << 24;
	 879         }
	 880     
	 881         /*
	 882          * We need to blast all three values with a single write.
	 883          */
	 884         pci_write_config_dword(dev, PCI_PRIMARY_BUS, buses);
	 885     
	 886         if (!is_cardbus) {
	 887             child->bridge_ctl = bctl;
	 888             max = pci_scan_child_bus(child);
	 889         } else {
	 890             /*
	 891              * For CardBus bridges, we leave 4 bus numbers
	 892              * as cards with a PCI-to-PCI bridge can be
	 893              * inserted later.
	 894              */
	 895             for (i = 0; i < CARDBUS_RESERVE_BUSNR; i++) {
	 896                 struct pci_bus *parent = bus;
	 897                 if (pci_find_bus(pci_domain_nr(bus),
	 898                             max+i+1))
	 899                     break;
	 900                 while (parent->parent) {
	 901                     if ((!pcibios_assign_all_busses()) &&
	 902                         (parent->busn_res.end > max) &&
	 903                         (parent->busn_res.end <= max+i)) {
	 904                         j = 1;
	 905                     }
	 906                     parent = parent->parent;
	 907                 }
	 908                 if (j) {
	 909                     /*
	 910                      * Often, there are two cardbus bridges
	 911                      * -- try to leave one valid bus number
	 912                      * for each one.
	 913                      */
	 914                     i /= 2;
	 915                     break;
	 916                 }
	 917             }
	 918             max += i;
	 919         }
	 920         /*
	 921          * Set the subordinate bus number to its real value.
	 922          */
	 923         pci_bus_update_busn_res_end(child, max);
	 924         pci_write_config_byte(dev, PCI_SUBORDINATE_BUS, max);
	 925     }
	 926     
	 927     sprintf(child->name,
	 928         (is_cardbus ? "PCI CardBus %04x:%02x" : "PCI Bus %04x:%02x"),
	 929         pci_domain_nr(bus), child->number);
	 930     
	 931     /* Has only triggered on CardBus, fixup is in yenta_socket */
	 932     while (bus->parent) {
	 933         if ((child->busn_res.end > bus->busn_res.end) ||
	 934             (child->number > bus->busn_res.end) ||
	 935             (child->number < bus->number) ||
	 936             (child->busn_res.end < bus->number)) {
	 937             dev_info(&child->dev, "%pR %s hidden behind%s bridge %s %pR\n",
	 938                 &child->busn_res,
	 939                 (bus->number > child->busn_res.end &&
	 940                  bus->busn_res.end < child->number) ?
	 941                     "wholly" : "partially",
	 942                 bus->self->transparent ? " transparent" : "",
	 943                 dev_name(&bus->dev),
	 944                 &bus->busn_res);
	 945         }
	 946         bus = bus->parent;
	 947     }
	 948     
	 949 out:
	 950     pci_write_config_word(dev, PCI_BRIDGE_CONTROL, bctl);
	 951     
	 952     return max;
	 953 }   
	 954 EXPORT_SYMBOL(pci_scan_bridge);

首先line772～line775从PCI桥的配置寄存器组中读取含有“主总线号”，“次总线号”等字段的长字，其中最低的字节为主总线号，在此之上的两个字节则依次为次总线号和子树中最大的总线号。再到line802,如果长字中较高的两个字节不全为0就进入if分支(pcibios_assign_all_busses()为空操作,返回0)。这两个字节不为0说明这条总线已经被枚举过一次，所以已经分配了次总线号和最大“子总线号”，一般而言，这发生在系统加电时有PCI BIOS进行的扫描枚举，对于这种情况，只有pass==0也就是第一趟扫描才真正执行if分支（看line809~810）。既然找到了bus就需要分配pci_bus结构等工作。可以看到调用了pci_add_new_bus()进行分配pci_bus，添加新总线，串联到父总线等等工作,还有就是会将新分配pci_bus的resource[]中四个指针指向桥设备pci_dev的resource数组相应的值。

	pci_add_new_bus（）
	  |
	  |
	  pci_alloc_child_bus（）
	
	 //drivers/pci/probe.c
	 	pci_alloc_child_bus(){
	 708     /* Set up default resource pointers and names.. */
	 709     for (i = 0; i < PCI_BRIDGE_RESOURCE_NUM; i++) {
	 710         child->resource[i] = &bridge->resource[PCI_BRIDGE_RESOURCES+i];
	 711         child->resource[i]->name = child->name;
	 712     }
	 713     bridge->subordinate = child;
	}

接下来就到了line828，因发现了新的总线，所以要对发现的次层总线进行扫描了，可以看到这里是对我们熟悉的pci_scan_child_bus（）的递归调用，因为此时CPU还在这个函数里呢。可见在对系统的PCI总线结构做深度优先的搜索。

如果不满足line802的条件呢？那么if的另一个分支就要在第二趟扫描进行时才进行。这是针对没有经过BIOS处理的PCI桥，第一趟扫描的目的只在于知道总线号已经用到多大了，这样才能在第二趟扫描中在此基础上继续分配总线号，并将分配的编号设置到PCI桥的配置寄存器里（line884），同样也要对新的总线分配结构，调用pci_add_new_bus()，还有也需要调用pci_scan_child_bus()做次层总线扫描。在这个函数里还需要注意命令寄存器的处理（PCI_BRIDGE_CONTROL）。

当所有递归调用返回后，就返回到pci_scan_root_bus()中啦，万里长征还差一点点。返回到这里的时候，系统一般情况下已经完成了对0号PCI总线的（递归的）扫描枚举任务，系统已经知道总线上连接的部分（大多数情况下是全部）设备了，并已经完成了各个数据结构的分配和串联任务。接下来就是调用pci_bus_add_devices(b)了，这个函数比较的重要，这里我们暂时先跳过。。。^_^以后会详细的讲啦。


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

