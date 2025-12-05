### xhttp上下行分离怎么玩的啊，看了官方教程和大佬的配置文件，感觉会玩了，还是连不上<br>
简直是一看就会，一做就废<br>
我有一个双栈vps，准备上行ipv6，下行ipv4<br>
先在服务端建立了一个入站<br>
vless+xhttp ，port:50001，path:/8e87b868<br>
然后导入到客户端，正常ping通<br>
ok，开始实现分离<br>
客户端address是ipv6，在客户端的extra写入了：<br>
```
{
  "downloadSettings": {
    "address": "my ipv4", 
    "port":50001, 
    "network": "xhttp", 
    "xhttpSettings": {
      "path": "/8e87b868", 
      "mode": "auto"
    },
    "security": "none",
  }
}
```
正常ping通，我以为xhttp上下行分离就这么简单，等会加上tls就行了<br>
但是我抓了一下包，发现所有连接都是走的ipv6，根本抓不到ipv4，虽然能跑通，但是完全不是我预想的样子<br>
然后我又问了问ai，他说这样不行，我得再建立一个通道balabala....，然乎我在服务端又建立了一个入站<br>
vless+xhttp+relity ，port:50002，path:/8e87b868<br>
然后在客户端的extra写入了：<br>
```
{
  "downloadSettings": {
    "address": "my ipv4", 
    "port":50002, 
    "network": "xhttp", 
    "xhttpSettings": {
      "path": "/8e87b868", 
      "mode": "auto"
    },
    "security": "reality",
        "realitySettings": {
          "serverName": "microsoft.com",
          "fingerprint": "chrome",
          "show": false,
          "publicKey": "vloNOW-Nudm4BflmuDbwomGvohLO2DVfurHk-KurG00",
          "shortId": "",
          "spiderX": "/",
          "mldsa65Verify": ""
        }
  }
}
```
当我以为万事大吉的时候，发现加上了这条extra之后根本ping不通，，，，<br>
抓个包看看，好像有和ipv4建立连接，有携带sni=microsoft.com的hello信息，似乎是上行不通？<br>
| source                                   |      destination|     protocol|length|info|
| -------------------------------------- | :----------------: | :----------------: |:----------------: |:----------------: |
| 192.168.152.23                       | my ipv4 | TLSv1.2 | 29  | Client Hello (SNI=microsoft. com) |
<br>
不信邪的我又加了一个入站<br>
vless+xhttp ，port:50003，path:/8e87b868<br>
然后在客户端的extra写入了：<br>

