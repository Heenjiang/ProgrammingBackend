# DNS 域名解析服务

## 人类不善于记忆IP地址

当我们在浏览器的输入框中敲入一串由 “ . ” 隔开的字符串域名后，点击回车键，浏览器就返回了我们想要访问的网页。对于用户来说，浏览器背后的工作是一个黑箱。但是我们不妨深入想一下，在Transport layer我们通过端口加IP地址来达成 进程之间的通信，在network layer我们通过IP地址来达到host到host的通信。

浏览器返回网页的过程中肯定是与远程某台server通信，请求了一个网页文件，那么他们是靠什么标识（地址）来通信的呢？

答案就在DNS中，我们知道IP用于表示网络中不同的主机（网络号和主机号），但是我们要是想访问某台服务器，记住它的IP显然是一件困难的事情。人类天生不擅长记忆没有意义的内容，所以人们想到，将我们的IP地址转化成好记忆的一串字符串（用点分开），然后最后在通讯的层次上本质还是由IP来负责，于是DNS横空出世负责解析域名（正向解析）。

## 域名的格式

域名其实也是像IP地址一样有着一定的层次结构的，从左到右依次是根域名（.），顶级域名，二级域名，三级域名

![img](https://segmentfault.com/img/remote/1460000039039278)

其实这里的三级域名可以算是二级域名的二级目录，每一级的域名都由相应的域名服务器存储着记录。

域名服务器是指管理域名的主机和相应的软件，它可以管理所在分层的域的相关信息。一个域名服务器所负责管里的分层叫作 **区 (ZONE)**。域名的每层都设有一个域名服务器：

- 根域名服务器
- 顶级域名服务器
- 权限域名服务器

下面这幅图就很直观了：

![img](https://segmentfault.com/img/remote/1460000039039289)

## DNS message的格式

![image-20220207185930249.png](https://github.com/Heenjiang/ProgrammingBackend/blob/main/pic/image-20220207185930249.png?raw=true)

•    One DNS message format for both DNS queries and responses

•    12 bytes of header fallowed by 4 variable length sections

•    Identification: 16-bit numbers for matching response with earlier requests

•    Flags:

1. ◦    QR (1-bit): 0: DNS query, 1: DNS response
2. ◦    Opcode (4-bits): 0-standard lookup, 1-inverse look up, 2-sever status request
3. ◦    AA (1-bit): authoritative answer (answer come from authoritative name sever requested domain)
4. ◦    TC (1-bit): truncated or not
5. ◦    RD (1-bit): recursion desired
6. ◦    RA (1-bit): recursion available
7. ◦    Zero (3 bits): zeros
8. ◦    Rcode (4-bits): 0-no error, 3-name error (nxdomain)

•    Number of questions: Number of queries included in Questions section.

•    Number of answer RRs: Number of resource records in the Answers section.

•    Number of authority RRs: Number of resource records in the Authority section.

•    Number of additional RRs: Number of resource records in the Additional section.

**What does DNS do? In which network layer do we find DNS?**

•    DNS maps from host names to IP address. It’s an application layer protocol UDP typically in the transport layer

**What does a DNS server store?**

•    Resource records. It stores those for the zone for which it is authoritative

## DNS 查询方式

具体 DNS 查询的方式有两种：

- 递归查询
- 迭代查询

所谓迭代就是，如果请求的接收者不知道所请求的内容，那么**接收者将扮演请求者**，发出有关请求，直到获得所需要的内容，然后将内容返回给最初的请求者。

👍 通俗点来说，在递归查询中，如果 A 请求 B，那么 B 作为请求的接收者一定要给 A 想要的答案；而迭代查询则是指，如果接收者 B 没有请求者 A 所需要的准确内容，接收者 B 将告诉请求者 A，如何去获得这个内容，但是自己并不去发出请求。

一般来说，**域名服务器之间的查询使用迭代查询方式，以免根域名服务器的压力过大**。通过下面这两个图就能很好的理解了 👇

1）递归查询：

![img](https://segmentfault.com/img/remote/1460000039039283)

2）迭代查询：

![img](https://segmentfault.com/img/remote/1460000039039290)

## 完整域名解析过程

OK，将我们上面所说的域名服务器之间的 DNS 查询请求过程和域名缓存结合起来，就是一个完整的 DNS 协议进行域名解析的过程。这里我们以正向解析为例（域名解析成 IP 地址）：

1）首先搜索**浏览器的 DNS 缓存**，缓存中维护一张域名与 IP 地址的对应表；

2）若没有命中，则继续搜索**操作系统的 DNS 缓存**；

3）若仍然没有命中，则操作系统将域名发送至**本地域名服务器**，本地域名服务器查询自己的 DNS 缓存，查找成功则返回结果（注意：主机和本地域名服务器之间的查询方式是**递归查询**）；

4）若本地域名服务器的 DNS 缓存没有命中，则本地域名服务器向上级域名服务器进行查询，通过以下方式进行**迭代查询**（注意：本地域名服务器和其他域名服务器之间的查询方式是迭代查询，防止根域名服务器压力过大）：

- 首先本地域名服务器向**根域名服务器**发起请求，根域名服务器是最高层次的，它并不会直接指明这个域名对应的 IP 地址，而是返回顶级域名服务器的地址，也就是说给本地域名服务器指明一条道路，让他去这里寻找答案
- 本地域名服务器拿到这个**顶级域名服务器**的地址后，就向其发起请求，获取**权限域名服务器**的地址
- 本地域名服务器根据权限域名服务器的地址向其发起请求，最终得到该域名对应的 IP 地址

4）本地域名服务器将得到的 IP 地址返回给操作系统，同时自己将 IP 地址缓存起来

5）操作系统将 IP 地址返回给浏览器，同时自己也将 IP 地址缓存起来

6）至此，浏览器就得到了域名对应的 IP 地址，并将 IP 地址缓存起来

配合下图直观理解：

![img](https://segmentfault.com/img/remote/1460000039039286)

## **DNS中的安全问题**

**What is DNS cache poisoning?**

•    Before processing a query a DNS server consults its cache for an answer.

•    Poisoning is tricking a DNS server into storing falsified DNS records.

•    If successful makes a phishing attack easy. 

**If DNS queries had no associated identification numbers, how might you poison a DNS server?**

•    Victim queries local DNS server for www.bankofireland.ie.

•    Attacker spoofs response. Falsified DNS record maps to IP address of www.evilbank.com.

•    Note the spoofed response must arrive before the real one.

•    If the real answer arrives before the spoofed one the attack is halted.

•    See diagram

 

**If DNS queries had incrementing identification numbers, how might you poison a DNS server?**

•    The identification number is a 16-bit number. 65K possibilities.

•    Until 2002 DNS queries used sequential IDs.

•    Alice registers her own DNS server for the evilbank.com domain.

•    She queries the target/victim DNS server for www.evilbank.com.

•    She notes the sequence number when the query arrives.

•    She queries the target DNS server for www.bankofireland.ie. Its query will have next sequence no.

•    She sends to the target DNS server spoofed replies from ns.bankofireland.ie in the right range.

•    The spoofed replies contain the predicted sequence number and one matches.

•    See digram

 

 **Given random DNS query identification numbers, how did Kaminsky poison a DNS server?**

•    After 2002 random IDs were introduced but did not completely solve the problem.

•    If the response to the www.bankofireland.ie query arrives before cache poisoned attack is halted.

•    Random query IDs mean specific targeting of www.bankofireland.ie is impossible.

•    For the attack to work attacker must cause the target DNS server to issue lots of DNS queries.

•    Then a spoofed response must only match one of the many DNS queries issued by the victim.

•    The technique employed is called subdomain DNS cache poisoning.

•    Attacker issues lots of requests for non-existent machines in the target domain.

•    Attacker spoofs responses containing (an empty answer section and) a falsified authority section.

•    This effectively says “I don't know but this is the nameserver to ask”.

•    These are “glue” records where the authority/additional sections give IP address of NS.

•    If we get a hit this bogus NS is cached by the target.

•    It takes approximately 10s to get a hit.

•    All subsequent queries to bankofireland.ie are forwarded to the attacker's DNS server.

•    So query for www.bankofireland.ie comes to the attacker’s DNS who returns bogus mapping.

•    The fix is to randomize both the source port and the ID number.

•    See diagram

 

**What is split-horizon DNS?**

•    We employ two DNS servers.

•    One is a recursive server and handles DNS requests from machines on the internal network.

•    Another is non-recursive and handles requests from machines external to the network (providing RRs only for publicly accessible machines on the company network).

•    Otherwise (like in DCU) reverse look-ups to get internal host names are possible.

 

**What is “reverse DNS sweeping”?**

•    Identify the network IP block assigned to an organisation.

•    Iterate over the entire IP block range performing reverse DNS look-ups against the organisation's NS.

•    Can help reveal more details about the internal organisation of the network.

•    E.g. attacker seeks to target a particular user and finds a machine called “Bob's PC”.

•    Forward DNS searches involve querying for common machine names.

 ![DNS posioning.jpg](https://github.com/Heenjiang/ProgrammingBackend/blob/main/pic/DNS%20posioning.jpg?raw=true)

![2.jpg](https://github.com/Heenjiang/ProgrammingBackend/blob/main/pic/2.jpg?raw=true)