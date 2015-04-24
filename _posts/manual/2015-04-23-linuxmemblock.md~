---
layout: post
title: Linux内核初期内存管理---memblock
description: 张正宇说,博客要短些.....
category: manual
---

##一些说明
以前只知道内核有个比较早的内存管理机制叫做bootmem的.后来的版本(笔者分析的是3.19)好像把bootmem弃用了,取而代之的是__alloc_memory_core_early(),而这个函数其实就是调用的memblock来分配内的.这个以后的博客会有分析.这两天分析linux各个阶段的内存管理,对这个memblock重视了起来....

我查了一些资料,对memblock的作用好像讲述的都不太一样.有说是bootmem的替代品;有说是在bootmem初始化前用于内存管理的;还有说是Kernel对于物理内存使用情况的记录.一开始也被弄糊涂了,但其实memblock算法是linux内核初始化阶段的一个内存分配器,本质上是取代了原来的bootmem算法.memblock实现比较简单,而它的作用就是在page allocator初始化之前来管理内存,完成分配和释放请求.


##声明&结构


首先来看下memblock结构的定义,文件是include/linux/memblock.h:

	 42 struct memblock {
	 43     bool bottom_up;  /* is bottom up direction? */
	 44     phys_addr_t current_limit;
	 45     struct memblock_type memory;
	 46     struct memblock_type reserved;
	 47 #ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	 48     struct memblock_type physmem;
	 49 #endif
	 50 }; 

<ul>
<li>bottom_up:表示分配器分配内存的方式,true:从低地址(内核映像的尾部)向高地址分配;false:也就是top-down,从高地址向地址分配内存.</li>
<li>current_limit:用于限制通过memblock_alloc的内存申请.</li>
<li>memory:是可用内存的集合.</li>
<li>reserved:已分配内存的集合.</li>
</ul>

更详细的结构定义:

	 35 struct memblock_type {
	 36     unsigned long cnt;  /* number of regions */
	 37     unsigned long max;  /* size of the allocated array */
	 38     phys_addr_t total_size; /* size of all regions */
	 39     struct memblock_region *regions;
	 40 }; 
	 41  

<ul>
<li>cnt:当前集合(memory或者reserved)中记录的内存区域个数.</li>
<li>max:当前集合(memory或者reserved)中可记录的内存区域的最大个数.</li>
<li>total_size:集合记录区域信息大小.</li>
<li>regions:内存区域结构指针.</li>
</ul>

	 26 struct memblock_region {
	 27     phys_addr_t base;
	 28     phys_addr_t size;
	 29     unsigned long flags;
	 30 #ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
	 31     int nid;
	 32 #endif        
	 33 }; 
	 34    

<ul>
<li>base:内存区域起始地址.</li>
<li>size:内存区域大小.</li>
<li>flags:标记.</li>
<li>nid:node号.</li>
</ul>

在编译时,会分配好memblock结构所需要的内存空间,文件是mm/memblock.c:

	  28 static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
	  29 static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
	  30 #ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	  31 static struct memblock_region memblock_physmem_init_regions[INIT_PHYSMEM_REGIONS] __initdata_memblock;
	  32 #endif
	  33  
	  34 struct memblock memblock __initdata_memblock = {
	  35     .memory.regions     = memblock_memory_init_regions,
	  36     .memory.cnt     = 1,    /* empty dummy entry */
	  37     .memory.max     = INIT_MEMBLOCK_REGIONS,
	  38    
	  39     .reserved.regions   = memblock_reserved_init_regions,
	  40     .reserved.cnt       = 1,    /* empty dummy entry */
	  41     .reserved.max       = INIT_MEMBLOCK_REGIONS,
	  42    
	  43 #ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	  44     .physmem.regions    = memblock_physmem_init_regions,
	  45     .physmem.cnt        = 1,    /* empty dummy entry */
	  46     .physmem.max        = INIT_PHYSMEM_REGIONS,
	  47 #endif
	  48    
	  49     .bottom_up      = false,
	  50     .current_limit      = MEMBLOCK_ALLOC_ANYWHERE,
	  51 }; 

其中INIT_MEMBLOCK_REGIONS为128,MEMBLOCK_ALLOC_ANYWHERE为~(phys_addr_t)0即为0xffffffff.内存分配方式为top-down.

内核编译时就初始化了这个变量,那这个memblock又是怎么运作的呢?


##用法

