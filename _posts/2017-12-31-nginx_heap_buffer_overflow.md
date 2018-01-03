---
layout: post
title: 线上nginx踩内存问题定位记录
---
最近线上nginx遇到了踩内存的问题，最终借助gdb、AddressSanitizer和章亦春大神的no-pool-nginx 的patch找到了问题所在。在这里简单记录一下整个定位的过程，以备后面查阅。
	
#一、GDB
线上版本灰度过程中，会出现core 的现象，大概跑一天左右出现一两个，线下测试无法复现，并且core的位置并不完全相同，根据gdb查看的信息，猜测是踩内存。选取其中一个core来分析，首先查看调用堆栈。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_7/blog_7_1.png"> 
  
进程挂在了从pool中分配内存的函数"ngx\_palloc\_small"中。这种情况，肯定是pool中有问题了。打印pool看一下
	
	(gdb) p *pool
	$1 = {d = {last = 0x27ffbc8 "", end = 0x14412b8 "", next = 0x1a3f210, failed = 31766912}, max = 31767456, current = 0x0, chain = 0x0, large = 0x0, cleanup = 0x14414d8, log = 0x0}

打印的pool中，可以看出，current、max、failed值明显不正确。另外，max、failed值看起来像指针，而非正确的数值。那么打印出16进制看一下。
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_7/blog_7_2.png">    

这里pool的管理结构"ngx\_pool\_t" 应该是被踩了，那么是被谁踩了呢？这里的pool是一批buff池中的pool，初始化时，对于buff池会统一初始化，并创建pool，所以这些buff的pool内存在heap上应该是连续的。那么看一下上一个buff的pool的情况。
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_7/blog_7_3.png">    

上一个buff的pool值比较正常。同时发现了一个问题，就是打印当前挂掉的pool的16进制时，其中一个指针刚好就是上一个buff的pool指针。这也刚好验证了当前导致程序挂掉的pool管理结构被上一个pool分配的内存踩了。在buff的pool使用的结构体中，其中有指向pool指针的结构体是"ngx\_http\_request\_t"。那么这里是不是"ngx\_http\_request\_t" 呢？我们可以打印看一下。怎么打印呢？首先找出pool指针在"ngx\_http\_request\_t"指针中的偏移，然后将指向(buff - 1)->pool指针的地址减去偏移就可以得到"ngx\_http\_request\_t"的起始地址。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_7/blog_7_4.png">  
    
打印出来的"ngx\_http\_request\_t" 的signature 等于1347703880（#define NGX\_HTTP\_MODULE           0x50545448   /* "HTTP" */），验证了确实当前挂掉的pool确实是被"ngx\_http\_request\_t" 结构体覆盖了。另外，在下图的打印中，奇怪的一点是，这个request的起始地址比上一个pool的end值还大，并且肯定也不是一个合法的"ngx\_pool\_data\_t"，因为后面的内存紧接着就是下一个buff的pool。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_7/blog_7_5.png">   

这里就比较奇怪了，难道这个pool也曾经被踩了，然后reset之后又正常了？至此就陷入了困境，对于真正的原因还是没有头绪。


# 二、使用AddressSanitizer
想起了google的AddressSanitizer，可以定位关于内存的各种错误，包括use after free, heap buffer overflow, stack overflow, memory leaks等等错误。具体可以参考https://github.com/google/sanitizers/wiki/AddressSanitizer。于是就想用AddressSanitizer来看一下。目前gcc 4.8 以上已经支持AddressSanitizer了。所以，执行nginx的configure时，加上一下配置，然后重新编译nginx就可以使用强大的AddressSanitizer工具。
	
	--with-cc-opt="-g -fPIC -fsanitize=address -fno-omit-frame-pointer -lasan" 
	--with-ld-opt="-g -fPIC -fsanitize=address -fno-omit-frame-pointer -lasan" 

