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