抛开其他的先不谈,如果要使用memblock,最上层函数一共就4个
<ul>
<li>memblock_add(phys_addr_t base, phys_addr_t size):向memory区中添加内存区域.</li>
<li>memblock_remove(phys_addr_t base, phys_addr_t size):向memory区中删除区域.</li>
<li>memblock_free(phys_addr_t base, phys_addr_t size):释放内存.</li>
<li>memblock_alloc(phys_addr_t size, phys_addr_t align):申请内存</li>
</ul>

题外话:笔者大致翻看了一下内核代码,发现很少使用memblock_free(),因为很多地方都是申请了内存做永久使用的。再者,其实在内核中通过memblock_alloc来分配内存其实比较少,一般都是在调用memblock底层的一些函数来简单粗暴的分配的.博文下面会详细讲述.

##内核生命周期中的memblock

如果从整个linux生命周期来讲,涉及到各种初始化等,这里来详细分析,因为还没有分析完内核,所以这里是分析到哪里就记录到哪里了.


###初始化

在内核初始化初期,物理内存会通过Int 0x15来被探测和整理,存放到e820中.而初始化就发生在这个以后.

	file:arch/x86/kernel/setup.c 
	1099     memblock_set_current_limit(ISA_END_ADDRESS); 
	1100     memblock_x86_fill(); 

虽然讲是初始化,其实由于一些原因,这个之前早就有一些内存通过memblock_reserve申明为保留内存了.比如为了建立内核页表需要扩展__brk,而扩展后的brk就立即被声明为已分配.而其实并不是正真通过memblock分配的,当然这些都是题外话.

	file:arch/x86/kernel/setup.c 
	1088     early_alloc_pgt_buf();
	1089               
	1090     /*        
	1091      * Need to conclude brk, before memblock_x86_fill()
	1092      *  it could use memblock_find_in_range, could overlap with
	1093      *  brk area.
	1094      */       
	1095     reserve_brk();

可以看到,setup_arch()函数通过memblock_x86_fill(),依据e820中的信息来初始化memblock.

	file:arch/x86/kernel/e820.c 
	1071 void __init memblock_x86_fill(void)
	1072 {   
	1073     int i;
	1074     u64 end;
	1075     
	1076     /*
	1077      * EFI may have more than 128 entries
	1078      * We are safe to enable resizing, beause memblock_x86_fill()
	1079      * is rather later for x86
	1080      */
	1081     memblock_allow_resize();
	1082     
	1083     for (i = 0; i < e820.nr_map; i++) {
	1084         struct e820entry *ei = &e820.map[i];
	1085     
	1086         end = ei->addr + ei->size;
	1087         if (end != (resource_size_t)end)
	1088             continue;
	1089     
	1090         if (ei->type != E820_RAM && ei->type != E820_RESERVED_KERN)
	1091             continue;
	1092     
	1093         memblock_add(ei->addr, ei->size);
	1094     }
	1095     
	1096     /* throw away partial pages */
	1097     memblock_trim_memory(PAGE_SIZE);
	1098     
	1099     memblock_dump_all();
	1100 } 

比较简单,通过e820中的信息memblock_add(),将内存添加到memblock中的memory中,当做可分配内存.后两个函数主要是修剪内存使之对齐和输出信息.

然后....就没了,没错,这样就初始化好了,简单又粗暴!


###实现细节

上面已经说过了,可以通过memblock_free()/memblock_alloc()释放和分配内存,如果内核你乖乖用这两个函数来管理,也还简单,可它偏偏不...在分配页表空间的时候它就是会调用memblock底层的函数直接分配,搞得我蛋疼...当然这样是有道理的.单纯的memblock_alloc()会从所有可用的内存空间中分配内存,而在某些场合,内核需要在特定的内存范围内分配内存.好了,话不多说,一起来看下吧.

首先是memblock_add():

	 file:mm/memblock.c
	 583 int __init_memblock memblock_add(phys_addr_t base, phys_addr_t size)
	 584 {
	 585     return memblock_add_range(&memblock.memory, base, size,
	 586                    MAX_NUMNODES, 0);
	 587 }

可以看到是对memblock_add_range的封装,只不过仅添加到memory区域,那同理,memblock_remove():

	 file:mm/memblock.c
	 680 int __init_memblock memblock_remove(phys_addr_t base, phys_addr_t size) 
	 681 {
	 682     return memblock_remove_range(&memblock.memory, base, size);
	 683 }

