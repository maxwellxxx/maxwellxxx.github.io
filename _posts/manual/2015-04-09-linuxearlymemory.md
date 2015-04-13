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

进入保护模式后,需要一个长跳转,隐式地设置cs:
	1.	movw	$__BOOT_DS, %cx		//数据段选择子
		movw	$__BOOT_TSS, %di	//TTS段选择子

		movl	%cr0, %edx
		orb	$X86_CR0_PE, %dl	# 开启Protected mode 
		movl	%edx, %cr0		//进入保护模式后需要一个长跳转
		# Transition to 32-bit mode
		.byte	0x66, 0xea		# ljmpl opcode	--| ljmpl字节码
	2:	.long	in_pm32			# offset  ---与上条组合成ljmpl in_pm32	  这时候cs已经为__BOOT_CS.这样可以跳到in_pm32
		.word	__BOOT_CS		# segment 代码段选择子!!!!!!!!!!!其实在这里就隐式的设置了代码段

就设置各个段选择子.所有段寄存器(ds、es、fs、gs、ss)都为设置为_BOOT_DS选择子.

!!!!!!!!!!1注意一旦进入保护模式,就开始以罗辑地址寻址了.即要对给定的地址首先进行段式管理转换,再进行页式管理转换.

跳入保护模式代码jmpl *%eax (这里eax值为0x100000),由于代码段基址为0,所以罗辑地址(段内偏移量)等于线性地址,再由于没有分页,所以线性地址就是物理地址.(请注意上面的内存分布).这里的0x100000为arch/x86/boot/compressed/head_32.S中的startup_32(),用于解压剩余的内核.






###第二次进入保护模式(第二次设置gdtr)

解压完内核后就应该跳入真正的内核,即内核中第二个startup_32().这个时候的整个vmlinux的编译链接地址都是从虚拟地址(线性地址)0xc0000000开始的,有必要重新设置下段寻址,这个是linux内核第二次设置段寻址,称为第二次进入保护模式.这一次设置的原因是在之前的处理过程中，指令地址是从物理地址0x100000开始的，而此时整个vmlinux的编译链接地址是从虚拟地址0xC0000000开始的，所以需要在这里重新设置boot_gdt的位置。

	L88-L107:
		ENTRY(startup_32)
		movl pa(stack_start),%ecx		//栈开始的物理地址
	
		/* test KEEP_SEGMENTS flag to see if the bootloader is asking
			us to not reload segments */
		testb $(1<<6), BP_loadflags(%esi)	//看是否需要设置保护模式环境
		jnz 2f

	/*
	 * Set segments to known values.
	 */
		lgdt pa(boot_gdt_descr)			//设置gdtr
		movl $(__BOOT_DS),%eax			//设置各个段选择子
		movl %eax,%ds
		movl %eax,%es
		movl %eax,%fs
		movl %eax,%gs
		movl %eax,%ss
	2:
		leal -__PAGE_OFFSET(%ecx),%esp		//设置栈顶

GDTR是一个长度为48bit的寄存器，内容为一个32位的基地址和一个16位的段限。其中32位的基址是指GDT在内存中的地址。lgdt后,加载boot_gdt_descr地址的内容,gdtr段限为__BOOT_DS+7,32位基址变为boot_gdt物理地址.

	L737:boot_gdt_descr:
	L738:	.word __BOOT_DS+7			//_BOOT_DS初始化时数据段选择子+7,=0x8f
	L739:	.long boot_gdt - __PAGE_OFFSET		//boot_gdt物理地址


	L757:ENTRY(boot_gdt)
	L758:	.fill GDT_ENTRY_BOOT_CS,8,0		//GDT_ENTRY_BOOT_CS=2
	L759:	.quad 0x00cf9a000000ffff		/* kernel 4GB code at 0x00000000 */
	L760:	.quad 0x00cf92000000ffff		/* kernel 4GB data at 0x00000000 */

.fill后申请了2个8字节内容为0的空间.L759表示4GB内核代码段描述符内容,起始地址为0x00000000,L760为4GB内核数据段描述符内容,起始地址为0x00000000.处于初始化阶段,不存在用户数据段和代码段.

