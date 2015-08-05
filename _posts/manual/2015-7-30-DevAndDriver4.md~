---
layout: post
title: Linux设备驱动模型及其他(4)
description: Hi，cortana！
category: manual
---

>上章讲到Linux写PCI设备枚举和配置的准备工作。

##PCI子系统初始化1（根总线结构初始化）
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
	 37 void pci_add_resource(struct list_head *resources, struct resource *res) 
	 38 {   
	 39     pci_add_resource_offset(resources, res, 0);
	 40 }   
	 41 EXPORT_SYMBOL(pci_add_resource);

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

这里一般情况下就会到default_ressources,line74,75。可以看到是将ioport_resource，iomem_resource两个resource添加到resources链表尾部。这两个resource是这么定义的：

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

ioport_resource，iomem_resource也就是反映着I/O空间和内存空间地址占用情况的两颗资源树，在以后会详细分析，包括这里的PCI桥窗口。

资源分配结束，返回到pcibios_scan_root()调用pci_scan_root_bus()就开始扫描工作了:

	bus = pci_scan_root_bus(NULL, busnum, &pci_root_ops, sd, &resources);
	//drivers/pci/probe.c 
	2060 struct pci_bus *pci_scan_root_bus(struct device *parent, int bus,
	2061         struct pci_ops *ops, void *sysdata, struct list_head *resources)
	2062 {
	2063     struct pci_host_bridge_window *window;
	2064     bool found = false;
	2065     struct pci_bus *b; 
	2066     int max; 
	2067
	2068     list_for_each_entry(window, resources, list)
	2069         if (window->res->flags & IORESOURCE_BUS) {
	2070             found = true;
	2071             break;
	2072         } 
	2073 
	2074     b = pci_create_root_bus(parent, bus, ops, sysdata, resources);
	2075     if (!b)
	2076         return NULL;
	2077 
	2078     if (!found) {
	2079         dev_info(&b->dev,
	2080          "No busn resource found for root bus, will use [bus %02x-ff]\n",
	2081             bus);
	2082         pci_bus_insert_busn_res(b, bus, 255);
	2083     }
	2084
	2085     max = pci_scan_child_bus(b);
	2086 
	2087     if (!found)
	2088         pci_bus_update_busn_res_end(b, max);
	2089 
	2090     pci_bus_add_devices(b);
	2091     return b;
	2092 } 
	2093 EXPORT_SYMBOL(pci_scan_root_bus);

