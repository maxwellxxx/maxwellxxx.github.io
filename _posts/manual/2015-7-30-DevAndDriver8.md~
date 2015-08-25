---
layout: post
title: Linux设备驱动模型及其他(8)
description: 处理中断请求路径互连
category: manual
---

>上章讲到找到了中断路径表pirq_table，并完成了其他根总线的扫描，并寻找到了中断路径控制器pirq_router并为其加载了驱动。

###中断配置（接上篇）

上篇在pcibios_irq_init（）中已经完成了前期的准备工作，接下来就开始要处理中断请求路径互连。

	//arch/x86/pci/irq.c  
	1117 void __init pcibios_irq_init(void)
	1118 {
	.................
	1149     x86_init.pci.fixup_irqs();
	.................
	1162 }

	 //arch/x86/kernel/x86_init.c
	 81     .pci = {
	 82         .init           = x86_default_pci_init,
	 83         .init_irq       = x86_default_pci_init_irq,
	 84         .fixup_irqs     = x86_default_pci_fixup_irqs,
	 85     },
	 
	 arch/x86/include/asm/pci_x86.h
	 205 # define x86_default_pci_fixup_irqs pcibios_fixup_irqs 

	//arch/x86/pci/irq.c  
	1023 void __init pcibios_fixup_irqs(void)
	1024 {  
	1025     struct pci_dev *dev = NULL;
	1026     u8 pin;
	1027    
	1028     DBG(KERN_DEBUG "PCI: IRQ fixup\n");
	1029     for_each_pci_dev(dev) {
	1030         /*
	1031          * If the BIOS has set an out of range IRQ number, just
	1032          * ignore it.  Also keep track of which IRQ's are
	1033          * already in use.
	1034          */
	1035         if (dev->irq >= 16) {
	1036             dev_dbg(&dev->dev, "ignoring bogus IRQ %d\n", dev->irq);
	1037             dev->irq = 0;
	1038         }
	1039         /*
	1040          * If the IRQ is already assigned to a PCI device,
	1041          * ignore its ISA use penalty
	1042          */
	1043         if (pirq_penalty[dev->irq] >= 100 &&
	1044                 pirq_penalty[dev->irq] < 100000)
	1045             pirq_penalty[dev->irq] = 0;
	1046         pirq_penalty[dev->irq]++;
	1047     }
	1048    
	1049     if (io_apic_assign_pci_irqs)
	1050         return;
	1051    
	1052     dev = NULL;
	1053     for_each_pci_dev(dev) {
	1054         pci_read_config_byte(dev, PCI_INTERRUPT_PIN, &pin);
	1055         if (!pin)
	1056             continue;
	1057    
	1058         /*
	1059          * Still no IRQ? Try to lookup one...
	1060          */
	1061         if (!dev->irq)
	1062             pcibios_lookup_irq(dev, 0);
	1063     }
	1064 }

如前所述，每个中断请求最终都是要连接到中断控制器上的某条中断请求输入线上。不过中断控制器的16条输入线并不是可以随便连接的，比如0号中断线给时钟专用。另外，还要尽可能将各项PCI设备均匀地分布到不同的中断请求线上。为此内核引入了一个数组pirq_penalty[]这个数组我们在上篇已经接触过了，通过这个数据和一个简单的算法，就能确定选择中断输入线。具体来看下定义和用处。

	  //arch/x86/pci/irq.c
	  31 /*  
	  32  * Never use: 0, 1, 2 (timer, keyboard, and cascade)
	  33  * Avoid using: 13, 14 and 15 (FP error and IDE).
	  34  * Penalize: 3, 4, 6, 7, 12 (known ISA uses: serial, floppy, parallel and mouse)
	  35  */
	  36 unsigned int pcibios_irq_mask = 0xfff8;
	  37     
	  38 static int pirq_penalty[16] = { 
	  39     1000000, 1000000, 1000000, 1000, 1000, 0, 1000, 1000,
	  40     0, 0, 0, 0, 1000, 100000, 100000, 100000
	  41 };   

