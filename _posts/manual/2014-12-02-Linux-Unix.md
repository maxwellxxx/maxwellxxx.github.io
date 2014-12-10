---
layout: post
title: Linux/Unix系统编程
description: 计划写满500个系统调用和标准库函数的使用方法和注意事项……看能不能坚持吧，笔记这回事，还是习惯纸笔……
category: manual
---
更新于：2014-12-10
开始各种填坑，不过决定以后要为每本书都开个坑，以前的书也要开个小坑………总之这个博客就是个大月亮！！！
![moon](/images/manual/moon.jpg)
不过这篇真的是个正经的存在，我会乱讲？？！！

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

###文件控制操作fcntl()(系统调用)

	#include<fcntl.h>
	int fcntl(int fd,int cmd,...);
	return depends on request-SUC,-1-ERR

cmd参数支持的操作很多,这里逐步加吧,后三个省略号根据cmd来看,可以有可以没有.

	cmd:
	F_GETFL   获取访问访问模式和状态标志
	F_SETFL   修改文件某些状态标志,允许修改的有O_APPEND,O_NONBLOCK,O_NOATIME,O_ASYNC,O_DIRECT.修改其他会被系统忽略
	F_DUPFD   复制文件操作符。
	
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
	retrun new fd -SUC,-1-ERR
	
这个函数基本同上个函数，但是会返回个一由newfd指定的文件描述符编号，如果指定的描述符已经打开，那此函数会先隐式的将它关闭……并且忽略关闭时的错误。更好的编码方式是在调用此函数前先显示调用close将newfd关闭。
	
	//另外一种灵活的方法复制文件描述符
	newfd=fcntl(oldfd,F_DUPFD,startfd);
	/*为oldfd创建副本，并且分配大于等于startfd且未使用的最小newfd。
	*/

复制的newfd与oldfd享有同一个打开文件句柄，文件偏移量，状态标志。然而newfd拥有自己一套文件描述符标志，且其close-on-exec处于关闭。

关于文件描述符号的分配等问题，将另外讨论。

	#include <unistd.h>
	int dup3(int oldfd,int newfd,int flags);
	retrun new fd -SUC,-1-ERR

dup3完成的工作与dup2相同，但是可以指定flags，但是只支持O_CLOEXEC。这将促使内核为新的文件描述符设置close-on-exec位。

###在文件特定偏移量处I/O：pread()和pwrite()(系统调用)

	#include <unistd.h>
	ssize_t pread(int fd,void *buf,size_t count,off_t offset);
		return number of bytes read,0-EOF,-1-ERR
	ssize_t pwrite(int fd,const void *buf,size_t count,off_t offset);
		return number of bytes written,-1-ERR

我们知道进程下所有线程共用一个文件描述符表，这两个函数可以在多线程对同一个文件I/O时避免受其他线程修改offset影响。
<ul>
	<li>这两个函数会在指定的offset偏移处进行操作，而且不改变当前偏移量。</li>
</ul>

###分散输入和集中输出readv()&writev()(系统调用)

	#include <sys/uio.h>
	ssize_t readv(int fd,const strcut iovec *iov,int iovcnt);
		return number of bytes read,0-EOF,-1-ERR
	ssize_t writev(int fd,const strcut iovec *iov,int iovcnt);
		return number of bytes written,-1-ERR

这两个系统调用用于一次传输多个缓冲区的数据。每组iov定义了一组用于传输的数据。iovcnt则指定了iov的个数。

	struct iovec
	{
		void *iov_base;		//start addr of buffer
		size_t iov_len;		//number of bytes to transfer to/from buffer
	}

readv()实现了从fd读取一片连续的数据，然后将其分散放置在iov指定区域里。这一动作从iov[0]开始一次填满每个缓冲。writev()则实现相反的动作
<ul>
	<li>这两个函数是atom的，即所与操作一个次性完成。即不受其他线程改变偏移量影响。</li>
	<li>这两个函数会改变文件偏移量。</li>
</ul>

###截断文件truncate()&ftruncate()(系统调用)

	#include <unistd.h>
	int truncate(const char *pathname,off_t length);
	int ftruncae(int fd,off_t length);

若文件当前长度大于参数lengtg，调用丢弃超出部分，如果小于length则在文件末添加一些列空字节或形成文件空洞。

###创建临时文件mkstemp()(库函数)

	#include <stdlib.h>
	int mkstemp(char *template)
		return fd-SUC,-1-ERR

参数为路径名形式，但是最后六个字符一定要是XXXXXX，因为这六个字符会被替换成。
<ul>
	<li>因为会对传入的参数进行修改，所以必须为字符数组，而非字符串常量。</li>
	<li>文件拥有者对建立的文件有读写权限（使用了O_EXCL），其他用户没有权限。</li>
	<li>在调用成功后，应该立即使用unlink(template)，以便在关闭后自动删除临时文件。</li>
</ul>

	#include<stdio.h>
	FILE *tmpfile(void);
		return file pointer-SUC,-1-ERR

创建一个名称唯一的临时文件，并使用O_EXCL，防止冲突。返回文件流供stdio库函数使用。文件流关闭后将自动删除文件。即隐式调用unlink()。





##进程

###获取(父)进程号getpid()&getppid()(系统调用)

	#include <unistd.h>
	pid_t getpid(void);
	pid_t getppid(void);
		always SUC

###访问和修改环境变量(库函数)

	#include <stdlib.h>
	char *getenv(const char *name);
		return pointer to value,NULL-no such variable
从环境变量里检索单个之，传入环境变量名，返回相应的字符串指针。（返回name=value 的value部分。）
<ul>
	<li>也可以在程序中使用全局变量的形式访问 extern char **environ</li>
	<li>环境变量的修改会被以后创建的子进程可见，即环境变量也是一种进程乃至程序通讯方法。</li>
</ul>

	#include <stdlib.h>
	int putenv(char *string);
		return 0-SUC,nonzero-ERR

string指向name=value字符串，由于只是将environ中一个元素指向string，所以string不应该是在栈中分配的字符串，而且以后修改string也将影响环境变量。

	#iclude <stdlib.h>
	int setenv(const char *name,const char *value,int overwrite);
		return 0-SUC,-1-ERR

函数为形如name=value的字符串分配一块内存缓冲，并将name和value所指的字符串复制到缓冲。如果name已经存在，且overwrite为0則不改变，如果为1则改变。
<ul>
	<li>name中不应该有‘=’号。</li>
</ul>

	#include <stdlib.h>
	int unsetenv(const char *name);
		return 0-SUC,-1-ERR