函数调用时parent为NULL，因为是根总线嘛。busnum为0，看这里的pci_root_ops，就是我们上一篇讲的读写配置空间寄存器的操作。函数首先从刚刚分配的资源树查找是否存在总线的资源（没有问题话肯定是可以找到的），找到后就直接创建根总线啦（因为根总线肯定是存在的嘛）！来看代码：

	//drivers/pci/probe.c
	1892 struct pci_bus *pci_create_root_bus(struct device *parent, int bus,
	1893         struct pci_ops *ops, void *sysdata, struct list_head *resources)
	1894 {  
	1895     int error;
	1896     struct pci_host_bridge *bridge;
	1897     struct pci_bus *b, *b2;
	1898     struct pci_host_bridge_window *window, *n;
	1899     struct resource *res;
	1900     resource_size_t offset;
	1901     char bus_addr[64];
	1902     char *fmt;
	1903    
	1904     b = pci_alloc_bus(NULL);
	1905     if (!b)
	1906         return NULL;
	1907    
	1908     b->sysdata = sysdata;
	1909     b->ops = ops;
	1910     b->number = b->busn_res.start = bus;
	1911     pci_bus_assign_domain_nr(b, parent);
	1912     b2 = pci_find_bus(pci_domain_nr(b), bus);
	1913     if (b2) {
	1914         /* If we already got to this bus through a different bridge, ignore it */
	1915         dev_dbg(&b2->dev, "bus already known\n");
	1916         goto err_out;
	1917     }
	1918    
	1919     bridge = pci_alloc_host_bridge(b);
	1920     if (!bridge)
	1921         goto err_out;
	1922    
	1923     bridge->dev.parent = parent;
	1924     bridge->dev.release = pci_release_host_bridge_dev;
	1925     dev_set_name(&bridge->dev, "pci%04x:%02x", pci_domain_nr(b), bus);
	1926     error = pcibios_root_bridge_prepare(bridge);
	1927     if (error) {
	1928         kfree(bridge);
	1929         goto err_out;
	1930     }
	1931    
	1932     error = device_register(&bridge->dev);
	1933     if (error) {
	1934         put_device(&bridge->dev);
	1935         goto err_out;
	1936     }
	1937     b->bridge = get_device(&bridge->dev);
	1938     device_enable_async_suspend(b->bridge);
	1939     pci_set_bus_of_node(b);
	1940     
	1941     if (!parent)
	1942         set_dev_node(b->bridge, pcibus_to_node(b));
	1943     
	1944     b->dev.class = &pcibus_class;
	1945     b->dev.parent = b->bridge;
	1946     dev_set_name(&b->dev, "%04x:%02x", pci_domain_nr(b), bus);
	1947     error = device_register(&b->dev);
	1948     if (error)
	1949         goto class_dev_reg_err;
	1950     
	1951     pcibios_add_bus(b);
	1952     
	1953     /* Create legacy_io and legacy_mem files for this bus */
	1954     pci_create_legacy_files(b);
	1955     
	1956     if (parent)
	1957         dev_info(parent, "PCI host bridge to bus %s\n", dev_name(&b->dev));
	1958     else
	1959         printk(KERN_INFO "PCI host bridge to bus %s\n", dev_name(&b->dev));
	1960     
	1961     /* Add initial resources to the bus */
	1962     list_for_each_entry_safe(window, n, resources, list) {
	1963         list_move_tail(&window->list, &bridge->windows);
	1964         res = window->res;
	1965         offset = window->offset;
	1966         if (res->flags & IORESOURCE_BUS)
	1967             pci_bus_insert_busn_res(b, bus, res->end);
	1968         else
	1969             pci_bus_add_resource(b, res, 0);
	1970         if (offset) {
	1971             if (resource_type(res) == IORESOURCE_IO)
	1972                 fmt = " (bus address [%#06llx-%#06llx])";
	1973             else
	1974                 fmt = " (bus address [%#010llx-%#010llx])";
	1975             snprintf(bus_addr, sizeof(bus_addr), fmt,
	1976                  (unsigned long long) (res->start - offset),
	1977                  (unsigned long long) (res->end - offset));
	1978         } else
	1979             bus_addr[0] = '\0';
	1980         dev_info(&b->dev, "root bus resource %pR%s\n", res, bus_addr);
	1981     }
	1982    
	1983     down_write(&pci_bus_sem);
	1984     list_add_tail(&b->node, &pci_root_buses);
	1985     up_write(&pci_bus_sem);
	1986    
	1987     return b;
	1988    
	1989 class_dev_reg_err:
	1990     put_device(&bridge->dev);
	1991     device_unregister(&bridge->dev);
	1992 err_out:
	1993     kfree(b);
	1994     return NULL;
	1995 }  

代码很长，慢慢分析。首先是是为根总线分配结构，这里先来看下这个结构：

	 //include/linux/pci.h
	 444 struct pci_bus {
	 445     struct list_head node;      /* 链表元素node：对于PCI根总线而言，其pci_bus结构通过node成员链接到本节一开始所述的根总线链表中，根总线链表的表头由一个list_head类型的全局变量pci_root_buses所描述。而对于非根pci总线，其pci_bus结构通过node成员链接到其父总线的子总线链表children中*/
	 446     struct pci_bus  *parent;    /*parent指针：指向该pci总线的父总线，即pci桥所在的那条总线*/
	 447     struct list_head children;  /*children指针：描述了这条PCI总线的子总线链表的表头。这条PCI总线的所有子总线都通过上述的node链表元素链接成一条子总线链表，而该链表的表头就由父总线的children指针所描述*/
	 448     struct list_head devices;   /* devices链表头：描述了这条PCI总线的逻辑设备链表的表头。除了链接在全局PCI设备链表中之外，每一个PCI逻辑设备也通过其 pci_dev结构中的bus_list成员链入其所在PCI总线的局部设备链表中，而这个局部的总线设备链表的表头就由pci_bus结构中的 devices成员所描述*/
	 449     struct pci_dev  *self;      /* bridge device as seen by parent */
	 450     struct list_head slots;     /* list of slots on this bus */
	 451     struct resource *resource[PCI_BRIDGE_RESOURCE_NUM];/* 资源指针数组：指向应路由到这条pci总线的地址空间资源，通常是指向对应桥设备的pci_dev结构中的资源数组resource［10：7］*/
	 452     struct list_head resources; /* address space routed to this bus */
	 453     struct resource busn_res;   /* bus numbers routed to this bus */
	 454    
	 455     struct pci_ops  *ops;       /* 指针ops：指向一个pci_ops结构，表示这条pci总线所使用的配置空间访问函数*/
	 456     struct msi_controller *msi; /* MSI controller */
	 457     void        *sysdata;   * 无类型指针sysdata：指向系统特定的扩展数据*/
	 458     struct proc_dir_entry *procdir; /* 指针procdir：指向该PCI总线在／proc文件系统中对应的目录项*/
	 459    
	 460     unsigned char   number;     /* bus number */
	 461     unsigned char   primary;    /* primary：表示引出这条PCI总线的“桥设备的主总线”（也即桥设备所在的PCI总线）编号，取值范围0－255*/
	 462     unsigned char   max_bus_speed;  /* enum pci_bus_speed */
	 463     unsigned char   cur_bus_speed;  /* enum pci_bus_speed */
	 464 #ifdef CONFIG_PCI_DOMAINS_GENERIC
	 465     int     domain_nr;
	 466 #endif
	 467    
	 468     char        name[48];
	 469    
	 470     unsigned short  bridge_ctl; /* manage NO_ISA/FBB/et al behaviors */
	 471     pci_bus_flags_t bus_flags;  /* inherited by child buses */
	 472     struct device       *bridge;
	 473     struct device       dev;	///！！！！！！！！！注意这里
	 474     struct bin_attribute    *legacy_io; /* legacy I/O for this bus */
	 475     struct bin_attribute    *legacy_mem; /* legacy mem */
	 476     unsigned int        is_added:1;
	 477 };

	 483 static struct pci_bus *pci_alloc_bus(struct pci_bus *parent)
	 484 { 
	 485     struct pci_bus *b;
	 486 
	 487     b = kzalloc(sizeof(*b), GFP_KERNEL);
	 488     if (!b)      
	 489         return NULL;
	 490 
	 491     INIT_LIST_HEAD(&b->node);
	 492     INIT_LIST_HEAD(&b->children);
	 493     INIT_LIST_HEAD(&b->devices);
	 494     INIT_LIST_HEAD(&b->slots);
	 495     INIT_LIST_HEAD(&b->resources);
	 496     b->max_bus_speed = PCI_SPEED_UNKNOWN;
	 497     b->cur_bus_speed = PCI_SPEED_UNKNOWN;
	 498 #ifdef CONFIG_PCI_DOMAINS_GENERIC
	 499     if (parent)
	 500         b->domain_nr = parent->domain_nr;
	 501 #endif
	 502     return b;
	 503 } 