重新编译后nginx，跑一晚上之后，会有如下错误：

	==17300== ERROR: AddressSanitizer: heap-buffer-overflow on address 0x606200a7c308 at pc 0x4686a4 bp 0x7ffd9c11a370 sp 0x7ffd9c11a360
	WRITE of size 1 at 0x606200a7c308 thread T0
    #0 0x4686a3 (/usr/local/l7/l7_nginx/nginx+0x4686a3)
    #1 0x85a43a (/usr/local/l7/l7_nginx/nginx+0x85a43a)
    #2 0x857dc3 (/usr/local/l7/l7_nginx/nginx+0x857dc3)
    #3 0x84ff96 (/usr/local/l7/l7_nginx/nginx+0x84ff96)
    #4 0x84f249 (/usr/local/l7/l7_nginx/nginx+0x84f249)
    #5 0x4721c0 (/usr/local/l7/l7_nginx/nginx+0x4721c0)
    #6 0x647bd0 (/usr/local/l7/l7_nginx/nginx+0x647bd0)
    #7 0x61ab55 (/usr/local/l7/l7_nginx/nginx+0x61ab55)
    #8 0x55b631 (/usr/local/l7/l7_nginx/nginx+0x55b631)
    #9 0x572c6d (/usr/local/l7/l7_nginx/nginx+0x572c6d)
    #10 0x57136b (/usr/local/l7/l7_nginx/nginx+0x57136b)
    #11 0x581a0f (/usr/local/l7/l7_nginx/nginx+0x581a0f)
    #12 0x55609d (/usr/local/l7/l7_nginx/nginx+0x55609d)
    #13 0x555c18 (/usr/local/l7/l7_nginx/nginx+0x555c18)
    #14 0x555aaa (/usr/local/l7/l7_nginx/nginx+0x555aaa)
    #15 0x57fa17 (/usr/local/l7/l7_nginx/nginx+0x57fa17)
    #16 0x57c273 (/usr/local/l7/l7_nginx/nginx+0x57c273)
    #17 0x57a17c (/usr/local/l7/l7_nginx/nginx+0x57a17c)
    #18 0x575f11 (/usr/local/l7/l7_nginx/nginx+0x575f11)
    #19 0x517770 (/usr/local/l7/l7_nginx/nginx+0x517770)
    #20 0x4e6430 (/usr/local/l7/l7_nginx/nginx+0x4e6430)
    #21 0x510646 (/usr/local/l7/l7_nginx/nginx+0x510646)
    #22 0x506f85 (/usr/local/l7/l7_nginx/nginx+0x506f85)
    #23 0x50d200 (/usr/local/l7/l7_nginx/nginx+0x50d200)
    #24 0x50c3d1 (/usr/local/l7/l7_nginx/nginx+0x50c3d1)
    #25 0x45d261 (/usr/local/l7/l7_nginx/nginx+0x45d261)
    #26 0x7f4432304b14 (/usr/lib64/libc-2.17.so+0x21b14)
    #27 0x45c3d0 (/usr/local/l7/l7_nginx/nginx+0x45c3d0)
	0x606200a7c308 is located 6 bytes to the right of 4098-byte region [0x606200a7b300,0x606200a7c302)

通过addr2line 可以找出heap-buffer-overflow的堆栈。

	root@:/data/# addr2line -e ./nginx 0x4686a3
	/data/nginx-1.13.5/src/core/ngx_palloc.c:304

	void *
	ngx_pcalloc(ngx_pool_t *pool, size_t size)
	{
    	void *p;

    	p = ngx_palloc(pool, size); // 304 行
    	if (p) {
       	ngx_memzero(p, size);
    	}

    	return p;
	}

程序挂在ngx\_memzero 这一行，因为ngx\_palloc返回的地址超过了end。这里表现和使用gdb分析得到的结果是一样的。但是为什么会ngx\_palloc 会返回不合法的地址呢？难道这里的pool值已经不对了吗？
考虑到nginx使用pool来管理内存，如果pool管理的内存内有写越界，其实是没办法检测到的，因为这些内存事实是合法的内存。在网上找到了章亦春大神（agentzh）的no-pool-nginx patch。打上这个patch，就可以替换nginx原生的pool，而是使用malloc、free来管理内存。这样，当pool内的内存出现问题，就会马上报错。
打上patch后，奇怪的是破了几天居然没有任何问题。之前跑一晚上基本上会出现一两次问题。在这里又重新没有了思路，打上no-pool-nginx的patch为什么又不出现呢？最终一切从代码出发，再一次仔细阅读ngx\_palloc\_small的代码，发现了问题。

	static ngx_inline void *
	ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
	{
    	u_char      *m;
    	ngx_pool_t  *p;

    	p = pool->current;

    	do {
        m = p->d.last;

        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }

        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;

            return m;
        }

        p = p->d.next;

    	} while (p);

    	return ngx_palloc_block(pool, size);
	}

Ngx\_palloc\_small这里在分配内存的时候，首先会将m(p->d.last) 对齐。对齐后m是可能大于end的，当时在比较时 ((size_t) (p->d.end - m) >= size) 转换成了size_t来比较，end - m为负数，但是转换成无符号的size_t后，就变成了大于size的正数。而我们自己写的第三方模块，因为笔误，将4096写成了4098，导致创建pool时传入的size不是对齐的。最终可能就分配一个错误的内存地址。
我给nginx官方提了一个defect(https://trac.nginx.org/nginx/ticket/1454
)。nginx官方的回复是之前有人提过这个问题，当前不会修复（https://trac.nginx.org/nginx/ticket/686）。另外，在官方文档中增加了对于pool的size的要求（http://nginx.org/en/docs/dev/development_guide.html#pool）：  
	
	ngx_create_pool(size, log) — Create a pool with specified block size. 
	The pool object returned is allocated in the pool as well.
 	The size should be at least NGX_MIN_POOL_SIZE and a multiple of NGX_POOL_ALIGNMENT.


#三、结语

尽管nginx官方不会修复这个问题，但是我觉得这里实现的并不好。首先代码注释上没有对于size有任何要求，实现时分配的内存大小事实上是对齐的，但是end指针却并没有指向真正分配内存的最后。这里估计我们不会是最后一个踩坑的。总体而言，可能还是对于nginx 官方文档不够熟悉吧，导致浪费了很多时间去定位这个问题。不过AddressSanitizer和no-pool-nginx patch确实非常有用，对于定位nginx的一些问题是非常有用的。另外，对于nginx的一些defect和最新进展需要及时掌握，才能帮助更好得定位问题。