数组大小为16，对应16条中断请求输入线。每个元素代表着因选用相应中断请求输入线而受到的“惩罚”，所以选择时要选择惩罚值最小的。初始时中断请求输入线0、1、2、13、14和15的惩罚为1000000，所以实际上永远都选择不到这几条输入线。而5、8、9、10、11为0，所以开始总是选择这几条线。对于同一条中断输入请求线，选用它的设备越多，则再次选择它受到的惩罚越大。所以，只要设备数量不是太大，那么中断请求输入线3、4、5、7和12基本也不会被用到。这样就保证了负荷的均匀分布。

在上篇时我们提到“-----在中断路径表pirq_table中还有个位图exclusive_irqs，位图中为1的位表示中断控制器的相应输入应该专用，而避免为多个中断源共用。所以对于这些中断请求输入线，代码中在一个数组pirq_penalty[]的相应元素上增加一个惩罚量“100”，使得中断路径互连时不太会选择这些作为对象。-----”这里的输入线是最好能保留给ISA总线上的设备用的。但是只是为了尽量避免PCI设备使用，如果发现PCI设备已经在使用了，就破罐子破摔，就不妨就让其他的PCI设备也来使用，干脆就把原先+100的值置为0，重新开始计算（line1043~line1045）。

另一种是irq>16的情况，显然是非法中断，直接将irq置为0，交给第二轮foreach处理。其实可以看到，如果是非法irq和irq为0的设备，都会在pirq_penalty[0]加一次计数，反正这个值已经够大了，多加几次无所谓。