pci_alloc_bus主要就是分配pci_bus结构，并初始化其中的链表头，这里注意结构里面的device！！！以后会讲到。回到pci_create_root_bus,继续初始化根总线的pci_bus，看到line1909这里就将读写配置空间的操作赋给它了。line1911～line1928我们略过。line1919为根总线分配桥设备，跟进去看下：

	 //drivers/pci/probe.c
	 517 static struct pci_host_bridge *pci_alloc_host_bridge(struct pci_bus *b)    
	 518 {
	 519     struct pci_host_bridge *bridge;
	 520    
	 521     bridge = kzalloc(sizeof(*bridge), GFP_KERNEL);
	 522     if (!bridge)
	 523         return NULL;
	 524    
	 525     INIT_LIST_HEAD(&bridge->windows);
	 526     bridge->bus = b;
	 527     return bridge;
	 528 } 
	
	 //include/linux/pci.h 
	 406 struct pci_host_bridge { 
	 407     struct device dev;		//！！！！！！注意这里哦！
	 408     struct pci_bus *bus;        /* root bus */
	 409     struct list_head windows;   /* pci_host_bridge_windows */
	 410     void (*release_fn)(struct pci_host_bridge *);
	 411     void *release_data;
	 412 }; 

同样注意结构体里的device！！！！ 这里工作很简单，分配结构，初始化链表头，设置bridge的bus。回到pci_create_root_bus，line1923～line1930，进一步对bridge设置。到line1932，这里就是注册设备啦～～。这些我们先留到以后分析，因为这些都涉及到设备驱动模型，整合在一起讲比较明白。然后就是对根总线结构的进一步设置，line1944~1955分别设置了总线设备的类和父设备（也就是host桥），并将总线设备进行注册等等。。。接着到line1961，为总线添加资源，也就是将上面的 ioport_resource，iomem_resource添加到根总线的结构的resources中。（因为这两种资源不是 IORESOURCE_BUS）

	 //drivers/pci/bus.c 
	 54 void pci_bus_add_resource(struct pci_bus *bus, struct resource *res,
	 55               unsigned int flags)
	 56 {
	 57     struct pci_bus_resource *bus_res;
	 58 
	 59     bus_res = kzalloc(sizeof(struct pci_bus_resource), GFP_KERNEL);
	 60     if (!bus_res) {
	 61         dev_err(&bus->dev, "can't add %pR resource\n", res);
	 62         return;
	 63     } 
	 64
	 65     bus_res->res = res;
	 66     bus_res->flags = flags;
	 67     list_add_tail(&bus_res->list, &bus->resources);
	 68 }

这些结束后，就是line1984了，将根总线链接到全局变量pci_root_buses！接着就返回到pci_scan_root_bus了，这个时候我们已经有了根总线以及host-bridge了。接下来就是来扫描这条根总线了。。。。下篇！

##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译



