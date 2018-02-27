---
layout: post
title: crash打印内核percpu变量值
---

最近又遇到内核crash，最后定位为因为snprintf引起的问题，最后定位出来是因为snprintf 使用不当导致的。

	snprintf(buff, len, host);
	
其中host是域名，存在url编码，比如p%20a%20n.baidu.com，对于存在 % 的字符串，snprintf 这种用法会有问题，导致core，正确的用法

	snprintf(buff, len, "%s", host);
	
最终定位出来，发现错误比较弱智，但是定位的过程还是值得记录。最开始，通过堆栈找到这一行代码有问题，但是不知道host和len的值，不知道为什么会有问题。下面重点记录一下如何找出host值。  

# 一、直接打印法
分析代码，host值是保存在percpu变量中。而percpu变量本身是一个全局变量，每一个cpu使用通过per_cpu_ptr宏来获取percpu变量的指针地址。
使用crash 分析内核core 文件的方法这里不再赘述，具体可以参考之前的博客：http://hsiven.github.io/2016/07/24/linux_crash.html

宏的定义如下：
	
	#define per_cpu_ptr(ptr, cpu)    SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)))

	#define SHIFT_PERCPU_PTR(__p, __offset)    RELOC_HIDE((__p), (__offset))

	#define RELOC_HIDE(ptr, off)                    \
  		({ unsigned long __ptr;                    \
     	__ptr = (unsigned long) (ptr);                \
    	(typeof(ptr)) (__ptr + (off)); })

	#define per_cpu_offset(x) (__per_cpu_offset(x))

通过宏定义，我们可以知道，percpu变量地址的获取可以通过当前CPU编号和全局相对地址，获取绝对地址。
具体如下所示：
	<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_8/blog_8_1.png">   
  
  
# 二、堆栈推导法
首先通过bt命令打印出内核挂掉时的堆栈。
	<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_8/blog_8_2.png"> 
	
这里可以看出来，主要挂掉的堆栈是：
	
	vsnprintf
	snprintf
	send_block_page

其中一些 ixgbe\_msix\_clean\_rings、move\_native\_irq 函数其实是当前堆栈中的一些残留，遍历打印之前的堆栈。当前寄存器的值如下：
	 
	RIP: ffffffff812bc6fd  RSP: ffff880060723540  RFLAGS: 00010206
	RAX: 0000000000000000  RBX: 0000000000000011  RCX: 000000000000006e
	RDX: 0000000000000003  RSI: ffffe8ef9c92e366  RDI: 00000000ffffffff
	RBP: ffff8800607235e0   R8: ffff8800607236fa   R9: ffff8800607235f0
	R10: 0000000000000000  R11: 0000000000000000  R12: ffffe8ef9c92e36a
	R13: ffff8800607236e9  R14: 00000000ffffffff  R15: 0000000000000014
	ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
	
