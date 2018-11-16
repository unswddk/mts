## L3VPN知识总结

#### 一. 路由是如何学到的？

##### 基本配置

![](29.png)


拓扑如上图所示，要完成如下要求：

+ Site 1 <=> Site 4
+ Site 2 <=> Site 5
+ Site 3 <=> Site 6 <=> Site 7
+ Site 4 <=> Site 1


假设SP网络<font color='red'>使用RSVP在骨干网区域建立LSPs，通过如下的标签进行跟对端PE进行标签转发。假如服务商使用LSP来建立LSP，就不像这样端对端的连接，是一种多点对单点的连接。
</font>

PE1的配置：
首先在PE1上建立三个VPN实例。
    VRF Red     
        Interface: if_1
        RD = RD_Red
        Export Target = Red
        Import Target = Red

    VRF Blue
        Interface: if_4
        RD = RD_Blue
        Export Target = Blue
        Import Target = Blue

    VRF Green
        Interface: if_3
        RD = RD_Green
        Export Target = Green
        Import Target = Green

PE2上配置Red和BLue的VRF：

    VRF Red
        Interface: if_2
        RD = RD_Red
        Export Target = Red
        Import Target = Red
    VRF Blue
        Interface: if_3
        RD = RD_Blue
        Export Target = Blue
        Import Target = Blue    

PE3上配置Green的VRF：

    VRF Green
        Interface: if_2, if_3
        RD = RD_Green
        Export Target = Green
        Import Target = Green

对应我司的设备配置如下：

```java
//1. 配置VPN实例
    ip vpn-instance Red
    ipv4-family
        route-distinguisher RD_Red  //配置VPN实例IPv4地址族的RD 22:1
                                    //VPN Target是BGP的扩展团体属性，用来控制VPN路由信息的接收和发布
        vpn-target 5:5 both
                                    //如果import和export分别引入,使用如下命令
                                    //vpn-target 3:3 export-extcommunity
                                    //vpn-target 4:4 import-extcommunity
//2. 接口与VPN实例绑定
    interface g if_1
    ip bingding vpn-instance Red    //将当前接口和VPN实例绑定
    ip address xxx.xxx.xxx.xxx xx   //配置接口的IP地址
```

#### 2. VPN路由信息下发

在客户站点向另外一个客户站点，能通过VPN发送数据之前，VPN的路由信息必须通过骨干区域下发到每一个客户站点。

##### 2.1 CE向Ingress PE下发路由 

CE路由公布IPv4路由前缀到直连的PE路由。有多重方式可以使PE路由器学习到直连CE路由的路由信息。

+ 静态路由
+ 与CE之间运行IGP（RIP，OSPF）
+ 与CE建立EBGP

PE路由器会检查所有的路由，如果这条路由符合本地import策略，就会在VRF中加入这条IPv4路由。这些路由都不会发布进SP的骨干区域。
在广播这条路由之前，PE 下发一条MPLS标签给这条路由。如果这条路由是通过端对端学习到的，标签分发是基于入接口。所以，路由都分发相同的标签。

##### PE1

假设PE1分发1001的标签给从site1中学习到的路由。1002标签给从site2中学习的路由，1003标签分给从site3上学习到的路由。PE1上建立三条MPLS路由，当PE1从骨干区域收到带有1001,1002，或1003的数据包，它能pop标签，然后根据数据包的标签直接向CE1，CE2，或CE3转发ipv4数据表

**MPLS Forwarding Table (PE 1)**

|Input Interface|Label|Action| Output Interface|
|---|---|--|---|
|If_2| 1001| Pop |if_1|
|If_2 |1002 |Pop |if_4|
|If_2 |1003 |Pop |if_3|

这样的结果就是，PE1上的VRF包含这些本地路由：

*VRF Red*
|Destination|BGP Next-Hop| interface| Bottom Label|Top Label|
|--|--|--|--|--|
|10.1/16|Direct|if_1|1001|-

*VRF Blue*
|Destination|BGP Next-Hop| interface| Bottom Label|Top Label|
|--|--|--|--|--|
|10.1/16|Direct|if_4|1002|-


*VRF Green*
|Destination|BGP Next-Hop| interface| Bottom Label|Top Label|
|--|--|--|--|--|
|10.1/16|Direct|if_3|1003|-


##### PE2

**MPLS Forwarding Table (PE 2)**

|Input Interface|Label|Action| Output Interface|
|---|---|--|---|
|If_2| 1004| Pop |if_2|
|If_2 |1005 |Pop |if_3|


