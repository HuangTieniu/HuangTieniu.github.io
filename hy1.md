Overview

TCP/IP协议栈满足不了现代IDC工作负载(workloads)的需求，主要有2个原因：(1)内核处理收发包需要消耗大量的CPU；(2)TCP不能满足应用对低延迟的需求：一方面，内核协议栈会带来数十ms的延迟；另一方面，TCP的拥塞控制算法、超时重传机制都会增加延迟。

RDMA在NIC内部实现传输协议，所以没有第一个问题；同时，通过zero-copy、kernel bypass避免了内核层面的延迟。

与TCP不同的是，RDMA需要一个无损(lossless)的网络。例如，交换机不能因为缓冲区溢出而丢包。为此，RoCE使用PFC(Priority-based Flow Control)带进行流控。一旦交换机的port的接收队列超过一定阀值(shreshold)时，就会向对端发送PFC pause frame，通知发送端停止继续发包。一旦接收队列低于另一个阀值时，就会发送一个pause with zero duration，通知发送端恢复发包。

PFC对数据流进行分类(class)，不同种类的数据流设置不同的优先级。比如将RoCE的数据流和TCP/IP等其它数据流设置不同的优先级。详细参考Considerations for Global Pause, PFC and QoS with Mellanox Switches and Adapters
Network Flow Classification

对于IP/Ethernet，有2种方式对网络流量分类：

    By using PCP bits on the VLAN header
    By using DSCP bits on the IP header

详细介绍参考Understanding QoS Configuration for RoCE。
Traffic Control Mechanisms

对于RoCE，有2个机制用于流控：Flow Control (PFC)和Congestion Control (DCQCN)，这两个机制可以同时，也可以分开工作。

    Flow Control (PFC)

PFC是一个链路层协议，只能针对port进行流控，粒度较粗。一旦发生拥塞，会导致整个端口停止pause。这是不合理的，参考Understanding RoCEv2 Congestion Management。为此，RoCE引入Congestion Control。

    Congestion Control (DCQCN)

DC-QCN是RoCE使用的拥塞控制协议，它基于Explicit Congestion Notification (ECN)。后面会详细介绍。
PFC

前面介绍有2种方式对网络流量进行分类，所以，PFC也有2种实现。
VLAN-based PFC

    VLAN tag

基于VLAN tag的Priority code point (PCP，3-bits)定义了8个Priority.

    VLAN-based PFC

In case of L2 network, PFC uses the priority bits within the VLAN tag (IEEE 802.1p) to differentiate up to eight types of flows that can be subject to flow control (each one independently).

    RoCE with VLAN-based PFC

HowTo Run RoCE and TCP over L2 Enabled with PFC.

## 将skb prio 0~7 映射到vlan prio 3
for i in {0..7}; do ip link set dev eth1.100 type vlan egress-qos-map $i:3 ; done

## enable PFC on TC3
mlnx_qos -i eth1 -f 0,0,0,1,0,0,0,0

例如：

[root@node1 ~]# cat /proc/net/vlan/eth1.100 
eth1.100  VID: 100       REORDER_HDR: 1  dev->priv_flags: 1001
         total frames received            0
          total bytes received            0
      Broadcast/Multicast Rcvd            0

      total frames transmitted            0
       total bytes transmitted            0
Device: eth1
INGRESS priority mappings: 0:0  1:0  2:0  3:0  4:0  5:0  6:0 7:0
 EGRESS priority mappings: 
[root@node1 ~]# for i in {0..7}; do ip link set dev eth1.100 type vlan egress-qos-map $i:3 ; done
[root@node1 ~]# cat /proc/net/vlan/eth1.100                                                      
eth1.100  VID: 100       REORDER_HDR: 1  dev->priv_flags: 1001
         total frames received            0
          total bytes received            0
      Broadcast/Multicast Rcvd            0

      total frames transmitted            0
       total bytes transmitted            0
Device: eth1
INGRESS priority mappings: 0:0  1:0  2:0  3:0  4:0  5:0  6:0 7:0
 EGRESS priority mappings: 0:3 1:3 2:3 3:3 4:3 5:3 6:3 7:3 

参考HowTo Set Egress Priority VLAN on Linux.

    问题

基于VLAN的PFC机制有2个主要问题：(1)交换机需要工作在trunk模式；(2)没有标准的方式实现VLAN PCP跨L3网络传输(VLAN是一个L2协议)。

DSCP-based PFC通过使用IP头部的DSCP字段解决了上面2个问题。
DSCP-based PFC

DSCP-based PFC requires both NICs and switches to classify and queue packets based on the DSCP value instead of the VLAN tag.

    DSCP vs TOS

The type of service (ToS) field in the IPv4 header has had various purposes over the years, and has been defined in different ways by five RFCs.[1] The modern redefinition of the ToS field is a six-bit Differentiated Services Code Point (DSCP) field[2] and a two-bit Explicit Congestion Notification (ECN) field.[3] While Differentiated Services is somewhat backwards compatible with ToS, ECN is not.

详细介绍参考：

    Type of service
    Differentiated services

PFC机制的一些问题

RDMA的PFC机制可能会导致一些问题：

    RDMA transport livelock

尽管PFC可以避免buffer overflow导致的丢包，但是，其它一些原因，比如FCS错误，也可能导致网络丢包。RDMA的go-back-0算法，每次出现丢包，都会导致整个message的所有packet都会重传，从而导致livelock。TCP有SACK算法，由于RDMA传输层在NIC实现，受限于硬件资源，NIC很难实现SACK算法。可以使用go-back-N算法来避免这个问题。

    PFC Deadlock

当PFC机制与Ethernet的广播机制工作时，可能导致出现PFC Deadlock。简单来说，就是PFC机制会导致相应的port停止发包，而Ethernet的广播包可能引起新的PFC pause依赖（比如port对端的server down掉)，从而引起循环依赖。广播和多播对于loseless是非常危险的，建议不要将其归于loseless classes。

    NIC PFC pause frame storm

由于PFC pause是传递的，所以很容器引起pause frame storm。比如，NIC因为bug导致接收缓冲区填满，NIC会一直对外发送pause frame。需要在NIC端和交换机端使用watchdog机制来防止pause storm。

    The Slow-receiver symptom

由于NIC的资源有限，它将大部分数据结构，比如QPC(Queue Pair Context) 和WQE (Work Queue Element)都放在host memory。而NIC只会缓存部分数据对象，一旦出现cache miss，NIC的处理速度就会下降。