移除name标识的环境变量。

当为了以安全方式执行set-user-ID程序时，可能需要清空环境变量以重建。这是可以使用将全局变量environ设为NULL。

	extern char **environ;
	environ=NULL:

也可以使用以下函数
	
	#define _BSD_SOURCE
	#include <stdlib.h>
	int clearenv(void);
		return 0-SUC,-1-ERR

<ul>
	<li>使用setenv()和clearenv()可能导致内存泄露，当使用setenv()时会分配内存，clearenv()时则不知道已经分配内存，所以不会释放，从而导致内存泄露。所以一般都在程序开始执行clearenv()，用于移除从父进程继承的环境变量。</li>
</ul>


###执行非局部跳转命令setjmp()&longjmp()(库函数)

<ul>
	<li>“非局部”是指跳转的目标为当前执行函数之外的某个位置。</li>
</ul>
C语言中的goto语句存在一个限制，即不能从当前的函数跳转到另一个函数。但是考虑到在错误处理时可能出现这样的场景：
<ul>
	<li>在一个深层嵌套的调用的函数中发生了错误，需要放弃当前任务，从多层函数调用中返回，并在更层级的函数中继续执行。甚至是在main()函数中。</li>
</ul>

那么可以使用如下的函数：

	#include <setjmp.h>
	int setjmp(jmp_buf env);
		return 0-initial call,nonzero-return via longjmp()
	void longjmp(jmp_buf env,int val);

setjmp()的调用为后续的longjmp()执行的跳转确立了跳转目标。该目标为程序发起setjmp()调用的位置。通过查看setjmp()的返回值，可以知道是第一次调用setjmp()返回，还是后续调用了longjmp()的伪返回。初始返回值为0,后续“伪”返回值为longjmp中val参数指定的任意值。根据返回值的不同，可以区分出跳转到同一目标的不同起跳位置。

<ul>
	<li>longjmp的val参数如果指定为0,实际执行时会替换成1。</li>
</ul>

而参数env为成功跳转的粘合剂，setjmp()函数把当前进程环境的各种信息保存到env中，调用longjmp时必须制定相同的env，一次来执行“伪”返回。由于两次调用在不同的函数（不然可以使用goto），所以env最好为全局变量。
<ul>
	<li>env除了保存了进程信息外，还保存了ip，和esp寄存器。其实是通过这两个值来实现返回的。</li>
</ul>

不要滥用！！！！！！！
还有要注意编译器优化的问题带啦的代码重组。





##内存分配

###分配堆上内存

所谓的堆，其实就是一段长度可变的连续虚拟内存。通常始于进程未初始化数据段末尾。随着内存的分配和释放增减，通常将堆的内存边界成为“program break”。

###调整program break brk()&sbrk()(系统调用)

	#include <unistd.h>
	int brk(void *end_data_segment);
		return 0-SUC,-1-ERR
	void *sbrk(intptr_t increment);
		return previous pb-SUC,-1-ERR

brk()会将pk设为end_data_segment指定位置，由于虚拟内存按页分配，所以会四舍五人到下一个内存边界。

sbrk()会将pk在原有的基础上增加increment。sbrk()会返回前一个pk地址，那么如果pk增加，返回的是增加的内存的起始地址。

	sbrk(0)//返回当前的pk，而不做改变。

###在堆上分配内存malloc()&free()(库函数)

	#include <stdlib.h>
	void *malloc(size_t size);
		return pointer to allocated memory -SUC,NULL-ERR
	void free(void *ptr);

###其他堆内存分配方法(库函数)

	#include <stdlib.h>
	void *calloc(size_t numitems,size_t size);
		return pointer to allocated memory -SUC,NULL-ERR

用与给一组相同的对象分配内存，numitems为对象数量，size为每个对象的大小。会将分配的内存初始化为0.

	#include <stdlib.h>
	void *realloc(void *ptr,size_t size);
		return pointer to allocated memory -SUC,NULL-ERR

用于调整（通常是增加）一块内存的大小，此内存必须要是由malloc分配。返回调整后的内存地址，与之前的相比，可能位置不同。如果返回NULL则不改动ptr指向的内存。
<ul>
	<li>如果原内存块后空间不足，则分配新内存，并将数据复制到新内存中。</li>
	<li>如果原内存块后能够分配内存，则直接扩展。</li>
</ul>

####分配对齐的内存memalign()&posix_memalign()
函数分配内存的起始位置要与2的整数u次幂边界对齐。

	#include <malloc.h>
	void *memalign(size_t boundary,size_t size);
		return pointer to allocated memory -SUC,NULL-ERR

分配size个字节的内存，起始地址是参数boudary的整数倍，而boudary必须是2的整数次幂。

	#include <stdlib.h>
	int posix_memalign(void **memptr,size_t alignment,size_t size);
		return 0-SUC,a positive error number-ERR

已经分配的内存地址有memptr返回。

###在堆栈上分配内存

	#include <alloca.h>
	void *alloca(size_t size); 
		return pointer to allocated block of memory

