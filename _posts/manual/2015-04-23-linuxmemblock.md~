---
layout: post
title: Linux内核初期内存管理---memblock
description: 张正宇说,博客要短些.....
category: manual
---

##一些说明
以前只知道内核有个比较早的内存管理机制叫做bootmem的.后来的版本(笔者分析的是3.19)好像把bootmem弃用了,取而代之的是__alloc_memory_core_early(),这个以后的博客会有分析.这两天分析linux各个阶段的内存管理,对这个memblock重视了起来....

我查了一些资料,对memblock的作用好像讲述的都不太一样.有说是bootmem的替代品;有说是在bootmem初始化前用于内存管理的;还有说是Kernel对于物理内存使用情况的记录.一开始也被弄糊涂了,还是自己来看下比较靠谱.


##初始化


首先来看下memblock结构的定义,文件是include/linux/memblock.h:

	 26 struct memblock_region {
	 27     phys_addr_t base;
	 28     phys_addr_t size;
	 29     unsigned long flags;
	 30 #ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
	 31     int nid;
	 32 #endif        
	 33 }; 
	 34    
	 35 struct memblock_type {
	 36     unsigned long cnt;  /* number of regions */
	 37     unsigned long max;  /* size of the allocated array */
	 38     phys_addr_t total_size; /* size of all regions */
	 39     struct memblock_region *regions;
	 40 }; 
	 41    
	 42 struct memblock {
	 43     bool bottom_up;  /* is bottom up direction? */
	 44     phys_addr_t current_limit;
	 45     struct memblock_type memory;
	 46     struct memblock_type reserved;
	 47 #ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	 48     struct memblock_type physmem;
	 49 #endif
	 50 }; 


接着是这个全局变量的初始化,文件是mm/memblock.c:

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


其中memblock.memory.regions是可用内存的集合(INIT_MEMBLOCK_REGIONS=128),memblock.memory记录了最大的区域个数和已经初始化过的内存区域个数.而memory.reserved.regions是需要保留的memory region以阻止分配的集合(INIT_MEMBLOCK_REGIONS=128)。

内核编译时就初始化了这个变量,那这个memblock又是怎么运作的呢?


##原理和用法

几个管理memblock的函数如下:
<ul>
<li>int memblock_add(phys_addr_t base, phys_addr_t size);向memory区域中添加给定的内存区.新加入的region需要经过检查，如果与原先的region有重叠，则需要合并在原先的memory region中，否则的话就新建一个memory region.</li>
<li>int memblock_remove(phys_addr_t base, phys_addr_t size);从memory中移除指定物理地址所指定的memory region.如果所指定的区域是存在区域的一部分，则涉及到调整region大小，或者将一个region拆分成为两个region.</li>
<li>int memblock_free(phys_addr_t base, phys_addr_t size);从reserved中移除指定物理地址所指定的memory region.如果所指定的区域是存在区域的一部分，则涉及到调整region大小，或者将一个region拆分成为两个region.</li>
<li>int memblock_reserve(phys_addr_t base, phys_addr_t size);向reserved区域中添加给定的内存区.新加入的region需要经过检查，如果与原先的region有重叠，则需要合并在原先的region中，否则的话就新建一个region.</li>
<li>phys_addr_t memblock_find_in_range_node(phys_addr_t size, phys_addr_t align, phys_addr_t start, phys_addr_t end,int nid);根据给定的范围从node中找到一块内存区域在memory中但是不在reserved中.</li>
<li>phys_addr_t memblock_find_in_range()实际调用上者,不过是从本节点找而已.</li>
</ul>

如果要分配内存,就简单了
<ul>
<li>1.memblock_find_in_range(),找到空闲内存区域.</li>
<li>2.memblock_reserve()直接将找到的内存加入到reserved区域就分配成功了.</li>
</ul>


题外话:笔者大致翻看了一下内核代码,发现很少使用memblock_free().

##内核生命周期中的memblock

如果从整个linux生命周期来讲,涉及到各种初始化等,这里来详细分析,因为还没有分析完内核,所以这里是分析到哪里就记录到哪里了.

