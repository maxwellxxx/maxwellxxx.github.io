---
layout: post
title: Linux/Unix系统编程
description: 计划写满500个系统调用和标准库函数的使用方法和注意事项……看能不能坚持吧，笔记这回事，还是习惯纸笔……
category: manual
---

开始各种填坑，不过决定以后要为每本书都开个坑，以前的书也要开个小坑………总之这个博客就是个大月亮！！！
![moon](/images/manual/moon.jpg)

来吧……

由于这本是手册……所以只是讲关于各个系统调用和一些标准库函数的使用方法和不为人知的注意事项。涉及一些分析总结的话，恩……又来一个坑。
##文件I/O:通用I/O模型

###打开一个文件open()(系统调用)

	#include <sys/stat.h>
	#include <fcntl.h>
	int open(const char *pathname,int flags,.../*mode_t mode*/)
		return fd-SUC，-1-err set errno

要打开的文件有参数pathname来标识。如果pathname是一符号链接，会对其进行解引用。
flags位为掩码，用于指定文件的访问模式：

	O_RDONLY   只读 --、
	O_WRONLY   只写 --|不能同时使用
	O_RDWR     读写 --/

	O_CLOEXEC  设置close-on-exec
	O_CREAT    不存在则创建
	O_DIRECT   无缓冲I/O
	O_DIRECTORY如果pathname不是目录,则失败
	O_EXCL     结合O_CREAT使用,专门用于创建文件
	O_LARGEFILE在32位系统中打开大文件
	O_NOATIME  调用read,不修改文件访问时间
	O_NOCTTY   不然pathname成为控制终端
	O_NOFOLLOW 对符号链接不做解引
	O_TRUNC    截断已有文件,使其长度为0

	0_APPEND   总在文件尾部追加数据
	O_ASYNC    当I/O操作可行时,产生signal通知进程
	O_DSYNC    同步I/O数据完整性
	O_NONBLOCK 以非阻塞方式打开
	O_SYNC     以同步方式写入
<ul>	
	<li>前三个为访问模式，不能同时使用。调用fcntl()的F_GETFL能够查看当前访问模式。</li>
	<li>第二部分为文件创建标志,不能检索不能修改.</li>
	<li>第三部分状态标志位,可用fcntl()修改和检索</li>
</ul>
其中O_CLOEXEC是为了防止多线程程序执行fcntl()时引发竞争场景,可能会使文件描述符泄露给不安全的程序.而O_CREATE和O_EXEL同时使用可以确保文件肯定是由当前进程创建的.

mode参数如果没有指定O_CREATE,可以省略.
mode_t
稍后补

出错处理
稍后补
###创建件一个文件create()(系统调用)

	#include <fcntl.h>
	int creat(const char *pathname,mode_t mode);
	return fd-SUC,-1-ERR

因为早期的UNIX中open中只有两个参数,无法创建文件,这个则用了专门创建文件.

###读取文件内容read()(系统调用)

	#include<unistd.h>
	ssize_t read(int fd,void *buffer,size_t count);
	return number of bytes read,0-EOF,-1-ERR

可能返回值小于count,可能是当前读取的位置靠近文件尾部.或者读取管道,FIFO,socket或者终端是,读到\n时read也会结束.

###数据写入文件write()(系统调用)

	#include<unistd.h>
	ssize_t write(int fd,void *buffer,size_t count);
	return number of written,-1-err

返回值可能小于count,成为"部分写",造成的原因是磁盘已满,或者进程资源对文件大小的限制.

###关闭文件close()(系统调用)

	#include<unistd.h>
	int close(int fd);
	return 0-SUC,-1-ERR

有必要检查错误,防止关闭失败,耗尽文件描述符资源.

###改变文件偏移量了lseek()(系统调用)

	#include<unistd.h>
	off_t lseek(int fd,off_t offset,int whence);
	return new offset-SUC,-1-ERR

参数offset指定了以字节为单位的数值(off_t为有符号数).

whence参数指定由哪个基点来解释offset

<ul>
	<li>SEEK_SET  将基点设置为文件起点.offset必须为正</li>
	<li>SEED_CUR  相对于当前的偏移,将偏移量调整offset个字节.offset可为负数</li>
	<li>SEEK_END  将文件偏移设置为起始于文件尾部后的offset个字节,此时,offset参数要从文件最后一个字节的下一个字节算起!offset可为负数</li>
</ul>

可以这样获取当前offset 

	curr=lseek(fd,0,SEEK_SET);

另外此函数不能用于管道,FIFO,socket或者终端!!!!!!


###通用模型以外的I/O操作ioctl()(系统调用)

	#include<sys/ioctl.h>
	int ioctl(int fd,int request,.../*argp*/);
	return depends on request-SUC,-1-ERR






##深入文件I/O

##文件控制操作fcntl()(系统调用)

	#include<fcntl.h>
	int fcntl(int fd,int cmd,...);
	return depends on request-SUC,-1-ERR

cmd参数支持的操作很多,这里逐步加吧,后三个省略号根据cmd来看,可以有可以没有.

	cmd:
	F_GETFL   获取访问访问模式和状态标志
	F_SETFL   修改文件某些状态标志,允许修改的有O_APPEND,O_NONBLOCK,O_NOATIME,O_ASYNC,O_DIRECT.修改其他会被系统忽略
	
F_GETFL返回的flags如果用于判断访问模式会有些复杂,原因是那三个常量不与状态标志位中的单个比特位对应,要用掩码O_ACCMODE与返回的flags相与.示例:

	int accessMode=flags&O_ACCMODE
	if(accessMode==O_WRONLY||accessMode==O_RDWR)
		printf("writable!");

使用fcntl()修改文件状态标志,尤其适合如下场景
<ul>
	<li>文件不是由调用程序打开的,不能通过open来控制文件状态标志.比如文件是3个标准输入输出描述符中的一个</li>
	<li>文件是通过open以外的系统调用打开的,比如pipe(),socket()</li>
</ul>
在修改时,应该先获取当前flags然后修改对应的比特位,然后在set.

###复制文件描述符(系统调用)

	#include <unistd.h>
	int dup(int oldfd);
	retrun new fd -SUC,-1-ERR

该调用复制一个已经打开的fd,返回一个新的fd,但是他们两个指向同一个文件句柄.

	#include<unistd.h>
	int dup2(int oldfd,int newfd);
	

