系统调用 listen 的第二个参数backlog的含义 
------
我们知道，tcp服务器上接受一个连接有以下步骤：

* 首先调用socket，创建一个新的fd
* 调用bind，绑定到本地地址上
* 调用listen, 将fd由主动套接字变成被动套接字，并可以接受连接
* 调用accept，开始接受连接

今天我们探寻一下listen这个系统调用。
<code>
	int listen(int sockfd, int backlog);
</code>  
listen的第一个参数是fd，这个很好理解，那么第二个参数backlog指的是什么。从man手册上看得的解释是：

	The backlog argument defines the maximum length to which the queue of pending connections for sockfd may grow. 
	
也就是说，backlog指定了pending connections连接队列的大小。pending connections指的是已经完成三次握手，但还没有被accept的连接。  
在内核中，系统为监听的套接字维护两个队列：

* 未完成连接队列（incomplete connection queue，或者叫半连接队列）  
  指的是已经由客户端发出syn请求并到达服务器，并且正在等待服务器完成对应的三次握手的处理。这些连接处于sync_recv状态。
* 已完成连接队列（completed connection queue）  
  指的是已经完成三次握手，但是还没有accept的连接的队列。这些连接处于established状态。
  
未完成连接队列的大小是由 /proc/sys/net/ipv4/tcp\_max\_syn\_backlog 设置。当然这个设置有效的前提是系统的syncookie 功能被禁用，如果syncookie被启动，那么通过proc设置未完成连接队列的大小是无效的（byw，可以通过 cat  /proc/sys/net/ipv4/tcp_syncookies 查看syncookie是否启用）。  
从前面已经知道，已完成连接队列的大小是由 listen的第二个参数backlog决定。但是backlog的大小不能超过/proc/sys/net/core/somaxconn，如果超过，则以somaxconn为准。  
关于这两个队列的统计，可以通过以下命令来查看ListenOverflows（netstat -s也可以查看这个统计）。

	cat /proc/net/netstat | awk '/TcpExt/ {print $21}'

ListenOverflows表示accept队列满了之后，丢弃新建连接的统计。对于未完成连接队列满了之后的丢弃连接的情况，目前没有发现有什么统计。

