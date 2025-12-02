## x2ray使用xhttp上下行分离，上行使用vless+xhttp套cdn（表现为本地ip<->cf‘s IP），下行使用vless+tcp+reality（表现为本地IP<->VPS‘s IP），使用wireshark抓包，发现建立连接时先和下行也就是vps’s ip建立连接，此意何为？

这是正常现象，因为在 X2Ray 的“上下行分离”里，下行链路（Reality）是由客户端主动预先建立的“回程通道”，并不是被动由上行触发。<br>
客户端会主动先连 Reality VPS，形成“下行隧道”。<br>
然后才通过上行（XHTTP/CDN）开始真正的流量传输。<br>
上下行分离本质上是“两个独立连接”<br>
uplink:   客户端 → CF → 伪装网站 → VPS（XHTTP）<br>
downlink: VPS → Reality → 客户端（TCP）<br>
两条链路没有强依赖顺序，也不是同一个 TCP 四元组。<br>
客户端启动时，先建立 Reality TCP（下行），用作将来服务器回传的数据路径<br>
Reality 的握手需要一定时间，因此客户端提早初始化<br>
再建立 XHTTP 上行通道（通过 CDN），发送的实际请求，两条链路都会被 X2Ray 绑定在同一个内层会话中<br>
X2Ray 需要一个“可立即可用”的回程链路<br>
Reality 下行链路对于服务器 → 客户端的数据传递至关重要。下行 Reality 是客户端主动连接，不是被调用<br>
Reality 的特性就是：不接受随机 inbound 连接，全部是客户端主动发起，握手需要时间<br>
不像 VMess、Trojan 可以双向读写一个 TCP，Reality 是明确方向的“拉线”<br>
## Wireshark抓包结果
![](https://github.com/wjk199511140034/xhttp-study/blob/main/wireshark1.png)
![](https://github.com/wjk199511140034/xhttp-study/blob/main/wireshark2.png)
<br>
特征 1：上行分包<br>
多条 TCP，小 AppData，分散发送，多次握手<br>
这就是 XHTTP 典型的：<br>
Fragmented Uplink（碎片化上行）<br>
特征 2：上行多路并发<br>
本地端口 4835/4836…交替使用，每条都传非常少的数据，典型的：<br>
Multiplexed Uplink（多路上行）<br>
特征 3：下行流式传输<br>
连续的 TLS1.3 AppData，长时间活跃的 Reality 隧道<br>
一直来自 VPS 443<br>
这就是典型 Reality / VLESS 特征：<br>
Streamed Downlink（流式下行）<br>


## 服务器上行出站使用6666端口，服务器下行出站使用443端口，需要在服务端下行出站设置回落到6666，并在客户端行节点设置downloadSettings，内容指向下行出站的配置，为何服务端和客户端设置相悖

上下行分离本质上是两个完全独立的连接，由 X2Ray 内部“绑定”成一个会话。服务端与客户端配置的目标不同，因此必须看起来“反向”。**<br>
理解“上下行分离”必须先接受一个事实：它不是单通道，而是 TWO TCP<br>
客户端 ---上行---> 服务器<br>
客户端 <---下行--- 服务器<br>
在 XHTTP + Reality 架构中：<br>
| 方向                                   |      协议          |     谁发起        |端口                |作用              |
| -------------------------------------- | :----------------: | :----------------: |:----------------: |:----------------: |
| 上行（uplink）                       | XHTTP (VLESS+HTTP)| 客户端 → 服务器 |Server:6666|发客户端→服务器的数据|
| 下行（downlink）                      | Reality (VLESS+TLS)| 客户端 → 服务器 |Server:443|发服务器→客户端的数据|

注意❗ 两条链路都是由 客户端主动发起，Reality 不接受被动 inbound，因此必须 client→server。<br>
服务器端为什么 Reality 下行要 “回落到 6666”<br>
"回落" 是 server 入站(inbound) 的逻辑，不是出站(outbound)。<br>
Reality inbound 收到客户端的 TLS 流量， 解密后，发现这是“下行链路的控制信道”， 必须把这个“控制信道”转发给上行处理逻辑（XHTTP inbound）<br>
而你的 XHTTP inbound 正好监听在 6666 端口，所以 Reality inbound 必须 fallBack（回落）到 6666<br>
<br>
那客户端为什么 downloadSettings 要指向“下行出站”配置？<br>
因为客户端需要告诉 X2Ray：<br>
“服务器给我回流的数据应该通过 Reality（下行）这条链路传回来。”<br>
downloadSettings 指的不是“服务器监听端口”，而是：<br>
指定“下行应该走哪一个 outbound 配置项”<br>
客户端意思是：服务器 → 客户端 的数据，必须走 reality-downlink（443），而不是走 XHTTP（6666）。<br>
也就是“我期待服务器会通过 Reality 那条线把数据给我”。<br>
<br>
为什么看起来“相悖”？因为两边处理的对象不同<br>
服务端Reality inbound（443）需要将下行控制流量“路由”给 XHTTP inbound（6666），所以表现为 443 → 6666<br>
客户端downloadSettings 是告诉本地“我要从 Reality outbound（443）收下行数据”，所以表现为 下行 = Reality（443）<br>
它们并不矛盾，作用完全不同：<br>
客户端的下行 outbound = 服务器的 Reality inbound<br>
客户端的上行 outbound = 服务器的 XHTTP inbound<br>
而服务端“回落到 6666”只是 Reality inbound 的内部路由转发逻辑，不是客户端用的配置。<br>
服务端上行连接使用xhttp，你会发现根本没有回落选项<br>
<br>