!!!!!!!!!!!!!!!内核运行到这个时候,所有段基址都是0x00000000开始,而内核链接的线性地址都是从虚拟地址0xc0000000,但是这个时候还没有开启分页,那如果要访问一个变量应该怎么寻址呢?其实仅仅使用pa和va就完成了罗辑地址和物理地址的互换.

	#define pa(X) ((X)-__PAGE_OFFSET)
	#define va(X) ((X)+__PAGE_OFFSET)
	__PAGE_OFFSET=0xC0000000（3G,3G-4G为内核空间,32bit(罗辑地址寻址)）
	
	例如刚刚的:
	lgdt pa(boot_gdt_descr)			//设置gdtr


###开启第一次分页(建立临时内核页表)

不过虽然有pa和va大法,但是依然不是长久之计,当务之急是开启分页,在内核编译链接时,就已经存在了一张全局目录:

	L662:ENTRY(initial_page_table)
	L663	.fill 1024,4,0	//填充一个页面(4k)的空间

在第一次开启分页时就把这张表作为全局页=目录,将其地址给cr3寄存器开启分页的.

	movl $pa(initial_page_table), %eax
	movl %eax,%cr3		/* set the page table pointer.. */
	movl $CR0_STATE,%eax
	movl %eax,%cr0		/* ..and set paging (PG) bit */

!!!!!注意这里第一次分配内存!!!!!


那这张全局目录又是如何初始化,各个页表、表项又是怎么初始化的呢?在链接vmlinux时,有一个叫做BRK段,其开始地址为__brk_base.这个段的作用是保留给用户通过brk()系统调用向内核申请内存空间用的.这里我们先不管它.现在内核需要分配页表空间,应该从哪里开始呢?就从__brk_base分配.那么就需要把__brk_base的物理地址给initial_page_table的第0项还有第767项,分配时同时从第0和第767开始分配,至于为什么是767项,下面会解释.

那么从__brk_base开始的第一个页面就变成第一个页表,第二个页面就变成第二个页表....有多少个页表取决于内核_end的地址.总之要将整个内核都完成从物理地址到虚拟地址的映射.完成后,全局目录和页表的情况是这样的:

![img](/images/memory/pgtpte.png)


这里很奇怪,第0个页表的第0项为什么是0x003,而不是0x0呢?因为内存是以4k对齐了,所以地址表项中存的地址低12位是不表示地址的,这里用作各种标志位.不仅是页表中有这种情况,全局目录中每项页不是存的_brk_base的物理地址,比如第0项存的是_brk_base+0x67,0x67也作为标志位.

PTE_IDENT_ATTR常量,可见定义arch/x86/include/asm/pgtable_types.h:
	#define PTE_IDENT_ATTR  0x003		/* PRESENT+RW */
	#define PDE_IDENT_ATTR  0x067		/PRESENT+RW+USER+DIRTY+ACCESSED */
	#define PGD_IDENT_ATTR 0x001		/* PRESENT (no other attributes) */
	PRESENT=1 页没被交换出内存.PRESENT=0 页被交换出内存,访问内存会产生缺页中断

还有就是刚刚留下的问题,就是为什么要在全局页表的第767项同时分配,原因是这样的:第一次启动分页时目的时将整个内核的物理地址空间映射到虚拟地址空间.而内核在编译链接vmlinux时是从线性地址0xc0000000开始的,解压时是先将vmlinux拷贝到0x1000000以后的内存空间的,然后将它解压到拷贝前的内核镜像地址(0x100000以后).那么通过逻辑地址寻址时0xc0000000线性地址以后的地址需要通过分页映射到物理地址0x00000000开始的空间.0xc00000000>>20=0xc00 0xc00/4=768.所以在分配页表时需要从全局目录表第0和第767项同时开始.

这些工作都完成后,就完成了将物理地址0x00000000到内核_end内存空间映射到线性地址0x00000000开始和0xc0000000开始的内存空间. 这样的话,用逻辑地址0x00000000或者0xc0000000类似的地址都能访问到物理地址0x00000000开始的空间.




