---
layout: post
title: Linux设备驱动模型及其他(3)
description: 今天是windows10发布的日子了!!
category: manual
---

>上章讲到PCI配置寄存器组，以及枚举和配置设备的大致工作原理。

##准备工作
上两章中已经提到过，Linux可以不使用BIOS提供的PCI服务来进行枚举和配置设备，而这些工作主要是通过读写0xCF8～0xCFF两个I/O寄存器来完成的。接下来我们就来看下Linux内核代码中是如何操作的。《Linux内核情景分析》中参考的内核版本是2.4.x的，这里我们分析的是3.1.9(虽然4.x都发布了)，这么多年来内核早以改的“面目全非”，主要是很多特性的加入，比如设备驱动模型等。这里如果没有特殊说明，我们还是按照3.1.9版本来分析。

首先来看几个底层的函数，也就是通过寄存器0xCF8和0xCFC读写目标设备上的配置寄存器，首先来看到

	arch/x86/pci/init.c:
	  8 static __init int pci_arch_init(void)
	  9 {
	 10 #ifdef CONFIG_PCI_DIRECT
	 11     int type = 0;
	 12
	 13     type = pci_direct_probe();
	 14 #endif
	 15 
	 16     if (!(pci_probe & PCI_PROBE_NOEARLY))
	 17         pci_mmcfg_early_init();
	 18 
	 19     if (x86_init.pci.arch_init && !x86_init.pci.arch_init())
	 20         return 0;
	 21 
	 22 #ifdef CONFIG_PCI_BIOS
	 23     pci_pcbios_init();
	 24 #endif
	 25     /*
	 26      * don't check for raw_pci_ops here because we want pcbios as last
	 27      * fallback, yet it's needed to run first to set pcibios_last_bus
	 28      * in case legacy PCI probing is used. otherwise detecting peer busses
	 29      * fails.
	 30      */
	 31 #ifdef CONFIG_PCI_DIRECT
	 32     pci_direct_init(type);
	 33 #endif
	 34     if (!raw_pci_ops && !raw_pci_ext_ops)
	 35         printk(KERN_ERR
	 36         "PCI: Fatal: No config space access function found\n");
	 37 
	 38     dmi_check_pciprobe();
	 39 
	 40     dmi_check_skip_isa_align();
	 41 
	 42     return 0;
	 43 }
	 44 arch_initcall(pci_arch_init); 

这个文件里就这么个函数了，但是这个函数很重要。先看最后的arch_initcall(pci_arch_init)，这句话非常的重要，作用相当于注册一个函数链，然后会在适当是时候调用，具体在神马时候呢？可以先告诉你：

	start_kernel()
	 |
	 |
	 reset_init()   //在这里就开始创建kernel_init进程啦！
	  |
	  |
	  kernel_init()
	   |
	   |
	   kernel_init_freeable()   //这个函数真的是做了一万件事情！！
	    |
	    |
	    do_basic_setup()
	     |
	     |
	     do_initcalls()      //就是它了，负责将xxx_initcall()注册的函数有顺序的执行！

关于do_initcalls()其实还大有学问，但这里不是我们讨论的重点，以后说不定会总结下，如果想了解，可以看这里：<a href="http://blog.csdn.net/ericghw/article/details/8302689">linux initcall机制</a>。言归正传，do_initcalls()自然会调用到上面的函数，那这个函数具体是在做神马咧？


