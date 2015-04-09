---
layout: post
title: Linux内核初始化阶段内存管理的几种阶段
description: 讲述从内核加载到建立完整的内存管理所经历的几种阶段
category: manual
---

本文旨在讲述从引导到完全建立内存管理体系过程中,内核对内存管理所经历的几种状态.

##一些讲在前面的话

在很久很久以前,linux内核还是支持直接从磁盘直接启动,也就是内核镜像自带了一个可以引导的MBR,按照套路计算机上电以后BIOS会将MBR加载到0000:7c00处执行.后来时过境迁,linux内核必须通过grub这些东西来引导了.本文的故事就从grub引导开始....

##内核镜像内存布局
现在计算机上电后BIOS会将grub载入到0000:7c00处执行.grub会根据配置从磁盘将内核载入到内存...过程这里就忽略了,载入完成后内存是这么个分布情况:

![sea](/images/memory/memorylocation.png)


实模式下共20根地址线,能访问到0x100000(1M)的内存.寄存器为16位.地址转换方式为"左移四位加偏移"比如es=0x1000,DI=0xffff,那么es:DI=0x1ffff.实模式下段寄存器存放各段基址,通过段+偏移来寻址. 偏移地址称为有效地址,表示操作数所在单元到段首的距离即逻辑地址的偏移地址. 换算出的地址称为线性地址,在实模式下也为物理地址.(关于虚拟地址,物理地址,线性地址,罗辑地址,请看博客)

那么实模式下能访问的最大物理地址为0xfffff,grub是通过暂时开启保护模式将内核镜像的一部分载入到0x100000的.

由于不同的bootloader由不同大小,x的值(即实模式加载的内存地址也不同),grub将x设为0x9000.grub载入内核大致过程如下:
<ul>
<li>1.       调用一个BIOS过程显示“Loading”信息。</li>
<li>2.       调用一个BIOS过程从磁盘装入内核映像的初始部分，即将内核映像的第一个512字节加载到物理地址0x0009000开始的内存中，而将setup程序的代码（参见后面的内存布局）从地址0x00090200开始存入RAM中。</li>
<li>3.       调用一个BIOS过程从磁盘中装载其余的内核映像，并把内核映像放入从低地址0x00010000（适用于使用make zImage编译的小内核映像）或者从高地址0x00100000（适用于使用make bzImage编译的大内核映像，也就是我们现在的情况）开始的RAM中。大内核映像的支持虽然本质上与其他启动模式相同，但是它却把数据放在不同的物理内存地址，以避免ISA黑洞问题。</li>
<li>4.       跳转到arch/x86/boot/header.S的_start处开始执行。</li>
</ul>

##x个阶段

###准备进入第一次保护模式(内存管理:野蛮)

这个阶段段寄存器保存各个段的基址.

这个时候是grub才将控制权交给内核,汇编代码会简单设置堆栈(ss,esp)和清理bss然后跳入C语言了.进入arch/x86/boot/main.c,调用go_to_protected_mode()进入保护模式(pm.c)

为了进入保护模式,需要先设置gdt,这个时候的gdt为boot_gdt,代码段和数据段描述符中的基址都为0.设置完后就开启保护模式.

	 arch/x86/boot/pm.c    
	 66 static void setup_gdt(void)
	 67 {           
	 68     /* There are machines which are known to not boot with the GDT
	 69        being 8-byte unaligned.  Intel recommends 16 byte alignment. */
	 70     static const u64 boot_gdt[] __attribute__((aligned(16))) = {
	 71         /* CS: code, read/execute, 4 GB, base 0 */
	 72         [GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
	 73         /* DS: data, read/write, 4 GB, base 0 */
	 74         [GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
	 75         /* TSS: 32-bit tss, 104 bytes, base 4096 */
	 76         /* We only have a TSS here to keep Intel VT happy;
	 77            we don't actually use it for anything. */
	 78         [GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
	 79     };      
	 80     /* Xen HVM incorrectly stores a pointer to the gdt_ptr, instead
	 81        of the gdt_ptr contents.  Thus, make it static so it will
	 82        stay in memory, at least long enough that we switch to the
	 83        proper kernel GDT. */
	 84     static struct gdt_ptr gdt;
	 85             
	 86     gdt.len = sizeof(boot_gdt)-1;
	 87     gdt.ptr = (u32)&boot_gdt + (ds() << 4);
	 88             
	 89     asm volatile("lgdtl %0" : : "m" (gdt));	//加载段描述符
	 90 } 

###第一次进入保护模式,为了解压内核(内存管理:野蛮)

进入保护模式后,就设置各个段选择子.所有段寄存器(ds、es、fs、gs、ss)都为设置为_BOOT_DS选择子.

跳入保护模式代码jmpl *%eax (这里eax值为0x100000),由于代码段基址为0,所以线性地址等于有效地址,再由于没有分页,所以线性地址就是物理地址.(请注意上面的内存分布).这里的0x100000为arch/x86/boot/compressed/head_32.S中的startup_32(),用于解压剩余的内核.


###准备第二次进入保护模式



