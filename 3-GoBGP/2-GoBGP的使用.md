# GoBGP的使用

参考:

- [GoBGP Github](https://github.com/osrg/gobgp)

GoBGP 是一个用 Go 实现的 BGP 控制平面。它负责建立 BGP 会话、交换路由、执行策略和最佳路径计算；它本身不转发业务报文。若要让 Linux 真正按照 GoBGP 学到的路由转发，需要通过 FRR/Quagga Zebra，或者由外部程序通过 gRPC 读取路由后写入内核 FIB。


## 1. 典型配置

```text
[global.config]
  as = 65000
  router-id = "10.0.0.1"
[[neighbors]]
  [neighbors.config]
    neighbor-address = "192.0.2.2"
    peer-as = 65001
```

这段配置的作用是：让本机 GoBGP 以 AS 65000 的身份，与地址为 192.0.2.2、属于 AS 65001 的路由器建立一条 eBGP 邻居关系。

```text
本机 GoBGP                           对端路由器
AS 65000                             AS 65001
Router ID 10.0.0.1                   地址 192.0.2.2
     └────────── eBGP / TCP 179 ──────────┘
```

下面详细解释一下上面的配置.

### 1.1 global.config段

1）as = 65000

指定 GoBGP 的本地 AS 号为 65000。本机发送 BGP OPEN 报文时，会告诉对端“我是 AS 65000”

>AS: 自治系统(Autonomous System)

2）router-id = "10.0.0.1"

指定本机 BGP Router ID，用于唯一标识这个 BGP 实例。它是一个 32 位标识符，看起来像 IPv4 地址，但：

- 不一定是接口地址
- 不负责建立 TCP 连接
- 建议使用稳定且唯一的地址，通常取 Loopback 地址

### 1.2 neighbors段

定义一个 BGP 邻居。每出现一次 `[[neighbors]]`，就表示增加一个邻居。

1）neighbor-address = "192.0.2.2"

指定对端 BGP 邻居地址。GoBGP 会尝试与`192.0.2.2:179`建立TCP连接。

2）peer-as = 65001

声明对端必须属于 AS 65001。

> ps: 因为本地AS为65000，对端AS为65001，两者不同，所以这是 eBGP 会话。

### 1.3 对应BGP路由交换过程

GoBGP启动后的过程大致是：

```text
读取配置
  → 添加邻居 192.0.2.2
  → 建立 TCP 179 连接
  → 交换 BGP OPEN
  → 校验对端 AS 是否为 65001
  → 协商 BGP 能力
  → 进入 Established
  → 交换路由
```
要成功建立会话，还需要满足以下条件：

1） **本机可以到达 192.0.2.2**

```bash
ping 192.0.2.2
ip route get 192.0.2.2
```

2）**对端也配置了反向邻居**

例如：
```bash
邻居地址：192.0.2.1
邻居 AS：65000
```

如果对端也是 GoBGP，其配置类似：

```text
[global.config]
  as = 65001
  router-id = "10.0.0.2"

[[neighbors]]
  [neighbors.config]
    neighbor-address = "192.0.2.1"
    peer-as = 65000
```

3）**防火墙允许 TCP 179**

4）**如果是普通 eBGP，双方地址通常需要直连。非直连时要配置 eBGP Multihop**

<br>

配置完成后可以检查：
```bash
# gobgp neighbor
Peer         AS     State
192.0.2.2    65001  Establ
```
成功时就会看到上述输出。

需要注意：这段配置只是建立 BGP 邻居，并不会自动产生要宣告的业务路由。若要宣告一个前缀，还需要添加路由，例如：

```bash
gobgp global rib add 10.20.0.0/16
```
然后 GoBGP 才能把该前缀通过已经建立的 eBGP 会话发送给 192.0.2.2

## 2. 通过GoBGP实现VIP

```bash
gobgp global rib -a ipv4 add $vip/32 nexthop $oss_gateway_ip
```
这个命令向 GoBGP 的全局 RIB 中注入一条 IPv4 /32 主机路由，并把 BGP NEXT_HOP 属性设置为 $oss_gateway_ip。

需要注意：
- 该命令不会把 $vip 配置到本机网卡。
- 不会启动 VIP 对应的服务。
- 默认只是把路径加入 GoBGP RIB；是否进入 Linux ip route 取决于 Zebra/FIB 集成。
- nexthop 是 BGP 路径属性，最终对外通告的下一跳还可能受 eBGP、iBGP、next-hop-self 或出口策略影响。











