---
layout: post
title: Linux设备驱动模型及其他(5)
description: 扫描根总线（枚举与配置设备）
category: manual
---

>上章讲到根总线结构的初始化

##扫描根总线（枚举设备）
这里首先回顾下pci子系统初始化来的函数调用流程：

	pci_subsys_init()	//subsys_initcall()（arch/x86/pci/legacy.c）
	  |
	  |
	  pci_legacy_init()	//做了主要的一些工作（arch/x86/pci/legacy.c）
	    |
	    |
	    pcibios_scan_root()  //（arch/x86/pci/common.c）
	      |
	      |
	      pci_scan_root_bus()	//分配根总线结构和扫描根总线（drivers/pci/probe.c）
	        |
	        |
	        pci_create_root_bus()   //为根总线分配结构，以及host桥设备。注意这里的设备注册。
	        |
	        |
	        pci_scan_child_bus()    //扫描总线
	        |
	        |
	        pci_bus_add_devices()

我们这里就主要来看pci_scan_child_bus,

	//drivers/pci/probe.c
	1830 unsigned int pci_scan_child_bus(struct pci_bus *bus)
	1831 { 
	1832     unsigned int devfn, pass, max = bus->busn_res.start;
	1833     struct pci_dev *dev;
	1834 
	1835     dev_dbg(&bus->dev, "scanning bus\n");
	1836 
	1837     /* Go find them, Rover! */
	1838     for (devfn = 0; devfn < 0x100; devfn += 8)
	1839         pci_scan_slot(bus, devfn);
	1840 
	1841     /* Reserve buses for SR-IOV capability. */
	1842     max += pci_iov_bus_range(bus);
	1843 
	1844     /*
	1845      * After performing arch-dependent fixup of the bus, look behind
	1846      * all PCI-to-PCI bridges on this bus.
	1847      */
	1848     if (!bus->is_added) {
	1849         dev_dbg(&bus->dev, "fixups for bus\n");
	1850         pcibios_fixup_bus(bus);
	1851         bus->is_added = 1;
	1852     }
	1853    
	1854     for (pass = 0; pass < 2; pass++)
	1855         list_for_each_entry(dev, &bus->devices, bus_list) {
	1856             if (pci_is_bridge(dev))
	1857                 max = pci_scan_bridge(bus, dev, max, pass);
	1858         }
	1859    
	1860     /*
	1861      * We've scanned the bus and so we know all about what's on
	1862      * the other side of any bridges that may be on this bus plus
	1863      * any devices.
	1864      *
	1865      * Return how far we've got finding sub-buses.
	1866      */
	1867     dev_dbg(&bus->dev, "bus scan returning with max=%02x\n", max);
	1868     return max;
	1869 }

扫描一条PCI总线的直接目的就是逐个地发现连接在总线上的PCI设备，为其建立起pci_dev数据结构并挂入相应的队列，这就是所谓的枚举。这里依次对每个PCI卡槽（接口）通过pci_scan_slot()进行扫描，每次扫描8个功能，即逻辑设备，也是每个PCI接口卡上的最大容量。！！这里的devfn是将5位设备号和3位功能号组合起来的“逻辑设备号”。来看一下line1839这个函数：

	//drivers/pci/probe.c
	1629 int pci_scan_slot(struct pci_bus *bus, int devfn)
	1630 { 
	1631     unsigned fn, nr = 0;
	1632     struct pci_dev *dev;
	1633 
	1634     if (only_one_child(bus) && (devfn > 0))
	1635         return 0; /* Already scanned the entire slot */
	1636 
	1637     dev = pci_scan_single_device(bus, devfn);
	1638     if (!dev)
	1639         return 0;
	1640     if (!dev->is_added)
	1641         nr++;
	1642 
	1643     for (fn = next_fn(bus, dev, 0); fn > 0; fn = next_fn(bus, dev, fn)) {
	1644         dev = pci_scan_single_device(bus, devfn + fn);
	1645         if (dev) {
	1646             if (!dev->is_added)
	1647                 nr++;
	1648             dev->multifunction = 1;
	1649         }
	1650     }
	1651 
	1652     /* only one slot has pcie device */
	1653     if (bus->self && nr)
	1654         pcie_aspm_init_link_state(bus->self);
	1655 
	1656     return nr;
	1657 } 
	1658 EXPORT_SYMBOL(pci_scan_slot);

	1556 struct pci_dev *pci_scan_single_device(struct pci_bus *bus, int devfn)
	1557 { 
	1558     struct pci_dev *dev;
	1559 
	1560     dev = pci_get_slot(bus, devfn);
	1561     if (dev) {
	1562         pci_dev_put(dev);
	1563         return dev;
	1564     }
	1565 
	1566     dev = pci_scan_device(bus, devfn);
	1567     if (!dev)
	1568         return NULL;
	1569 
	1570     pci_device_add(dev, bus);
	1571 
	1572     return dev;
	1573 } 
	1574 EXPORT_SYMBOL(pci_scan_single_device);