*VRF Red*
|Destination|BGP Next-Hop| interface| Bottom Label|Top Label|
|--|--|--|--|--|
|10.2/16|Direct|if_2|1004|-

*VRF blue*
|Destination|BGP Next-Hop| interface| Bottom Label|Top Label|
|--|--|--|--|--|
|10.2/16|Direct|if_3|1005|-

##### PE3

**MPLS Forwarding Table (PE 2)**
|Input Interface|Label|Action| Output Interface|
|---|---|--|---|
|If_2| 1006| Pop |if_2|
|If_2 |1007 |Pop |if_3|

*VRF Green*
|Destination|BGP Next-Hop| interface| Bottom Label|Top Label|
|--|--|--|--|--|
|10.3/16|Direct|if_2|1006|-
|10.3/16|Direct|if_3|1007|-

#### PE1和PE2跨骨干网下发路由 
Ingress PE routers use MP-IBGP to distribute routes received from directly connected sites to egress PE routers. 
PE1通过MP-IBGP分发从直连端口接收到的路由。PE路由器需要保存MP-IBGP全互联或者是使用路由反射器来保证路由信息能够下发到PE路由器中。在ingress PE路由分发本地VPN路由到它的MP-IBGP对等体之前，它将每一个IPv4前缀通过添加VRF中定义的RD值转换为VPN-IPv4前缀。路由的广播如下：

+ 路由的VPN-IPv4路由前缀
+ 一个包含Ingress PE的环回口的BGP下一跳。这个地址被编码为一个VPN-IPv4地址，RD是0因为MP-BGP要求下一跳是同样的地址族当路由被广播的时候。
+ PE会分发一个MPLS标签给路由，在他通过直连CE路由器学习到这条路由的时候。
+ 一条路由的Target Attribute是基于本地对VRF的export target配置策略来决定的。所有的路由都会携带这个RT值。

当一个ingress PE路由广播它的本地VPN-IPv4路由向它的MP-IBGP对等体的时候，它能要么发送VRFs中的所有的路由给它所有的MP-IBGP对等体，或者他能生成一个单独的广播为每一个对等体不包含了特定的VPN路由它不会分享给一个指定的对等体。这是通过ORF来允许一个BGP speaker 去announce 给它的对等体或者路由反射器，路由的集合会被引入一个或者多个这个PE 路由器的VRFs。当一个Egress PE路由器通过一个对等体接收到一个VPN-IPv4路由，它会将这个路由的信息跟所有的VRF的所有直连的VPN的import策略。
假如一个路由携带的RT能够匹配至少一个这个PE上的VRF中的路由引入策略，这个VPN-IPv4路由就会被加入到他的VPN_IPv4的RIB中。这个VPN_IPv4 RIB是一个很大的路由信息库，包含了所有的路由，能至少满足这个egress PE路由的VRFs中的一个引入策略。这个表示唯一的依靠RD来分开路由的，因为它是唯一的一个表包含所有的来自所有VNPs直连到PE上的路由。在这个表中的路由应该全局唯一因为重叠的IPv4地址有可能被加入唯一个的RD值。
BGP的路径选择发生在这个表中在路由s 被引入他们的目的VRF中之前。注意用户错误配置RDs能够导致VPN-IPv4路由在这个表征有相同的结构，但他们应该是不同的。
假如这种情况发生，BGP路径选择就会只选择其中的一条加入VRF中。
基于这个原因，RFC 2574bis建议使用全局唯一的公开ASNs和IPv4地址来定义RDs。

##### Ingress PE 路由发布
这部分描述，路由是如何通过Ingress PE发布并穿过MPLS-VPN的骨干区域到达Egress PE的。

**PE 1 路由发布**

**PE 1 发布下面的路由给它的每一个MP-IBGP对等体。**

 <!-- the following routes to each of its MP-IBGP peers. -->
    Destination = RD_Red:10.1/16
    Label = 1001
    BGP Next Hop = PE 1
    Route Target = Red

    Destination = RD_Blue:10.1/16
    Label = 1002
    BGP Next Hop = PE 1
    Route Target = Blue

    Destination = RD_Green:10.1/16
    Label = 1003
    BGP Next Hop = PE 1
    Route Target = Green

**PE 2 路由发布**

**PE 2 发布下面的路由信息给它的每一个MP-IBGP对等体.**

    Destination = RD_Red:10.2/16
    Label = 1004
    BGP Next Hop = PE 2
    Route Target = Red

    Destination = RD_Blue:10.2/16
    Label = 1005
    BGP Next Hop = PE 2
    Route Target = Blue
