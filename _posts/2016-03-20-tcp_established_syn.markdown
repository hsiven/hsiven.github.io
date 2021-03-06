---
layout: post
title: Established 状态的连接对于syn包的处理
---

前几天和同事一起项目评审时，碰到一个问题：就是在Estalished状态的连接如果收到syn包会如何处理？当时不是特别清楚，觉得可能是重新建立新的连接，或者回ack包，或者回reset。那么，在linux 协议栈中，如果处于Estabished状态的连接收到了syn包，到底会怎么处理呢？让我们从内核协议栈代码中来探寻这个答案。  

首先，说一下TCP层收到包的处理流程（这里不讨论各个函数的细节处理逻辑，具体可查代码，就简单的说一下代码处理路径）。从IP层的收数据包通过tcp_v4_rcv达到TCP层.tcp_v4_rcv中首先查找socket，判断socket当前的状态，如果是TCP_TIME_WAIT，单独处理，这里暂时不讨论，对于其他正常的数据包，调用tcp_v4_do_rcv处理。在tcp_v4_do_rcv函数中，判断状态为TCP_ESTABLISHED,则调用tcp_rcv_established处理这个数据包。  

tcp_rcv_established 中对于数据包的处理分为两类：fast_path 和 solw_path.正常情况下，数据包按序到达，都可以按照fast path直接将数据包放到receive queue中。进入fast path工作的条件包括：

 * 没有乱序数据包
 * 接受窗口不为0
 * 还有接受缓存空间
 * 没有紧急数据
 
在slow_path处理中，首先调用 tcp_validate_incoming 验证数据包的有效性。在tcp_validate_incoming 中检查seq no，如果是乱序的数据包，并且不是rst包，则会重新发送ack数据包：  

	/* Step 1: check sequence number */
	if (!tcp_sequence(tp, TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq)) {
		/* RFC793, page 37: "In all states except SYN-SENT, all reset
		 * (RST) segments are validated by checking their SEQ-fields."
		 * And page 69: "If an incoming segment is not acceptable,
		 * an acknowledgment should be sent in reply (unless the RST
		 * bit is set, if so drop the segment and return)".
		 */
		if (!th->rst)
			tcp_send_dupack(sk, skb);
		goto discard;
	}


tcp_sequence函数检查seq是否在合理的窗口范围之内：  
	

	static inline int tcp_sequence(struct tcp_sock *tp, u32 seq, u32 end_seq)
	{
		return	!before(end_seq, tp->rcv_wup) &&
			!after(seq, tp->rcv_nxt + tcp_receive_window(tp));
	}

  
  rcv\_nxt和rcv\_wup两个变量在tcp.h中定义，分别表示期望接收的下一个段、上一个已经确认的段。


第二步检查rst数据包，第三步检查安全性，目前没有实现，第四步检查syn包。如果不是乱序的syn包，则表示这个连接出错了，所以会reset这个连接。  	  
	
	/* step 4: Check for a SYN in window. */
	if (th->syn && !before(TCP_SKB_CB(skb)->seq, tp->rcv_nxt)) {
		if (syn_inerr)
			TCP_INC_STATS_BH(sock_net(sk), TCP_MIB_INERRS);
		NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_TCPABORTONSYN);
		tcp_reset(sk);
		return -1;
	}

所以，对于开头说的问题，处于ESTABISHED状态的tcp连接，如果收到syn包，会有两种情况：

 * 如果这个syn包的seq是乱序的，则会回ack包
 * 如果这个syn包的seq在合法范围内，则会reset这个连接。


