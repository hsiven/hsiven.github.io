reahat 企业版性能调优网络部分
-----
这篇文章是翻译自https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/main-network.html，主要是网络部分调优的介绍，和目前的工作关系很大，所以仔细看了一下，并翻译第八章的网络性能提升部分的摘要，以备后续查阅。


前言

随着时间的推移，Red Hat企业版的网络协议栈已经升级了很多优化特性。对于大部分的情况，使用系统自动配置的网络设置就能提供较好的性能。  
大部分情况下，网络性能问题一般都是硬件故障或者是基础设施故障。这些因素都超出了本篇文章的讨论范围。本片文章讨论的性能问题和解决方案对于优化网络系统非常有帮助。  
网络系统是操作系统中非常精巧的子系统。这也是开源社区和Red Hat投入大量的工作来实现自动优化网络性能的原因。因此，对于大部分情况，你不需要为了性能重新配置网络系统。

1. 提升网络性能的方法

Red Hat企业版6.1提供了以下几种提高网络性能的方法：

* Receive Packet Steering(RPS)

RPS 的功能是让网卡的一个rx queue的软终端可以分布其他CPU上。这种方法可以帮助解决网卡的一个硬件队列成为瓶颈而性能上不去的情况。如果想开启RPS，可以修改/sys/class/net/ethX/queues/rx-N/rps_cpus 的值（如ffff）。注意，在设置cpu的时候，要注意缓存有效性和numa特性。

* Receive Flow Steering

* getsockopt support for tcp thin-streams

* Transparent Proxy Support


2. 优化网络设置

诊断网络性能问题的一些工具：
	netstat
	dropwatch
	ip
	ethtool
	
Socket receive buffer size

SO_REVBUF

8.3 Overview of Packet Reception

	

	