**PE 3 路由发布**
**PE 3 advertises the following routes to each of its MP-IBGP peers.**

    Destination = RD_Green:10.2/16
    Label = 1006
    BGP Next Hop = PE 3
    Route Target = Green
    Destination = RD_Green:10.3/16
    Label = 1007
    BGP Next Hop = PE 3
    Route Target = Green

#### Egress PE 路由安装

这一部分描述Egress PE如何学习过滤然后install 从ingress PE接收到的路由
**PE 1 Route Installation**
    PE 1 installs the following route from peer PE 2 into VRF Red.
    Destination = RD_Red:10.2/16
    Label = 1004
    BGP Next Hop = PE 2
    Route Target = Red’

    PE 1 installs the following route from peer PE 2 into VRF Blue.
    Destination = RD_Blue:10.2/16
    Label = 1005
    BGP Next Hop = PE 2
    Route Target = Blue

    PE 1 installs the following routes from peer PE 3 into VRF Green.
    Destination = RD_Green:10.2/16
    Label = 1006
    BGP Next Hop = PE 3
    Route Target = Green

    Destination = RD_Green:10.3/16
    Label = 1007
    BGP Next Hop = PE 3
    Route Target = Green

当所有路由交换之后，PE1的VRF如下：

*VRF Red*
|Destination|BGP Next-hop|Interface|Botom Label|Top Label|
|---|---|---|----|---|
10.1/16|Direct|IF_1|1001|-
10.2/16|PE2|IF_2|1004|11

*VRF Blue*
|Destination|BGP Next-hop|Interface|Botom Label|Top Label|
|---|---|---|----|---|
10.1/16|Direct|IF_4|1002|-
10.2/16|PE2|IF_2|1005|11
*VRF Green*
|Destination|BGP Next-hop|Interface|Botom Label|Top Label|
|---|---|---|----|---|
10.1/16|Direct|IF_3|1003|-
10.2/16|PE3|IF_2|1006|66
10.3/16|PE3|IF_2|1006|66

**PE 2 Route Installation**

**PE 2 installs the following route from peer PE 1 into VRF Red.**

    Destination = RD_Red:10.1/16
    Label = 1001
    BGP Next Hop = PE 1
    Route Target = Red
**PE 2 installs the following route from peer PE 1 into VRF Blue.**

    Destination = RS_Blue:10.1/16
    Label = 1002
    BGP Next Hop = PE 1
    Route Target = Blue

当所有路由交换之后，PE2的VRF如下：
*VRF Red*
|Destination|BGP Next-hop|Interface|Botom Label|Top Label|
|---|---|---|----|---|
10.1/16|PE1|IF_1|1001|22
10.2/16|direct|IF_2|1004|—

*VRF Blue*
|Destination|BGP Next-hop|Interface|Botom Label|Top Label|
|---|---|---|----|---|
10.1/16|PE1|IF_1|1002|22
10.2/16|Direct|IF_2|1005|-
**PE 3 Route Installation**
**PE 3 installs the following route from peer PE 1 into VRF Green.**
    Destination = RD_Green:10.1/16
    Label = 1003
    BGP Next Hop = PE 1
    Route Target = Green

当所有路由交换之后，PE3的VRF如下：
*VRF Green*
|Destination|BGP Next-hop|Interface|Botom Label|Top Label|
|---|---|---|----|---|
10.1/16|PE1|IF_1|1002|55
10.2/16|Direct|IF_2|1006|-
10.3/16|Direct|IF_3|1007|-

#### Egress PE 向 CE 下发 路由

如果Egress PE路由器的VRF中存在一个路由可以用来传递一个直连的CE路由器的数据包，这个PE路由器能下发这个路由给这个CE路由器。CE路由器可以使用下面几种方式从他直连的PE路由器来学习VPN路由。
+ 跟PE之间运行IGP
 <!-- Run an IGP (RIPv2, OSPF) with the PE router. -->
+ 与PE建立一个EBGP连接
<!-- Establish an EBGP connection with the PE router. -->
另外更简单的还可以：
<!-- Alternatively, the PE router can simply do the following. -->
+ PE-to-CR之间的路由协议能够下发一个缺省路由指向PE路由器。
+  通过一个指向PE路由器的静态缺省路由来configure CE
<!-- The CE can be configured with a static default route pointing to the PE router. -->
<!-- The PE-to-CE routing protocol can distribute a default route pointing to the PE router. -->
当所有的路由从PE下发到了CE之后，CE的路由表中就有了以下信息：
CE 1 Routing Table
Destination| Next-Hop| Interface
---|----|----
10.1/16| Direct| if_x
10.2/16 |PE 1  | if_z