分析send\_block\_page函数和堆栈，挂在第三个snprintf字段，其中send\_block\_page 的第三个参数存储的就是host字段。按照x86寄存器传参的约定，函数前六个参数的依次是RDI, RSI, RDX, RCX, R8和R9。所以host传参在rcx，反汇编该函数，可以知道rcx mov到r12之后，r12就一直保存着host值。r12寄存器在snprintf 函数中没有使用，在vsnprintf 函数中，会在压栈时保存到栈上。所以，我们可以通过rsp寄存器的值，找出压栈之前寄存器r12的值。

	crash> dis send_block_page
	0xffffffffa0026860 <send_block_page>:   push   %rbp
	0xffffffffa0026861 <send_block_page+1>: xor    %eax,%eax
	0xffffffffa0026863 <send_block_page+3>: mov    %rsp,%rbp
	0xffffffffa0026866 <send_block_page+6>: push   %r15
	0xffffffffa0026868 <send_block_page+8>: push   %r14
	0xffffffffa002686a <send_block_page+10>:        mov    %r8d,%r14d
	0xffffffffa002686d <send_block_page+13>:        add    $0x1,%r14d
	0xffffffffa0026871 <send_block_page+17>:        push   %r13
	0xffffffffa0026873 <send_block_page+19>:        lea    -0x240(%rbp),%r13
	0xffffffffa002687a <send_block_page+26>:        push   %r12
	0xffffffffa002687c <send_block_page+28>:        mov    %rcx,%r12
	0xffffffffa002687f <send_block_page+31>:        mov    $0x40,%ecx
	0xffffffffa0026884 <send_block_page+36>:        push   %rbx
	0xffffffffa0026885 <send_block_page+37>:        mov    %rdi,%rbx
	0xffffffffa0026888 <send_block_page+40>:        mov    %r13,%rdi
	0xffffffffa002688b <send_block_page+43>:        sub    $0x268,%rsp
	0xffffffffa0026892 <send_block_page+50>:        mov    %esi,-0x264(%rbp)
	0xffffffffa0026898 <send_block_page+56>:        mov    %edx,-0x268(%rbp)
	0xffffffffa002689e <send_block_page+62>:        rep stos %rax,%es:(%rdi)
	0xffffffffa00268a1 <send_block_page+65>:        mov    $0xffffffffa0040fc0,%rdx
	0xffffffffa00268a8 <send_block_page+72>:        mov    $0x200,%esi
	0xffffffffa00268ad <send_block_page+77>:        mov    %r13,%rdi
	0xffffffffa00268b0 <send_block_page+80>:        callq  0xffffffff812bca60 <snprintf>
	0xffffffffa00268b5 <send_block_page+85>:        mov    $0x200,%ecx
	0xffffffffa00268ba <send_block_page+90>:        movslq %eax,%rdi
	0xffffffffa00268bd <send_block_page+93>:        mov    %eax,%r15d
	0xffffffffa00268c0 <send_block_page+96>:        mov    %ecx,%esi
	0xffffffffa00268c2 <send_block_page+98>:        lea    0x0(%r13,%rdi,1),%rdi
	0xffffffffa00268c7 <send_block_page+103>:       mov    $0xffffffffa003ea80,%rdx
	0xffffffffa00268ce <send_block_page+110>:       sub    %eax,%esi
	0xffffffffa00268d0 <send_block_page+112>:       xor    %eax,%eax
	0xffffffffa00268d2 <send_block_page+114>:       mov    %ecx,-0x270(%rbp)
	0xffffffffa00268d8 <send_block_page+120>:       movslq %esi,%rsi
	0xffffffffa00268db <send_block_page+123>:       callq  0xffffffff812bca60 <snprintf>
	0xffffffffa00268e0 <send_block_page+128>:       lea    (%rax,%r15,1),%r15d
	0xffffffffa00268e4 <send_block_page+132>:       movslq %r14d,%rsi
	0xffffffffa00268e7 <send_block_page+135>:       mov    %r12,%rdx
	0xffffffffa00268ea <send_block_page+138>:       xor    %eax,%eax
	0xffffffffa00268ec <send_block_page+140>:       movslq %r15d,%rdi
	0xffffffffa00268ef <send_block_page+143>:       lea    0x0(%r13,%rdi,1),%rdi
	0xffffffffa00268f4 <send_block_page+148>:       callq  0xffffffff812bca60 <snprintf>
	0xffffffffa00268f9 <send_block_page+153>:       mov    -0x270(%rbp),%ecx
	
	crash> dis snprintf
	0xffffffff812bca60 <snprintf>:  push   %rbp
	0xffffffff812bca61 <snprintf+1>:        mov    %rsp,%rbp
	0xffffffff812bca64 <snprintf+4>:        sub    $0x50,%rsp
	0xffffffff812bca68 <snprintf+8>:        lea    0x10(%rbp),%rax
	0xffffffff812bca6c <snprintf+12>:       mov    %rcx,-0x18(%rbp)
	0xffffffff812bca70 <snprintf+16>:       lea    -0x50(%rbp),%rcx
	0xffffffff812bca74 <snprintf+20>:       mov    %r8,-0x10(%rbp)
	0xffffffff812bca78 <snprintf+24>:       mov    %r9,-0x8(%rbp)
	0xffffffff812bca7c <snprintf+28>:       mov    %rax,-0x48(%rbp)
	0xffffffff812bca80 <snprintf+32>:       lea    -0x30(%rbp),%rax
	0xffffffff812bca84 <snprintf+36>:       movl   $0x18,-0x50(%rbp)
	0xffffffff812bca8b <snprintf+43>:       mov    %rax,-0x40(%rbp)
	0xffffffff812bca8f <snprintf+47>:       callq  0xffffffff812bbff0 <vsnprintf>
	0xffffffff812bca94 <snprintf+52>:       leaveq
	0xffffffff812bca95 <snprintf+53>:       retq   
	0xffffffff812bca96 <snprintf+54>:       nopw   %cs:0x0(%rax,%rax,1)
	
	crash> dis vsnprintf
	0xffffffff812bbff0 <vsnprintf>: push   %rbp
	0xffffffff812bbff1 <vsnprintf+1>:       mov    %rcx,%r9
	0xffffffff812bbff4 <vsnprintf+4>:       mov    %rsp,%rbp
	0xffffffff812bbff7 <vsnprintf+7>:       push   %r15
	0xffffffff812bbff9 <vsnprintf+9>:       push   %r14
	0xffffffff812bbffb <vsnprintf+11>:      push   %r13
	0xffffffff812bbffd <vsnprintf+13>:      push   %r12
	0xffffffff812bbfff <vsnprintf+15>:      push   %rbx
	0xffffffff812bc000 <vsnprintf+16>:      sub    $0x78,%rsp
	0xffffffff812bc004 <vsnprintf+20>:      mov    %rsi,-0x60(%rbp)
	0xffffffff812bc008 <vsnprintf+24>:      mov    -0x60(%rbp),%ecx
	0xffffffff812bc00b <vsnprintf+27>:      mov    %rdi,-0x58(%rbp)
	0xffffffff812bc00f <vsnprintf+31>:      mov    %rdx,%rsi
	0xffffffff812bc012 <vsnprintf+34>:      test   %ecx,%ecx
	0xffffffff812bc014 <vsnprintf+36>:      js     0xffffffff812bc95d <vsnprintf+2413>
	0xffffffff812bc01a <vsnprintf+42>:      mov    -0x58(%rbp),%r8
	0xffffffff812bc01e <vsnprintf+46>:      add    -0x60(%rbp),%r8
	0xffffffff812bc022 <vsnprintf+50>:      jae    0xffffffff812bc036 <vsnprintf+70>
	...
	
