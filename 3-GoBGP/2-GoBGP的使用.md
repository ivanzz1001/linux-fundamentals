# GoBGP的使用

参考:

- [GoBGP Github](https://github.com/osrg/gobgp)

GoBGP 是一个用 Go 实现的 BGP 控制平面。它负责建立 BGP 会话、交换路由、执行策略和最佳路径计算；它本身不转发业务报文。若要让 Linux 真正按照 GoBGP 学到的路由转发，需要通过 FRR/Quagga Zebra，或者由外部程序通过 gRPC 读取路由后写入内核 FIB。


## 1. 整体架构

```mermaid
flowchart LR
    C["配置文件<br/>TOML / YAML / JSON"] --> D["gobgpd"]
    CLI["gobgp CLI"] -->|gRPC| API["GoBGP gRPC API"]
    APP["控制器 / Python / Go 程序"] -->|gRPC| API
    API --> D

    P["BGP Peer"] <-->|TCP 179<br/>OPEN / UPDATE / KEEPALIVE| FSM["Peer FSM"]
    FSM --> PARSE["BGP 报文解析与校验"]
    PARSE --> IMP["Import Policy"]
    IMP --> RIB["Global RIB<br/>按 AFI/SAFI 分类"]
    RIB --> BEST["最佳路径 / Multipath 计算"]

    BEST --> EXP["Export Policy"]
    EXP --> FSM

    BEST -->|ZAPI，可选| Z["FRR / Quagga Zebra"]
    Z -->|Netlink| FIB["Linux Kernel FIB"]
    FIB --> DATA["数据包转发"]
```