CE 2 Routing Table
Destination| Next-Hop| Interface
---|----|----
10.1/16| Direct| if_x
10.2/16 |PE 1  | if_z

CE 3 Routing Table

Destination| Next-Hop| Interface
---|----|----
10.1/16| Direct| if_x
10.2/16 |PE 1  | if_z
10.3/16 |PE 1 |if_z

CE 4 Routing Table
Destination| Next-Hop| Interface
---|----|----
10.1/16 |PE 2  | if_z
10.2/16| Direct| if_x

CE 5 Routing Table
Destination| Next-Hop| Interface
---|----|----
10.1/16 |PE 2  | if_z
10.2/16| Direct| if_x

CE 6 Routing Table
Destination| Next-Hop| Interface
---|----|----
10.1/16 |PE 2  | if_z
10.2/16| Direct| if_x
10.3/16| PE 2| if_z

CE 7 Routing Table
Destination| Next-Hop| Interface
---|----|----
10.1/16 |PE 2  | if_z
10.2/16| PE 2| if_z
10.3/16| Direct| if_x


到此所有的路由学习完毕。

#### 二. 数据包如何转发？
Transmitting customer traffic from one VPN site to another VPN site involves a number of
different forwarding decisions.

从一个VPN站点到另外一个VPN站点转发客户流量包含了一系列不同的转发策略

+ 源CE路由到ingress PE路由器的策略
+ Ingress PE路由器转发策略
+ 每一个P路由器的转发策略
+ 出口PE路由器的到目的地CE路由器的转发策略

1. Source CE Router to Ingress PE Router Forwarding

当一个CE路由器接收到一个同同站点的系统内IPv4数据包，CE路由器通过最长匹配查找和转发IPv4数据包给他直连的PE路由器。
2. Ingress PE Router Forwarding
当一个PE路由器从一个CE收到IPv4数据包，这个PE会在VRF进行中查找。数据包的目的地地址匹配ipv4前缀，如果在VRF中找到一个匹配项，返回下一跳和出接口信息。
如果这个数据包的出接口是与数据包关联着同一个VRF，下一跳要么是另外一台CE设备，要么是同一个VPN的一个不同的直连站点CE。在一个PE中的一个单独的VRF存在对应着一个VPN的所有来至直连站点的路由。
如果这个数据包的出接口和进接口关联着练个两个不同的VRFs，他们直连的两个站点有至少一个VPN是相同的，然后每个站点有分开的转发表。为了转发数据包，有必要在关联着出接口的VRF中需要查询数据包的目的地址。
如果出接口没有关联VRF，这个数据包必须至少一跳穿过服务商的骨干网到达一个远端的PE路由器。如果这个数据包必须穿过骨干区域，它至少有两个下一跳，一个BGP下一跳，一个IGP下一跳。

+ BGP下一跳是这个入口PE初始发布的VPN-IPv4路由。BGP下一跳是分配和分发标签来通过MP-IBGP，确认发布这条路由的站点直连的关系。PE路由器压入标签在这个数据包的标签栈，它变成底层标签。或者内层标签。
+ 这个IGP下一跳是第一跳在这条LSP中去往BGO的下一跳。这个IGP下一跳会被分配一个LDP或者RSVP标签来到达BGP的下一跳路由器。这个标签是上层标签或者是外层标签。

因此，从CE收取数据包，然后创建标签栈的PE路由器就是入口LSR，然后BGP下一跳是这条穿过骨干区域的LSP的出口LSR。假如BGP的下一跳和IGP的下一跳是同一个路由器，然后如果使用倒数第二跳弹出，这个数据包能够只通过BGP标签转发。

##### P Router Forwarding

MPLS的骨干区域交换标签，在每一条都会交换上层标签，知道到达倒数第二跳，在这里上层标签被弹出，数据被发往目的PE路由器。

##### PE Router to Destination CE Router Forwarding
当PE路由器收到数据包，它会先查MPLS路由对底层的标签，看这个路由对应的标签和接口信息。如果匹配，底层标签被弹出，IPv4数据包被之间送往与Label关联的CE路由器。这里注意跟站点直连的VRF并不需要被访问。


实例 #1: Forwarding VPN Red Traffic from Site 1 to Site 4
假设在站点1的主机10.1.2.3想要发送一个数据包到在站点4的10.2.9.3的服务器
 
 从site 1向site 4转发Red VPN流量如下图：
![](30.png)