其实第一轮的foreach只是对已经被BIOS配置好的PCI设备中断做了一次验证，而其实有些设备可能没有被配置好或者配置好了但是dev->irq中没有记录，就要进行第二轮的foreach。首先从寄存器PCI_INTERRUPT_PIN读入pin值，如果pin值为非0，则表明此设备有中断功能。如果此时它的irq为0，就表明还不知道它和中断控制器的哪条输入线连接，这时候就需要通过pcibios_lookup_irq(dev, 0)来寻找。

	 //arch/x86/pci/irq.c 
	 878 static int pcibios_lookup_irq(struct pci_dev *dev, int assign)
	 879 {   
	 880     u8 pin;
	 881     struct irq_info *info;
	 882     int i, pirq, newirq;
	 883     int irq = 0;
	 884     u32 mask;
	 885     struct irq_router *r = &pirq_router;
	 886     struct pci_dev *dev2 = NULL;
	 887     char *msg = NULL;
	 888     
	 889     /* Find IRQ pin */
	 890     pci_read_config_byte(dev, PCI_INTERRUPT_PIN, &pin);
	 891     if (!pin) {
	 892         dev_dbg(&dev->dev, "no interrupt pin\n");
	 893         return 0;
	 894     }
	 895     
	 896     if (io_apic_assign_pci_irqs)
	 897         return 0;
	 898     
	 899     /* Find IRQ routing entry */
	 900     
	 901     if (!pirq_table)
	 902         return 0;
	 903     
	 904     info = pirq_get_info(dev);
	 905     if (!info) {
	 906         dev_dbg(&dev->dev, "PCI INT %c not found in routing table\n",
	 907             'A' + pin - 1);
	 908         return 0;
	 909     }
	 910     pirq = info->irq[pin - 1].link;
	 911     mask = info->irq[pin - 1].bitmap;
	 912     if (!pirq) {
	 913         dev_dbg(&dev->dev, "PCI INT %c not routed\n", 'A' + pin - 1);
	 914         return 0;
	 915     }
	 916     dev_dbg(&dev->dev, "PCI INT %c -> PIRQ %02x, mask %04x, excl %04x",
	 917         'A' + pin - 1, pirq, mask, pirq_table->exclusive_irqs);
	 918     mask &= pcibios_irq_mask;
	 .........................
	 ////HP 专用处理
	 ////ACER 专用处理
	 .........................
	 938     /*
	 939      * Find the best IRQ to assign: use the one
	 940      * reported by the device if possible.
	 941      */
	 942     newirq = dev->irq;
	 943     if (newirq && !((1 << newirq) & mask)) {
	 944         if (pci_probe & PCI_USE_PIRQ_MASK)
	 945             newirq = 0;
	 946         else
	 947             dev_warn(&dev->dev, "IRQ %d doesn't match PIRQ mask "
	 948                  "%#x; try pci=usepirqmask\n", newirq, mask);
	 949     }
	 950     if (!newirq && assign) {
	 951         for (i = 0; i < 16; i++) {
	 952             if (!(mask & (1 << i)))
	 953                 continue;
	 954             if (pirq_penalty[i] < pirq_penalty[newirq] &&
	 955                 can_request_irq(i, IRQF_SHARED))
	 956                 newirq = i;
	 957         }
	 958     }
	 959     dev_dbg(&dev->dev, "PCI INT %c -> newirq %d", 'A' + pin - 1, newirq);
	 960     
	 961     /* Check if it is hardcoded */
	 962     if ((pirq & 0xf0) == 0xf0) {
	 963         irq = pirq & 0xf;
	 964         msg = "hardcoded";
	 965     } else if (r->get && (irq = r->get(pirq_router_dev, dev, pirq)) && \
	 966     ((!(pci_probe & PCI_USE_PIRQ_MASK)) || ((1 << irq) & mask))) {
	 967         msg = "found";
	 968         eisa_set_level_irq(irq);
	 969     } else if (newirq && r->set &&
	 970         (dev->class >> 8) != PCI_CLASS_DISPLAY_VGA) {
	 971         if (r->set(pirq_router_dev, dev, pirq, newirq)) {
	 972             eisa_set_level_irq(newirq);
	 973             msg = "assigned";
	 974             irq = newirq;
	 975         }
	 976     }
	 977    
	 978     if (!irq) {
	 979         if (newirq && mask == (1 << newirq)) {
	 980             msg = "guessed";
	 981             irq = newirq;
	 982         } else {
	 983             dev_dbg(&dev->dev, "can't route interrupt\n");
	 984             return 0;
	 985         }
	 986     }
	 987     dev_info(&dev->dev, "%s PCI INT %c -> IRQ %d\n", msg, 'A' + pin - 1, irq);
	 988    
	 989     /* Update IRQ for all devices with the same pirq value */
	 990     for_each_pci_dev(dev2) {
	 991         pci_read_config_byte(dev2, PCI_INTERRUPT_PIN, &pin);
	 992         if (!pin)
	 993             continue;
	 994    
	 995         info = pirq_get_info(dev2);
	 996         if (!info)
	 997             continue;
	 998         if (info->irq[pin - 1].link == pirq) {
	 999             /*
	1000              * We refuse to override the dev->irq
	1001              * information. Give a warning!
	1002              */
	1003             if (dev2->irq && dev2->irq != irq && \
	1004             (!(pci_probe & PCI_USE_PIRQ_MASK) || \
	1005             ((1 << dev2->irq) & mask))) {
	1006 #ifndef CONFIG_PCI_MSI
	1007                 dev_info(&dev2->dev, "IRQ routing conflict: "
	1008                      "have IRQ %d, want IRQ %d\n",
	1009                      dev2->irq, irq);
	1010 #endif 
	1011                 continue;
	1012             }
	1013             dev2->irq = irq;
	1014             pirq_penalty[irq]++;
	1015             if (dev != dev2)
	1016                 dev_info(&dev->dev, "sharing IRQ %d with %s\n",
	1017                      irq, pci_name(dev2));
	1018         }
	1019     }
	1020     return 1;
	1021 }  

	 864 static struct irq_info *pirq_get_info(struct pci_dev *dev) 
	 865 {
	 866     struct irq_routing_table *rt = pirq_table;
	 867     int entries = (rt->size - sizeof(struct irq_routing_table)) /
	 868         sizeof(struct irq_info);
	 869     struct irq_info *info;
	 870  
	 871     for (info = rt->slots; entries--; info++)
	 872         if (info->bus == dev->bus->number &&    
	 873             PCI_SLOT(info->devfn) == PCI_SLOT(dev->devfn))
	 874             return info;
	 875     return NULL; 
	 876 }    

