---
layout: post
title: 扩展Ubuntu Swap分区
description: Android编译需要一万G的内存,简直大
category: project
---

##引子
你可能花了几天下完了android5.0.2源代码,make -j65535!编译到最后链接共享库,既然需要8GB左右的内存,实验室渣渣电脑果断受不了,加上当初你自负的认为4g内存开发已经妥妥的够用,所以没有分配swap分区....等吧,CPU烧起来,硬盘要爆炸,重启吧!现在后悔药来了...两种药方,任君挑选.

##药方1:T_T我不会重新分区啊!
首先停止所有swap分区

	sudo swapoff -a

简单粗暴,不需要重新分区.到某个目录

	sudo dd if=/dev/zero of=swap.img bs=1024 count=400000 (//4GB)
	sudo mkswap swap.img    //格式化
	sudo swapon swap.img    //激活!妥妥的啦

永久添加,在/etc/fstab中添加

	/swap.img路径 swap swap defaults 0 0

##药方2:^_^高级玩家	
首先停止所有swap分区

	sudo swapoff -a

用fdisk命令加swap分区的盘符，（fdisk /dev/sda7）剔除swap分区，干掉分区，然后n添加分区（添加时硬盘必须要有可用空间，然后再用t将新添的分区id改为82（linux swap类型），w更新分区表（w之前的操作是无效的）。(数据诚宝贵,操作需谨慎!)

	sudo mkswap /dev/sda2    //格式化sda2分区为swap分区
	sudo swapon /dev/sda2    //激活!妥妥的啦

永久添加,在/etc/fstab中添加

	/dev/sda2 swap swap defaults 0 0

##吐槽
GFW墙的一手好网.清华tuna镜像棒棒的.清华是个好学校.android编译简直要屌丝的命!!(END)

