---
layout: post
title: RCU 实现原理 
---

最近遇到了一个场景，读多写少，而且对于读这一端的性能要求极高，读写锁也没法满足要求，所以研究了一下RCU（Read-Copy-Update）。RCU中读端几乎没有性能损耗，更新端的性能稍微差一些。对于RCU的具体实现，还没有全部、彻底弄清楚，今天尝试介绍一下RCU，下面是文章的 http://lwn.net/Articles/305782/ 的一个摘要翻译吧。  
RCU2002年加入到内核2.6版本中，经历过两个版本，第一个是经典版本--classic rcu，第二个是使用tree实现的tree RCU。下面简单的介绍一下这两种实现思想（具体的实现分析后续再分析）。  

一、 经典RCU的实现  
经典的RCU实现中有一个关键的概念，就是读(read_rcu)临界区（critial section）受限于内核代码，并且不允许阻塞（block）。这也就意味着，任何时候对于一个特定的cpu，如果处于阻塞状态（blocking），或者退出了内核态，那么我们就知道RCU的read-side的临界区已经完成(或者对于read_rcu_bh，在临界区不允许抢占，如果cpu处于可抢占的状态，则表示这个cpu的读端临界区已经完成)。这种状态称之为“quietcent state”（静态？好吧，我也不知道该怎么翻译）。在每一个CPU都至少经历过一个quietcent state，RCU的grace period（好吧，这个我也不知道该怎么翻译，竞争阶段？）结束了。
  
经典的RCU最重要的数据结构是 rcu_ctrlblk，这个结构中有一个cpumask域，每个cpu都有一个bit。每一个CPU在grace period的开始时设置为1，当经过过quietcent state之后就需要将bit clear。因为多个CPU可能需要并发的clear bit，当cpu个数比较多时，性能就会下降。另外，经典的RCU会唤醒正在睡眠的cpu（因为每个CPU都需要clear bit），这点限制了linux的省电能力。  


二、分级RCU的实现  
为了解决经典的RCU竞争的问题，分级RCU采用了下图的结构。
每一个rcu_node 有自己的lock，因此cpu0和cpu1竞争同一个锁，同理，cpu2和cpu3竞争一个锁。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_3/blog_3_pic_1.jpg">

下面的序列图解释了如何探测grace periods。在第一幅图中，如红色方块所示，没有CPU经历过quietstate。假设所有六个cpu同时尝试RCU都经历过quietstate，那么每一组都只有一个cpu获取到锁，如第二幅图所示，假设这三个幸运的cpu分别是0，3，5（图中用绿色的方块显示）。一旦这些cpus结束了，那么其他的cpus就是获取锁。这些cpus发现他们是这组cpu当中的最后一个，那么就会尝试获取更上一层rcu_node的锁。第四、五、六依次显示cpu1、cpu2、cpu4获取更上一层的锁。图六显示了所有的cpu都经历了quietstate，所以这一次的grace period结束了。

<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_3/blog_3_pic_2.jpg">

在上面的图中，从没有超过三个CPUs竞争一个锁，相对于经典的RCU实现，可能six CPUs同时竞争锁。除此之外，分级RCU减少了唤醒空闲cpu，节省了电能消耗。