###第三次开启保护模式(第三次设置gdtr)

因为开启了分页所以需要设置一次gdtr.Linux x86 的分段管理是通过 GDTR 来实现的,那么现在就来总结一下 Linux 启动以来到现在,共设置了几次 GDTR:
<ul>
<li>1. 第一次还是 cpu 处于实模式的时候,运行 arch\x86\boot\pm.c 下 setup_gdt()函数的代码。该函数,设置了两个 GDT 项,一个是代码段可读/执行的,另一个是数据段可读写的,都是从 0-4G 直接映射到 0-4G,也就是虚拟地址和线性地址相等。</li>
<li>2. 第二次是在内核解压缩以后,用解压缩后的内核代码 arch\x86\kernel\head_32.S 再次对gdt 进行设置,这一次的设置效果和上一次是一样的。</li>
<li>3. 第三次同样是在 arch\x86\kernel\head_32.S 中,只不过是在开启了页面寻址之后,通过分页寻址得到编译好的全局描述符表 gdt 的地址。这一次效果就跟前两次不一样了,为内核最终使用的全局描述符表,同时也设置了 IDT。</li>
</ul>

Linux 启动以来自此加载的 gdt 已有以上若干个段的描述符在编译 vmlinux 的时候初始化了,其他没被初始化的地方暂时保留。

##文明世界---setup_arch()



###第二次设置分页(第二次设置cr3)

在start_arch()中会再次设置一次cr3:

	 873     /*
	 874      * copy kernel address range established so far and switch
	 875      * to the proper swapper page table
	 876      */
	 877     clone_pgd_range(swapper_pg_dir     + KERNEL_PGD_BOUNDARY,              
	 878             initial_page_table + KERNEL_PGD_BOUNDARY,
	 879             KERNEL_PGD_PTRS);
	 880     
	 881     load_cr3(swapper_pg_dir);

在上面,我们初始化了initial_page_table作为全局页目录表.这里把它复制给swapper_pg_dir,在这以后,swapper_pg_dir就一直当做全局目录表使用了....随后设置cr3,弃用以前的initial_page_table.

	定义在:arch/x86/kernel/head_32.S 
	659 initial_pg_pmd:
	660     .fill 1024*KPMDS,4,0
	661 #else  
	662 ENTRY(initial_page_table)
	663     .fill 1024,4,0
	664 #endif 
	665 initial_pg_fixmap:
	666     .fill 1024,4,0
	667 ENTRY(empty_zero_page)
	668     .fill 4096,1,0
	669 ENTRY(swapper_pg_dir)
	670     .fill 1024,4,0


###真正的内存管理

到现在为止,linux内核以上面状态进行了一些列工作(此处略过好多....)....终于来到第一个真正的内核管理函数:setup_memory_map().


我们在执行实模式下代码 main 函数的时候调用了一个 detect_memory_e820()函数,当时该函数通过 BIOS 服务程序 int 0x15 获得系统启动后的所有可用空间,共有boot_params.e820_entries个可用空间,每块空间作为 boot_params.e820_map[]数组的元素,存放着他们的起始地址、大小和元素。


这里说个题外话,探测一个 PC 机内存的最好方法是就是通过调用 INT 0x15,eax = 0xe820来实现。这个功能在 2002 年以后被所有 PC 机所使用,这是唯一能够探测超过 4G 大小内存的方案,当然,这个方法也可以被认为是内存的最终检测方法。实际上,这个函数返回一个非排序列表,这个列表包含了那些没有使用的内存信息的项,并且可能返回存在覆盖的区域。在 linux 中每个列表项被存放在 ES:EDI 指定的内存区域中。每个项均有一定的格式:即 2个8字节字 段,一个2字节字段。我们前面看见了,对于内存探测的实现由函数detect_memory_e820 来实现的,在这个函数中,使用了一个do...while()循环来实现,并将所探测的内容写入boot_params.e820_map 数组中。