先看10行的CONFIG_PCI_DIRECT，第一篇就说明了,Linux可以在编译时选择这个选项，来绕过BIOS的PCI服务，就体现在这里啦！！进而调用pci_direct_probe()，这个函数也很重要，里面有上篇文章一个问题的答案：

	arch/x86/pci/direct.c
	283 int __init pci_direct_probe(void)
	284 {  
	285     if ((pci_probe & PCI_PROBE_CONF1) == 0)
	286         goto type2;
	287     if (!request_region(0xCF8, 8, "PCI conf1"))
	288         goto type2;
	289    
	290     if (pci_check_type1()) {
	291         raw_pci_ops = &pci_direct_conf1;
	292         port_cf9_safe = true;
	293         return 1;
	294     }
	295     release_region(0xCF8, 8);
	296    
	297  type2:
	298     if ((pci_probe & PCI_PROBE_CONF2) == 0)
	299         return 0;
	300     if (!request_region(0xCF8, 4, "PCI conf2"))
	301         return 0;
	302     if (!request_region(0xC000, 0x1000, "PCI conf2"))
	303         goto fail2;
	304    
	305     if (pci_check_type2()) {
	306         raw_pci_ops = &pci_direct_conf2;
	307         port_cf9_safe = true;
	308         return 2;
	309     }
	310    
	311     release_region(0xC000, 0x1000);
	312  fail2:
	313     release_region(0xCF8, 4);
	314     return 0;
	315 } 

	//这里的pci_probe是个全局变量，定义在（arch/x86/pci/common.c）	
	 23 unsigned int pci_probe = PCI_PROBE_BIOS | PCI_PROBE_CONF1 | PCI_PROBE_CONF2 | 
 	 24                 PCI_PROBE_MMCONF;

已经讲过“宿主——PCI桥”的I/O口一定是在地址为0xCF8~0xCFF处，其中0xCF8~0xCFB为地址口，0xCFC～0xCFF为数据口。这种配置就是这里所谓的PCI_PROBE_CONF1，当然还有PCI_PROBE_CONF2，但是好像已经完全弃用了。。。所以在这里就可以看到是优先尝试TYPE1的，这里注意第287行！request_region()这个函数，还记得cat /proc/ioports么？就是在这里的！！这个函数负责注册资源，这个函数放在后面重点介绍。

291行，将raw_pci_ops的值赋为pci_direct_conf1的地址，也就TYPE1型的读写PCI配置空间的函数：

	//raw_pci_ops为一个全局数据结构实例，作用是记录PCI配置空间寄存器读写操作的函数。（arch/x86/pci/common.c）
	 38 const struct pci_raw_ops *__read_mostly raw_pci_ops
	****************************************************************
	//pci_direct_conf1定义（arch/x86/pci/direct.c ）
	 82 const struct pci_raw_ops pci_direct_conf1 = {
	 83     .read =     pci_conf1_read,
	 84     .write =    pci_conf1_write,
	 85 };
	****************************************************************
	//pci_raw_ops结构定义（arch/x86/include/asm/pci_x86.h）
	 98 struct pci_raw_ops { 
	 99     int (*read)(unsigned int domain, unsigned int bus, unsigned int devfn,
	100                         int reg, int len, u32 *val);
	101     int (*write)(unsigned int domain, unsigned int bus, unsigned int devfn,
	102                         int reg, int len, u32 val);
	103 };

情况已经很明了了,正真的读写函数是pci_conf1_read和pci_conf1_write,我们这里只分析下read的实现,因为代码也说了__read_mostly 偷个懒(^^):

	 arch/x86/pci/direct.c
	 10 /*
	 11  * Functions for accessing PCI base (first 256 bytes) and extended
	 12  * (4096 bytes per PCI function) configuration space with type 1
	 13  * accesses.
	 14  */
	 15  
	 16 #define PCI_CONF1_ADDRESS(bus, devfn, reg) \
	 17     (0x80000000 | ((reg & 0xF00) << 16) | (bus << 16) \
	 18     | (devfn << 8) | (reg & 0xFC))
	 19    
	 20 static int pci_conf1_read(unsigned int seg, unsigned int bus,
	 21               unsigned int devfn, int reg, int len, u32 *value)
	 22 {  
	 23     unsigned long flags;
	 24    
	 25     if (seg || (bus > 255) || (devfn > 255) || (reg > 4095)) {
	 26         *value = -1;
	 27         return -EINVAL;
	 28     }
	 29    
	 30     raw_spin_lock_irqsave(&pci_config_lock, flags);
	 31    
	 32     outl(PCI_CONF1_ADDRESS(bus, devfn, reg), 0xCF8);
	 33    
	 34     switch (len) {
	 35     case 1:
	 36         *value = inb(0xCFC + (reg & 3));
	 37         break;
	 38     case 2:
	 39         *value = inw(0xCFC + (reg & 2));
	 40         break;
	 41     case 4:
	 42         *value = inl(0xCFC);
	 43         break;
	 44     }
	 45    
	 46     raw_spin_unlock_irqrestore(&pci_config_lock, flags);
	 47    
	 48     return 0;
	 49 }  

