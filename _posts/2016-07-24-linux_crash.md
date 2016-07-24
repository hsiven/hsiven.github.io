---
layout: post
title: 两则内核crash定位案例
---

最近碰到了两起内核core，都比较难定位。不过最终组里大神hyman帮忙定位出了原因，现在在这里简单记录一下，以备需要需要时查阅。

# 一、递归造成的内核栈溢出

由于我内核编码经验不足，在内核中使用了递归，造成了内核栈溢出的问题。下面介绍一下定位出是由于递归造成core，并打印出堆栈的过程。在这使用了crash工具。具体crash可自行搜索。

具体步骤如下：

* ./crash System.map-2.6.32.43-tlinux-1.0.4.m-default 
vmlinux-2.6.32.43-tlinux-1.0.4.m-default ./2016-07-19-09\:51/vmcore

* mod -s tvs *.ko，加载ko的符号

* dmesg，打印内核crash时的log信息  


			
		[ 1088.796252] WARNING: at kernel/softirq.c:159 local_bh_enable_ip+0x7c/0xa0()
		[ 1088.796253] Hardware name: PowerEdge R630
		[ 1088.796254] Modules linked in: tvs
		[ 1088.796257] Pid: 9354, comm: tvsadm Not tainted 2.6.32.43-tlinux-1.0.4.m-default #1
		[ 1088.796258] Call Trace:
		[ 1088.796261]  [<ffffffff81049b9c>] ? local_bh_enable_ip+0x7c/0xa0
		[ 1088.796264]  [<ffffffff810445c7>] warn_slowpath_common+0x87/0xb0
		[ 1088.796266]  [<ffffffff810445ff>] warn_slowpath_null+0xf/0x20
		[ 1088.796268]  [<ffffffff81049b9c>] local_bh_enable_ip+0x7c/0xa0
		[ 1088.796272]  [<ffffffff817fc41f>] _spin_unlock_bh+0xf/0x20
		[ 1088.796275]  [<ffffffff816e220f>] release_sock+0xaf/0xc0
		[ 1088.796277]  [<ffffffff817202ec>] ip_setsockopt+0x9c/0xa0
		[ 1088.796280]  [<ffffffff8173bb89>] udp_setsockopt+0x19/0x40
		[ 1088.796282]  [<ffffffff816e19df>] sock_common_setsockopt+0xf/0x20
		[ 1088.796284]  [<ffffffff816df21c>] sys_setsockopt+0x6c/0xc0
		[ 1088.796287]  [<ffffffff8100bc6b>] system_call_fastpath+0x16/0x1b
		[ 1088.796289] ---[ end trace fed7807705e651e8 ]---
		[ 1088.796325] tvsadm[9354]: segfault at 7f06206a3e10 ip 00007f06206a3e10 sp 00007fff23e035f8 error 14
		[ 1088.796328] BUG: unable to handle kernel paging request at 00000001a1797e00
		[ 1088.796356] IP: [<ffffffff81036372>] task_curr+0x12/0x30
		[ 1088.796374] PGD 206cf18067 PUD 0 
		[ 1088.796397] Thread overran stack, or stack corrupted
		[ 1088.796409] Oops: 0000 [#1] SMP 
		[ 1088.796433] last sysfs file: /sys/devices/system/cpu/online
		[ 1088.796446] CPU 0 
		[ 1088.796463] Modules linked in: tvs
		[ 1088.796484] Pid: 9354, comm: tvsadm Tainted: G        W  2.6.32.43-tlinux-1.0.4.m-default #1 PowerEdge R630
		[ 1088.796508] RIP: 0010:[<ffffffff81036372>]  [<ffffffff81036372>] task_curr+0x12/0x30
		[ 1088.796534] RSP: 0000:ffff882043f57d18  EFLAGS: 00010046
		[ 1088.796550] RAX: 0000000000011e00 RBX: 0000000000000000 RCX: 000000000000000a
		[ 1088.796570] RDX: 0000000043f57da0 RSI: ffff882076c06040 RDI: ffff882076c06040
		[ 1088.796591] RBP: ffff882043f57d18 R08: 0000000000000000 R09: 0000000000000000
		[ 1088.796612] R10: 0000000000000000 R11: 0000000000030001 R12: ffff882076c06040
		[ 1088.796633] R13: 000000000000000a R14: 000000000000000b R15: 000000000000000a
		[ 1088.796655] FS:  00007f0621aaa720(0000) GS:ffff880060600000(0000) knlGS:0000000000000000
		[ 1088.796860] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
		[ 1088.796967] CR2: 00000001a1797e00 CR3: 0000002076fd1000 CR4: 00000000000406f0
		[ 1088.797079] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
		[ 1088.797189] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
		[ 1088.797299] Process tvsadm (pid: 9354, threadinfo ffff882043f56000, task ffff882076c06040)
		[ 1088.797499] Stack:
		[ 1088.797596]  ffff882043f57d68 ffffffff810508b8 ffffffff81c32ec0 ffff882077d17080
		[ 1088.797718] <0> 0000000a76c06040 ffff882043f57e18 ffff8810769e64f0 ffff882076c06040
		[ 1088.797937] <0> 000000000000000b ffff882076c064d0 ffff882043f57da8 ffffffff81050c67
		[ 1088.798250] Call Trace:
		[ 1088.798355]  [<ffffffff810508b8>] complete_signal+0xe8/0x200
		[ 1088.798466]  [<ffffffff81050c67>] __send_signal+0x157/0x290
		[ 1088.798575]  [<ffffffff81050dac>] send_signal+0xc/0x10
		[ 1088.798682]  [<ffffffff8105136b>] specific_send_sig_info+0xb/0x10
		[ 1088.798791]  [<ffffffff8105186a>] force_sig_info+0x9a/0xf0
		[ 1088.798898]  [<ffffffff8102aa2f>] force_sig_info_fault+0x3f/0x50
		[ 1088.799007]  [<ffffffff810c0000>] ? kill_block_super+0x40/0x50
		[ 1088.799116]  [<ffffffff812a2c4d>] ? __ratelimit+0xbd/0xe0
		[ 1088.799224]  [<ffffffff8102adff>] __bad_area_nosemaphore+0xbf/0x1c0
		[ 1088.799334]  [<ffffffff8102af9e>] bad_area_nosemaphore+0xe/0x10
		[ 1088.799447]  [<ffffffff8102b4d6>] do_page_fault+0x196/0x2d0
		[ 1088.799559]  [<ffffffff817fc68f>] page_fault+0x1f/0x30
		[ 1088.799665] Code: 48 29 c6 48 c1 fe 03 48 01 f0 48 89 07 c9 c3 66 66 66 90 66 66 90 66 66 90 48 8b 57 08 55 48 c7 c0 00 1e 01 00 48 89 e5 8b 52 18 <48> 8b 14 d5 00 91 cd 81 48 3b bc 10 98 07 00 00 c9 0f 94 c0 0f 
		[ 1088.800241] RIP  [<ffffffff81036372>] task_curr+0x12/0x30
		[ 1088.800350]  RSP <ffff882043f57d18>
		[ 1088.800451] CR2: 00000001a1797e00


其实，通过dmesg打印，也能看出一些信息了。上面标蓝色的信息，其实就是表示栈溢出了，但具体在哪里出问题了目前还不知道。从下面这句看出来，是在task_curr 这里面出问题了（这里也可以通过bt 来查看）。
		
	[ 1088.796356] IP: [<ffffffff81036372>] task_curr+0x12/0x30

那么我们接下来反汇编 task_curr 看看有没有什么发现。

* disassemble task_curr    

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_5/disassemble_task_curr.png">    


从dmesg中可以看出，是task_curr + 0x12 这一句出问题的，所以也就是上图中红色框框的那一句，这句是要将地址为（%rdx* 8 - 0x7e326f00）中的值赋给%rdx。这一句crash了，那么%rdx 中的值肯定是有问题的。从dmesg中我们可以看到寄存器中的值如下所示：

	[ 1088.796508] RIP: 0010:[<ffffffff81036372>]  [<ffffffff81036372>] task_curr+0x12/0x30
	[ 1088.796534] RSP: 0000:ffff882043f57d18  EFLAGS: 00010046
	[ 1088.796550] RAX: 0000000000011e00 RBX: 0000000000000000 RCX: 000000000000000a
	[ 1088.796570] RDX: 0000000043f57da0 RSI: ffff882076c06040 RDI: ffff882076c06040
	[ 1088.796591] RBP: ffff882043f57d18 R08: 0000000000000000 R09: 0000000000000000
	[ 1088.796612] R10: 0000000000000000 R11: 0000000000030001 R12: ffff882076c06040
	[ 1088.796633] R13: 000000000000000a R14: 000000000000000b R15: 000000000000000a。

我们再看一下linux内核代码中task_curr的实现：

	/**
	 * task_curr - is this task currently executing on a CPU?
	 * @p: the task in question.
	 */
	inline int task_curr(const struct task_struct *p)
	{
		return cpu_curr(task_cpu(p)) == p;
	}


X86_64中使用寄存器来传参，从右到左前六个整数或者指针的参数分别放在RDI、RSI、RDX、RCX、R8、R9 这六个寄存器中（具体可见：
https://en.wikipedia.org/wiki/X86_calling_conventions#System_V_AMD64_ABI ）。所以从汇编的第一句我们可以知道RDI中是参数p，也就是一个task_struct指针。Task_struct 是内核中管理进程的机构体，其中有一个stack指针指向当前进程的内核栈。那么我们可以打印出task_struct看看。struct task_struct 0xffff882076c06040，其中0xffff882076c06040 是寄存器%rdi的内容。上面我们已经推断出rdi指向当前的task_struct。

* struct task_struct ffff882076c06040
		
		crash> struct task_struct ffff882076c06040
		struct task_struct {
		  state = 0, 
		  stack = 0xffff882043f56000, 
		  usage = {
		    counter = 2
		  }, 
		  flags = 4202816, 
		  ptrace = 0, 
		  lock_depth = -1, 
		  prio = 120, 
		  static_prio = 120, 
		  normal_prio = 120, 
		  rt_priority = 0, 
		  sched_class = 0xffffffff8180e600, 
		  se = {
		    load = {
		      weight = 1024, 
		      inv_weight = 4194304
		    }, 
		    run_node = {
		      rb_parent_color = 1, 
		      rb_right = 0x0, 
		      rb_left = 0x0
		    }, 
		    group_node = {
		      next = 0xffff880060611ea0, 
		      prev = 0xffff880060611ea0
		    }, 
		    on_rq = 1, 
		    exec_start = 1088794391276, 
		    sum_exec_runtime = 13429719, 
		    vruntime = 7543356257, 
		    prev_sum_exec_runtime = 13429719, 
		    last_wakeup = 0, 
		.........
	
从这里我们可以找到stack指针的地址，也就是0xffff882043f56000。在这里，我们看一下内核栈的布局。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_5/linux_stack_layout.png">

从上面这张图可以看出，stack指向的就是thread_info 结构体。而内核栈和thread_info 结构体共享内核栈空间（一共8k），thread_info就存储在内核栈的底部，内核栈是从高地址往低地址生长的，一旦溢出首先就破坏了thread_info。thread_info里存放着指向进程的指针等关键数据，迟早会被访问到，那时系统崩溃就是必然的事。接下来我们我们看看thread_info 这个结构现在是怎么样的？

* struct thread_info 0xffff882043f56000

		crash> struct thread_info 0xffff882043f56000
		struct thread_info {
		  task = 0x23, 
		  exec_domain = 0x0, 
		  flags = 0, 
		  status = 60, 
		  cpu = 1140161952, 
		  preempt_count = -30688, 
		  addr_limit = {
		    seg = 154618818560
		  }, 
		  restart_block = {
		    fn = 0xffffc9001f7ff460, 
		    {
		      {
		        arg0 = 3458905248413614496, 
		        arg1 = 2, 
		        arg2 = 0, 
		        arg3 = 0
		      }, 
		      futex = {
		        uaddr = 0x30007fff536d81a0, 
		        val = 2, 
		        flags = 0, 
		        bitset = 0, 
		        time = 0, 
		        uaddr2 = 0x0
		      }, 
		      nanosleep = {
		        index = 1399685536, 
		        rmtp = 0x2, 
		        compat_rmtp = 0x0, 
		        expires = 0
		....
		
	
从上面可以看出，task指针，cpu值这些明显不对，所以这一块内存肯定是被覆盖掉了。那么我们接下来看看整个内核栈里面到底有什么内容。

* x /1024gx 0xffff882043f56000

		crash> x /1024gx 0xffff882043f56000
		0xffff882043f56000:     0x0000000000000023      0x0000000000000000
		0xffff882043f56010:     0x0000003c00000000      0xffff882043f57da0
		0xffff882043f56020:     0x00000023fffff000      0xffffc9001f7ff460
		0xffff882043f56030:     0x30007fff536d81a0      0x0000000000000002
		0xffff882043f56040:     0x0000000000000000      0x0000000000000000
		0xffff882043f56050:     0x0000000000000000      0x0000000000000000
		0xffff882043f56060:     0x0000000000000080      0xffffc9001f7ff460
		0xffff882043f56070:     0x0000000000000002      0x0000000000000002
		0xffff882043f56080:     0x0000000000000000      0xffff882043f56158
		0xffff882043f56090:     0xffffffffa0022f73      0x0000000000000023
		0xffff882043f560a0:     0x0000000000000000      0x0000000000000000
		0xffff882043f560b0:     0x0000000000000000      0x0000000000000000
		0xffff882043f560c0:     0x0000000000000000      0x0000000000000000
		0xffff882043f560d0:     0x0000000000000022      0x0000000000000000
		0xffff882043f560e0:     0x0000003c00000000      0xffff882043f57da0
		0xffff882043f560f0:     0x0000002200000000      0xffffc9001f7ff440
		0xffff882043f56100:     0x3000000000000000      0x0000000000000000
		0xffff882043f56110:     0x0000000000000000      0x0000000000000000
		0xffff882043f56120:     0x0000000000000000      0x0000000000000000
		0xffff882043f56130:     0x0000000000000080      0xffffc9001f7ff440
		0xffff882043f56140:     0x0000000000000002      0x0000000000000002
		0xffff882043f56150:     0x0000000000000000      0xffff882043f56228
		0xffff882043f56160:     0xffffffffa0022f73      0x0000000000000022
		0xffff882043f56170:     0x0000000000000000      0x0000000000000000
		0xffff882043f56180:     0x0000000000000000      0x0000000000000000
		0xffff882043f56190:     0x0000000000000000      0x0000000000000000
		0xffff882043f561a0:     0x0000000000000021      0x0000000000000000
		0xffff882043f561b0:     0x0000003c00000000      0xffff882043f57da0
		0xffff882043f561c0:     0x0000002100000000      0xffffc9001f7ff420
		0xffff882043f561d0:     0x3000000000000000      0x0000000000000000
		0xffff882043f561e0:     0x0000000000000000      0x0000000000000000
		0xffff882043f561f0:     0x0000000000000000      0x0000000000000000
		0xffff882043f56200:     0x0000000000000080      0xffffc9001f7ff420
		0xffff882043f56210:     0x0000000000000002      0x0000000000000002
		.............. 
		
从栈中没有看到明显的ascii码，不过有一些指针，可以尝试打印出内容看看。当打印到0xffffffffa0022f73这个地址时，发现了一些线索.

* x /16gx 0xffffffffa0022f73

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_5/build_children_trie.png">

从这里可以看出，这个地址是代码段里的。反汇编build_children_trie可以看出，这个地址是递归调用函数的返回地址。所以至此，可以判断定是这个递归函数导致了栈溢出。

# 二、sprintf引起的惨剧

定位到上面内核crash是由于递归造成的原因之后，就开始修改。修改完后进行稳定性测试，发现不停的stop.sh admin，然后start.sh admin，此时流量大概80w pps左右，此时会大概平均一两个小时后，会打印一些莫名其妙段错误信息，如下：

	Message from syslogd@TENCENT64 at Jul 13 21:16:00 ...
	 kernel:[12318.634247] ------------[ cut here ]------------
	
	Message from syslogd@TENCENT64 at Jul 13 21:16:00 ...
	 kernel:[12318.634277] invalid opcode: 0000 [#1] SMP 
	
	Message from syslogd@TENCENT64 at Jul 13 21:16:00 ...
	 kernel:[12318.634302] last sysfs file: /sys/devices/system/cpu/online
	
	Message from syslogd@TENCENT64 at Jul 13 21:16:00 ...
	 kernel:[12318.635476] Stack:
	
	Message from syslogd@TENCENT64 at Jul 13 21:16:00 ...
	 kernel:[12318.636231] Call Trace:
	
	Message from syslogd@TENCENT64 at Jul 13 21:16:00 ...
	 kernel:[12318.637550] Code: 18 48 83 c4 18 5b 41 5c 41 5d 41 5e 41 5f c9 c3 66 90 49 8b 57 10 48 89 42 08 48 89 10 4c 89 40 08 49 89 47 10 eb 89 0f 0b eb fe <0f> 0b eb fe 31 c9 44 89 ea 44 89 e6 48 89 df e8 23 fb ff ff 65 

Dmesg中有如下信息：

	[90457.033433] general protection fault: 0000 [#1] SMP 
	[90457.033461] last sysfs file: /sys/devices/system/cpu/online
	[90457.033475] CPU 26 
	[90457.033492] Modules linked in: tvs [last unloaded: tvs]
	[90457.033522] Pid: 173, comm: events/26 Tainted: G        W  2.6.32.43-tlinux-1.0.4.m-default #1 PowerEdge R630
	[90457.033546] RIP: 0010:[<ffffffff810b64ce>]  [<ffffffff810b64ce>] free_block+0x8e/0x180
	[90457.033573] RSP: 0018:ffff8810796a7db0  EFLAGS: 00010003
	[90457.033590] RAX: 00705c14707070a8 RBX: 2020202020203020 RCX: 0000000000000000
	[90457.033610] RDX: ffffea0000000000 RSI: ffff8804657dd018 RDI: 2020202020203020
	[90457.033631] RBP: ffff8810796a7de0 R08: 0000000000000000 R09: ffffffff81cd9888
	[90457.033652] R10: 0000000000000000 R11: 00000000303a3030 R12: ffff8804657dd018
	[90457.033673] R13: ffff88045a8b81c0 R14: 0000000000000000 R15: 0000000009a33cd7
	[90457.033694] FS:  0000000000000000(0000) GS:ffff8800607a0000(0000) knlGS:0000000000000000
	[90457.033814] CS:  0010 DS: 0018 ES: 0018 CR0: 000000008005003b
	[90457.033923] CR2: 00007feee136e38e CR3: 0000001075d92000 CR4: 00000000000406e0
	[90457.034034] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
	[90457.034145] DR3: 0000000000000000 DR6: 00000000ffff0ff0 DR7: 0000000000000400
	[90457.034260] Process events/26 (pid: 173, threadinfo ffff8810796a6000, task ffff8810796a4840)
	[90457.034466] Stack:
	[90457.034566]  0000000000000000 ffff8804657dd018 ffff8804657dd000 0000000009a33cd7
	[90457.034691] <0> ffff8804b9cbe100 0000000000000000 ffff8810796a7e20 ffffffff810b6637
	[90457.034913] <0> ffff88045a8b81c0 ffff8804b9cbe140 ffff8804b9cbe0c0 ffff88045a8b81c0
	[90457.035229] Call Trace:
	[90457.035328]  [<ffffffff810b6637>] drain_array+0x77/0xf0
	[90457.035432]  [<ffffffff810b7601>] cache_reap+0x61/0x220
	[90457.035536]  [<ffffffff810b75a0>] ? cache_reap+0x0/0x220
	[90457.035736]  [<ffffffff810567ca>] worker_thread+0x13a/0x210
	[90457.035840]  [<ffffffff81059ce0>] ? autoremove_wake_function+0x0/0x40
	[90457.035948]  [<ffffffff81056690>] ? worker_thread+0x0/0x210
	[90457.036052]  [<ffffffff81059b6e>] kthread+0x8e/0xa0
	[90457.036155]  [<ffffffff8100cb9a>] child_rip+0xa/0x20
	[90457.036257]  [<ffffffff81059ae0>] ? kthread+0x0/0xa0
	[90457.036361]  [<ffffffff8100cb90>] ? child_rip+0x0/0x20
	[90457.036462] Code: 8b 1c 24 48 89 df e8 02 89 f7 ff 48 c1 e8 0c 48 8d 14 c5 00 00 00 00 48 c1 e0 06 48 29 d0 48 ba 00 00 00 00 00 ea ff ff 48 01 d0 <48> 8b 08 66 85 c9 0f 88 c7 00 00 00 84 c9 0f 89 ce 00 00 00 48 
	[90457.037046] RIP  [<ffffffff810b64ce>] free_block+0x8e/0x180
	[90457.037159]  RSP <ffff8810796a7db0>
	[90457.037788] ---[ end trace 7825b6c7db7d9b56 ]---
	
但内核并没有crash，并没有vcore帮助定位问题，所以只能从dmesg中去找蛛丝马迹。可以知道在free_block 中出现了问题，地址为 free_block+0x8e。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_5/disassemble_free_block.png">

也就是上图红色框框中的那一句。现在也就是说%rax寄存器的地址是有问题的。但是为什么有问题就不得而知了。另外，这个是内核的函数，并不是我们内核模块中的函数，这里出问题了说明有内存被踩了，但在哪里踩了，怎么踩的完全是一头雾水！

请教组里大神hyman，他分析后给出了一个建议，因为free_block函数是slab对象回收的实现，而kmalloc也是使用的是slab内存，看看代码是不是kmalloc的那一块内存有问题。搜了一下代码，新加的代码中有两处kmalloc，陈老师建议将两处kmalloc分别改一个地方为static变量，然后在不同的机器上跑跑看。修改完代码后，终于在一台机器上出现了core。利用crash工具，分析core，发现是在tvs_mirror_set_ip 这里面出问题的。通过反汇编对应源代码找出来，是在访问静态变量mirror_ip_array时出现了问题（具体过程和案例一类似，不再赘述）。

	for (i = 0; i < MAX_MIRROR_IP; i++)
		MIRROR_IP_UNUSE(&mirror_ip_array[i]);
		
打印出mirror_ip_array 的地址时发现地址是一系列字符串数值2020202020203020。通过反汇编查看整个模块的静态变量布局，刚好发现mirror_ip_array在block_content后面。而block_content就是刚才从kmalloc转换成static的变量。所以确定了是block_content踩内存，导致后面mirror_ip_array的指针被写成了字符串。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_5/block_content.png">

至此，也就发现了问题的真正原因，是因为使用sprintf函数写字符串，而sprintf函数并没有判断边界，导致会概率性的出现踩内存，覆盖了mirror_ip_array指针（在此建议大家以后不再使用strcpy、sprintf这些不安全的字符串操作函数，都是泪的教训）。

案例二使用crash定位的方法比案例一要简单，主要的难点在于思路，从free_block知道是在释放slab内存时除了问题，而kmalloc真是使用的是slab，从而猜测是kmalloc出来的内存出来的问题。之后就是去尝试使用static变量，让踩了内存后能够出现core信息，这样就能定位出到底哪里除了问题。