size指定要在栈上分配的内存大小,通过它分配的内存。
<ul> 
	<li>不能用free来释放，因为跳出栈帧时会自动释放。</li>
	<li>不能在函数的参数列表中使:func(x,alloca(size)</li>
	<li>在信号处理程序中使用longjmp，用alloca更有优势，因为用malloc分配的内存不能有效free,而alloca通过解栈就释放了。</li>
</ul>





##用户和组

###获取用户和组的信息

####从密码文件/etc/passpw获取记录getpwnam()&getpwuid()(库函数)

	#include <pwd.h>
	struct passwd *getpwnam(const char *name);
	struct passwd *getpwuid(uid_t uid);
		return pointer -SUC,NULL-ERR

为name提供一个登录名，或者为uid提供一个用户ID。返回一个数据结构：

	struct passwd
	{
		char *pw_name;	//Login name
		char *pw_passwd	//Encrypted password
		uid_t pw_uid;	//User ID
		gid_t pw_gid;	//Group ID
		char *pw_gecos;	//comment (user information)
		char *pw_dir;	//initial working directory(home)
		char *pw_shell;	//Login shell
	}

当没有开启shadow密码的情况下，pw_passwd才会包含有效信息。
<ul>
	<li>注意，这里返回的指针都指向一个静态分配的结构，所以任何一次调用都会改写该数据结构！！！！</li>
	<li>所以这两个函数都是不可以重入的</li>
</ul>

####从组文件/etc/group获取记录getgrnam()&getgrgid()(库函数)

	#include <grp.h>
	struct group *getgrnam(const char *name);
	struct group *getgrgid(gid_t gid);
		return pointer -SUC,NULL-ERR

两者通过组名和组ID来返回一个数据结构指针：
	
	struct group
	{
		char *gr_name;
		char *gr_passwd;
		gid_t gr_gid;
		char **gr_mem;		//menbers in this group listed in /etc/group
	}

<ul>
	<li>两者返回的指针同样指向一个静态分配的结构，任何一次调用都会改写数据。</li>
	<li>两者不可重入</li>
</ul>

####扫描密码文件和组文件中的所有记录(库函数)

这里是一组函数配合使用

	#include <pwd.h>
	struct passwd *getpwent(void);
		return pointer-SUC,NULL or end of stream-ERR
	void setpwent(void);
	void endpwent(void);

getpwent()会从密码文件中逐条返回记录，如果出错或到末尾，返回NULL。一经调用就自动打开密码文件，处理结束后需要调用endpwent()关闭。在处理文件中途，可以调用setpwent()返回到文件起始处。

相同的还有从组文件中扫描：

	#include <shadow.h>
	struct group* getgrent(void);
		return pointer-SUC,NULL or end of stream-ERR
	void setgrent(void);
	void endgrent();

同上面一组函数。

<ul>
	<li>注意，这两组返回的指向数据结构也是静态分配的！！！</li>
	<li>所以，每次调用getpwnam(),getpwuid(),getpwent()其中一个，都会改变数据结构!!group系的函数同理!!</li>
</ul>

####从shadow密码文件中获取记录(库函数)

下列函数作用基本同上！从shadow返回个别记录或者扫描返回记录。

	#include <shadow.h>
	struct spwd* getspwnam(const char *name);
		return pointer-SUC,NULL-ERR
	strcut spwd* getspent(void);
		return pointer-SUC,NULL or end of stream-ERR
	void setspent(void);
	void endspent(void);

类似与上面函数，返回结构体也为静态分配。

	strcut spwd
	{
		char *sp_namp;	//login name
		char *sp_pwdp;	//encrypted password
		long sp_lstchg;	//Time of last password change(days since 1970.1.1)
		long sp_min;	
		long sp_max;
		long sp_warn;
		long sp_inact;
		long sp_expire;
		unsigned long sp_falg;
	}
<ul>
	<li>访问shadow必须为root或者shadow组用户，不然会出权限错误。</li>
</ul>
###密码加密和用户认证(系统调用)

由于UNIX系统密码采用单向加密，所以无法通过加密过的密码来还原原始密码，所以认证方式是将待验证密码加密于/etc/shadow中的密码进行匹配。加密方法封装在crypt()。

	#define _XOPEN_SOURCE
	#include <unistd.h>
	char *crypt(const char *key,const char *salt);
		return pointer to encrypted password-SUC,-1-ERR

key为要加密的密码，salt为指向2个字符的字符串，用于搅动DES加密算法（保证安全性）。（S盒变换么……轻喷）。返回加密后的字符串指针。
<ul>
	<li>返回的字符串为静态分配，请注意！</li>
	<li>在返回的字符串中，前两个字符是对salt的拷贝，也就是说，如果要比对shadow中的密码，salt的值就是shadow中密码的前两个字符！</li>
	<li>在linux中如果要使用crypt()需要开启-lcrypt编译选项，以便链接crypt库。</li>
	<li>在完成加密后，应该尽快释放明文密码。</li>
</ul>





##进程凭证

每个进程都有一套数字用于表示用户的ID（UID）和组ID（GID），有时，也将这些ID称之为进程凭证，具体如下：
<ul>
	<li>实际用户ID（real user ID）和实际组ID（real gourp ID）</li>
	<li>有效用户ID（effective user ID）和有效组ID（effective group ID）</li>
	<li>保存的set-user-ID（saved set-user-ID）和保存的set-group-ID（seved set-group-ID）</li>
	<li>文件系统用户ID（file-system User ID）和文件系统组ID（file-system Group ID）（linux 专有）</li>
	<li>辅助组ID</li>
</ul>
在/proc/PID/status文件中会按照实际、有效、保存、文件系统的顺序将这些ID列出来。

这里解释各个的含义：
<ul>
	<li>实际ID:即运行当前程序的用户ID以及用户所在的组ID</li>
	<li>有效ID:即当前运行程序实际拥有哪个用户ID和组ID的权限</li>
	<li>set-user-ID程序会将进程的有效ID置成可执行文件的属主用户ID，set-group-id同理。</li>
	<li>保存的set-user-ID和保存的set-group-id在程序初始化时有有效id复制过来，无论当前程序是否是一个set-user（group）-id程序。</li>
	<li>文件系统ID基本等同于相应的有效ID</li>
	<li>辅助组ID一般都从父进程继承。</li>
</ul>
这里需要注意的是：
<ul>
	<li>在Linux中，set-user-id和set-group-id权限位对shell脚本不适用，理由见详细分析。</li>
</ul>
可用下列命令为可执行程序加上set-user-id和set-group-id权限位：

	$ su
	# ls -l example
	-rwxr-xr-x	1 root 		root
	#chmod u+s example       //加上set-user-id权限位
	# ls -l example
	-rwsr-xr-x	1 root 		root
	#chmod g+x example	 //加上set-group-id权限位
	# ls -l example
	-rwsr-sr-x	1 root 		root
	
其他的另见详细分析。

###获取和修改进程凭证

TIPS：Linux将超级用户划分成各种不同的能力，其中包括：
<ul>
	<li>CAP_SETUID能力允许进程任意修改其用户ID。</li>
	<li>CAP_SETGID能力允许进程任意修改起组ID</li>
</ul>

####获取和修改实际、有效和保存设置的标识（系统调用）

获取实际和有效ID
	
	#include <unistd.h>
	uid_t getuid(void);
		return real user ID of calling process
	gid_t getgid(void);
		return real group ID of calling process
	uid_t geteuid(void);
		return effective user id of calling process
	gid_t getegid(void);
		return effective group id of calling process

修改有效ID
	
	#include <unistd.h>
	int setuid(uid_t uid);
	int setgid(gid_t gid);
		return 0-SUC,-1-ERR

setuid()会根据给出的uid参数值来修改调用进程的有效用户ID，也可能会修改实际用户ID和保存set-user-id。系统调用setgid()实现了相似的功能。

另外两个函数遵守如下规则
<ul>
	<li>规则1：当非特权进程调用时，仅能修改进程的有效ID，而且仅能将有效ID修改成相应的实际用户ID或者保存的set-user-id。那么这就意味着，对于非特权用户只有执行一个set-user-id程序是，函数才有实际的意义。</li>
	<li>规则2：当特权级进程以一个非0的参数调用函数时，其实际、有效、保存ID都会被置为指定的值，而且这一操作是单项的，一旦特权级进程以此方式修改了ID，那么所有的特权全部丢失，且不可逆。如果要避免这些情况，可使用seteuid(),setegid()。</li>
</ul>
对于规则2：如果是调用setgid()不会造成特权丢失，因为特权有userid决定。

对一个特权进程来说，永久放弃特权做法如下：

	if(setuid(getuid())==-1)
		errExit(setuid);

进程能够使用seteuid()来修改其有效用户ID，或者setegid()来修改有效组ID：

	#include <unistd.h>
	int seteuid(uid_t uid);
	int setegid(gid_t gid);
		return 0-SUC,-1-ERR

这两个函数遵循如下规则：
<ul>
	<li>规则1：当非特权进程调用时，仅能将有效ID修改成实际或者保存的ID。</li>
	<li>规则2：当特权级进程调用时，可将有效ID（仅仅修改有效ID）修改为任意值。如果一个特权进程将有效ID修改成非0则会丢失特权，但是可以通过规则1来恢复特权。</li>
</ul>

例，特权进程放弃进程，并恢复特权：

	euid=geteuid();			//保存euid用以恢复
	if(seteuid(getuid())==-1)	//丧失特权
		errExit(seteuid);
	if(seteuid(euid))		//恢复特权
		errExit(seteuid);
		
####修改实际ID和有效ID

	#include <unistd.h>
	int setreuid(uid_t ruid,uid_t euid)
	int setregid(gid_t rgid,gid_t egid)
		return 0-SUC,-1-ERR

两个函数第一个参数指定新的实际ID，第二个参数指定新的有效ID，如果有一个不想修改，置为-1即可。

两个函数也遵守几个规则：
<ul>
	<li>规则1：当非特权进程调用时，仅能将实际Id设为当前实际ID或者有效id，且能将有效ID设为当前实际ID、有效ID或者保存ID。</li>
	<li>规则2：特权级进程能设置实际ID和有效ID为任意值。</li>
</ul>

规则3：不管进程特权与否，只要满足下列条件之一，就将保存set-user-id设置为新的有效用户ID
<ul>
	<li>ruid不为-1（即设置实际ID，即便是设置为当前值）</li>
	<li>对于有效ID所设置的值不同于系统调用之前的实际用户ID</li>
</ul>

####获取实际，有效和保存设置ID（Linux下非标准系统调用）

	#define _GNU_SOURCE
	#include <unistd.h>
	
	int getresuid(uid_t *ruid,uid_t *euid,uid_t *suid);
	int getresgid(gid_t *rgid,gid_t *egid,gid_t *sgid);

####修改实际，有效和保存设置ID（Linux下非标准系统调用）

太懒了，反正各种都系统都鲜有支持，所以这里不罗列。

###获取和修改文件系统ID（系统调用）
太懒了，反正Linux这两个功能已经没有用了，所以忽略！


###总结
这里差张表！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！






##时间

程序一半会关注两种时间：
<ul>
	<li>真实时间：度量这一时间的起点有二：一为某个标准点；二为进程生命周期内的某个固定点（通常为程序启动时）。前者为"日历时间"适用于对数据库记录或者文件打上时间戳的程序，后者为“流逝时间”主要针对于需要周期性操作或者定期从外部输入设备进行度量的程序。</li>
	<li>进程时间：一个进程使用的CPU时间总量，适用于对程序、算法性能的检测和优化。</li>
</ul>

###日历时间

无论地理位置如何，UNIX系统内部对时间的表示方式均以Epoch以来的秒数来度量，即1970年1月1日早上凌晨开始。

系统调用gettimeofday()：

	#include <sys/time.h>
	int gettimeofday(struct timeval *tv,struct timezone *tz);
		return 0-SUC，-1-ERR

将日历时间返回到tv指向的缓冲区中。
	
	struct timeval
	{
		time_t 		tv_sec;
		suseconds_t 	tv_usec;
	}

time_t为有符号整型。参数tz是历史遗留产物，调用时设为NULL即可。

系统调用time():
	
	#include <time.h>
	time_t time(time_t *timep);
		return number of seconds science the Epoch-SUC,-1-ERR

如果timep不为NULL，还会将日历时间返回到timep指向的缓冲中。调用时可以采用简单的调用方式：

	t=time(NULL);

防止timep为无效地址带来的错误。

###时间转换函数

####将time_t转换成可打印的格式

	#include <time.h>
	char *ctime(const time_t *timep);
		return pointer to statically allocated string terminated by newline and \0-SUC,NULL-ERR

将time_t指针作为参数传入，返回一个长达26字节的字符串，含标准时间和日期如下所示：
	
	Wed Jun   8  14:22:34 2011

字符串已经包含换行和终止符。
<ul>
	<li>注意：这里返回的字符串是静态分配的，下一次调用ctime()会将其覆盖。所以函数不能重入。</li>
	<li>调用ctime(),gmtime().localTime(),asctime()中的任意一个函数，都可能会覆盖有其他函数的返回！</li>
</ul>

####time_t和分解时间之间的转换

函数gmtime()和localTime()将一个time_t转换成所谓的分解时间。返回的分解时间放在一个经静态分配的结构里。

	#Include <time.h>
	struct tm *gmtime(const time_t *timep);
	struct tm *localtime(const time_t *timep);
		return pointer to statically allocated struct-SUC,NULL-ERR

gmtime()将日历时间转换为一个对应的UTC的分解时间。localtime()需要考虑时区和夏令时，返回对应与系统本地时间的一个分解时间。
tm的结构如下：
	
	struct tm
	{
		int tm_sec;	//(0-60)
		int tm_min;	//(0-59)
		int tm_hour;	//(0-23)
		int tm_mday;	//day of the month(1-31)
		int tm_mon;	//month(0-11)
		int tm_year;	//year since 1900
		int tm_wday;	//day of the week(sunday=0)
		int tm_yday;	//day int the year(0-365,1 Jan=0)
		int tm_isdst;	//???		
	}

<ul>
	<li>注意：这里返回的结构也是静态分配的。</li>
</ul>

####分解时间和打印格式之间的转换

	#include <time.h>
	char *asctime(const struct tm *timeptr);
		return pointer to statically allocated string terminated by newline and \0-SUC,NULL-ERR

返回同ctime()。
<ul>
	<li>注意：这里返回的字符串也是静态分配的。</li>
</ul>

当要吧一个分解时间转换为打印格式时，strftime()可以提供更精确的控制：

	#include <time.h>
	size_t strftime(char *outstr,size_t maxsize,const char *format,const struct tm *timeptr);
		return number of bytes placed in outstr(excluding terminating null byte)-SUC,0-ERR

outstr中返回的字符串按照format参数定义的格式做了格式化，MaxSize参数指定了outstr的最大长度，不同于ctime()和asctime(),strftime()返回的字符串不会包含换行符，除非format中本身包含。

如果成功，strftime()返回outstr所指缓冲区的长度，且不包含终止符。如果结果字符串的总长度，含终止空字节，长度超过了Maxsize参数，则返回零，出错，切不能确定outstr中的内容。

格式化表：

稍后补上！！！

###更新系统时钟

一般很少被调用，有时间补上！！！

###进程时间(系统调用)

为了记录的目的，内核把CPU时间分为两个部分：
<ul>
	<li>用户CPU时间是在用户模式下执行花费的时间数量。有时也称为虚拟时间，对于程序来说，是它已经得到的CPU时间。</li>
	<li>系统CPU时间是在内核模式中执行花费的时间数量，这是内核用于系统调用的时间或者代表程序执行其他任务(例如,服务页错误)的时间。</li>
</ul>

系统调用times(),检索进程时间信息，并把结果返回到buf结构体中

	#include <sys/times.h>
	clock_t times(struct tms *buf);
		return number of clock ticks(sysconf(_SC_CLK_TCK))since "arbitrary" time in past on success,or -1-ERR

buf指向的结构如下

	struct tms
	{
		clock_t tms_utime;	//User mode CPU times
		clock_t tms_stime;	//system CPU times by caller
		clock_t tms_cutime;	//User CPU time of all (wait for) children
		clock_t tms_cstime;	//system CPU time of all (wait for) children
	}

tms前两个字段为目前为止的用户时间和系统CPU时间，最后两个是父进程(比如，times的调用者)执行了系统调用wait()的所有已经终止的子京城所使用的CPU时间。

数据类型clock_t是时钟计时单元为单位度量时间的整型值，习惯用于计算tms结构体的4个字段。可以使用sysconf(_SC_CLK_TCK)来获取每秒包含的时钟计时单位，然后用这个数字可以得到秒。

如果成功函数返回自过去的任意点流逝的以时钟计时单位为单位的时间，然而我们不知道这个时间点是什么时候，所以这个返回值唯一的用处是，调用两次后做差，可以得到两次调用间流逝的时间。但是这个时间也是不可靠的，因为clock_t可能溢出后清零。所以可靠的计算流逝时间是调用gettimeofday()做差。

函数clock()提供了一个简单的接口来取得进程时间。

	#include <time.h>
	clock_t clock(void);
		return total CPU time used by the calling process measured in CLOCKS_PER_SEC-SUC,-1-ERR

这里返回的结果是计量单位是宏定义CLOCKS_PER_SEC，所以我们需要除这个值才能得到秒数。

clock()返回的时间包括所有子进程使用的CPU时间，但是在LINUX上，不包括！！！！！！！！





##系统限制和选项

但凡UNIX实现，无不对各种系统特性和资源加以限制，SUSv3规定，针对其所规范的每个限制，所有实现都必须支持一个最小值。在大多数情况下，会将这些最小值定义为<limits.h>文件中的常量，其命名则冠以字符串POSIX，而且通常还会包含字符串_MAX，因此常量的命名形式如_POSIX_XXX_MAX。

每个限制都有一个名称，与上述最小值的名称对应，但确实_POSIX前缀。某个实现可以在<limits.h>中以该名字定义一个常量，用于表示该实现的相应限制，若已然定义，则该限制的值至少等于或者大于前述的最大值即(XXX_MAX>=_POSIX_XXX_MAX)。

SUSv3将限制归为三类，运行时恒定，路径名变量值和运行时可增加值：
<ul>
	<li><运行时恒定：指某一限制，若已然在<limits.h>文件中定义，则对于这个实现而言固定不变。然而该值可能是不确定的(因为该值可能会依赖于可用的内存空间),因而在<limits.h>文件中会忽略其定义。在这种情况下，应用程序可以使用sysconf()来获取运行时值。</li>
	<li>路径变量值：指与路径名（文件、目录、终端）相关的限制，每个限制可能是相对于某个系统实现的常量，也可能随着文件系统的不同而不同。在限制可能因为路径名发生变化的时候变化，这时候可以使用pathconf()或则fpathconf()来获取。</li>
	<li>运行时可增加值：指某一限制，相对于特定的实现固定，且运行此实现的所有系统至少支持这一值。然而，特定系统在运行是可能会增加该值，应用程序可以使用sysconf()来获取系统支持的实际值。</li>
</ul>

###在运行时获取系统限制(和选项)sysconf()(系统调用)

	#include <unistd.h>
	long sysconf(int name);
		return value of limit specified by name,-1 if limit is indeterminate or an err.

参数name应为定义于<unistd.h>文件中的_SC_系列常量之一，限制值将作为函数结果返回。

若无法确定某一个限制，则返回-1。但调用发生错误也会返回-1，区分这两中错误的方法是在调用前设置errno为0，如果调用返回-1，但是errno不为0，则是调用错误。

SUSv3规定，针对特定限制，调用sysconf()所获取的值在调用进程的生命期中不会改变。

限制名和_SC_对比表稍后补上！！！！

###运行时获取文件相关的限制(和选项)pathconf()fpathconf()(系统调用)

	#include <unistd.h>
	long pathconf(const char *pathname,int name);
	long fpathconf(int fd,int name);
		return value of limit specified by name,-1 if limit is indeterminate or an err.

参数name应为定义在<unistd.h>中_PC_系列的常量之一。错误处理同sysconf()。

###系统选项

除了对系统的各种资源加以限制外，SUSv3还规定了UNIX实现可支持的各种选项。包括实时信号，POSIX共享内存、任务控制以及POSIX线程之类功能的支持。除了少数特例外，并未要求实现支持这些选项。相反，对于实现在编译及运行时是否支持某个特定特性，允许实现自行给出建议。

通过在<unistd.h>中定义的相应常量，实现能够在编译时通过其对特定SUSv3选项的支持。此类常量通常会冠以(_POSIX_或者_XOPEN_)，以表示其源于何种标准。

各个常量一经定义，必须为为下列值之一：
<ul>
	<li>-1,表示实现不支持该选项。</li>
	<li>0，表示实现可能支持该选项，需要在应用程序运行时来检查是否获得支持。</li>
	<li>大于0，则表示支持该选项。</li>
</ul>

如果当常量定义为0时，可以使用sysconf()和pathconf()在运行时检查是否支持。传入的name其命名规则通常与编译时常量形式相同，只是前缀为_SC_或者_PC_取代。

表稍后补上！！！





##系统和进程信息

由于主要讨论了/proc文件系统，放到总结分析去，这里仅仅介绍uname()系统调用。

###系统标识uname()(系统调用)

	#include <sys/utsname.h>
	int uname(strcut utsname *utsbuf);
		return 0-SUC,-1-ERR

utsbuf参数指向一个utsname结构

	#define _UTSNAME_LENGTH 65
	struct utsname
	{
		char sysname[_UTSNAME_LENGTH];
		char nodename[_UTSNAME_LENGTH];
		char release[_UTSNAME_LENGTH];
		char version[_UTSNAME_LENGTH];
		char machine[_UTSNAME_LENGTH];
	#ifdef _GNU_SOURCE
		char domainname[_UTSNAME_LENGTH];
	#endif
	};

结构中的sysname,release,version,machine字段由内核自动生成。在Linux下/proc/sys/kernel中的三个文件提供了与ustname中sysname,release,version字段相同的信息。





##文件I/O缓冲

出于效率的考虑，系统I/O调用（即内核）和标准C语言库I/O函数（即stdio函数）在操作磁盘文件时会对数据进行缓冲。

###文件I/O的内核缓冲：缓冲区高速缓存

read()和write()系统调用在操作磁盘文件时不会直接发起磁盘访问，而是仅仅在用户空间缓冲区和内核缓冲区高速缓存之间复制数据。例如，如下调用将3个字节的数据从用户空间内存传递到内核空间的缓冲区：

	write(fd,"abc",3);

write()随即返回，在后续的某个时刻，内核会将其缓冲区中的数据写入磁盘（即刷新至磁盘）。如果在此调用期间，另一个进程企图读取文件，那么会从内核的缓冲区中提供数据，而不是从文件中读取。

与此同理，读取时内核会从磁盘中读取数据并存到内核缓冲区，read()调用将直接从缓冲区取数据，直至将缓冲区数据取完。这时内核会再从文件中读下一段内容到缓冲区。

Linux内核对高速缓冲区没有设置上限，仅受限于物理内存的大小。

<ul>
	<li>内核缓冲的设计旨在减少对磁盘I/O的次数，从而提高效率。</li>
</ul>

###stdio库的缓冲

<ul>
	<li>库缓冲的设计旨在缓冲大块数据以减少系统调用的次数。</li>
	<li>C语言的库I/O函数fprintf(),fscanf(),fgets(),fputs(),fputc(),fgetc()就是这么做的。</li>
</ul>
因此，使用库函数，可以使编程者免于对数据的缓冲。

####设置一个stdio流的缓冲模式
调用一个setvbuf()，可以控制stdio库使用缓冲的形式。

	#include <stdio.h>
	int setvbuf(FILE *stream,char *buf,int mode,size_t size);

参数stream标识将要修改那个文件流的缓冲模式。打开流后，一定要在调用其他任何stdio函数前调用setvbuf()。setvbuf()将影响后续在指定流上进行的所有stdio操作。

参数buf和size则针对参数stream要使用的缓冲区，指定这些参数有如下两种方式
<ul>
	<li>如果参数buf不为NULL，那么其指向size大小的内存块以作为stream的缓冲区。因为stdio库将要使用buf所指的缓冲区，所以该缓冲区必须是动态或者静态在堆上分配的。</li>
	<li>若buf为NULL，那么stdio库会为stream自动分配一个缓冲区(除非选择非缓冲的I/O).SUSv3允许，但是不强制要求库实现使用size来确定其缓冲区的大小，glibc实现会根据该场景下忽略size。</li>
</ul>

参数mode制定缓冲的类型，并具有下列值之一：
<ul>
	<li>_IONBF:不对I/O进行缓冲，每个stdio库函数立即调用read()或者write()，并且忽略buf和size参数，可以制定这两个参数为NULL和0。stderr默认属于这个类型，从而保证错误信息能及时输出。</li>
	<li>_IOLBF:采用行I/O缓冲。指代终端设备的流默认属于这一类型，对于输出流，在输出一个换行符(除非缓冲区填满)前将缓冲数据。对于输入流，每次读取一行。</li>
	<li>_IOFBF:采用全缓冲。单次读、写数据(通过read()或者write()系统调用)的大小与缓冲区相同。指代磁盘的流默认此缓冲模式。</li>
</ul>

	#include <stdio.h>
	void setbuf(FILE *stream,char *buf)

setbuf(fp,buf)调用除了不返回结果外，就相当于：setvbuf(fp,buf,(buf!=NULL) ? _IOFBF:_IONBF,BUF_SIZE);

	#include <stdio.h>
	void setbuffer(FIEL *stream,char *buf,size_t size);

setbuffer函数类似于setbuf()函数，但允许调用指定buf缓冲区大小。

对setbuffer(fp,buf,size)调用相当于：setvbuf(fp,buf,(buf!=NULL) ? _IOFBF:_IONBF,size);

####刷新stdio缓冲区

无论当前采用何种缓冲模式，在任何时刻都可以使用fflush()库函数强制将stdio输出流中的数据刷新到内核缓冲区。

	#include<stdio.h>
	int fflush(FILE *stream)
		return 0-SUC,-1-ERR

<ul>
	<li>如果参数为NULL，则刷新所有stdio缓冲区。</li>
	<li>也能将fflush()应用与输入流，这将丢弃业已缓冲的数据。</li>
	<li>关闭相应流时，将自动刷新其stdio缓冲区。</li>
</ul>
在包括glibc库在内的许多C函数库实现中，若stdin和stdou指向同一个终端，那么无论何时从stdin中读取输入时，都将隐含调用一次fflush(stdout)函数，这将刷新写入stdout的任何提示，但不包括终止换行符。

###控制文件I/O的内核缓冲

强制刷新内核缓冲区到输出文件是有可能的。有时这个也很有必要，例如，当应用程序（如数据库的日志文件）要确保在继续操作前将输出真正写入磁盘（或者至少写入磁盘的硬件高速缓存中。）

####同步I/O数据完整性和同步I/O文件完整性

SUSv3将同步I/O完成定义为：某一I/O操作，要么已成功完成到磁盘的数据传递，要么被诊断为不成功。

SUSv3定义了两种不同类型的同步I/O完成，二者之间的区别涉及用于描述文件的元数据，亦即内核针对文件而存储的数据

定义的第一种同步I/O完成类型是synchronized I/O data integrity completion.(同步I/O数据完整性)，旨在确保针对文件的一次更新传递了足够的信息（到磁盘），以便于之后对数据的获取。
<ul>
	<li>就读操作而言，这意味着被请求的文件数据已经（从磁盘）传递给进程。若存在任何影响到所请求数据的挂起的写操作，那么在执行读操作前，会将这些数据快递到磁盘。</li>	
	<li>就写操作而言，这意味着写请求所指定的数据已传递（到磁盘）完毕，且用于获取数据的所有文件元数据也已传递（到磁盘）完毕。此处的要点在于要获取文件数据，并非需要传递所有经过修改的文件元数据属性。发生修改的文件元数据中需要传递的属性之一是文件大小（如果写操作确实扩展了文件）。相形之下，如果是文件时间戳发生了变化，就无需在下次获取数据前将其传递到磁盘。</li>
</ul>
Synchronized I/O file integrity completion（同步I/O文件完整性）是另一种同步I/O完成，也是上述synchronized I/O data integrity completion的超集。该I/O完成模式的区别在于在对文件的一次更新过程中，要将所有发生更新的文件元数据传递到磁盘上，即使有些在后续对文件数据的读操作中并不重要。

####用于控制文件I/O内核缓冲的系统调用

fsync()系统调用将值缓冲数据和打开文件描述符fd相关的所有元数据都刷新到磁盘上。调用fsync()会强制使文件处于Synchronized I/O file integrity completion状态。

	#include <unistd.h>
	int fsync(int fd);
		return 0-suc,-1-ERR

仅在对磁盘设备（或者至少其高速缓存）的传递完成后，fsync()调用才会返回。

fdatasync()系统调用的运作类似与fsync()，但是会强制是文件处于synchronized I/O data integrity completion的状态。

	#include <unistd.h>
	int fdatasync(int fd);
		return 0-SUC,-1-ERR

fdatasync()可能会减少对磁盘的操作次数，由fsync()调用请求的两次变为一次。例如：若仅仅修改了文件数据，但是没有改变大小，那么调用fdatasync()只是强行更新了数据，而不会更新元数据。相比之下，fsync()会更新元数据。因为文件的数据可元数据通常驻留在磁盘的不同区域，更新这些数据需要反复寻道。

sync()系统调用会使包含更新文件信息的所有内核缓冲区（即数据块，指针块，元数据等）刷新到磁盘上。
	
	#include <unistd.h>
	void sync(void);

在linux实现中，sync()调用仅在所有数据已经传递到磁盘上（至少是高速缓存）时返回。

####使所有写入同步：O_SYNC

调用open()函数时指定O_SYNC标志，则会使所有后续输出同步。

	fd=open(pathname,O_WRONLY|O_SYNC);

调用open()后，每个write()调用会自动将文件数据和元数据刷新到磁盘上。

###就I/O模式向内核提出意见posix_fadvise()系统调用

	#define _XOPEN_SOURCE 600
	#include <fcntl.h>
	int posiz_fadvise(int fd,off_t offset,off_t len,int advice)
		return 0-SUC,or a positive error number on err

内核可以（但是不必要）根据posix_fadvise()所提供的信息来优化对缓冲区高速缓存的使用，进而提高进程和整个系统的性能。

参数fd所指为一文件描述符，调用期望通知内核进程对fd指代文件的访问模式。参数offset和len确定了建议所适用的文件区域，offset指定区域起始的偏移量，len指定了区域的大小。len为0表示从offset开始，到文件尾。

参数advice表示进程期望对文件采用的访问模式，具体为下列之一：
<ul>
	<li>POSIX_FADV_NORMAL:进程对访问模式没有特别建议。如果没有建议，这就是默认行为，在linux中，该操作将文件预读窗口大小设置为128kb</li>
	<li>POSIX_FADV_SEQUENTIAKL:进程预计会从低偏移量到高偏移量顺序读取数据。在linux中，该操作将文件预读窗口设置为默认值的两倍。</li>
	<li>POSIX_FADV_RANDOM:进程预计随机顺序访问数据。在linux中，该选项会禁止文件预读。</li>
	<li>POSIX_FADV_WILLNEED:进程预计会在不久的将来访问指定的文件区域。内核将有offset和len指定区域的文件数据预先读取到缓冲区高速缓存中。对于数据保留多久，内核没有保证。</li>
	<li>POSIX_FADV_DONTNEED:进程预计在不久的将来不会访问指定的文件区域，这一操作给内核的建议是释放相关的高速缓存页面（如果存在的话）</li>
	<li>POSIX_FADV_NOREUSE:进程预计会一次性访问文件区域，而且不会复用，这等于提示内核对指定区域访问一次后即释放页面（linux中还没有实现）</li>
</ul>

###绕过缓冲区高速缓存:直接I/O

可以针对一个单独的文件或者一个块设备执行直接I/O。要做到这点，需要在调用open()打开文件或者设备时指定O_DIRECT标志。

注意，如果一个进程以O_DIRECT方式打开文件，但是另一个进程以普通方式打开同一个文件，则直接I/O所读写的数据与缓冲区高速缓冲中的数据可能不一致，需要避免！！

因为直接I/O涉及对磁盘的直接访问，所以在执行I/O时，必须遵守一些限制：
<ul>
	<li>用于传输数据的缓冲区，其内存边界必须对齐为块大小的整数倍。</li>
	<li>数据传输的开始，亦即文件和设备的偏移量，必须为块大小的整数倍。</li>
	<li>待传输数据的长度必须是块大小的整数倍。</li>
</ul>
不遵守上述规定，将会导致EINVAL错误。这里的块大小指设备的物理块大小（即512bytes）。

###混合使用库函数和系统调用进行文件I/O

在同一文件上执行I/O时，还可以将系统调用和标准C语言库函数混合使用。fileno()和fdopen()函数有助于完成这一工作：

	#include <>stdio.h>
	int fileno(FILE *stream);
		return file descriptor on success,-1-ERR
	FILE*fdopen(int fd,const char *mode);
		return(new) file pointer on success,-1-ERR

给定一个流，fileno()将返回相应的文件描述符(即stdio库在该流上已经打开的文件描述符)。随机可以在诸如read(),write(),dup()和fcntl()之类的I/O系统调用中正常使用fd。

fdopen()函数与fileno()功能相反，给定一个文件描述符，函数将创建一个使用该描述符进行文件I/O的相应流。mode参数与fopen()中的mode参数含义相同。r为读，w为写，a为追加。若参数和文件描述符fd的访问模式不一样，则对fdopen的调用将返回失败。

使用stdio函数，并结合系统调用来实现对文件I/O的操作时，必须将缓冲问题牢记于心。I/O系统调用会直接将数据传递到内核缓冲区高速缓存。而stdio库函数会等到用户空间缓冲区填满在调用write()将其传递到内核缓冲。





##系统编程概念

以下忽略，见详细分析
<ul>
	<li>设备专用文件</li>
	<li>磁盘和分区</li>
	<li>文件系统</li>
	<li>i节点</li>
	<li>虚拟文件系统（VFS）</li>
	<li>日志文件系统</li>
	<li>单根目录层级和挂载点</li>	
</ul>

###文件系统的挂载和卸载

系统调用mount()和umount()运行特权级进程（CAP_SYS_ADMIN）以挂载或卸载文件系统。

以下三个文件包含了当前已挂载或者可挂载的文件系统信息：
<ul>
	<li>/proc/mounts：可查看当前已挂载文件系统的列表。此文件是内核数据结构的接口，所以总是包含已挂载文件系统的精确信息。</li>
	<li>/etc/mtab：包含的信息和上一个文件类似，只是略微详细一点，特别包含了mount(8)传递给文件系统的专有项，这在上一个文件里没有出现。但是因为系统调用mount()和umount()不更新这个文件，所以这个文件中的内容可能不准确。</li>
	<li>/etc/fstab：由系统管理员手工维护，包含了对系统支持的所有文件系统的描述，该文件可供mount(8),umount(8)以及fsck(8)使用。</li>
</ul>
以上三个文件格式相同，如下所示：

	/dev/sda9 /boot ext3 rw 0 0

<ul>
	<li>1、已挂载的设备名</li>
	<li>2、设备的挂载点</li>
	<li>3、文件系统类型</li>
	<li>4、挂载标志，如上就是已读写挂载</li>
	<li>5、一个数字，dump(8)会使用其来控制对文件系统备份。只有/etc/fstab文件中才用的到该字段。</li>
	<li>6、一个数字，在系统引导时，用于控制fsck(8)对文件系统的检查。</li>
</ul>

###挂载文件系统mount()(系统调用)
	
	#include <sys/mount.h>
	int mount(const char *source,const char *target,const char *fstype,unsigned long mountflags, const void *data);
		return 0-SUC,-1-ERR
	
mount系统调用用于将由source指定设备包含的文件系统挂载到target指定的目录下。这两个参数命名为source和target的原因是mount除了将文件系统挂载到一目录下，还可以执行其他工作。

参数fstype是一串字符串，用来标识设备所含的文件系统，比如ext4或btrfs。

参数mountflags为掩码：
<ul>
	<li>MS_BING:建立绑定挂载。</li>
	<li>MS_DIRSYNC:同步更新路径</li>
	<li>MS_MANDLOCK:允许强制锁定文件</li>
	<li>MS_MOVE:以原子操作将挂载点移到新位置</li>
	<li>MS_NOATIME:不更新文件的最后访问时间</li>
	<li>MS_NODEV:不允许访问设备</li>
	<li>MS_NODIRTIME:不更新目录的最后访问时间</li>
	<li>MS_NOEXEC:不允许程序执行</li>
	<li>MS_RDONLY:禁用set-user-ID和set-group-ID程序</li>
	<li>MS_REG:递归挂载</li>
	<li>MS_RELATIME:只有当最后访问时间早于修改时间或者最后状态变更时间时，才对前者进行更新</li>
	<li>MS_REMOUNT:使用新的mountflags和data重新挂载</li>
	<li>MS_STRICTATIME:总是更新最后访问时间</li>
	<li>MS_SYNCHRONOUS:使得所有文件和目录同步更新</li>
</ul>

###卸载文件系统：umount()和umount2()

	#include <sys/mount.h>
	int umount(const char *target);
		return 0-SUC,-1-ERR

target参数指定待卸载文件系统的挂载点。无法卸载正在使用中(busy)文件系统，意即这一文件系统上有文件被打开，或者进程的当前工作目录驻留在次文件系统下。此时调用会返回EBUSY。

	#include <sys/mount.h>
	int umount2(const char *target,int flags);
		return 0-SUC,-1-ERR

系统调用umount2比mount的扩展版，可以对卸载操作进行更精密的操作。flags为0或者有多个值OR而成，flags如下：
<ul>
	<li>MNT_DETACH:执行lazy卸载。对挂载点进行标记，一方面允许已使用了挂载点的进程得以继续使用，同时禁止任何其他进程对该挂载点发起新的访问。当所有进程不在访问点时，系统会卸载相应的文件系统。</li>
	<li>MNT_EXPIRE:将挂载点标记为到期(expired)？？？？？</li>
	<li>MNT_FORCE:即便文件系统（只对NFS挂载有效）处与在用状态，依然将其强行卸载。采用这一选项可能会造成数据丢失。</li>
	<li>UMOUNT_NOFOLLOW:若target为符号链接，则不对其进行解引用。该标志专为某些set-user-ID-root程序而设计的，此类程序允许非特权级用户执行卸载操作，意在避免安全问题。</li>
</ul>

###略去一些高级挂载特性，还有虚拟内存文件系统，tmpfs。

###获得文件系统有关的信息statvfs()

	#include <sys/statvfs.h>
	int statvfs(const char *pathname,struct statvfs *statvfsbuf);
	int fstatvfs(int fd,struct statvfs *statvfsbuf);
		return 0-SUC,-1-ERR

两者的唯一区别是用于标识文件系统的方式，前者需要指定pathname，后者需要打开文件的描述符。两者均会返回一个结构：
	
	struct statvfs
	{
		unsigned long f_bsize;	//filesystem block size
		unsigned long f_frsize;	//fundamental filesystem block size
		
		fsblkcnt_t f_blocks;	//Total number of blocks in file system
		fsblkcnt_t f_bfree;	//Total number of free blocks
		fsblkcnt_t f_bavail;	//number of free blocks available to unprivileged process

		fsfilcnt_t f_files;	//total number of i-nodes
		fsfilcnt_t f_ffree;	//total number of free i-nodes
		fsfilcnt_t f_favail;	//number of i-nodes available to unprivileged process
		
		unsigned long f_fsid;	//filesystem ID
		unsigned long f_flag;	//mount flags
		unsigned long f_namemax;//maximum length of filenames on this filesystem
	}





##文件属性