首先看宏定义,PCI_CONF1_ADDRESS,可以看到完全是按照第二章中的要求合成的设备综合地址,如果忘了,还是看下图:
***
![JCQ](/images/JCQ.png)
***
正真读的时候也完全是按照流程,先向地址寄存器0XCF8写综合地址(line32).然后根据请求的长度len来分情况处理,分别从内容寄存器0xCFC读一个byte,word,long。！！！注意配置空间是小端的形式，所以留意这里的处理手法,写的时候也要注意转换！！！


好了，现在TYPE已经确定为1了，相关的读写操作也已经设置好了，我们回到pci_arch_init()里，看到31行调用了pci_direct_init(),这个函数没干嘛，跳过！至此就没什么了，后面两个是DMI相关，这里我们不关心。


最后无论是读还是写操作必须封装为统一的接口，让后续的所有操作统一来调用，而不用在意到底是TYPE1还是TYPE2，就在arch/x86/pci/common.c中：

	 41 int raw_pci_read(unsigned int domain, unsigned int bus, unsigned int devfn,
	 42                         int reg, int len, u32 *val)
	 43 {  
	 44     if (domain == 0 && reg < 256 && raw_pci_ops)
	 45         return raw_pci_ops->read(domain, bus, devfn, reg, len, val);
	 46     if (raw_pci_ext_ops)
	 47         return raw_pci_ext_ops->read(domain, bus, devfn, reg, len, val);
	 48     return -EINVAL;
	 49 }  
	 50    
	 51 int raw_pci_write(unsigned int domain, unsigned int bus, unsigned int devfn,
	 52                         int reg, int len, u32 val)
	 53 {  
	 54     if (domain == 0 && reg < 256 && raw_pci_ops)
	 55         return raw_pci_ops->write(domain, bus, devfn, reg, len, val);
	 56     if (raw_pci_ext_ops)
	 57         return raw_pci_ext_ops->write(domain, bus, devfn, reg, len, val);
	 58     return -EINVAL;
	 59 }  
	 60    
	 61 static int pci_read(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 *value)
	 62 {  
	 63     return raw_pci_read(pci_domain_nr(bus), bus->number,
	 64                  devfn, where, size, value);
	 65 }  
	 66    
	 67 static int pci_write(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 value)
	 68 {  
	 69     return raw_pci_write(pci_domain_nr(bus), bus->number,
	 70                   devfn, where, size, value);
	 71 } 
	 72     
	 73 struct pci_ops pci_root_ops = {
	 74     .read = pci_read,
	 75     .write = pci_write,
	 76 };  