所以r12 保存的栈上位置应该，rsp + 0x78 + 0x8.
	<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_8/blog_8_3.png"> 

打印出的域名和第一种方法匹配。

#三、猜测法
上面两种方法都是通过分析得到。如果实在不行，还可以通过猜测法得到。打印出寄存器的值后，将寄存器中看起来像地址的值打印出来看看。

	RIP: ffffffff812bc6fd  RSP: ffff880060723540  RFLAGS: 00010206
	RAX: 0000000000000000  RBX: 0000000000000011  RCX: 000000000000006e
	RDX: 0000000000000003  RSI: ffffe8ef9c92e366  RDI: 00000000ffffffff
	RBP: ffff8800607235e0   R8: ffff8800607236fa   R9: ffff8800607235f0
	R10: 0000000000000000  R11: 0000000000000000  R12: ffffe8ef9c92e36a
	R13: ffff8800607236e9  R14: 00000000ffffffff  R15: 0000000000000014
	ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018

当打印r12中的地址时，发现有一点像域名，然后向前打印一些就可以得到我们想要看到的域名信息。
	<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_8/blog_8_4.png"> 
	
#总结
总体而言，还是需要对于堆栈、汇编、内核更加深入了解，才能在定位问题时得心应手。csapp有时间还是需要好好的复习一下相关章节。