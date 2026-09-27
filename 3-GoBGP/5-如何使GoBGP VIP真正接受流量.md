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
应该看到类似:

```bash
local 10.20.0.100 dev lo proto kernel scope host
```
`local路由`表示目的地址属于本机，数据包会被交给本地协议栈，而不是继续转发。
> 参看 [Linux ip-route 文档](https://man7.org/linux/man-pages/man8/ip-route.8.html)

这里不需要打开:

```bash
net.ipv4.ip_forward
```
因为数据包的目的地址是本机 VIP，不是让服务器充当普通路由器。

### 2.2 让应用监听 VIP

- 应用可以只监听 VIP

  ```bash
  10.20.0.100:443
  ```
  例如`Nginx`:
  
  ```bash
  server {
      listen 10.20.0.100:443 ssl;
      server_name _;
  }
  ```
  检查监听情况应该可以看到:

  ```bash
  # ss -lntp
  LISTEN 0 511 10.20.0.100:443
  ```


- 也可以监听所有本地地址

  ```bash
  0.0.0.0:443
  ```
   检查监听情况应该可以看到:

  ```bash
  # ss -lntp
  LISTEN 0 511 0.0.0.0:443
  ```

先在服务器本机测试：

```bash
# curl -k https://10.20.0.100/
```
如果本机都不能访问，先不要发布 BGP 路由。

### 2.3 通过 GoBGP 发布 VIP

如果服务器与 ToR 建立的是直连 eBGP，通常可以直接添加：

```bash
gobgp global rib add 10.20.0.100/32
```

或者显式指定服务器的 underlay 地址为下一跳：
```bash
gobgp global rib add 10.20.0.100/32 nexthop 192.0.2.10
```

GoBGP 官方 CLI 支持通过 nexthop 设置路径的 BGP NEXT_HOP 属性。[GoBGP CLI](https://github.com/osrg/gobgp/blob/master/docs/sources/cli-command-syntax.md)

需要注意：下一跳应当指向“能够到达这台服务器的地址”。通常是服务器物理接口或 Loopback underlay 地址，而不是上游网关地址。错误地把上游网关设置成下一跳，可能形成路由环路。

检查 GoBGP:

```bash
# gobgp neighbor
# gobgp global rib 10.20.0.100/32
# gobgp neighbor 192.0.2.1 adj-out
```

还必须在 ToR 上确认实际收到的路由类似(ps: 不同厂商路由器确认实际收到对应路由的方法可能不同）：

```bash
10.20.0.100/32 via 192.0.2.10
```

不能只看 GoBGP 本地 RIB。

### 2.4 从外部测试

从其他服务器上执行：
```bash
# curl -k https://10.20.0.100/
```
同时在 VIP 服务器抓包：
```bash
# sudo tcpdump -ni eth0 host 10.20.0.100
```
如果能抓到请求但应用没有响应，问题通常在：
- 应用没有监听 VIP。
- 本机防火墙拦截。
- Linux 没有 local 10.20.0.100 路由。
  
反向路径检查或回程路由有问题。

## 3. 方式二-使用 dummy 接口

如果 VIP 较多，可以创建专用 dummy 接口：

```bash
sudo ip link add vip0 type dummy
sudo ip link set vip0 up
sudo ip addr add 10.20.0.100/32 dev vip0
```
检查:

```bash
ip addr show dev vip0
ip route show table local
```

然后应用监听 VIP，再通过 GoBGP 宣告：

```bash
# gobgp global rib add 10.20.0.100/32
```

其工作原理与 Loopback 相同，只是 VIP 管理更加独立：

```text
eth0：物理网络/BGP邻居
vip0：业务VIP
```
生产环境通常通过 systemd-networkd、NetworkManager 或发行版网络配置持久化，否则重启后 dummy 接口和 VIP 会消失。

## 4. 方式三-不配置接口地址，使用 local route

也可以只建立本地路由：
```bash
sudo ip route add local 10.20.0.100/32 dev lo
```

如果应用需要明确绑定这个没有配置在接口上的地址，还需要：
```bash
sudo sysctl -w net.ipv4.ip_nonlocal_bind=1
```
然后再启动应用并发布路由。但要注意：
> ip_nonlocal_bind=1 只允许程序 bind() 非本地地址，不会单独让入站数据包自动变成本地流量。

Linux 内核文档对它的定义就是“允许进程绑定非本地 IP”。[Linux IP sysctl](https://kernel.org/doc/html/latest/networking/ip-sysctl.html)

因此至少还需要：
```bash
ip route add local 10.20.0.100/32 dev lo
```
总体上，直接把 /32 配置到 lo 或 dummy 更清晰、容易排障

## 5. 什么时候需要 IPVS?

如果当前节点本身就是四层负载均衡器，收到 VIP 流量后还要再分发给后端服务器，才需要 IPVS：

```bash
客户端
  → VIP
  → GoBGP节点/IPVS
  ├─ Real Server 1
  └─ Real Server 2
```

如果 Nginx、OSS 服务或其他应用直接运行在宣告 VIP 的服务器上，则不需要 IPVS：
```bash
客户端
  → VIP
  → 本机应用
```

## 6. 回程路由和 rp_filter

如果请求和响应使用不同接口，严格反向路径检查可能丢包。先查看：
```bash
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.eth0.rp_filter
```
存在非对称路由时，可考虑在相关接口使用 loose 模式：

```bash
# sudo sysctl -w net.ipv4.conf.eth0.rp_filter=2
```

其中：
- rp_filter=1: 严格模式，入接口必须是到源地址的最佳反向路径。
- rp_filter=2：宽松模式，只要求源地址通过任意接口可达。

内核文档建议复杂或非对称路由使用 loose 模式。[Linux rp_filter 文档](https://kernel.org/doc/html/latest/networking/ip-sysctl.html)

不要无条件关闭所有接口的反向路径检查，应先通过抓包和路由表确认确实是该问题。

## 7. 上下线顺序

- 上线
  ```bash
  添加VIP到lo/dummy
    → 启动应用
    → 本机健康检查成功
    → GoBGP发布VIP/32
  ```

- 下线
  ```bash
  GoBGP撤销VIP/32
    → 等待网络收敛和连接排空
    → 停止应用
    → 删除本地VIP
  ```

撤销命令:

```bash
gobgp global rib del 10.20.0.100/32
```

推荐的最终组合是：

```bash
# 1. 让Linux接受VIP
sudo ip addr add 10.20.0.100/32 dev lo

# 2. 启动并检查应用
ss -lntp
curl -k https://10.20.0.100/

# 3. 服务健康后才发布
gobgp global rib add 10.20.0.100/32

# 4. 确认路由确实发给上游
gobgp global rib 10.20.0.100/32
gobgp neighbor 192.0.2.1 adj-out
```
其中最关键的判断链路是：

```bash
ToR有VIP/32路由
+ Linux local表有VIP
+ 应用监听VIP端口
= VIP可以实际接收业务流量
```
  