寻找什么呢？前面讲过，如果知道一个设备的中断请求连到了路径互连器的哪条输入线，并且发现了这条线已经在路径互连器中连接到了中断控制器，也就知道了或者说发现了该设备的中断请求的最终去向。但是，如果在路径互连器中尚未连接呢？此时是否要在中断控制器的输入线中做出选择并完成连接呢？这就有pci_bios_irq()第二个参数决定，这里设置为0，表示如果没有连接就以后再说。

看代码，首先读寄存器PCI_INTERRUPT_PIN，以得知其中断请求连在PCI总线的哪条线上（INTA～INTD），读到的pin是从1开始的，所以可以看到下面在对pin进行操作时都是-1来修正的（如line910）。line904，调用pirq_get_info()从BIOS提供的中断请求路径表中，找到总线和插槽所在的路径信息。

pirq_get_info()根据总线号和插槽号来找到相应的irq_info数据结构，这个结构中的数组irq[4]记录着这个特定的插槽的4条中断请求线（INTA～INTD）跟中断路径互连器的连接。对于每条中断请求，数据结构中提供了两个字段。其中link的主体是路径互连器的输入线号；bitmap则是一个位图，表示哪些中断控制器的输入线可以作为选择目标。在bitmap的基础上还有全局屏蔽位图，line918，pcibios_irq_mask，定义为0xfff8，表示中断请求号0、1、2不能选用。至line918我们已经得到了pirq和mask。

后面还有一部分是为特定厂家服务的代码，这里不分析。到line929～line960，如果参数assign不为0的话，就需要尽量为设备分配一个新中断号，当然这里传入的assign为0，所以不管它。接着到这里：

	 961     /* Check if it is hardcoded */
	 962     if ((pirq & 0xf0) == 0xf0) {
	 963         irq = pirq & 0xf;
	 964         msg = "hardcoded";
	 965     } else if (r->get && (irq = r->get(pirq_router_dev, dev, pirq)) && \
	 966     ((!(pci_probe & PCI_USE_PIRQ_MASK)) || ((1 << irq) & mask))) {
	 967         msg = "found";
	 968         eisa_set_level_irq(irq);
	 969     } else if (newirq && r->set &&
	 970         (dev->class >> 8) != PCI_CLASS_DISPLAY_VGA) {
	 971         if (r->set(pirq_router_dev, dev, pirq, newirq)) {
	 972             eisa_set_level_irq(newirq);
	 973             msg = "assigned";
	 974             irq = newirq;
	 975         }
	 976     }

在BIOS提供的中断路径表中，字段link字段的高4位如果全为1，则表示路径互连器内部采用的是“硬连接”，此时其低4位就是中断控制器的输入线号；否则表示路径互连器内部的连接可以通过get和set两个函数来读出或设置。对于一般的路径互连器，只要get为非空，都是可以读出连接目标的。这里的r-get就用到在上篇博客中内核设置的irq_router啦。如果读出的irq为非0，那就表示有所连接的目标，否则就是尚未连接到中断控制器。如果没有连接的话就要看上面line929～line960做的工作了，如果分配了新的irq就需要set啦，当然这里的处理是：如果没有连接，就放着不管。

如果是已经连接到中断控制器，那么我们就搞清楚了目标设备中断请求的最终去向，就把这个信息记录到dev->irq中就可以啦。但是自己搞好了，也不要忘了别人啊。。。所以看最后有一次foreach，如果系统中有设备的pirq和我们处理的是一样的，就分享这个信息，也就是设置dev->irq，并进行惩罚计数。这样做的好处是避免一些重复的操作。

至此对于BIOS已经配置好中断的设备（从PIN到中断路径互连器到中断控制器本身就配置好的），我们就已经弄清楚中断去向了（dev->irq已经明了），下面就是要为PCI设备的各个地址空间分配总线地址了。下篇见！

##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译