setup_memory_map()会根据boot_params 中 e820_map 字段的值来设置全局变量 e820 的值。这个全局变量是一个e820map 结构:

	struct e820map {
	__u32 nr_map;
	struct e820entry map[E820_X_MAX];//E820MAX定义为128
	};

	struct e820entry {
	    __u64 addr;    /* start of memory segment */
	    __u64 size;    /* size of memory segment */
	    __u32 type;    /* type of memory segment */
	} __attribute__((packed));

这里略过其他过程,比如内存消毒等等....最后将boot_params.e820_map[]数组中的所有内存区物理内存区物理信息拷贝到全局变量e820中.并将e820中的数据拷贝到e820_saved中当做备份.


###max_pfn
回到setup_arch(),内核根据e820中的数据得到max_pfn:

	1058     max_pfn = e820_end_of_ram_pfn();

函数根据e820的数据来获得32位可用物理内存地址的最大值并右移PAGE_SHIFT,也就是12位最后由函数 e820_end_pfn 返回这个 20 位的值,保存在内部变量 max_pfn 中,作为总的页面数量.


###高端内存

start_arch()中接着,find_low_pfn_range();它根据max_pfn是否大于MAXMEM_PFN来判断是否启用了高端内存.

	644 void __init find_low_pfn_range(void) 
	645 {       
	646     /* it could update max_pfn */
	647      
	648     if (max_pfn <= MAXMEM_PFN)
	649         lowmem_pfn_init();
	650     else
	651         highmem_pfn_init();		//现在系统一般进入到这里了
	652 }  

这个MAXMEM_PFN跟max_pfn一样,也是除去低12位的20位的一个数值。如果启用了高端内存,就调用highmem_pfn_init()函数将全局变量max_low_pfn设置为MAXMEM_PFN;并设置全局变量 highmem_pages 为 max_pfn - MAXMEM_PFN,作为高端页面的数量:

	607 static void __init highmem_pfn_init(void)
	608 {
	609     max_low_pfn = MAXMEM_PFN;
	610  
	611     if (highmem_pages == -1)
	612         highmem_pages = max_pfn - MAXMEM_PFN;
	613  
	614     if (highmem_pages + MAXMEM_PFN < max_pfn)
	615         max_pfn = MAXMEM_PFN + highmem_pages;
	616  
	617     if (highmem_pages + MAXMEM_PFN > max_pfn) {
	618         printk(KERN_WARNING MSG_HIGHMEM_TOO_SMALL,
	619             pages_to_mb(max_pfn - MAXMEM_PFN),
	620             pages_to_mb(highmem_pages));
	621         highmem_pages = 0;
	622     }   
	623 #ifndef CONFIG_HIGHMEM
	624     /* Maximum memory usable is what is directly addressable */
	625     printk(KERN_WARNING "Warning only %ldMB will be used.\n", MAXMEM>>20);
	626     if (max_pfn > MAX_NONPAE_PFN)
	627         printk(KERN_WARNING "Use a HIGHMEM64G enabled kernel.\n");
	628     else
	629         printk(KERN_WARNING "Use a HIGHMEM enabled kernel.\n");
	630     max_pfn = MAXMEM_PFN;
	....
	639 }  

###小结

内核运行到这里,我们来做下总结,看下现在的状态.
<ul>
<li>1. 已经设置了最终gdt,开启了段式内存管理,内核代码和数据段都是0x00000000.这样的结果就是逻辑地址经过硬件平台段式管理转换后得到的线性地址和逻辑地址相同.</li>
<li>2. 已经设置了cr3,开启了分页式内存管理.最后的全局目录表是swapper_pg_dir,页表还是在__brk_base开始.将物理地址0x00000000-_end的空间映射到虚拟地址(线性地址)0x00000000开始和0xc0000000开始(即全局目录同时从0项和767项分配).这样导致的结果是由虚拟地址0xc0000000开始链接的内核可以正常通过逻辑地址寻址.</li>
<li>3. 内核通过int 0x15获取物理内存布局,并存在e820全局数组中.</li>
</ul>

而且需要注意的是,这个时候内核基本不能动态分配管理内存,上面少数(我记得是唯一)动态分配内存的方式也仅仅是从brk中分配页表.
###着手建立最终内核页表

