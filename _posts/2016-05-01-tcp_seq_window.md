---
layout: post
title: tcp连接中的序列号和接受窗口大小浅析 
---

最近在工作遇到了很多和tcp头部的序列号、窗口大小的问题，现在来梳理一下。  
首先看一下tcp的头部。在linux中的tcp头部定义(little-endian)：

		struct tcphdr {
		    __be16	 source;
		    __be16	 dest;
		    __be32	 seq;
		    __be32	 ack_seq;
		    __u16	 res1:4,
		    doff:4,
		    fin:1,
		    syn:1,
		    rst:1,
		    psh:1,
		    ack:1,
		    urg:1,
		    ece:1,
		    cwr:1;
		    __be16 window;
		    __sum16 check;
		    __be16 urg_ptr;
		};
		
这篇文章中关注的是seq、ack_seq、window这三个字段。seq是当前发送的tcp报文中第一个字节数据的序列号。如果tcp头部中中的ack被置为1，那个ack_seq代表的是当前接受方希望接受的下个数据的序列号。假设三次握手中，第一个syn包中的序列号为1，可以通过下面这张图片来看一下seq、ack_seq的变化。


<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_4/tcp_flow_seq.png">



接受方需要对tcp报文中的seq进行合法性校验。假设当前tcp报文的序列号为seq，报文大小为 seg_len,那么seq和seg_len需要满足以下两个条件之一：

* 	seq 大于等于接收方希望收到的下一个报文并且seq小于接受方希望收到的下一个报文加上当前接受方的窗口大小。
*  或者(seq + seg_len - 1)满足上述要求，在接收方希望收到的下一个报文和希望收到的下一个报文加上窗口大小的中间。其中 （seq + seg_len - 1） 表示的是这一次报文的最后一字节数据的序列号。

在seastar的tcp协议栈中实现如下：

	template <typename InetTraits>
	bool tcp<InetTraits>::tcb::segment_acceptable(tcp_seq seg_seq, unsigned seg_len) {
        if (seg_len == 0 && _rcv.window == 0) {
     	     // SEG.SEQ = RCV.NXT
     	     return seg_seq == _rcv.next;
        } else if (seg_len == 0 && _rcv.window > 0) {
            // RCV.NXT =< SEG.SEQ < RCV.NXT+RCV.WND
           return (_rcv.next <= seg_seq) && (seg_seq < _rcv.next + _rcv.window);
        } else if (seg_len > 0 && _rcv.window > 0) {
     	     // RCV.NXT =< SEG.SEQ < RCV.NXT+RCV.WND
      	     //    or
      	     // RCV.NXT =< SEG.SEQ+SEG.LEN-1 < RCV.NXT+RCV.WND
           bool x = (_rcv.next <= seg_seq) && seg_seq < (_rcv.next + _rcv.window);
      	    bool y = (_rcv.next <= seg_seq + seg_len - 1) && (seg_seq + seg_len - 1 < _rcv.next + _rcv.window);
      	    return x || y;
        } else  {
     	     // SEG.LEN > 0 RCV.WND = 0, not acceptable
     	     return false;
   	     }
    }

  tcp头部中中有一个16位的接受窗口字段，这个字段通告发送方目前接受方的接受窗口大小。因为16位数最大为65535，所以通知的window最大为65535（64k）.为了处理窗口大于65535的情况，tcp还设计了Window Scale Option的tcp选项。tcp window scale 选项有三个字节，格式如下：
    

             +---------+---------+---------+

             | Kind=3  |Length=3 |shift.cnt|

             +---------+---------+---------+
  
tcp window scale option只能在syn包中和syn+ack的回包中使用，并且只有syn中带有window scale选项，syn + ack才会带这个选项。window scale选项在其他的数据包中将会被忽略。tcp连接的一方如果准备使用window scale，就应该在syn包中发送这个选项，即使scale factor是1。scale factor应该是2的次幂，以对数存储发送，发送方和接收方通过移位运算来计算窗口大小。tcp window scale是可选选项。如果window scale开启，那么tcp发送一方将会将本身的接受窗口右移shift.cnt位，然后作为接受窗口大小发送给另一方。但是，值得注意的是，在syn包和syn+ack数据包中，window的大小是最原始的大小，没有右移。只有在后续的数据包中才会右移。所以想要追踪两端都支持window scale的tcp连接，需要将window大小左移scale factor。另外，scale factor可以在连接的握手包（syn包和syn + ack）中看到。另外，tcp连接的两端窗口的scale factor可以不同。

linux内核中调用tcp_select_initial_window来初始化window大小。初始化rcv_wscale的方法是：
	
	if (wscale_ok) {
	    /* Set window scaling on max possible window
	    * See RFC1323 for an explanation of the limit to 14
	    */
	    space = max_t(u32, sysctl_tcp_rmem[2], sysctl_rmem_max);
	    space = min_t(u32, space, *window_clamp);
	    while (space > 65535 && (*rcv_wscale) < 14) {
		    space >>= 1;
		    (*rcv_wscale)++;
	    }
    }

接收窗口取最大值为max(tcp_rmem[2], rmem_max)，本连接接收窗口的最大值为为min(max(tcp_rmem[2], rmem_max), window_clamp)。
那么我们需要多大的窗口扩大因子，才能用16位来表示最大的接收窗口呢？
如果接收窗口的最大值受限于tcp_rmem[2] = 4194304，那么rcv_wscale = 7，窗口扩大倍数为128。  
接受窗口的大小在三次握手后确定，但在tcp连接传送数据的过程中会有相关的调整。每次发送一个TCP数据段，都要构建TCP首部，这时会调用tcp_select_window选择接收窗口大小。
窗口大小选择的基本算法：

* 计算当前接收窗口的剩余大小cur_win。
*  计算新的接收窗口大小new_win，这个值为剩余接收缓存的3/4，且不能超过rcv_ssthresh（表示接受缓存的当前阈值）
* 取cur_win和new_win中值较大者作为接收窗口大小。

计算cur_win的方法：

	/* Compute the actual receive window we are currently advertising.
 	* Rcv_nxt can be after the window if our peer push more data
 	* than the offered window.
 	*/
	static inline u32 tcp_receive_window(const struct tcp_sock *tp)
	{
	    s32 win = tp->rcv_wup + tp->rcv_wnd - tp->rcv_nxt;

	    if (win < 0)
		    win = 0;
	    return (u32) win;
	}

rcv_wup 表示上一次发送更新接受窗口大小时的rcv_nxt,其实也就是当前接受窗口的最左端。rcv_wnd表示当前接受窗口的大小，rcv_nxt表示当前期望接受到的下一个序列号。
计算新的窗口大小调用__tcp_select_window（代码比较长，就不贴了，可以看源代码）。  

最后,看一下关于和接受窗口大小相关的系统参数，包括

* tcp_moderate_rcvbuf  
是否自动调节TCP接收缓冲区的大小，默认值为1。
* tcp_adv_win_scale  
在tcp_moderate_rcvbuf启用的情况下，用来对计算接收缓冲区和接收窗口的参数进行微调，默认值为2。2表示应用缓冲区是tcp_rmem指定的1/4.
* tcp_rmem  
包括三个参数：min default max。
tcp_rmem[1] — default ：接收缓冲区长度的初始值，用来初始化sock的sk_rcvbuf，默认为87380字节。
tcp_rmem[2] — max：接收缓冲区长度的最大值，用来调整sock的sk_rcvbuf，默认为4194304，一般是2000多个数据包。 