这里直接去看pci_scan_single_device()，因为是第一次扫描，所以肯定是直接到line1566，我们看下这个函数在做些什么：
	
	//drivers/pci/probe.c
	1455 /*  
	1456  * Read the config data for a PCI device, sanity-check it
	1457  * and fill in the dev structure...
	1458  */ 
	1459 static struct pci_dev *pci_scan_device(struct pci_bus *bus, int devfn)
	1460 {   
	1461     struct pci_dev *dev;
	1462     u32 l;
	1463     
	1464     if (!pci_bus_read_dev_vendor_id(bus, devfn, &l, 60*1000))
	1465         return NULL;
	1466     
	1467     dev = pci_alloc_dev(bus);
	1468     if (!dev)
	1469         return NULL;
	1470     
	1471     dev->devfn = devfn;
	1472     dev->vendor = l & 0xffff;
	1473     dev->device = (l >> 16) & 0xffff;
	1474     
	1475     pci_set_of_node(dev);
	1476     
	1477     if (pci_setup_device(dev)) {
	1478         pci_bus_put(dev->bus);
	1479         kfree(dev);
	1480         return NULL;
	1481     }
	1482     
	1483     return dev;
	1484 } 

看注释的话应该也很清楚了，就是去读给定的“逻辑设备”的配置空间了，还有就是填充pci_dev结构。这里比较关键，好好分析下，首先是line1464，这里其实仅仅是个探测，看到底有没有设备，而真正的工作在pci_setup_device。还是先看如何探测：

	1415 bool pci_bus_read_dev_vendor_id(struct pci_bus *bus, int devfn, u32 *l,
	1416                 int crs_timeout)
	1417 {  
	1418     int delay = 1;
	1419    
	1420     if (pci_bus_read_config_dword(bus, devfn, PCI_VENDOR_ID, l))
	1421         return false;
	1422    
	1423     /* some broken boards return 0 or ~0 if a slot is empty: */
	1424     if (*l == 0xffffffff || *l == 0x00000000 ||
	1425         *l == 0x0000ffff || *l == 0xffff0000)
	1426         return false;
	1427    
	1428     /*
	1429      * Configuration Request Retry Status.  Some root ports return the
	1430      * actual device ID instead of the synthetic ID (0xFFFF) required
	1431      * by the PCIe spec.  Ignore the device ID and only check for
	1432      * (vendor id == 1).
	1433      */
	1434     while ((*l & 0xffff) == 0x0001) {
	1435         if (!crs_timeout)
	1436             return false;
	1437    
	1438         msleep(delay);
	1439         delay *= 2;
	1440         if (pci_bus_read_config_dword(bus, devfn, PCI_VENDOR_ID, l))
	1441             return false;
	1442         /* Card hasn't responded in 60 seconds?  Must be stuck. */
	1443         if (delay > crs_timeout) {
	1444             printk(KERN_WARNING "pci %04x:%02x:%02x.%d: not responding\n",
	1445                    pci_domain_nr(bus), bus->number, PCI_SLOT(devfn),
	1446                    PCI_FUNC(devfn));
	1447             return false;
	1448         }
	1449     }
	1450    
	1451     return true;
	1452 }  
	1453 EXPORT_SYMBOL(pci_bus_read_dev_vendor_id);

