---
layout: post
title: Linux内核初始化阶段内存管理的几种阶段
description: 讲述从内核加载到建立完整的内存管理所经历的几种阶段
category: manual
---

本文旨在讲述从引导到完全建立内存管理体系过程中,内核对内存管理所经历的几种状态.

##一些讲在前面的话

在很久很久以前,linux内核还是支持直接从磁盘直接启动,也就是内核镜像自带了一个可以引导的MBR,按照套路计算机上电以后BIOS会将MBR加载到0000:7c00处执行.后来时过境迁,linux内核必须通过grub这些东西来引导了.本文的故事就从grub引导开始....

现在计算机上电后BIOS会将grub载入到0000:7c00处执行.grub会根据配置从磁盘将内核载入到内存...过程这里就忽略了,载入完成后内存是这么个分布情况:

		~			~
		|protected-mode kernel	|
		#上面就是内核保护模式的代码,即vmlinux.bin(包括解压代码(compressed下代码)和压缩后的vmlinux.bin)
		#这部分是grub暂时开保护模式后将代码放入的,实模式下不能访问到100000以上内存
	100000	+-----------------------+	
		|I/O memory hole	|
	0A0000	+-----------------------+	
		|Reserved for BIOS	|
		~			~	
		|Command Line		|
	x+10000	+-----------------------+	
		|	stack/heap	|	for used by the kernel real-mode
	x+08000	+-----------------------+	
		|kernel setup		|	//曾经这两部分是分开的源文件,后来第一扇区
		|kernel boot sector	|	//被遗弃后,所有代码并入header.s中(real-mode)
		#上面就是bzimage中的setup.bin
	x	+-----------------------+
		|boot loader		|	<-boot sector entry point 0000:7c00
	001000	+-----------------------+	
		|Reserved for MBR/BIOS	|
	000800	+-----------------------+
		|Typically used by MBR	|
	000600	+-----------------------+	
		|  Bios use only	|
	000000	+-----------------------+