首先我们来看几个全局变量的值(假设开启高端内存).
<ul>
<li>max_pfn=32位可用物理内存地址的最大值并右移PAGE_SHIFT(20位):所有可用物理内存总页框数.</li>
<li>max_low_pfn=MAXMEM_PFN:低端内存总页框数</li>
<li>highmem_pages = max_pfn - MAXMEM_PFN:高端内存总页框数</li>
</ul>

有了这些信息就开始建立最终内核页表了,这是个什么东西呢?如果大家还没失忆的话,应该记得前面已经初始化过一次页表了,而那个页表是临时内核页表.为什么说它是临时的呢?因为临时页表个数只取决于_end的位置,也就是仅仅把解压缩后的内核代码映射出来了,目的是可以通过逻辑地址正常寻址.全局目录表是initial_page_table,在setup_arch()初期复制给了swapper_pg_dir.


那最终页表是怎么样的呢?最终页表需要映射所有可用的RAM(注意这里包括了已经映射了的内核代码,页目录)以页为单位分成多个页,每个页一个比特,提供一个初始阶段内存的分配和释放管理平台(!!!!!!!!!!!),全局目录表是swapper_pg_dir.


(maxwellxxx!!!!这个可能是老版内核的分配方式,需要考证是否现在还建立这个bit map)

先不管这个bit map和初期内存分配和释放平台,首先是建立最终内核页表,建立时分为三种情况:
<ul>
<li>第一种:RAM小于896MB时,也就是没有高端内存的情况.</li>
这种情况是把所有的RAM全部映射到0xc0000000,由内核页表所提供的最终映射必须把从0xc0000000开始的线性地址转化为从0开始的物理地址,主内核页全局目录仍然保存在swapper_pg_dir变量中。

<li>第二种:RAM在896Mb到4GB之间,也就是存在高端内存的情况.</li>
这种情况就不能把所有的RAM全部映射到内核空间了,Linux在初始化阶段只是把一个具有896MB的RAM映射到内核线性地址空间。如果一个程序需要对现有RAM的其余部分寻址，那就必须把某些其他的线性地址间隔映射到所需的RAM，做法就是修改某些页表项的值。内核使用与前一种情况相同的代码来初始化页全局目录。

<li>第三种:RAM在4GB以上.</li>
现代计算机，特别是些高性能的服务器内存远远超过4GB，那么内核页表初始化怎么做呢；更确切地说，我们处理以下发生的情况：
• CPU模式支持物理地址扩展（PAE）
• RAM容量大于4GB
• 内核以PAE支持来编译

尽管PAE处理36位物理地址，但是线性地址依然是32位地址。如前所述，Linux映射一个896MB的RAM到内核地址空间；剩余RAM留着不映射，并由动态重映射来处理。与前一种情况的主要差异是使用三级分页模型。其实，即使我们的CPU支持PAE，但是也只能有寻址能力为64GB的内核页表，所以，如果要建立更高性能的服务器，建议改善动态重映射算法，或者干脆升级为64位的处理器。
<ul>


我们这里仅讨论第二种情况.注意,是要映射到线性地址0xc0000000开始的地址,也就意味着只要从全局目录项767开始就行了.

既然要建立映射,按照从前建立临时页表的套路,应该要有个全局目录表,然后分配若干页表并初始化完成映射.这里全局目录表已经有了,不过现在的分页机制还得靠它,暂时还有用,先不管它.那么就剩下分配页表空间了...


关于分配页表空间,这里说下,曾经的内核是简单粗暴,直接从e820中找一块连续内存供所有页表空间使用.而我分析的这个内核版本(3.19.x)已经不这么玩了,它是这么搞的:

!!!!!注意这里第二次分配内存!!!!!