看到这里的pci_bus_read_config_dword么？就是我们在第三篇里面讲述的了，这里终于用到了。实际就是去调用bus->ops下的函数，插槽为空或者读取失败就直接返回false（只要厂商标号或者设备号不全为1或0就认为是有效的设备）。如果能成功读到vendorID就接着调用pci_alloc_dev（）为其分配pci_dev结构，期间进行一些初始化设置。然后设置devfn、vendorID、deviceID等等。。。具体看下其中涉及的函数：

	1399 struct pci_dev *pci_alloc_dev(struct pci_bus *bus)
	1400 {   
	1401     struct pci_dev *dev;
	1402     
	1403     dev = kzalloc(sizeof(struct pci_dev), GFP_KERNEL);
	1404     if (!dev)
	1405         return NULL; 
	1406           
	1407     INIT_LIST_HEAD(&dev->bus_list);
	1408     dev->dev.type = &pci_dev_type;
	1409     dev->bus = pci_bus_get(bus);
	1410     
	1411     return dev;
	1412 } 

这个函数还是比较简单，比较麻烦的是另一个，pci_setup_device(），这个函数做了很多实质性的工作，包括各个不同PCI设备类型的处理等。。。：

	//drivers/pci/probe.c 
	1087 /**
	1088  * pci_setup_device - fill in class and map information of a device
	1089  * @dev: the device structure to fill
	1090  *
	1091  * Initialize the device structure with information about the device's 
	1092  * vendor,class,memory and IO-space addresses,IRQ lines etc.
	1093  * Called at initialisation of the PCI subsystem and by CardBus services.
	1094  * Returns 0 on success and negative if unknown type of device (not normal,
	1095  * bridge or CardBus).
	1096  */
	1097 int pci_setup_device(struct pci_dev *dev)
	1098 { 
	1099     u32 class;
	1100     u8 hdr_type;
	1101     struct pci_slot *slot;
	1102     int pos = 0;
	1103     struct pci_bus_region region;
	1104     struct resource *res;
	1105
	1106     if (pci_read_config_byte(dev, PCI_HEADER_TYPE, &hdr_type))
	1107         return -EIO;
	1108 
	1109     dev->sysdata = dev->bus->sysdata;
	1110     dev->dev.parent = dev->bus->bridge;
	1111     dev->dev.bus = &pci_bus_type;
	1112     dev->hdr_type = hdr_type & 0x7f;
	1113     dev->multifunction = !!(hdr_type & 0x80);
	1114     dev->error_state = pci_channel_io_normal;
	1115     set_pcie_port_type(dev);
	1116 
	...........................	
	1129     pci_read_config_dword(dev, PCI_CLASS_REVISION, &class);
	...........................
	1145     class = dev->class >> 8;

	1239 } 

首先读取头部类型，曾经已经提到过，共分为0.1.2三种头部类型，而这里就是对这三种类型的设备进行不同的处理，一开始都好理解。直到line1112，头部类型共8位，其中低的7位标示类型，最高的那一位则标识此设备是否为多功能设备。一个物理PCI设备（接口卡）可以是多功能的，也可以是单功能的，头部类型最高位是1则表示为多功能设备，单功能的逻辑设备号必定为8的倍数（应为低三位的功能号为0）。那可以理解line1112~1113的工作了。另外还会获取的设备类型（class code），接着往下看:
	
	//drivers/pci/probe.c    pci_setup_device()
	.........
	1147     switch (dev->hdr_type) {            /* header type */
	1148     case PCI_HEADER_TYPE_NORMAL:            /* standard header */
	1149         if (class == PCI_CLASS_BRIDGE_PCI)
	1150             goto bad;
	1151         pci_read_irq(dev);
	1152         pci_read_bases(dev, 6, PCI_ROM_ADDRESS);
	1153         pci_read_config_word(dev, PCI_SUBSYSTEM_VENDOR_ID, &dev->subsystem_vendor);
	1154         pci_read_config_word(dev, PCI_SUBSYSTEM_ID, &dev->subsystem_device);
	............

针对普通设备,首先通过pci_read_irq(dev)去获取IRQ相关的两个寄存器内容并保存到pci_dev的相关结构中，反应着该设备的中断请求信号线与总线和系统的连接方式，具体请参考第二篇中的说明。

PCI设备中一般都带有一些RAM和ROM空间，通常的控制/状态寄存器和数据寄存器也往往以RAM区间的形式出现，这些地址以后都要先映射到系统总线上，再进一步映射到内核的虚拟空间。看line1152，通过pci_read_bases()把这些区间的大小和当前的地址读进来。对于一般的PCI设备最多可以有6个这样的RAM区域。我们来看下这个函数是怎么工作的：

	 //drivers/pci/probe.c 
	 314 static void pci_read_bases(struct pci_dev *dev, unsigned int howmany, int rom) 
	 315 {
	 316     unsigned int pos, reg;
	 317  
	 318     for (pos = 0; pos < howmany; pos++) {
	 319         struct resource *res = &dev->resource[pos];
	 320         reg = PCI_BASE_ADDRESS_0 + (pos << 2);
	 321         pos += __pci_read_base(dev, pci_bar_unknown, res, reg);
	 322     }
	 323  
	 324     if (rom) {
	 325         struct resource *res = &dev->resource[PCI_ROM_RESOURCE];
	 326         dev->rom_base_reg = rom;
	 327         res->flags = IORESOURCE_MEM | IORESOURCE_PREFETCH |
	 328                 IORESOURCE_READONLY | IORESOURCE_CACHEABLE |
	 329                 IORESOURCE_SIZEALIGN;
	 330         __pci_read_base(dev, pci_bar_mem32, res, rom);
	 331     }
	 332 }

这里的dev->resource[]是一个resource结构的数组，用来记录设备上各个地址区间的起始地址和长度，数组大小为12，这是因为除PCI设备的6个常规地址区间外还可以有一个扩充ROM区间，同时还要考虑设备为PCI桥的情况。在设备的配置寄存器组中，用于6个常规地址区间的长字是连续的，所以这里通过一个for循环可以直接来读取。表面看的确是风平浪静，我们来看下__pci_read_base()。

	 //drivers/pci/probe.c 
	 161 /**
	 162  * pci_read_base - read a PCI BAR
	 163  * @dev: the PCI device
	 164  * @type: type of the BAR
	 165  * @res: resource buffer to be filled in
	 166  * @pos: BAR position in the config space
	 167  *
	 168  * Returns 1 if the BAR is 64-bit, or 0 if 32-bit.
	 169  */
	 170 int __pci_read_base(struct pci_dev *dev, enum pci_bar_type type,
	 171             struct resource *res, unsigned int pos)
	 172 {  
	 173     u32 l, sz, mask;
	 174     u64 l64, sz64, mask64;
	 175     u16 orig_cmd;
	 176     struct pci_bus_region region, inverted_region;
	 177    
	 178     mask = type ? PCI_ROM_ADDRESS_MASK : ~0;
	 .................................
	 189     res->name = pci_name(dev);
	 190    
	 191     pci_read_config_dword(dev, pos, &l);
	 192     pci_write_config_dword(dev, pos, l | mask);
	 193     pci_read_config_dword(dev, pos, &sz);
	 194     pci_write_config_dword(dev, pos, l);
	 ..........................................
	 202     if (sz == 0xffffffff)
	 203         sz = 0;
	 ..........................................
	 209     if (l == 0xffffffff)
	 210         l = 0;
	 211    
	 212     if (type == pci_bar_unknown) {
	 213         res->flags = decode_bar(dev, l);
	 214         res->flags |= IORESOURCE_SIZEALIGN;
	 215         if (res->flags & IORESOURCE_IO) {
	 216             l64 = l & PCI_BASE_ADDRESS_IO_MASK;
	 217             sz64 = sz & PCI_BASE_ADDRESS_IO_MASK;
	 218             mask64 = PCI_BASE_ADDRESS_IO_MASK & (u32)IO_SPACE_LIMIT;
	 219         } else {
	 220             l64 = l & PCI_BASE_ADDRESS_MEM_MASK;
	 221             sz64 = sz & PCI_BASE_ADDRESS_MEM_MASK;
	 222             mask64 = (u32)PCI_BASE_ADDRESS_MEM_MASK;
	 223         }
	 .............................................
	 230    
	 231     if (res->flags & IORESOURCE_MEM_64) {
	 232         pci_read_config_dword(dev, pos + 4, &l);
	 233         pci_write_config_dword(dev, pos + 4, ~0);
	 234         pci_read_config_dword(dev, pos + 4, &sz);
	 235         pci_write_config_dword(dev, pos + 4, l);
	 236    
	 237         l64 |= ((u64)l << 32);
	 238         sz64 |= ((u64)sz << 32);
	 239         mask64 |= ((u64)~0 << 32);
	 240     }
	 241    
	 242     if (!dev->mmio_always_on && (orig_cmd & PCI_COMMAND_DECODE_ENABLE))
	 243         pci_write_config_word(dev, PCI_COMMAND, orig_cmd);
	 244    
	 245     if (!sz64)
	 246         goto fail;
	 247    
	 248     sz64 = pci_size(l64, sz64, mask64);
	 249     if (!sz64) {
	 250         dev_info(&dev->dev, FW_BUG "reg 0x%x: invalid BAR (can't size)\n",
	 251              pos);
	 252         goto fail;
	 253     }
	 254   
	 255     if (res->flags & IORESOURCE_MEM_64) {
	 256         if ((sizeof(dma_addr_t) < 8 || sizeof(resource_size_t) < 8) &&
	 257             sz64 > 0x100000000ULL) {
	 258             res->flags |= IORESOURCE_UNSET | IORESOURCE_DISABLED;
	 259             res->start = 0;
	 260             res->end = 0;
	 261             dev_err(&dev->dev, "reg 0x%x: can't handle BAR larger than 4GB (size %#010llx)\n",
	 262                 pos, (unsigned long long)sz64);
	 263             goto out;
	 264         }
	 265    
	 266         if ((sizeof(dma_addr_t) < 8) && l) {
	 267             /* Above 32-bit boundary; try to reallocate */
	 268             res->flags |= IORESOURCE_UNSET;
	 269             res->start = 0;
	 270             res->end = sz64;
	 271             dev_info(&dev->dev, "reg 0x%x: can't handle BAR above 4GB (bus address %#010llx)\n",
	 272                  pos, (unsigned long long)l64);
	 273             goto out;
	 274         }
	 275     }
	 276    
	 277     region.start = l64;
	 278     region.end = l64 + sz64;
	 279    
	 280     pcibios_bus_to_resource(dev->bus, res, &region);
	 281     pcibios_resource_to_bus(dev->bus, &inverted_region, res);
	 282    
	 283     /*
	 284      * If "A" is a BAR value (a bus address), "bus_to_resource(A)" is
	 285      * the corresponding resource address (the physical address used by
	 286      * the CPU.  Converting that resource address back to a bus address
	 287      * should yield the original BAR value:
	 288      *
	 289      *     resource_to_bus(bus_to_resource(A)) == A
	 290      *
	 291      * If it doesn't, CPU accesses to "bus_to_resource(A)" will not
	 292      * be claimed by the device.
	 293      */
	 294     if (inverted_region.start != region.start) {
	 295         res->flags |= IORESOURCE_UNSET;
	 296         res->start = 0;
	 297         res->end = region.end - region.start;
	 298         dev_info(&dev->dev, "reg 0x%x: initial BAR value %#010llx invalid\n",
	 299              pos, (unsigned long long)region.start);
	 300     }
	 301    
	 302     goto out;
	 303    
	 304    
	 305 fail:
	 306     res->flags = 0;
	 307 out:
	 308     if (res->flags)
	 309         dev_printk(KERN_DEBUG, &dev->dev, "reg 0x%x: %pR\n", pos, res);
	 310     
	 311     return (res->flags & IORESOURCE_MEM_64) ? 1 : 0;
	 312 } 

首先注意这里mask为～0，然后看到line191，这个时候读出来的先是区间的起始地址的组合信息，放置到l中。接着向寄存器写全1，再读取寄存器，这时候读取出来的就是区间长度的组合信息了，存在sz。最后再将l中的信息回写到寄存器中。接着就对读取的组合起始地址数据进行解析【1】：
<ul>
<li>首先，其最低位，也就是bit0标识了区间的类型，0表示是个存储器区间，1表示是个I/O地址（即寄存器）区间。注意这仅仅是表明可以通过什么样的操作（I/O指令或访问内存指令）来访问这个区间，而与其内容是否为寄存器无关，寄存器也可以通过存储单元的形式来实现（“memory mapped”）</li>
<li>如果是存储器的区间，那么其高28为就是起始地址的高28位（起始地址低4为一定是0）</li>
<li>如果是I/O区间，那么其高29位就是起始地址的高29位（其实地址低3为一定是0）</li>
<li>对于存储器区间，如果bit3为1，就表示这个区间的操作可以流水线化，成为“可预取”（prefetchable），否则就不能流水线化，只能一个一个单元的读写（见后）</li>
<li>还有bit2为1表示采用64位地址，为0表示采用32位地址</li>
<li>最后，bit1为1表示区间大小超过1mb，为0表示区间大小在1MB一下</li>
</ul>

__pci_read_base中的line213就是从读取的组合起始地址进行解析并将结果给flag，根据上面的解释以及下列的宏定义应该很好理解。

	 drivers/pci/probe.c
	 126 static inline unsigned long decode_bar(struct pci_dev *dev, u32 bar)
	 127 {  
	 128     u32 mem_type;
	 129     unsigned long flags;
	 130    
	 131     if ((bar & PCI_BASE_ADDRESS_SPACE) == PCI_BASE_ADDRESS_SPACE_IO) {
	 132         flags = bar & ~PCI_BASE_ADDRESS_IO_MASK;
	 133         flags |= IORESOURCE_IO;
	 134         return flags;
	 135     }
	 136    
	 137     flags = bar & ~PCI_BASE_ADDRESS_MEM_MASK;
	 138     flags |= IORESOURCE_MEM;
	 139     if (flags & PCI_BASE_ADDRESS_MEM_PREFETCH)
	 140         flags |= IORESOURCE_PREFETCH;
	 141    
	 142     mem_type = bar & PCI_BASE_ADDRESS_MEM_TYPE_MASK;
	 143     switch (mem_type) {
	 144     case PCI_BASE_ADDRESS_MEM_TYPE_32:
	 145         break;
	 146     case PCI_BASE_ADDRESS_MEM_TYPE_1M:
	 147         /* 1M mem BAR treated as 32-bit BAR */
	 148         break;
	 149     case PCI_BASE_ADDRESS_MEM_TYPE_64:
	 150         flags |= IORESOURCE_MEM_64;
	 151         break;
	 152     default:
	 153         /* mem unknown type treated as 32-bit BAR */
	 154         break;
	 155     }
	 156     return flags;
	 157 }
	 //include/uapi/linux/pci_regs.h
	 91 #define  PCI_BASE_ADDRESS_SPACE     0x01    /* 0 = memory, 1 = I/O */ 
	 92 #define  PCI_BASE_ADDRESS_SPACE_IO  0x01
	 93 #define  PCI_BASE_ADDRESS_SPACE_MEMORY  0x00
	 94 #define  PCI_BASE_ADDRESS_MEM_TYPE_MASK 0x06
	 95 #define  PCI_BASE_ADDRESS_MEM_TYPE_32   0x00    /* 32 bit address */
	 96 #define  PCI_BASE_ADDRESS_MEM_TYPE_1M   0x02    /* Below 1M [obsolete] */
	 97 #define  PCI_BASE_ADDRESS_MEM_TYPE_64   0x04    /* 64 bit address */
	 98 #define  PCI_BASE_ADDRESS_MEM_PREFETCH  0x08    /* prefetchable? */
	 99 #define  PCI_BASE_ADDRESS_MEM_MASK  (~0x0fUL)
	100 #define  PCI_BASE_ADDRESS_IO_MASK   (~0x03UL)

回到__pci_read_base()中，接下来line214～line223就是通过对flag来区分是I/O区间，还是存储器区间，并分别取得开始地址和大小。line231～line241，如果是64地址的存储区间，还需要读下一个配置寄存器。line248，根据规则对空大小的值进行换算，得到真实的空间大小，调用函数pci_size（），有兴趣的可以自己分析下。到line277～line281就将开始地址和大小记录到pci_dev相应的resource[]中，还涉及到桥窗口的设置。最后如果是64位地址空间就需要返回1通知上层调用跳过一个寄存器。回到pci_read_base()中就能看出来。完成常规的6个存储区外，设备上还可能提供一个扩充的ROM区间，见pci_read_base()中line324，这里就不讨论了。

再次回到上面的pci_setup_device（）中的line1153，继续读取配置空间，获取子系统号和子系统厂商号，对pci_dev进行配置。line1162，如果是IDE的存储设备进行特殊的配置。

	1162         if (class == PCI_CLASS_STORAGE_IDE) {
	1163             u8 progif;
	1164             pci_read_config_byte(dev, PCI_CLASS_PROG, &progif);
	1165             if ((progif & 1) == 0) {
	1166                 region.start = 0x1F0;
	1167                 region.end = 0x1F7;
	1168                 res = &dev->resource[0];
	1169                 res->flags = LEGACY_IO_RESOURCE;
	1170                 pcibios_bus_to_resource(dev->bus, res, &region);
	1171                 dev_info(&dev->dev, "legacy IDE quirk: reg 0x10: %pR\n",
	1172                      res);
	1173                 region.start = 0x3F6;
	1174                 region.end = 0x3F6;
	1175                 res = &dev->resource[1];
	1176                 res->flags = LEGACY_IO_RESOURCE;
	1177                 pcibios_bus_to_resource(dev->bus, res, &region);
	1178                 dev_info(&dev->dev, "legacy IDE quirk: reg 0x14: %pR\n",
	1179                      res);
	1180             }
	1181             if ((progif & 4) == 0) {
	1182                 region.start = 0x170;
	1183                 region.end = 0x177;
	1184                 res = &dev->resource[2];
	1185                 res->flags = LEGACY_IO_RESOURCE;
	1186                 pcibios_bus_to_resource(dev->bus, res, &region);
	1187                 dev_info(&dev->dev, "legacy IDE quirk: reg 0x18: %pR\n",
	1188                      res);
	1189                 region.start = 0x376;
	1190                 region.end = 0x376;
	1191                 res = &dev->resource[3];
	1192                 res->flags = LEGACY_IO_RESOURCE;
	1193                 pcibios_bus_to_resource(dev->bus, res, &region);
	1194                 dev_info(&dev->dev, "legacy IDE quirk: reg 0x1c: %pR\n",
	1195                      res);
	1196             }
	1197         }
	1198         break;

到此就处理完正常头部类型了，接下来就是桥的头部类型处理：

	1200     case PCI_HEADER_TYPE_BRIDGE:            /* bridge header */
	1201         if (class != PCI_CLASS_BRIDGE_PCI)
	1202             goto bad;
	1203         /* The PCI-to-PCI bridge spec requires that subtractive
	1204            decoding (i.e. transparent) bridge must have programming
	1205            interface code of 0x01. */
	1206         pci_read_irq(dev);
	1207         dev->transparent = ((dev->class & 0xff) == 1);
	1208         pci_read_bases(dev, 2, PCI_ROM_ADDRESS1);
	1209         set_pcie_hotplug_bridge(dev);
	1210         pos = pci_find_capability(dev, PCI_CAP_ID_SSVID);
	1211         if (pos) {
	1212             pci_read_config_word(dev, pos + PCI_SSVID_VENDOR_ID, &dev->subsystem_vendor);
	1213             pci_read_config_word(dev, pos + PCI_SSVID_DEVICE_ID, &dev->subsystem_device);
	1214         }
	1215         break;

对PCI-PCI桥设备的处理就比较简单了，许多字段，如中断请求等对PCI桥设备没有意义，调用pci_read_base（），可以看到PCI桥设备的存储区间比较少，就两个。当然如果是桥设备，还要下一层PCI总线的扫描，不过这个工作在后面进行，具体是哪里呢？就是在pci_scan_child_bus（）中了，可以先看下：

	//drivers/pci/probe.c     pci_scan_child_bus(struct pci_bus *bus)
	1854     for (pass = 0; pass < 2; pass++)
	1855         list_for_each_entry(dev, &bus->devices, bus_list) {
	1856             if (pci_is_bridge(dev))
	1857                 max = pci_scan_bridge(bus, dev, max, pass);
	1858         }  

到此为之，我们已经发现了一个设备，并以前将对应的pci_dev准备好了，这下终于可以返回了：

		pci_setup_device()
	        |
	        |
	        |  return 0,1
	     pci_scan_device()
	     |
	     |
	     |   return pri_dev *
	  pci_scan_single_device()

pci_scan_single_device()调用pci_device_add(dev, bus)来将pci_dev串到根总线的bus->devices链表中，其中还调用了device_add(&dev->dev)将device“注册到系统中”，这里就涉及到了linux的驱动模型，我们在以后详细讨论。

所有这些工作结束后，就返回到pci_scan_slot中了，继续扫描插槽中的其他功能（逻辑设备），扫描完一个插槽后，返回到pci_scan_child_bus，继续扫描其他插槽。这样就完成了根总线的扫描和设备配置任务。接下来就是探测次层总线啦！！下回分解。


！！！！！！！！！这里涉及到一个设备fixup的问题，就不讨论了！！！！！！！！！！！！！！！！

因为函数调用关系比较复杂，我们来整理下：

	pci_subsys_init()	//subsys_initcall()（arch/x86/pci/legacy.c）
	  |
	  |
	  pci_legacy_init()	//做了主要的一些工作（arch/x86/pci/legacy.c）
	    |
	    |
	    pcibios_scan_root()  //（arch/x86/pci/common.c）
	      |
	      |
	      pci_scan_root_bus()	//分配根总线结构和扫描根总线（drivers/pci/probe.c）
	        |
	        |
	        pci_create_root_bus()   //为根总线分配结构，以及host桥设备。注意这里的设备注册(*)。
	        |
	        |
	        pci_scan_child_bus()    //扫描总线
	       	|  |			//利用for循环扫描所有的插槽
	        |  |			// for (devfn = 0; devfn < 0x100; devfn += 8)
	        |  pci_scan_slot()	//扫描一个插槽中的所有子功能（逻辑设备）
	        |  | |			//利用for循环扫描所有子功能
	        |  | |	
	        |  | pci_scan_single_device()  //主要调用pci_scan_device探测并配置一个逻辑设备
	        |  |   |
	        |  |   |
	        |  |   pci_scan_device（）     //通过读写配置寄存器探测并配置一个逻辑设备的pci_dev结构
	        |  |     |		      //主要设计到厂商号，flags，功能号，resource等等
	        |  |     |		      //主要工作还是pci_setup_device来做
	        |  |	 pci_setup_device（）  //读写配置空间，对不同头类型进行分配处理，设置pci_dev结构
	        |  |     |
	        |  |	 | return
	        |  |   pci_device_add()	      //将pci_dev设备关联到bus，”注册“pci_dev->device等（*）
	        |  |     |
	        |  |     |
	        |  |     device_add(&dev->dev) //!!!!!非常重要（**）
	        |  |
	        |  pci_scan_bridge()	      //扫描次层总线
	        |
	        pci_bus_add_devices()         //？？？！！！好像跟寻找驱动有关？？ 在这里才将dev->added置位(****)

带*为涉及到驱动模型的东西，是非常重要的东西，这里是提醒自己不要忘记，以后会分析。
##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译



