---
layout: post
title: Linux设备驱动模型及其他(1)
description: Linux设备驱动模型及其他(1)
category: manual
---

##引子

一开始考虑这样一段代码

	inf = open("/floppy/TEST", O_RDONLY, 0);
	outf = open("/tmp/test", O_WRONLY | O_CREAT | O_TRUNC, 0600);
	do {
		len = read(inf, buf, 4096);
		write(outf, buf, len);
	} while (len);
	close(outf);
	close(inf);

代码很简单,就是将TEST拷贝为test.众所周知,无论open还是write都最终会陷入内核,由内核完成正真的打开和读写,问题是到底是怎么打开怎么读写...可能对Linux内核稍微有些了解的朋友还知道VFS...但这还远远不够.


##总线

我们使用的计算机有很多硬件设备,包括CPU,硬盘,显卡,鼠标,键盘...而CPU是他们的老大,一个硬件设备如果要正常工作,那么它就得和CPU搞好关系,而它们之间的纽带就是总线.所有的外设都并不是直接连接到CPU的,而是需要总线把它们连接起来.总线来来负责设备和CPU之间以及各个设备之间的通讯.一下是一些总线类型:
<ul>
<li>PCI</li>
<li>ISA</li>
<li>SBus</li>
<li>IEEE1394</li>
<li>USB</li>
<li>SCSI</li>
<li>串口并口</li>
</ul>
而目前无论哪种处理器体系结构,系统都不会只有一种总线,通常都是一些总线的组合.比如常见的x86系统结构一般是:CPU通过FSB(前端总线)连接到内存,通过PCI桥连接到PCI总线.而PCI总线有可以通过PCI-PCI桥,或者PCI-ISA桥连接到ISA总线,结构如图:
![bus1](/images/bus.gif) ![bus2](/images/bus2.jpg)