```
{
  "downloadSettings": {
    "address": "my ipv4", 
    "port":50003, 
    "network": "xhttp", 
    "xhttpSettings": {
      "path": "/8e87b868", 
      "mode": "auto"
    },
    "security": "none",
  }
}
```
又是正常ping通，但是抓包只有的ipv6，，，，<br>
### 上行ipv6 vless+xhttp port:5001，下行ipv4 vless+tcp+reality port:5002 fallback 5001<br>
继续读大佬的帖子[#4118](https://github.com/XTLS/Xray-core/discussions/4118) ，继续学习，发现我应该是没有设置回落，于是马上去改，由于面板中xhttp没有回落，自已看了一下官方文档，上面明确写了[回落只能用于 TCP+TLS 传输组合](https://xtls.github.io/config/features/fallback.html#fallbacks-%E9%85%8D%E7%BD%AE)，但是tcp+reality能用吗？不知道，试试呗！<br>
于是现在的服务端有两个inbound<br>
```1：vless+xhttp ，port:5001```<br>
```2：vless+tcp+relity ，port:5002，fallback 5001```<br>
将1的配置生成分享链接，导入到客户端，address要改成vps的ipv6，然后再添加extra<br>
终于让我跑通了！<br>
兴致勃勃的去抓包，这次终于对劲了！<br>
<img width="1221" height="839" alt="未标题-1" src="https://github.com/user-attachments/assets/407555a8-608f-403a-93db-74720025a091" /><br>
<br>
可以看到非常有意思的是，实际上客户端是先和下行的reality建立连接，然后才和上行XHTTP建立连接<br>
因为在这个配置中，reality相当于“前置”，他负责将看不懂的协议回落到指定端口，至此，小白终于完成了一次真正意义上的上下行分离<br>
上行ipv6 vless+xhttp，下行ipv4 vless+tcp+reality<br>
<br>
下面将具体设置过程贴出来，生啃代码实在是要命，这里都是使用面板做的<br>
1.先添加第一个入站<br>
![未标题-2](https://github.com/user-attachments/assets/fd80dd81-ca37-440c-b7b4-4eaab359135c)<br>
2.再添加第二个入站<br>
<img width="680" height="2246" alt="未标题-3" src="https://github.com/user-attachments/assets/6b136bae-0c94-4ffb-9955-d1ee7b29e8e9" /><br>
3.将第一个入站导出到V2RayN，注意将address改成ipv6地址<br>
然后复制下面这一段，添加到V2RayN里面。注意替换you ipv4，serverName,path,publicKey，要和你的reality对应<br>

```
{
  "downloadSettings": {
    "address": "your ipv4", 
    "port":5002, 
    "network": "xhttp", 
    "xhttpSettings": {
      "path": "/8e87b868", 
      "mode": "auto"
    },
    "security": "reality",
        "realitySettings": {
          "serverName": "microsoft.com",
          "fingerprint": "chrome",
          "show": false,
          "publicKey": "vloNOW-Nudm4BflmuDbwomGvohLO2DVfurHk-KurG00",
          "shortId": "",
          "spiderX": "/",
          "mldsa65Verify": ""
        }
  }
}
```
~~嫌麻烦的可以将第二个入站也导出到V2RayN，然后右键导出->导出所选配置到剪切板，新建一个txt文档，ctrl+V，找到realitySettings，将花括号里面的内容都复制下来，替换到刚才那一段文字中对应位置~~        似乎更麻烦了？<br>
4.将改好的文字粘贴到蓝圈这里<br>
![未标题-4](https://github.com/user-attachments/assets/686fca94-c87d-47b3-a000-3aa1b66adcb7)<br>


5.粘贴进去，然后确定，对着这个节点测延迟，应该就能看到实际数据了<br>

到这里应该已经明白了回落的大概原理，后面再尝试更多内容<br>
这个上行并没有加密，仅作为学习实验使用，请勿在正式使用中裸xhttp连接<br>
上行加上tls或者reality，再修改下客户端中的配置，就可以愉快的上下行分离了<br>
你也可以将上行的address改成ipv4，下行downloadSettings的address改成ipv6，实现上行ipv4，下行ipv6<br>
其实明白了原理就一通百通，在这个基础上，将你的域名托管到cf，然后添加一个A记录(或者AAAA记录)，指向你的VPS，将上行的address改成你的域名，就可以上行域名，下行ip<br>
点开小黄云，将你的域名填到SNI和host里面，address填优选的ip或者域名，就可以实现上行cdn，下行reality直连。<br>
同样的也可以将上下行反过来，上行直连下行cdn<br>
### some problem using same port<br>
继续拜读大佬神贴，在后边发现了不用Nginx的版本[#4118(comment)](https://github.com/XTLS/Xray-core/discussions/4118#discussioncomment-11593313 )，于是马上来学习<br>
在我之前的尝试中，虽然成功的实现了上下行分离，但是始终是跑在两个端口的，为什么复用同一个端口没有成功？<br>
总是上下行一起走ipv6，也就是downloadSettings没有起效，我还不懂哪里搞错了<br>
问了问AI，他说是因为没有TLS加密，我心里表示非常怀疑，不过既然有大佬的模板，那我继续抄作业<br>
准备实现上行vless+xhttp+tls套cdn，下行ipv4直连<br>
在cf托管一个域名，设置a.1995.xyz指向我的ipv6，<br>
因为我的ipv4没有443端口，所以我将端口设置为了12345，而cf走代理的话只允许80系和443系端口，所以我开了Origin rule 回源到12345<br>
将配置导出到客户端，并将address改成a.1995.xyz，port改成443，可以成功ping的通，可以测速，证明回源成功<br>
按照刚才的理解写一段extra，粘贴进客户端<br>
现在的配置是<br>
```1 服务端：vless+xhttp+tls ，port:12345，host：a.1995.xyz，SNI：a.1995.xyz，path：/ad1aa914```<br>
```2 客户端 ：vless+xhttp+tls ，address:a.1995.xyz，port:443，host：a.1995.xyz，SNI：a.1995.xyz，path：/ad1aa914```<br>
```3 CF：hostname eq a.1995.xyz--> port:12345```<br>
```
4. Extra:
 {
            "downloadSettings": {
              "address": "my ipv4",  
              "port":12345,
              "network": "xhttp",
              "security": "tls",
              "tlsSettings": {
                "serverName": "a.1995.xyz",  
                "allowInsecure": false, 
                "alpn":  ["h2", "http/1.1"],
                "fingerprint": "chrome"
              },
              "xhttpSettings": {
                "host": "a.1995.xyz", 
                "path": "/ad1aa914", 
                "mode": "auto"
              }
            }
          }
```
<br>
我发现本来可以跑的通的，但是加上这一段Extra之后就无法联通了，不知道是为什么。看了下日志，好像是证书错误？？<br>

```
2025/12/04 17:42:34.086873 [Warning] core: Xray 25.10.15 started
2025/12/04 17:42:35.153922 [Info] [707253193] proxy/socks: TCP Connect request to tcp:www.google.com:443
2025/12/04 17:42:35.153922 [Info] [707253193] app/dispatcher: taking detour [proxy10830] for [tcp:www.google.com:443]
2025/12/04 17:42:35.153922 from tcp:127.0.0.1:9892 accepted tcp:www.google.com:443 [mixed10830 -> proxy10830]
2025/12/04 17:42:35.153922 [Debug] [707253193] transport/internet/splithttp: XMUX: creating xmuxClient because xmuxClients is empty
2025/12/04 17:42:35.153922 [Info] [707253193] transport/internet/splithttp: XHTTP is dialing to tcp:a.1995.xyz:443, mode packet-up, HTTP version 2, host a.1995.xyz
2025/12/04 17:42:35.153922 [Debug] [707253193] transport/internet/splithttp: XMUX: creating xmuxClient because xmuxClients is empty
2025/12/04 17:42:35.153922 [Info] [707253193] transport/internet/splithttp: XHTTP is downloading from tcp:[**my ipv4**]:12345, mode stream-down, HTTP version 2, host a.1995.xyz
2025/12/04 17:42:35.154569 [Debug] [707253193] transport/internet: dialing to tcp:[**my ipv4**]:12345
2025/12/04 17:42:37.072728 [Info] [707253193] transport/internet/splithttp: failed to GET https://a.1995.xyz/ad1aa914/477a35fb-59ee-466c-8a78-7c21ffc1fb16 > Get "https://a.1995.xyz/ad1aa914/477a35fb-59ee-466c-8a78-7c21ffc1fb16": tls: failed to verify certificate: x509: certificate signed by unknown authority
2025/12/04 17:42:37.072728 [Info] [707253193] proxy/vless/outbound: tunneling request to tcp:www.google.com:443 via a.1995.xyz:443
2025/12/04 17:42:37 测试完成
```
<br>
但是不加这一段Extra可以正常通讯，证书是没问题的，为什么改了下行ip就不行呢？搞不懂<br>
然后用wireshark抓了下包，发现可以和下行的ipv4沟通，但是和ipv6没有成功握手，这是又是为什么？<br>
<img width="1492" height="407" alt="未标题-1" src="https://github.com/user-attachments/assets/90ca37b8-ebda-4059-82db-444ccab3724c" /><br>

难道和我的回源设置有关系吗？想上下行复用同一个端口咋就这么难？<br>
### Finally 上行套cdn vless+xhttp+tls port：12345，下行直连ipv4 vless++xhttp+tls port：12345<br>
搞了一会，还是没搞好，试着改成`allowInsecure=true`，倒是不报证书错误了，开始报404<br>
```
2025/12/04 21:30:49.124564 transport/internet/splithttp: XHTTP is downloading from tcp:[***my ipv4***]:12345, mode stream-down, HTTP version 2, host a.1995.xyz 
2025/12/04 21:30:49.317261 [Debug] [3553020593] transport/internet: dialing to tcp:[***my ipv4***]:12345
2025/12/04 21:30:49.592117 [Info] [3553020593] proxy/vless/outbound: tunneling request to tcp:www.google.com:443 via a.1995.xyz:443 
2025/12/04 21:30:49.592117 [Debug] [3553020593] transport/internet: dialing to tcp:a.1995.xyz:443 
2025/12/04 21:30:49.722148 [Info] [3553020593] transport/internet/splithttp: unexpected status 404 
2025/12/04 21:30:49 测试完成
```
再去问AI，他像模像样的解释了一大堆<br>
> 请求到达后 XHTTP 没能匹配到正确的 path/host/方法（因此返回 404） —— 通常是因为上游 TLS 握手并没有把正确的 SNI/Host/HTTP2 请求传递给 XHTTP handler，或是请求方法/头不满足 XHTTP 期望（XHTTP 常用 POST/GRPC 风格），或 xhttpSettings.host 配置导致拒绝。<br>

不知道AI对不对，难道真的是我的证书有问题？我的证书是白嫖的Sectigo，但是为何不设置上下行分离的时候可以用呢？<br>
为了保险起见，干脆随大流整个acme的免费证书吧<br>
这次换了证书之后，果然马上通了！抓个包一看，和分端口几乎一样，先和下行的ipv4建立连接<br>
即使没有前置的reality回落，也是先建立下行通路的连接，这是xray的特性？不是很了解，总之，能通了就好<br>
![未标题-1](https://github.com/user-attachments/assets/2a5e3f61-d7aa-4c20-9a42-00f12a9e4bc1)<br>
<br>
小白终于完成了二次上下行分离，上行ipv6(domain) vless+xhttp+tls套cdn，下行直连ipv4 vless++xhttp+tls，上下行统一端口12345，并且使用cf回源（套娃呢搁这？）<br>
如果你的VPS上下行都有完整的端口可用，就不必这么麻烦<br>
但是大多数nat小鸡，只有ipv6和高位ipv4，所以能实现非标准端口上行cdn+下行直连，对一些回程线路好的小鸡还是~~有意义的~~（有线路好的NAT小鸡？）.<br>
<br>
下面将具体设置过程贴出来，生啃代码实在是要命，这里都是使用面板做的<br>
1.CF添加一个AAAA记录指向自己的ipv6，开启小黄云（回源用的，如果你有完整的ipv4和v6端口可以不开）这个不用教了吧？注意SSL/TLS为FULL并且开启gRPC，老生常谈了<br>
2.面板后台申请证书，不同的面板方式不一样，具体请参考面板官方教程，我这个是alireza的[x-ui](https://github.com/alireza0/x-ui)，一般申请完证书之后都能自动安装，安装完证书你可以用[https://youdomain.com/:x-ui's port/x-ui's path](#)进入面板<br>
<img width="435" height="610" alt="image" src="https://github.com/user-attachments/assets/d4a14081-002c-4a6d-83a9-0842783aca25" /><br>
<br>
3. 添加入站，走同一个端口的话只需要添加一个入站，我这使用是非标准高位端口，如果你有完整的ipv4和v6端口，直接填443就好了<br>
![未标题-2](https://github.com/user-attachments/assets/2de50bd3-3373-4ee8-a6c7-0c57e84e24db)<br>
<br>
4.填好之后将配置导入到V2ray，修改address和port，如果你有完整的ipv4和v6端口，只需要修改address，端口本来就是443<br>
<img width="1109" height="861" alt="未标题-2" src="https://github.com/user-attachments/assets/62ead0e8-87bd-4c9d-894f-07ef9d10b579" /><br>
5.然后去CF，左侧面板，找到“规则”->"概述"->“Origin Rules”->“创建规则”<br>
主要就是修改红圈部分<br>
<img width="1901" height="2992" alt="2025-12-04 23 16 57 dash cloudflare com d481adf59767" src="https://github.com/user-attachments/assets/f263793c-4e2e-4d23-bb79-ad3c50221ba2" /><br>
6.复制这一段，填入v2rayN的extra，注意替换you ipv4，serverName，host，path，要和你的配置对应（如果你有完整的ipv4和v6端口，端口填443）<br>
```
 {
            "downloadSettings": {
              "address": "my ipv4",  
              "port":12345,
              "network": "xhttp",
              "security": "tls",
              "tlsSettings": {
                "serverName": "a.1995.xyz",  
                "allowInsecure": false, 
                "alpn":  ["h2", "http/1.1"],
                "fingerprint": "chrome"
              },
              "xhttpSettings": {
                "host": "a.1995.xyz", 
                "path": "/ad1aa914", 
                "mode": "auto"
              }
            }
          }
```
7.粘贴到这里，然后点确定就行了，这会你应该可以上行vless+xhttp+tls+cdn，下行直连vless+xhttp+tls了<br>
<img width="1103" height="865" alt="未标题-3" src="https://github.com/user-attachments/assets/3298741b-43e5-4dc2-b094-0f1052d70266" /><br>
8.想要优选IP或者优选域名的话，筛选后填到客户端的address里面就行了<br>
9.同样的，你可以将上下行调换，实现上行直连vless+xhttp+tls，下行vless+xhttp+tls+cdn<br>
看到这应该可以基本掌握了xhttp上下行分离的操作和原理了，并且可以使用web-ui和gui尝试着去配置自己的xhttp了<br>
以后就可以去探索R主席说的各种姿势了，，，，<br>
### some note
个人理解笔记，大佬请忽略，理解不一定对，纯属个人瞎猜：<br>
首先上行需要设置成xhttp，然后只需要在客户端填入一段extra就可以完成上下行分离操作<br>
一开始的三段代码其实离成功只有一步，因为我json写的有语法错误，最后一行参数后面不能加逗号<br>
但是马虎的我一开始没有发现，导致downloadSettings根本没有生效，所以一开始写了Extra虽然能跑，但是一直走上行的ipv6，相当于没有写Extra<br>
<br>
上下行分离的逻辑：虽然这很反直觉，但是上下行分离不是服务端决定的，它是由客户端决定的，即客户端检测到了extra:{downloadSettings}，要先测试下行连接通断，然后他和下行进行一些列复杂的沟通，再去上行通讯<br>

在一开始的尝试中，如果上下行协议完全相同，只是想上下行走不同ip或上下行单边过cdn，那么完全可以使用同一个端口，即上下行端口复用(废话，本来就是用同一个端口)<br>
如果上下行协议不相同，比如上行xhttp，下行xhttp+reality，那么就需要用到tcp特有的回落<br>

可以看一下大佬的操作[#4118](https://github.com/XTLS/Xray-core/discussions/4118)，reality是前置inbound，fallback都丢到`/dev/shm/xhttp_client_upload.sock`，然后xhttp再监听`/dev/shm/xhttp_client_upload.sock`，而且xhttp里面也没有写tls，tls是通过Nginx实现的<br>
``` 
 "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
       "streamSettings": {
          "network": "raw",
          "security": "reality",
         }
        "fallbacks": [
          {
            "dest": "/dev/shm/xhttp_client_upload.sock",
          }
        ]
      }
     {
     "listen": "/dev/shm/xhttp_client_upload.sock,0666",
      "protocol": "vless",
      "streamSettings": {
        "network": "xhttp",
      }
     }
]
```
<br>

我测试了mKCP和xhttp回落不会成功，因为官方写了[回落只能用于TCP+TLS传输组合](https://xtls.github.io/config/features/fallback.html)，但是TCP+Reality和纯TCP居然也能成功回落？<br>
根据我的推测，不敲代码，纯web-ui操作的话<br>
如果端口复用（上下行同一个端口），上下行协议必须完全一致，只能实现：

| 上行                 |         下行|通断                          |
| ---------------- | :--------------------------: | :---------------------:|
| xhttp+reality    | xhttp+reality | ✔|
| xhttp+tls          | xhttp+tls       | ✔| 
| xhttp                | xhttp             | ✔| 
<br>
当然，上下行虽然协议必须一致，但是可用不同的ip(双栈ip和多ip的vps)，或者上下行不同域名<br>
<br>
你可能会问这么做的意义是什么是呢？其实xhttp+tls和纯xhttp是可以过cdn的，你可以上下行优选不同的ip，以达到最低的延迟<br>
<br>
如果使用tcp的fallback，可以使上下行走在不同端口，但由于下行连接限定只能用tcp，所以只能是（理论上）：

| 上行                |                          下行|通断                          |
| --------------- | :---------------------: | :---------------------:|
| xhttp+reality  | tcp+reality(fallback) | ❌unexpected EOF | 
| xhttp+tls        | tcp+reality(fallback) | ❌unexpected EOF | 
| xhttp              | tcp+reality(fallback) | ✔| <br>
| xhttp+reality  | tcp+tls(fallback)       | ❌unexpected EOF | 
| xhttp+tls        | tcp+tls(fallback)       | ❌unexpected EOF | 
| xhttp              | tcp+tls(fallback)       | ✔| <br>
| xhttp+reality  | tcp(fallback)             | ❌unexpected EOF | 
| xhttp+tls        | tcp(fallback)             | ❌unexpected EOF | 
| xhttp              | tcp(fallback)             | ✔| 

<br>
可以在此基础上，尝试去更改上下行的ip，域名，或者尝试去过cdn实现加速等等<br>
更复杂更高级的玩法可能需要Nginx前置回落，或者直接手搓json了吧。。。。总之小白的纯ui玩xhttp到这里也该结束了<br>
<br>
<br>
不过呢，由上表可以看到，只有上行xhttp时可以生效，这是为什么呢？<br>
<br>
以下是上行xhttp+tls的log，上行使用reality时log也一样<br>
可以看到客户端已经和ipv4 ipv6都握手了，最后不知道为何跳一个unexpected EOF<br>
<br>

```
2025/12/05 15:40:08.904103 [Warning] core: Xray 25.12.2 started 
2025/12/05 15:40:09.976143 [Info] [2274171885] proxy/socks: TCP Connect request to tcp:www.google.com:443 
2025/12/05 15:40:09.976143 [Info] [2274171885] app/dispatcher: taking detour [proxy10830] for [tcp:www.google.com:443] 
2025/12/05 15:40:09.976143 from tcp:127.0.0.1:3378 accepted tcp:www.google.com:443 [mixed10830 -> proxy10830] 
2025/12/05 15:40:09.976143 [Debug] [2274171885] transport/internet/splithttp: XMUX: creating xmuxClient because xmuxClients is empty 2025/12/05 15:40:09.976143 [Info] [2274171885] transport/internet/splithttp: XHTTP is dialing to tcp:[**my ipv6**]:5001, mode packet-up, HTTP version 2, host a.1995.xyz 
2025/12/05 15:40:09.976143 [Debug] [2274171885] transport/internet/splithttp: XMUX: creating xmuxClient because xmuxClients is empty 
2025/12/05 15:40:09.976143 [Info] [2274171885] transport/internet/splithttp: XHTTP is downloading from tcp:[**my ipv4**]:5002, mode stream-down, HTTP version 2, host mofumofu.me 
2025/12/05 15:40:09.976692 [Debug] [2274171885] transport/internet: dialing to tcp:[**my ipv4**]:5002 
2025/12/05 15:40:10.546195 [Info] [2274171885] proxy/vless/outbound: tunneling request to tcp:www.google.com:443 via [**my ipv6**]:5001 
2025/12/05 15:40:10.546702 [Debug] [2274171885] transport/internet: dialing to tcp:[**my ipv6**]:50301 
2025/12/05 15:40:10.730552 [Info] [2274171885] transport/internet/splithttp: failed to GET https://mofumofu.me/ad1aa914/c2575b26-17dc-4f83-8920-78cad1477788 > Get "https://mofumofu.me/ad1aa914/c2575b26-17dc-4f83-8920-78cad1477788": unexpected EOF 
2025/12/05 15:40:10 测试完成
```
<br>
上行xhttp的log

```
2025/12/05 16:04:18.209989 [Warning] core: Xray 25.12.2 started
2025/12/05 16:04:19.286191 [Info] [3503482820] proxy/socks: TCP Connect request to tcp:www.google.com:443
2025/12/05 16:04:19.286191 [Info] [3503482820] app/dispatcher: taking detour [proxy10829] for [tcp:www.google.com:443]
2025/12/05 16:04:19.286191 from tcp:127.0.0.1:3948 accepted tcp:www.google.com:443 [mixed10829 -> proxy10829]
2025/12/05 16:04:19.286191 [Debug] [3503482820] transport/internet/splithttp: XMUX: creating xmuxClient because xmuxClients is empty
2025/12/05 16:04:19.286191 [Info] [3503482820] transport/internet/splithttp: XHTTP is dialing to tcp:[**my ipv6**]:5001, mode packet-up, HTTP version 1.1, host ch2.19951115.xyz
2025/12/05 16:04:19.286191 [Debug] [3503482820] transport/internet/splithttp: XMUX: creating xmuxClient because xmuxClients is empty
2025/12/05 16:04:19.286191 [Info] [3503482820] transport/internet/splithttp: XHTTP is downloading from tcp:[**my ipv4**]:5002, mode stream-down, HTTP version 2, host mofumofu.me
2025/12/05 16:04:19.286191 [Debug] [3503482820] transport/internet: dialing to tcp:[**my ipv4**]:5002
2025/12/05 16:04:19.501544 [Info] [3503482820] proxy/vless/outbound: tunneling request to tcp:www.google.com:443 via [**my ipv6**]:5001
2025/12/05 16:04:19.502346 [Debug] [3503482820] transport/internet: dialing to tcp:[**my ipv6**]:5001
2025/12/05 16:04:19.820136 [Debug] [3503482820] transport/internet: dialing to tcp:[**my ipv6**]:5001
2025/12/05 16:04:19.850559 [Debug] [3503482820] transport/internet: dialing to tcp:[**my ipv6**]:5001
2025/12/05 16:04:20 测试完成
```
<br>
