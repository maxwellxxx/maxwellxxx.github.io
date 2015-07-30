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
	  7    in the right sequence from here. */
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



那现在完成的工作就是为了枚举和配置做准备，实际上流程如何，且看下篇。


##参考目录
[1]《Linux内核情景分析》[中]毛德操等 [著]

[2]《深入Linux内核架构》[德]Wolfgang Mauerer 著 [中]郭旭 译