再看下memblock_free():

	 file:mm/memblock.c
	 686 int __init_memblock memblock_free(phys_addr_t base, phys_addr_t size)
	 687 {  
	 688     memblock_dbg("   memblock_free: [%#016llx-%#016llx] %pF\n",
	 689              (unsigned long long)base,
	 690              (unsigned long long)base + size - 1,
	 691              (void *)_RET_IP_);
	 692    
	 693     kmemleak_free_part(__va(base), size);
	 694     return memblock_remove_range(&memblock.reserved, base, size);
	 695 }  

可以看到从memory中添加和删除可用内存以及释放内存其实都比较简单.而分配就不一样了....

	file:mm/memblock.c
	1094 phys_addr_t __init memblock_alloc(phys_addr_t size, phys_addr_t align)
	1095 {
	1096     return memblock_alloc_base(size, align, MEMBLOCK_ALLOC_ACCESSIBLE);
	1097 }

	1081 phys_addr_t __init memblock_alloc_base(phys_addr_t size, phys_addr_t align, phys_addr_t max_addr)
	1082 {
	1083     phys_addr_t alloc;
	1084
	1085     alloc = __memblock_alloc_base(size, align, max_addr);
	1086
	1087     if (alloc == 0)
	1088         panic("ERROR: Failed to allocate 0x%llx bytes below 0x%llx.\n",
	1089               (unsigned long long) size, (unsigned long long) max_addr);
	1090
	1091     return alloc;
	1092 } 

	1076 phys_addr_t __init __memblock_alloc_base(phys_addr_t size, phys_addr_t align, phys_addr_t max_addr) 
	1077 {
	1078     return memblock_alloc_base_nid(size, align, max_addr, NUMA_NO_NODE);
	1079 }

	1064 static phys_addr_t __init memblock_alloc_base_nid(phys_addr_t size,
	1065                     phys_addr_t align, phys_addr_t max_addr,
	1066                     int nid)
	1067 {
	1068     return memblock_alloc_range_nid(size, align, 0, max_addr, nid);
	1069 } 

	1037 static phys_addr_t __init memblock_alloc_range_nid(phys_addr_t size,
	1038                     phys_addr_t align, phys_addr_t start,
	1039                     phys_addr_t end, int nid)
	1040 {
	1041     phys_addr_t found;
	1042
	1043     if (!align)
	1044         align = SMP_CACHE_BYTES;
	1045
	1046     found = memblock_find_in_range_node(size, align, start, end, nid);
	1047     if (found && !memblock_reserve(found, size)) {
	1048         /* 
	1049          * The min_count is set to 0 so that memblock allocations are
	1050          * never reported as leaks.
	1051          */
	1052         kmemleak_alloc(__va(found), size, 0, 0);
	1053         return found;
	1054     }
	1055     return 0;
	1056 }

有点长,不要意思,可是代码也不是我写的.可以做个小结了,memblock_alloc(phys_addr_t size, phys_addr_t align)其实就是在当前NODE在内存范围0-MEMBLOCK_ALLOC_ACCESSIBLE(其实是current_limit)中分配一个大小为size的内存区域.

那么问题就来了,memblock_alloc()很粗暴的从能用的内存里分配,而刚刚就提到,有些情况下需要从特定的内存范围内分配内存.解决方法类似memblock_alloc_range_nid中使用的,其实就是memblock_find_in_range_node指定内存区域和大小查找内存区域,memblock_reserve后将其标为已经分配....

