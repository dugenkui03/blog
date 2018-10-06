#### 一.引言

##### 1.1基本介绍

网络层只向上提供简单灵活的、无连接的、尽最大努力交付的数据包服务。每一个分组（即IP数据报）都是独立发送的，与其前后分组无关（不进行编号）。

网络层不提供服务质量的允诺，即说传送的分组可能出错、丢失、重复或者失序，**主机中的运输层**负责通信的可靠性。不像传统电信网一样先建立可靠的连接在传输数据是因为计算机有很强的差错处理能力。

传统电信网虚电路服务与数据报服务对比：
![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fpgr3bfb3mj30t506vt98.jpg)
![](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1fpgr3mrbykj30sp093759.jpg)

##### 1.2网络层主要协议、中间设备和XX交付概念

###### 主要协议

![](https://wx1.sinaimg.cn/mw690/006Xp67Kly1fpgrl1xorbj30d60asdg6.jpg)
1. 地址解析协议ARP（address resolution protocol）：
2. 逆地址解析协议RARP（reverse address resolution protocol）
3. 网际层报文协议ICMP（internet control message protocol）
4. 网际组管理协议IGMP（internet group management protocol）

**[注]：画在上层的协议对下层协议是调用关系；**

###### 中间设备

将网络连接在一起需要一些中间设备，根据中间设备所在的层次，可以有以下4种：

1. 网络层以上使用的中间设备叫**网关（gateway）**；
2. 网络层的中间设备叫路由器（router）；
3. 数据链路层叫网桥或者桥接器（bridge）；
4. 物理层叫转发器（repeater）；


###### 直接交付和间接交付

![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fpgry45tphj30qd0hfwgh.jpg)

如上图所示主机H~1~想H~2~发送数据报：如果主机H~1~查找自己路由表发现H~2~就在本网络（主机中也有路由表，详情查看ARP协议），便不用通过任何路由器而**直接交付**。否则必须把数据报交给某个路由器进行间接交付。

##### 1.3 IP地址的分类

###### 简介

32位4字节的IP地址由网络号和主机号组成：

- IP地址::={<网络号>,<主机号>}

###### 常用的三种类别地址

![](https://wx4.sinaimg.cn/mw690/006Xp67Kly1fpgsa6oqmyj30k50bjaao.jpg)

- A、B、C类地址的网络号非别有1、2、3个字节，其首位的**类别号**分别是0、10、110；
- A、B、C类地址用于一对一的单播，D类地址用于多播，E类地址保留为以后用；

<font color=red>**全0或者全1的网络号和主机号**</font>
- A类地址可指派的网络号有`2^7-1`个，因为全0表示this，意思是本网络，而除了类别为全1的（127）保留做本地软件环回测试（loopback test）用：目的地址的网络号为127时数据报不会被发送到任何网络；
- A类地址主机号3字节：全0时代表本主机所连接到的大哥网络（例如ip为5.6.7.8，则所连接到的A类网络是5.0.0.0），而主机号全1代表该网络上的所有主机；
- IP地址空间共有2^32个，A类网络占50%；
- B、C类网络号不可全为0，而主机号全0和全1也做保留。两只可指派的网络总数分别为25%和12.5%；
- ![](https://wx1.sinaimg.cn/mw1024/006Xp67Kly1fpgstqqem3j30so08k758.jpg)

**IP地址的指派范围**
![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fpgsrk3g32j30so05bgm8.jpg)
- A:1~126
- B:128.1~191.255;191=“10(类别号）111111”
- C:192.0.1~223.255.255；223=“110（类别号）11111”

**IP地址有以下特点：**
1. IP地址是分等级的结构，而且路由器仅根据目的主机IP的网络号来转发分组；
2. IP地址是一个主机（或路由器）和一条链路的接口，当主机连接到多个网络上时（比如路由器），需要多个网络号，这种主机称为多归属主机（multihomed host）；
3. 仅仅用网络层以下的转发器或网桥连接起来的若干个局域网仍是一个网络，因为他们具有相同的网络号net-id;
4. 互联网中ip地址示意图
![](https://wx2.sinaimg.cn/mw1024/006Xp67Kly1fpgtt5fkp8j30r40cvmyf.jpg)

- 路由器之间如果不分配地址则称之为无编号网络（unnumbered network）或无名网络（anonymous network）；

#### 二.物理地址、ARP协议和IP数据报格式

##### 2.1 物理地址
物理地址MAC存在于以太网适配器中，48位6字节，是数据链路层和物理层使用的地址；
![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fpgu4pjfamj30m108at96.jpg)
MAC（Medium Access Control）地址封装在MAC帧中，数据报在不同网络上传送时，其MAC帧首部的源地址和目的地址是**不断变化的**。**局域网的链路层只能看见MAC帧**;
![](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1fpgu7krcvhj30rs0dbt9w.jpg)

##### 2.2ARP协议
ARP协议负责将**本局域网**的IP地址转换到MAC地址，解决办法就是在主机ARP高速缓存（ARP cache）中存放一个IP地址到硬件地址的映射表。映射表内容是动态更新的。

###### ARP协议执行步骤：
前边讲到当主机H~1~向主机H~2~发送数据报时，会查看主机H~2~是否在本地局域网上，是的话进行**直接交付**。查看H~2~是否在本地局域网的过程由ARP协议负责：
1. 主机A查看ARP缓存是否有主机B的IP地址，如果有则将此地址放在MAC帧中并将数据发往此MAC地址，否则的话自动运行ARP协议：
2. APR进程在本局域网广播发送一个ARP请求分组，内容为“我的IP是X.X.X.X，硬件地址是Y-Y-Y-Y，需要知道IP地址为x.x.x.x主机的硬件地址”；
3. 本局域网上所有主机运行的ARP进程都收到了此ARP请求分组；
4. 对应目的IP的主机B**单播**回应信息“我是x.x.x.x,硬件地址是y-y-y-y”，并<font color=red>**将请求主机的IP和硬件地址放进ARP cache中**</font>；
5. 主机A收到主机B的ARP响应分组后，就在其ARP cache中写入主机B的IP到MAC地址的映射；
![](https://wx1.sinaimg.cn/mw1024/006Xp67Kly1fpgxdtxb7tj30pb0kojt0.jpg)

- 以上，APR作用于同一局域网，而且不同网络的主机也不需要知道对方的IP和MAC映射；

###### 为什么不直接用MAC地址而用IP地址进行通信呢？

世界上各种异构网络使用不同的硬件地址，如果通过硬件地址通信则需要进行复杂的硬件地址转换，而统一的IP地址使得各种异构网络就像在同一个网络上一样（屏蔽了异构网络中不同主机的差异）方便，而ARP协议的运行由底层软件自动运行。

##### 2.2 IP数据报的格式

IP数据报固定部分20字节，可选字段最长可达40字节。详情如下：
![](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1fpgxq6upfbj30im09ujrt.jpg)

- version：产品协议的版本
- IHL（Internet header length 首部长度）：单位是4字节，使用可选字段Options时如果长度不足4字节则用Padding字段填充；
- type of server：区分服务，一般不用
- total length 总长度：首部和数据之和的长度，单位字节，所以IP数据报总长度是2^16-1=65535字节
    1. IP数据报在数据链路层是否会被分片取决于数据链路层帧格式中的数据字段最大长度，即最大传输单元MTC（maximum transfer unit）。**网络层的IP数据报被封装成帧时，一定不能超过数据链路层的MTU**；
    2. IP数据报被分片则首部total length长度则被修改为分片后每一个分片额首部长度与数据长度的总和；
    3. 另.涉及字段：标志flags中的MF（morefragment）和片偏移Fragment offset;
- Flags:占3位，目前只有前两位有意义：
    1. MF（more fragment）：MF=1时表示源IP数据报被分片而且后边还有分片的数据报；
    2. DF（don‘t fragment):DF=1时表示此数据报不允许分片；
- fragment offset 片偏移：占13位，单位8字节，代表“较长的分组在分片后，某片在原分组中的相对位置”。片偏移是以8个字节为偏移单位，而且分片长度一定是8字节的整倍数。数据报分片示例：
![](https://wx2.sinaimg.cn/mw1024/006Xp67Kly1fpgyeu972oj30pt0hpq4l.jpg)
<font color=red>**注：分片后IP数据报的总长度、MF和片偏移的值都会改变**</font>;
- TTL time to live：TTL的功能是**跳数限制**，烦狗场无法交付的数据报无限制的在因特网中兜圈子。<font color=red>路由器在转发数据报前将TTL值减1，并判断其是否为0，是的话丢弃此报文，不在转发</font>。8位最大长度255，而且ttl=1表示此报文只在本局域网中传送；
- protocol：数据报所携带数据是哪个协议的，使目的主机知道应该上交给哪个处理过程。常用协议和对应字段值如下：
![](https://wx2.sinaimg.cn/mw1024/006Xp67Kly1fpgyquz94hj30st02hmx5.jpg)
- header checksum 首部校验和：每经过一个路由器都要重新计算一次，因为MF、fragment offset、total length和TTL都可能发生变化，不检验数据是为了减少工作量。流程如下
    1. 发送方发送前将header check数目字段置为0；
    2. 16位字“反码求和”后，将其反码放入checksum字段；
    3. 接收方将16位字“反码求和”后取反码，如果为全为0则正确；
    4. 接收方此时变成发送方，重复1~3直到发送到目的地；
- 源地址和目的地址各占32位8字节；
