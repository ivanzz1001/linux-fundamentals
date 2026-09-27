# 如何使GoBGP VIP真正接收流量

要让 VIP 真正接收流量，需要同时完成三件事：

1. Linux 将 VIP 视为本机地址。
1. 应用监听 VIP 对应端口。
1. GoBGP 把 VIP 路由发布给上游。

下面以推荐的“VIP 绑定 Loopback”方式说明。

## 1. 示例拓扑

```text
服务器物理地址：192.0.2.10/24，eth0
上游 ToR：      192.0.2.1
VIP：           10.20.0.100/32
业务端口：      TCP 443
```

最终流量路径：

```text
客户端
  → 上游路由器
  → 查到 10.20.0.100/32 经由 192.0.2.10
  → 数据包从服务器 eth0 进入
  → Linux 发现 10.20.0.100 是本地地址
  → 投递给监听 10.20.0.100:443 的应用
```

## 2. 方式1-把 VIP 配置到 Loopback（推荐）

### 2.1 添加VIP

```bash
# sudo ip addr add 10.20.0.100/32 dev lo
```

检查：
```bash
# ip addr show dev lo
# ip route show table local
```

