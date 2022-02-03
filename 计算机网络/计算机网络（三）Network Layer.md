# 计算机网络（三）Network Layer

网络层负责将上层交付的segment（比如TCP segment或者UDP datagram）传输到指定的host，网络层使用的是IP地址。负责的是host到host的数据传送。在这一层主要负责的协议是IP Protocol

## IP datagram格式

![image-20220125113922616](C:\Users\Enjiang He\AppData\Roaming\Typora\typora-user-images\image-20220125113922616.png)

### 字段含义

•    4-bit version:

​	◦    The IP version, 4 in this case

•    4-bit header length:

​	◦    The number of 32-bit words (4-bytes) in the header including any options.

​	◦    15 is the biggest number for this

​	◦    If no options we would have 5

​	◦    That leaves 40 bytes in total for options.

•    8-bit type of service (TOS): TOS short for Type of Service

​	◦    Minimize delay, maximize throughput, maximize reliability, minimize monetary cost. Only one can be turned on. All off means normal service.

​	◦    Different applications may set the TOS differently.

​	◦    Routers may use it to make routing decisions. Or they may ignore it.

•    16-bit total length:

​	◦    Size of IP datagram in bytes.

​	◦    Max size of IP datagram: 2**16 – 1 = 65535 bytes

•    16-bit identification:

​	◦    Normally increments by 1 for each datagram sent.

​	◦    Plays role in fragmentation and reassembly.

•    3-bit flags:

​	◦    Support fragments bits, DF do not fragment bit, MF more fragments bit

•    13-bit fragmentation offset:

​	◦    Used to support fragmentation. Tells us where this fragment goes in the overall IP datagram

•    8-bit time-to-live TTL:

​	◦    Every time an IP datagram hits a router, the router decrements this value.

​	◦    If it hits zero the packet is dropped and an ICMP TTL, exceeded message is returned to the sender

​	◦    Prevent packets wandering the Internet forever.

•    8-bit protocol:

​	◦    The protocol being transported TCP, UDP, ICMP

•    16-bit header checksum:

​	◦    Calculated over the header. If invalid datagram is quietly discarded.

•    32-bit source IP address: Self explanatory.

•    32-bit destination IP address: Self explanatory.

•    Options:

​	◦    Ask routers to record their IP in the datagram, ask them to record IP and timestamp, etc.

​	◦    Records are added to the options field.

•    Data:

•    The payload. Some encapsulated TCP segment or UDP datagram

当网络层收到上次的数据包后，会组装好IP datagram，然后此时根据网卡的routing table决定将这个IP datagram交给router或者是本地子网中的其他host；如果是交给router，那么就是发送到外网的数据，所以此时router中会有一张routing table来表明应该从哪一个接口发出（这张表的是由路由算法来维护更新的，通常是分级维护的，也就是同一个ISP下的所有router都有一张表，同一个ISP网络中必须使用同样的路由算法；而AS之间又是另一个层级），就这样经过一个个router到达目的主机的router，然后收取IP datagram再然后转发给目的主机。目的主机收到后会检查，组装，交付给上层。

IP fragment

因为IP的datagram的最大长度为：65535 ，而这个size在交付下层datalink层传输的时候是超过了MTU的，所以需要对超过下层传输协议MTU的IP datagram进行segment。

What does the **traceroute** program do? How does it work?

•    Traces the routers passed through by a packet along its route

•    Works by seeing TTL to 1, then 2, then 3, etc. Assumes route not changing from one ping to the next

 

6. What is ICMP used for? In which layer of the TCP/IP model does ICMP live?

•    ICMP (Internet Control Message Protocol) is used to report network diagnostics.

•    Network layer. IP helper

 

7. How does an ICMP smurf attack work?

•    Sending ICMP ping to a network broadcast address from a spoofed source IP

•    All response to the victim who never requested anything

•    Imbalance in size of request and response taken advantage of in other attacks. DNS, NTP

 

8. When does IP fragmentation arise?

•    Physical layer imposes upper limit on the size of the frame that can be transmitted.

•    IP layer queries an interface to obtain its MTU.

•    The network layer must be able to rebuild an IP datagram whose size exceeds the MTU.

•    IP layer fragments datagrams whose size exceeds MTU and labels them for later reassembly.

•    More likely with UDP than TCP queries local MTU during connection establishment and takes it into account during its conversion

 

9. Which IP header fields are involved in supporting IP fragmentation?

•    The same identification field is copied into each fragment.

•    The more fragments bit is enabled in all fragments except for the last one.

•    The fragment offset field contains offset in 8-byte units of this fragment from beginning of datagram.

 

10. Fragmentation is bad news. Why?

•    Extra work for routers along the way

•    Extra work for recipient who has to rebuild the datagram. If one fragment is lost they’re all lost and must all be resent.

•    Can hide attacks that we disperse across multiple fragments. **Scapy**

 

11. What is path MTU discovery?

•    Use the DF flag to find the path MTU. If fragmentation required the packet of size X is dropped and we get an ICMP error message. We know MTU < X, iterate to find MTU by varying X.

 

12. When does a header-based vulnerability arise?

•    When an implementation of a protocol fails to deal securely with a protocol header created in violation of the standard e.g. specifying an illegal packet size.

•    If a standard does not specify how to deal with certain situations (enabling all flags when doing so does not make sense) the way they are dealt with may reveal the underlying operating system.

•    Fuzzing can bring such issues to light.

 

13. Describe one real-world header-based vulnerability and attack.

•    The “ping of death”. 16 bits available to specify IP datagram size giving a max of 65,535 bytes.

•    Implementations may assume the reassembled datagram must fit in 65,535 bytes.

•    However by specifying suitably large fragment offset and fragment size values it is possible to describe a reassembled datagram larger than the maximum allowed.

•    Some implementations crashed when the reassembly buffer overflowed.

•    Despite name “ping of death” the vulnerability not with the ICMP protocol but an IP issue.

16. How does a router map an IP address to a corresponding network path?

•    Longest prefix matching.

•    Table says packets to 136.206.0.0/16 should go one route and packets to 136.206.115.0/24 another.

•    Packet to 136.206.115.23 arrives. Which path should it take? The most specific match is chosen.