start_arch中的1088 early_alloc_pgt_buf();它是为前期的页表分配空间.具体分配了两种页表,看注释是分配了12k给初始化时用页表,12k给了ISA空间映射用页表.也就是分配12k的页表空间作为前期内存分页用,其余的再动态分配,而不像以前那样分配全部页表空间.

	117 /* need 3 4k for initial PMD_SIZE,  3 4k for 0-ISA_END_ADDRESS */
	118 #define INIT_PGT_BUF_SIZE   (6 * PAGE_SIZE)
	120 void  __init early_alloc_pgt_buf(void) 
	121 {
	122     unsigned long tables = INIT_PGT_BUF_SIZE;	6*4096
	123     phys_addr_t base;
	124      
	125     base = __pa(extend_brk(tables, PAGE_SIZE));	//扩展brk,但是不能超过brk_limit
	126      
	127     pgt_buf_start = base >> PAGE_SHIFT;		//pgt开始的页框数
	128     pgt_buf_end = pgt_buf_start;			//已分配pgt结束位置的页框数
	129     pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);//分配的pgt空间结尾页框数
	130 }  

可以看到是从brk中分配内存作为页表空间,就和之前建立页表时一样,这边其实进行第二次分页,需要重做之前的工作.注意扩展时没有覆盖以前的页表!

	262 void * __init extend_brk(size_t size, size_t align)
	263 {        
	264     size_t mask = align - 1;
	265     void *ret;
	266          
	267     BUG_ON(_brk_start == 0);
	268     BUG_ON(align & mask);
	269          
	270     _brk_end = (_brk_end + mask) & ~mask;	//没有覆盖曾经的页表空间
	271     BUG_ON((char *)(_brk_end + size) > __brk_limit);
	272          
	273     ret = (void *)_brk_end;
	274     _brk_end += size;			//分配size(6*4096)
	275          
	276     memset(ret, 0, size);
	277          
	278     return ret;
	279 } 


分配完初始阶段的部分页表空间后就要开始真正的建立页表了!

	555 void __init init_mem_mapping(void)
	556 {
	557     unsigned long end;
	558    
	559     probe_page_size_mask();
	560    
	561 #ifdef CONFIG_X86_64
	562     end = max_pfn << PAGE_SHIFT;		//64位处理器
	563 #else					//获取最大内存地址
	564     end = max_low_pfn << PAGE_SHIFT;	//END=896mb!!!!!!!!!
	565 #endif
	566    
	567     /* the ISA range is always mapped regardless of memory holes */
	568     init_memory_mapping(0, ISA_END_ADDRESS);//0x100000
	569    
	570     /*
	571      * If the allocation is in bottom-up direction, we setup direct mapping
	572      * in bottom-up, otherwise we setup direct mapping in top-down.
	573      */
	574     if (memblock_bottom_up()) {
	575         unsigned long kernel_end = __pa_symbol(_end);
	576    
	577         /*
	578          * we need two separate calls here. This is because we want to
	579          * allocate page tables above the kernel. So we first map
	580          * [kernel_end, end) to make memory above the kernel be mapped
	581          * as soon as possible. And then use page tables allocated above
	582          * the kernel to map [ISA_END_ADDRESS, kernel_end).
	583          */
	584         memory_map_bottom_up(kernel_end, end);
	585         memory_map_bottom_up(ISA_END_ADDRESS, kernel_end);
	586     } else {
	587         memory_map_top_down(ISA_END_ADDRESS, end);
	588     }
	589    
	590 #ifdef CONFIG_X86_64
	591     if (max_pfn > max_low_pfn) {
	592         /* can we preseve max_low_pfn ?*/
	593         max_low_pfn = max_pfn;
	594     }
	595 #else
	596     early_ioremap_page_table_range_init();		
	597 #endif
	598    
	599     load_cr3(swapper_pg_dir);
	600     __flush_tlb_all();
	601    
	602     early_memtest(0, max_pfn_mapped << PAGE_SHIFT);
	603 }  

根据那篇转的高端内存映射的博文,我们可以知道,这边只要负责映射低端内存空也就是0-896mb.而这部分内存根据前面那个函数可以看到,也分为两个部分开始映射:
<ul>
<li>首先映射0-ISA_END_ADDRESS(0x100000)的RAM</li>
<li>接着映射0x100000~896mb的RAM</li>
</ul>