还没完，这里的pci_root_ops会赋值给0号总线的pci_bus结构，关于这个结构，以后会涉及到，这里先简单提一下：

	bus = pci_scan_root_bus(NULL, busnum, &pci_root_ops, sd, &resources); 
	      					   |
	      	 				   |
	      b = pci_create_root_bus(parent, bus, ops, sysdata, resources);

	//pci_bus结构 （include/linux/pci.h）
	 444 struct pci_bus {
	 445     struct list_head node;      /* node in list of buses */ 
	 446     struct pci_bus  *parent;    /* parent bus this bridge is on */
	 447     struct list_head children;  /* list of child buses */
	 448     struct list_head devices;   /* list of devices on this bus */
	 449     struct pci_dev  *self;      /* bridge device as seen by parent */
	 450     struct list_head slots;     /* list of slots on this bus */
	 451     struct resource *resource[PCI_BRIDGE_RESOURCE_NUM];
	 452     struct list_head resources; /* address space routed to this bus */
	 453     struct resource busn_res;   /* bus numbers routed to this bus */
	 454
	 455     struct pci_ops  *ops;       /* configuration access functions */
	 456     struct msi_controller *msi; /* MSI controller */
	 457     void        *sysdata;   /* hook for sys-specific extension */
	 .........................
	 477 }; 

到这里你以为万事大吉了么？其实还没有，这里需要看到（drivers/pci/access.c）
 
	 24 #define PCI_byte_BAD 0
	 25 #define PCI_word_BAD (pos & 1)
	 26 #define PCI_dword_BAD (pos & 3)
	 27 
	 28 #define PCI_OP_READ(size,type,len) \
	 29 int pci_bus_read_config_##size \
	 30     (struct pci_bus *bus, unsigned int devfn, int pos, type *value) \
	 31 {                                   \
	 32     int res;                            \
	 33     unsigned long flags;                        \
	 34     u32 data = 0;                           \
	 35     if (PCI_##size##_BAD) return PCIBIOS_BAD_REGISTER_NUMBER;   \
	 36     raw_spin_lock_irqsave(&pci_lock, flags);            \
	 37     res = bus->ops->read(bus, devfn, pos, len, &data);      \
	 38     *value = (type)data;                        \
	 39     raw_spin_unlock_irqrestore(&pci_lock, flags);       \
	 40     return res;                         \
	 41 } 
	 42 
	 43 #define PCI_OP_WRITE(size,type,len) \
	 44 int pci_bus_write_config_##size \
	 45     (struct pci_bus *bus, unsigned int devfn, int pos, type value)  \
	 46 {                                   \
	 47     int res;                            \
	 48     unsigned long flags;                        \
	 49     if (PCI_##size##_BAD) return PCIBIOS_BAD_REGISTER_NUMBER;   \
	 50     raw_spin_lock_irqsave(&pci_lock, flags);            \
	 51     res = bus->ops->write(bus, devfn, pos, len, value);     \
	 52     raw_spin_unlock_irqrestore(&pci_lock, flags);       \
	 53     return res;                         \
	 54 }
	 55
	 56 PCI_OP_READ(byte, u8, 1)
	 57 PCI_OP_READ(word, u16, 2)
	 58 PCI_OP_READ(dword, u32, 4)
	 59 PCI_OP_WRITE(byte, u8, 1)
	 60 PCI_OP_WRITE(word, u16, 2)
	 61 PCI_OP_WRITE(dword, u32, 4)
	 62
	 63 EXPORT_SYMBOL(pci_bus_read_config_byte);
	 64 EXPORT_SYMBOL(pci_bus_read_config_word);
	 65 EXPORT_SYMBOL(pci_bus_read_config_dword);
	 66 EXPORT_SYMBOL(pci_bus_write_config_byte);
	 67 EXPORT_SYMBOL(pci_bus_write_config_word);
	 68 EXPORT_SYMBOL(pci_bus_write_config_dword);

看line29宏定义，size作为字符串替换，联合line58就制造出了pci_bus_read_config_dword,最后再export。这里的一系列宏定义最终生产出了line63～line68这一系列的函数，而以后在内核在枚举和配置时也就是通过调用这6个导出函数实现的。（而这六个函数并不能通过cscope等代码导航工具探测到。。。所以要注意啦！）

那现在完成的工作就是为了枚举和配置做准备，真的是万事具备，只欠东风了，而实际上流程如何，且看下篇。


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译