看下实现吧:

	 file:mm/memblock.c
	 191 phys_addr_t __init_memblock memblock_find_in_range_node(phys_addr_t size,
	 192                     phys_addr_t align, phys_addr_t start,
	 193                     phys_addr_t end, int nid)
	 194 {  
	 195     phys_addr_t kernel_end, ret;
	 196    
	 197     /* pump up @end */
	 198     if (end == MEMBLOCK_ALLOC_ACCESSIBLE)
	 199         end = memblock.current_limit;
	 200    
	 201     /* avoid allocating the first page */
	 202     start = max_t(phys_addr_t, start, PAGE_SIZE);
	 203     end = max(start, end);
	 204     kernel_end = __pa_symbol(_end);
	 205    
	 206     /*
	 207      * try bottom-up allocation only when bottom-up mode
	 208      * is set and @end is above the kernel image.
	 209      */
	 210     if (memblock_bottom_up() && end > kernel_end) {
	 211         phys_addr_t bottom_up_start;
	 212    
	 213         /* make sure we will allocate above the kernel */
	 214         bottom_up_start = max(start, kernel_end);
	 215    
	 216         /* ok, try bottom-up allocation first */
	 217         ret = __memblock_find_range_bottom_up(bottom_up_start, end,
	 218                               size, align, nid);
	 219         if (ret)
	 220             return ret;
	 221    
	 222         /*
	 223          * we always limit bottom-up allocation above the kernel,
	 224          * but top-down allocation doesn't have the limit, so
	 225          * retrying top-down allocation may succeed when bottom-up
	 226          * allocation failed.
	 227          *
	 228          * bottom-up allocation is expected to be fail very rarely,
	 229          * so we use WARN_ONCE() here to see the stack trace if
	 230          * fail happens.
	 231          */
	 232         WARN_ONCE(1, "memblock: bottom-up allocation failed, "
	 233                  "memory hotunplug may be affected\n");
	 234     }

如果从memblock_alloc()过来,end就是MEMBLOCK_ALLOC_ACCESSIBLE,这个时候会设置为current_limit.如果不通过memblock_alloc分配,内存范围就是指定的范围.紧接着对start做调整，为的是避免申请到第一个页面。memblock_bottom_up()返回的是memblock.bottom_up，前面初始化的时候也知道这个值是false（在numa初始化时会设置为true），所以初始化前期应该调用的是__memblock_find_range_top_down()去查找内存:

 	 file:mm/memblock.c
	 148 static phys_addr_t __init_memblock
	 149 __memblock_find_range_top_down(phys_addr_t start, phys_addr_t end,
	 150                    phys_addr_t size, phys_addr_t align, int nid)
	 151 {   
	 152     phys_addr_t this_start, this_end, cand;
	 153     u64 i;
	 154     
	 155     for_each_free_mem_range_reverse(i, nid, &this_start, &this_end, NULL) {
	 156         this_start = clamp(this_start, start, end);
	 157         this_end = clamp(this_end, start, end);
	 158  
	 159         if (this_end < size)
	 160             continue;
	 161  
	 162         cand = round_down(this_end - size, align);
	 163         if (cand >= this_start)
	 164             return cand;
	 165     }
	 166  
	 167     return 0;
	 168 }  

函数通过使用for_each_free_mem_range_reverse宏封装调用__next_free_mem_range_rev()函数，此函数逐一将memblock.memory里面的内存块信息提取出来与memblock.reserved的各项信息进行检验，确保返回的this_start和this_end不会是分配过的内存块。然后通过clamp取中间值，判断大小是否满足，满足的情况下，将自末端向前（因为这是top-down申请方式）的size大小的空间的起始地址（前提该地址不会超出this_start）返回回去。至此满足要求的内存块算是找到了。

找到内存块后就是标记成已分配:

	 file:mm/memblock.c
	 712 int __init_memblock memblock_reserve(phys_addr_t base, phys_addr_t size) 
	 713 {
	 714     return memblock_reserve_region(base, size, MAX_NUMNODES, 0);
	 715 }

	 697 static int __init_memblock memblock_reserve_region(phys_addr_t base,
	 698                            phys_addr_t size,
	 699                            int nid,
	 700                            unsigned long flags)
	 701 {
	 702     struct memblock_type *_rgn = &memblock.reserved;
	 703
	 704     memblock_dbg("memblock_reserve: [%#016llx-%#016llx] flags %#02lx %pF\n",
	 705              (unsigned long long)base,
	 706              (unsigned long long)base + size - 1,
	 707              flags, (void *)_RET_IP_);
	 708
	 709     return memblock_add_range(_rgn, base, size, nid, flags);
	 710 }

可以看到也是把内存块信息添加到reserved区域中.

memblock_add_region()函数：
<li>如果memblock算法管理内存为空的时候，则将当前空间添加进去</li>
<li>不为空的情况下，则先检查是否存在内存重叠的情况，如果有的话，则剔除重叠部分，然后将其余非重叠的部分添加进去</li>
<li>如果出现region[]数组空间不够的情况，则通过memblock_double_array()添加新的region[]空间</li>
<li>最后通过memblock_merge_regions()把紧挨着的内存合并了。</li>

memblock_remove_range实现差不多相反的功能.


##总结

memblock内存管理是将所有的物理内存放到memblock.memory中作为可用内存来管理,分配过的内存只加入到memblock.reserved中,并不从memory中移出.同理释放内存也会加入到memory中.也就是说,memory在fill过后基本就是不动的了.申请和分配内存仅仅修改reserved就达到目的.在初始化阶段没有那么多复杂的内存操作场景，甚至很多地方都是申请了内存做永久使用的,所以这样的内存管理方式已经足够凑合着用了...毕竟内核也不指望用它一辈子.
